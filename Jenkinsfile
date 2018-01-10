node('mvn') {
   // define commands
   def mvnCmd = "mvn -s configuration/cicd-settings.xml"

   stage ('Build') {
     git branch: 'eap-7', url: 'http://gogs:3000/gogs/openshift-tasks.git'
     sh "${mvnCmd} clean install -DskipTests=true"
   }

   stage ('Test') {
     sh "${mvnCmd} test"
     step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
   }
  
   stage ('Analysis (Security, Bugs, etc)') {
     sh "${mvnCmd} site -DskipTests=true"
    
     step([$class: 'CheckStylePublisher', unstableTotalAll:'300'])
     step([$class: 'PmdPublisher', unstableTotalAll:'20'])
     step([$class: 'FindBugsPublisher', pattern: '**/findbugsXml.xml', unstableTotalAll:'20'])
     step([$class: 'JacocoPublisher'])
     publishHTML (target: [keepAll: true, reportDir: 'target/site', reportFiles: 'project-info.html', reportName: "Site Report"])
   }

   stage ('Push to Nexus') {
    sh "${mvnCmd} deploy -DskipTests=true"
   }

   stage ('Deploy DEV') {
     sh "rm -rf oc-build && mkdir -p oc-build/deployments"
     sh "cp target/openshift-tasks.war oc-build/deployments/ROOT.war"
     // clean up. keep the image stream
     sh "oc delete bc,dc,svc,route -l app=tasks -n dev-david"
     // create build. override the exit code since it complains about exising imagestream
     sh "oc new-build --name=tasks --image-stream=jboss-eap70-openshift:1.5 --binary=true --labels=app=tasks -n dev-david || true"
     // build image
     sh "oc start-build tasks --from-dir=oc-build --wait=true -n dev-david"
     // deploy image
     sh "oc new-app tasks:latest -n dev-david"
     sh "oc expose svc/tasks -n dev-david"
   }

   stage ('Deploy STAGE') {
     timeout(time:15, unit:'MINUTES') {
        input message: "Promote to STAGE?", ok: "Promote"
     }

     def v = version()
     // tag for stage
     sh "oc tag dev-david/tasks:latest stage-david/tasks:${v}"
     // clean up. keep the imagestream
     sh "oc delete bc,dc,svc,route -l app=tasks -n stage-david"
     // deploy stage image
     sh "oc new-app tasks:${v} -n stage-david"
     sh "oc expose svc/tasks -n stage-david"
   }
}

def version() {
  def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
