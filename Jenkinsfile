pipeline {
  agent {
    label 'docker-compose'
  }

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '60', artifactNumToKeepStr: '30'))
  }

  triggers {
    cron 'H H/4 * * *'
  }

  stages {
    stage('pull-engine') {
      steps {
        pullEngineImage()             
      }
    }

    stage('test') {
      steps {
        script {
          def examples = [
            'ivy': { assertIvyIsRunningInDemoMode() },
            'ivy-systemdb-postgres': { assertIvyIsNotRunningInDemoMode() },
            'ivy-systemdb-mysql': { assertIvyIsNotRunningInDemoMode() },
            'ivy-systemdb-mariadb': { assertIvyIsNotRunningInDemoMode() },
            'ivy-systemdb-mssql': { assertIvyIsNotRunningInDemoMode() },
            'ivy-sso-saml-apache-keycloak': { assertSaml() },
            'ivy-sso-openidc-apache-keycloak': { assertSaml() },
            'ivy-deploy-app': { assertAppIsDeployed("myApp") },
            'ivy-elasticsearch': { assertElasticsearch() },  
            'ivy-elasticsearch-cluster': { assertElasticsearchCluster() },
            'ivy-environment-variables': { assertIvyIsNotRunningInDemoMode() },
            'ivy-logging': { assertIvyConsoleLog("ivy-logging", "Loaded configurations of '/etc/axonivy-engine") },
            'ivy-reverse-proxy-nginx': { assertFrontendServerNginx() },
            'ivy-reverse-proxy-apache': { assertFrontendServerApache() },
            'ivy-openldap': { assertOpenLdap() },
            'ivy-patching': { assertPatching() },
            'ivy-secrets': { assertIvyIsNotRunningInDemoMode() },
            'ivy-valve': { assertValve() },
            'ivy-custom-errorpage': { assertCustomErrorPage() },
          ]

          examples.each { entry ->
            def example = entry.key;
            def assertion = entry.value;

            echo "==========================================================="
            echo "START TESTING EXAMPLE $example"
            try {
              dockerComposeUp(example)
              waitUntilIvyIsRunning(example)
              assertion.call()
              assertNoErrorOrWarnInIvyLog(example)
            } catch (ex) {
              currentBuild.result = 'UNSTABLE'
              echo ex.getMessage()
              def log = "warn-${example}.log"
              sh "echo SAMPLE ${example} FAILED >> ${log}"              
              sh "echo =========================================================== >> ${log}"
              
              sh "echo \"Error Message: ${ex.getMessage()}\" >> ${log}"   
              sh "echo =========================================================== >> ${log}"
              
              sh "echo DOCKER-COMPOSE BUILD LOG: >> ${log}"  
              sh "cat docker-compose-build.log >> ${log}"
              sh "echo =========================================================== >> ${log}"

              sh "echo DOCKER-COMPOSE UP LOG: >> ${log}"
              sh "docker-compose -f ${example}/docker-compose.yml logs >> ${log}"
            } finally {
              sh 'rm docker-compose-build.log'
              echo getIvyConsoleLog(example)
              dockerComposeDown(example)
              echo "==========================================================="
            }
          }
        }
      }
    }

    stage('archive') {
      steps {        
        archiveArtifacts allowEmptyArchive: true, artifacts: 'warn*.log'
      }
    }
  }
}


def pullEngineImage() {
  sh 'docker pull axonivy/axonivy-engine:dev'
}

def dockerComposeUp(example) {
  sh "docker-compose -f $example/docker-compose.yml build >> docker-compose-build.log"
  sh "docker-compose -f $example/docker-compose.yml up -d"
}

def dockerComposeDown(example) {
  sh "docker-compose -f $example/docker-compose.yml down"
}

def waitUntilIvyIsRunning(def example) {
  timeout(2) {
    waitUntil {
      def exitCode = sh script: "docker-compose -f $example/docker-compose.yml exec -T ivy wget -t 1 -q http://localhost:8080/ -O /dev/null", returnStatus: true
      return exitCode == 0;
    }
  }
}

def assertIvyIsRunningInDemoMode() {
  if (!isIvyRunningInDemoMode()) {
    throw new Exception("ivy is not running in demo mode")
  }
  if (isIvyRunningInMaintenanceMode()) {
    throw new Exception("ivy is running in maintenance mode");
  }
}

def assertIvyIsNotRunningInDemoMode() {
  if (isIvyRunningInDemoMode()) {      
    throw new Exception("ivy is running in demo mode")
  }
  if (isIvyRunningInMaintenanceMode()) {
    throw new Exception("ivy is running in maintenance mode");
  }
}

def assertSaml() {
  sleep 120
  def response = sh (script: 'curl -k -L https://localhost', returnStdout: true)
  if (!response.contains('Log in to ivy-demo')) {
    throw new Exception("not redirected to keycloak login page " + response)    
  }
}

def isIvyRunningInDemoMode() {
  def response = sh (script: "wget -qO- http://localhost:8080/info/index.jsp", returnStdout: true)
  return response.contains('Demo Mode')
}

def isIvyRunningInMaintenanceMode() {
  def response = sh (script: "wget -qO- http://localhost:8080/info/index.jsp", returnStdout: true)
  return response.contains('Maintenance Mode')
}

def assertOpenLdap() {
  waitUntilAppIsReady('ldap')
  // using basic auth mechanism to login (process servlet has basic auth filter)
  // even if no login is required for process start, the request will fail, if authentication is wrong
  def response = sh (script: "curl 'http://localhost:8080/ldap/pro/quick-start-tutorial/148655DDB7BB6588/start.ivp' --user rwei:rwei -L -i -b cookie.txt -s -o /dev/null -D/dev/stdout", returnStdout: true)
  if (response.contains("401")) { // 
    throw new Exception("could not login to app ldap as rwei/rwei");
  }
}

def assertAppIsDeployed(appName) {
  waitUntilAppIsReady(appName)
  def response = sh (script: "wget -qO- http://localhost:8080/$appName/", returnStdout: true)
  if (!followDefaultPageRedirect("http://localhost:8080", response).contains("Application: $appName")) {
    throw new Exception("app $appName is not deployed");
  }
}

def waitUntilAppIsReady(def appName) {
  // apps will be deployed async while starting engines
  // we could make a http request to the new app, but if its not exiting this will create ERROR messages in logs
  sleep 10
}

def assertIvyConsoleLog(example, message) {
  def log = getIvyConsoleLog(example)
  if (!log.contains(message)) {
    throw new Exception("console log of ivy does not contain $message");
  }
}

def assertNoErrorOrWarnInIvyLog(example) {
  def log = getIvyConsoleLog(example)
  if (log.contains("WARN") || log.contains("ERROR")) {
    throw new Exception("console log of ivy contains WARN/ERROR messages");
  }
}

def getIvyConsoleLog(example) {
  return sh (script: "docker-compose -f $example/docker-compose.yml logs ivy", returnStdout: true)
}

def assertPatching() {
  def sample = "ivy-patching"
  assertIvyConsoleLog(sample, "Install patches for classes: ch.ivyteam.ivy.search.internal.SearchManager")
  assertIvyConsoleLog(sample, "This Global Variable has been patched for Demo Purpose")
  assertIvyConsoleLog(sample, "starting patched Lucene base seach manager")
}

def assertValve() {
  def log =  sh (script: "docker exec ivy-valve_ivy_1 cat logs/ivy.log", returnStdout: true)
  if (!log.contains("Header -->")) {
    throw new Exception("ivy.log of ivy does not contains Header -->");
  }  
}

def assertFrontendServerNginx() {
  def response = sh (script: "wget -qO- http://localhost/", returnStdout: true)
  if (!followDefaultPageRedirect("http://localhost", response).contains('Welcome')) {
    throw new Exception("frontend server does not redirect to portal login page");
  }
}

def assertFrontendServerApache() {
  def response = sh (script: "wget -qO- http://localhost/info/index.jsp", returnStdout: true)
  if (!response.contains('Demo')) {
    throw new Exception("frontend server does not route to ivy");
  }
}

def assertElasticsearch() {
  // 1. Deploy Test Project
  sh "docker cp test.iar ivy-elasticsearch_ivy_1:/usr/lib/axonivy-engine/deploy/test.zip"  
  sleep(5) // wait until is deployed

  // 2. Execute Process which create business data
  sleep(10) // FIXME: sometimes es cluster is still not available. we have to fix that in the engine startup, wait on elasticsearch.
  sh "curl 'http://localhost:8080/test/pro/test/1665799EBA281E4C/start.ivp'"

  // 3. Query Elastic Search
  checkBusinessDataIndex(9200); 
}

def assertElasticsearchCluster() {
  // 1. Deploy Test Project
  sh "docker cp test.iar ivy-elasticsearch-cluster_ivy_1:/usr/lib/axonivy-engine/deploy/test.zip"  
  sleep(5) // wait until is deployed

  // 2. Execute Process which create business data
  sh "curl 'http://localhost:8080/test/pro/test/1665799EBA281E4C/start.ivp'"

  // 3. Query Elastic Search
  checkBusinessDataIndex(9201);
  checkBusinessDataIndex(9202);
  checkBusinessDataIndex(9203);  
}

def checkBusinessDataIndex(port) {
  timeout(2) {
    waitUntil {
      def url = "http://localhost:$port/_cat/indices"
      def response = sh (script: "curl $url", returnStdout: true)
      def elasticSearchIndex = "ivy.businessdata-test.testbusinessdata";  
      echo "elastic search response: $response"
      return response.contains(elasticSearchIndex);
    }
  }
}

def assertCustomErrorPage() {
  def response = sh (script: "curl http://localhost:8080/faces/notfound.xhtml", returnStdout: true)
  if (!response.contains('Please contact the system administrator')) {
    throw new Exception("could not load custom error page");
  }
}

def followDefaultPageRedirect(base, redirectPage) {
  def url = redirectPage.split('<meta http-equiv=\"refresh\" content=\"0; URL=')[1]
  url = base + url.split('\" />')[0]
  return sh (script: "wget -qO- $url", returnStdout: true)
}
