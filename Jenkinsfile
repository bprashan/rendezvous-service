node('ubuntu18.04-OnDemand') {

// Stage for checking out the sourceCode
stage('scm checkout'){
  checkout scm
}


// Stage to build the project
stage('build rendezvous package'){
  sh '''
  export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  ./gradlew clean build
  '''
}

// Stage to create the Rendezvous package and archive
stage('archive artifacts'){
  zip archive: true, dir: 'build/libs/RendezvousService*.jar', zipFile: 'RendezvousService.zip'
}

}