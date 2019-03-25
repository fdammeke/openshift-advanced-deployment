user9
# README
This file doesn't contain any copy-pastable commands but a general guideline in how to configure stuff! All users and passwords in this file are for DEMO purpose only and don't serve any real-life means.

# RANDOM
## Alter a route
```sh
oc patch route/bluegreen -p '{"spec":{"to":{"name":"green"}}}'
oc patch route/bluegreen -p '"spec": {"alternateBackends": [{"kind": "Service","name": "blue","weight": 0}],"to":{"weight":100}}'
oc set route-backends bluegreen blue=0 green=100
```
## Add healthchecks
```sh
oc set probe dc/blue --readiness --get-url=http://:8080/index.php --initial-delay-seconds=30
oc set probe dc/blue --liveness --get-url=http://:8080/index.php --initial-delay-seconds=30
```
## Create new app from container image
```sh
oc new-app --docker-image=docker.io/wkulhanek/logtofile:latest
```

## Sidecar containers
https://www.mirantis.com/blog/multi-container-pods-and-container-communication-in-kubernetes/

### Create the buildconfig
```sh
oc create -f sidecar/dc.yaml
```
### Source
``` yaml
    spec:
      containers:
      - name: logtofile
        image: docker.io/wkulhanek/logtofile@sha256:ef013bc12e3d6baa56ff1b8b9323bdecfec69d2898ce7b4a276089e55fdebe0c
        imagePullPolicy: Always
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /tmp
          name: output
      - name: logtostdout
        command:
        - /bin/sh
        - -c
        - tail -f /tmp/datelog.txt
        image: docker.io/busybox:latest
        imagePullPolicy: Always
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /tmp
          name: output
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: output
```
### Openshift OAuth proxy in front of a webapp
https://raw.githubusercontent.com/openshift/oauth-proxy/master/contrib/sidecar.yaml

## mongodb statefulset
https://github.com/cvallance/mongo-k8s-sidecar/blob/master/example/StatefulSet/mongo-statefulset.yaml


# ci/cd tools lab
## nexus
```sh
oc new-app --docker-image=sonatype/nexus3:latest
oc expose service nexus3
oc rollout pause dc/nexus3
oc set probe dc/nexus \
	--readiness \
	--failure-threshold 3 \
	--initial-delay-seconds 30 \
	--get-url=http://:8081/
oc set resources dc nexus3 --requests 'cpu=500m,memory=1Gi'
oc set resources dc nexus3 --limits 'cpu=2,memory=2Gi'
oc rollout resume dc/nexus3
oc create service clusterip nexus-registry --tcp=5000
oc create route edge nexus3-registry --service=nexus-registry
oc annotate route nexus3 console.alpha.openshift.io/overview-app-route=true
oc annotate route nexus-registry console.alpha.openshift.io/overview-app-route=false
```
## persistent volume claim
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus3
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
## SonarQube
```sh
oc process --parameters -n openshift postgresql-persistent
oc new-app --template=postgresql-persistent --param=POSTGRESQL_USER=admin --param=POSTGRESQL_PASSWORD=admin123 --param=POSTGRESQL_DATABASE=sonarqube --param=VOLUME_CAPACITY=1Gi
oc new-app --docker-image=wkulhanek/sonarqube:6.7.5 --param=SONARQUBE_JDBC_USERNAME=admin --param=SONARQUBE_JDBC_PASSWORD=admin123 --param=SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql/sonarqube
oc rollout pause dc/sonarqube
oc expose service sonarqube
```
## persistent volume claim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: sonarqube
  name: sonarqube
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
## assign PVC to dc/sonarqube, set some resource limits
```sh
oc rollout pause dc/sonarqube
oc set volume dc/sonarqube --remove --confirm
oc set volume dc/sonarqube --add --overwrite --name=sonarqube --mount-path=/opt/sonarqube/data
oc set resources dc sonarqube --requests 'cpu=1,memory=1.5Gi'
oc set resources dc sonarqube --limits 'cpu=2,memory=3Gi'
oc patch dc sonarqube --patch '{"spec":{"strategy":{"type":"Recreate"}}}'
oc set probe dc sonarqube \
	--readiness \
	--failure-threshold 3 \
	--initial-delay-seconds 30 \
	--get-url=http://:9000/
oc rollout resume dc/sonarqube
```
## deploy gogs
```sh
oc new-app --template=postgresql-persistent --param=POSTGRESQL_USER=admin --param=POSTGRESQL_PASSWORD=admin123 --param=POSTGRESQL_DATABASE=gogs --param=VOLUME_CAPACITY=1Gi --name=postgresql-gogs
oc new-app --docker-image=wkulhanek/gogs:11.34
oc expose service gogs
oc set volume dc/gogs --remove --confirm
oc set volume dc gogs -t persistentvolumeclaim  --claim-name gogs --add --overwrite --name=gogs --mount-path=/data
oc create configmap gogs --from-file=app.ini
oc set volume dc/gogs -t configmap --configmap-name="gogs" --add --overwrite --name=gogs-config --mount-path=/opt/gogs/custom/conf
```
## deploy jenkins
```sh
oc process --parameters -n openshift jenkins-persistent
oc new-app --template=jenkins-persistent --param=MEMORY_LIMIT=2Gi --param=VOLUME_CAPACITY=4Gi --param=DISABLE_ADMINISTRATIVE_MONITORS=true
oc new-build  -D $'FROM docker.io/openshift/jenkins-agent-maven-35-centos7:v3.11\n
      USER root\nRUN yum -y install skopeo && yum clean all\n
      USER 1001' --name=jenkins-agent-appdev -n user9-jenkins
oc policy add-role-to-user view system:serviceaccount:user9-jenkins:default -n user9-nexus
oc policy add-role-to-user view system:serviceaccount:user9-jenkins:default -n user9-sonarqube
oc policy add-role-to-user view system:serviceaccount:user9-jenkins:default -n user9-gogs

```

## create pipeline
``` groovy
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
```

https://github.com/fdammeke/openshift-advanced-deployment.git

```sh
oc new-app --template=eap71-basic-s2i --param APPLICATION_NAME=tasks --param SOURCE_REPOSITORY_URL=http://gogs.user9-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks-private.git --param SOURCE_REPOSITORY_REF=master --param CONTEXT_DIR=/ --param MAVEN_MIRROR_URL=http://nexus3.user9-nexus.svc.cluster.local:8081/repository/maven-all-public -l app=tasks
oc create secret generic gogs \
    --from-literal=username=fabian \
    --from-literal=password=fabian \
    --type=kubernetes.io/basic-auth
oc set build-secret --source bc/tasks gogs
oc patch bc/tasks -p '{"spec":{"strategy":{"sourceStrategy":{"forcePull":false}}}}'
oc new-app redhat-openjdk18-openshift:1.2~https://github.com/redhat-gpte-devopsautomation/ola.git --name ola -l app=ola
oc expose service ola
```

https://docs.openshift.com/container-platform/3.6/dev_guide/dev_tutorials/binary_builds.html
```sh
cat Dockerfile | oc new-build -D - --name=ola-binary
oc start-build ola-binary --from-dir='.'
oc new-app --docker-image=docker-registry.default.svc:5000/user9-build/ola-binary:latest -l app=ola-binary --name=ola-binary --insecure-registry
```

## Chained builds
```sh
oc new-build  docker.io/jorgemoralespou/s2i-go~https://github.com/tonykay/ose-chained-builds --context-dir=/go-scratch/hello_world --name builder
oc new-build builder:lastest \
    -D $'FROM FROM scratch\nCOPY main /main\nEXPOSE 8080\nENTRYPOINT ["/main"]' \
    --name=runtime \
    --source-image=builder \
    --source-image-path=/opt/app-root/src/go/src/main/main
oc delete all -l build=runtime
oc new-build -D $'FROM scratch\nCOPY main /main\nEXPOSE 8080\nENTRYPOINT ["/main"]' --name=runtime --source-image=builder --source-image-path='/opt/app-root/src/go/src/main/main:.'
oc new-app runtime -l app=runtime
oc expose service runtime
```
# pipelines
```sh
oc new-project -tasks-dev --display-name "Tasks Development"
oc policy add-role-to-user edit system:serviceaccount:user9-jenkins:jenkins -n user9-tasks-dev

oc new-build --binary=true --name="tasks" jboss-eap71-openshift:1.3 -n user9-tasks-dev
oc new-app user9-tasks-dev/tasks:0.0-0 --name=tasks --allow-missing-imagestream-tags=true -n user9-tasks-dev
oc set triggers dc/tasks --remove-all -n user9-tasks-dev
oc expose dc tasks --port 8080 -n user9-tasks-dev
oc expose svc tasks -n user9-tasks-dev
oc set probe dc/tasks -n user9-tasks-dev --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/

#oc create configmap tasks-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n user9-tasks-dev
oc create configmap tasks-config --from-file=configuration/ -n user9-tasks-dev

oc set volume dc/tasks --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-config -n user9-tasks-dev
oc set volume dc/tasks --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-config -n user9-tasks-dev
```
```sh
## Set up Production Project
oc new-project user9-tasks-prod --display-name "Tasks Production"
oc policy add-role-to-group system:image-puller system:serviceaccounts:user9-tasks-prod -n user9-tasks-dev
oc policy add-role-to-user edit system:serviceaccount:user9-jenkins:jenkins -n user9-tasks-prod

## Create Blue Application
oc new-app user9-tasks-dev/tasks:0.0 --name=tasks-blue --allow-missing-imagestream-tags=true -n user9-tasks-prod
oc set triggers dc/tasks-blue --remove-all -n user9-tasks-prod
oc expose dc tasks-blue --port 8080 -n user9-tasks-prod
oc set probe dc tasks-blue -n user9-tasks-prod --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/
# oc create configmap tasks-blue-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n user9-tasks-prod
oc create configmap tasks-blue-config --from-file=configuration/ -n user9-tasks-prod
oc set volume dc/tasks-blue --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-blue-config -n user9-tasks-prod
oc set volume dc/tasks-blue --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-blue-config -n user9-tasks-prod

## Create Green Application
oc new-app user9-tasks-dev/tasks:0.0 --name=tasks-green --allow-missing-imagestream-tags=true -n user9-tasks-prod
oc set triggers dc/tasks-green --remove-all -n user9-tasks-prod
oc expose dc tasks-green --port 8080 -n user9-tasks-prod
oc set probe dc tasks-green -n user9-tasks-prod --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/
# oc create configmap tasks-green-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n user9-tasks-prod
oc create configmap tasks-green-config --from-file=configuration/ -n user9-tasks-dev
oc set volume dc/tasks-green --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-green-config -n user9-tasks-prod
oc set volume dc/tasks-green --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-green-config -n user9-tasks-prod

## Expose Blue service as route to make blue application active
oc expose svc/tasks-blue --name tasks -n user9-tasks-prod
```

## create the git repo credentials
```sh
oc create secret generic gogs \
    --from-literal=username=fabian \
    --from-literal=password=fabian \
    --type=kubernetes.io/basic-auth
oc set build-secret --source bc/tasks gogs
```

## allow jenkins to update dc and routes in prod
```sh
oc policy add-role-to-user edit system:serviceaccount:user9-jenkins:default -n user9-tasks-prod

oc create secret docker-registry --docker-server=nexus-registry-user9-nexus.apps.7a47.openshift.opentlc.com --docker-username=admin --docker-password=admin123 --docker-email="computing@cegeka.com" nexus -n user9-tasks-prod


oc secrets add serviceaccount/default secrets/nexus --for=pull -n user9-tasks-prod
```

# OpenShift build pipeline
## Source secret
```sh
oc create secret generic gogs \
    --from-literal=username=fabian \
    --from-literal=password=fabian \
    --type=kubernetes.io/basic-auth \
    -n user9-jenkins
oc set build-secret --source bc/tasks gogs
```
## Create pipeline
```sh
oc create -f ./full-pipeline/buildconfig.yaml
```
