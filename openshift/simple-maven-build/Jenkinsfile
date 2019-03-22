def gogsurl = ''
pipeline {
  agent { node { label 'maven-appdev' } }
  parameters {
    string(name: 'user', defaultValue: 'user9', description: 'who is running this pipeline?')
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    ansiColor('xterm')
  }
  stages {
    stage ('get gogs url') {
      steps {
        script {
          gogsurl = sh(
            script: "oc get route gogs -n ${params.user}-gogs --template='{{ .spec.host }}'",
            returnStdout: true
          ).trim()
        }
      }
    }
    stage ('get scm') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git', url: "http://${gogsurl}/CICDLabs/openshift-tasks.git"]]])
      }
    }
    stage('maven build') {
      steps {
        sh("mvn clean install -DskipTests=true -s ./nexus_settings.xml")
      }
    }
    stage('maven deploy') {
      steps {
        sh("mvn -s ./nexus_settings.xml deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://\$(oc get route nexus3 -n ${params.user}-nexus --template='{{ .spec.host }}')/repository/releases")
      }
    }
    stage('skopeo copy') {
      steps {
        sh("skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds=openshift:\$(oc whoami -t) --dest-creds=admin:admin123 docker://docker-registry-default.apps.7a47.openshift.opentlc.com/${params.user}-jenkins/jenkins-agent-appdev docker://\$(oc get route nexus-registry -n ${params.user}-nexus --template='{{ .spec.host }}')/${params.user}-jenkins/jenkins-agent-maven-appdev:v3.11")
      }
    }
    stage('maven sonarcube') {
      steps {
        sh("mvn sonar:sonar -s ./nexus_settings.xml -Dsonar.host.url=http://\$(oc get route sonarqube -n ${params.user}-sonarqube --template='{{ .spec.host }}')")
      }
    }
  }
}
