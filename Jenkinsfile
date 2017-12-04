echo "Jenkins pipeline for simple-java-maven-app"

node ('master') {
  git url: 'https://github.com/feggreed/simple-java-maven-app.git'
  def v = version()
  if (v) {
      echo "Build version ${v}"
  } else {
      echo "Build version NOT Provided"
  }
  def mvnHome = tool 'M3'
  env.PATH = "${mvnHome}/bin:${env.PATH}"
  //sh 'mvn -B verify'
  sh "${mvnHome}/bin/mvn -B -Dmaven.test.failure.ignore verify"
  step([$class: 'ArtifactArchiver', artifacts: '**/target/*.jar', fingerprint: true])
  step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
}
def version() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version'
    matcher ? matcher[0][1] : null
}
