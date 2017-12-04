echo "Jenkins pipeline for simple-java-maven-app"

node {
  git url: 'https://github.com/feggreed/simple-java-maven-app.git'
  def mvnHome = tool 'M3'
  env.PATH = "${mvnHome}/bin:${env.PATH}"
  sh 'mvn -B verify'
}