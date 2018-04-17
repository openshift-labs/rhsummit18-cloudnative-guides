#### Create Production Environment


Create secret for Git credentials:

~~~shell
oc create secret generic git-credentials --from-literal=username={{ GIT_USERNAME }} --from-literal=password={{ GIT_PASSWORD }}
oc label secret git-credentials credential.sync.jenkins.openshift.io=true
~~~


Grant Jenkins access:

~~~shell
# oc policy add-role-to-user edit system:serviceaccount:dev:jenkins -n prod{{ PROJECT_SUFFIX }}
oc policy add-role-to-user system:deployer system:serviceaccount:dev:jenkins -n prod{{ PROJECT_SUFFIX }}
~~~

Create a pipeline:

~~~groovy
def releaseTag

pipeline {
  agent {
      label 'maven'
  }
  stages {
    stage('Release Code') {
      environment {
        SCM_GIT_URL = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
      }
      steps {
        sh "git config --local user.email 'jenkins@cicd.com'"
        sh "git config --local user.name 'jenkins'"

        script {
          releaseTag = readMavenPom().getVersion().replace("-SNAPSHOT", "")
          openshift.withCluster() {
            withCredentials([usernamePassword(credentialsId: "${openshift.project()}-git-credentials", usernameVariable: "GIT_USERNAME", passwordVariable: "GIT_PASSWORD")]) {
              sh "mvn --batch-mode release:clean release:prepare release:perform -s .settings.xml"
            }
          }
        }
      }
    }
    stage('Release Image') {
      steps {
        script {
          openshift.withCluster() {
            echo "Releasing catalog image version ${releaseTag}"
            openshift.tag("${openshift.project()}/catalog:latest", "${openshift.project()}/catalog:${releaseTag}")
          }
        }
      }
    }    
    stage('Promote to PROD') {
      steps {
        script {
          openshift.withCluster() {
            def devNamespace = openshift.project()
            openshift.withProject(env.PROD_PROJECT) {
              openshift.tag("${devNamespace}/catalog:${releaseTag}", "${openshift.project()}/catalog:prod")
            }
          }
        }
      }
    }    
  }
}
~~~

Commit the pipeline in the catalog git repo

Go to DEV project, **Add to Project** > *Import YAML/JSON* and paste the following to create an OpenShift pipeline 
using the `Jenkinsfile.release` pipeline defined in Git repo:

~~~yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: catalog-release
spec:
  runPolicy: Serial
  source:
    git:
      ref: master
      uri: "http://{{ GIT_HOSTNAME }}/{{ GIT_USERNAME }}/catalog.git"
    type: Git
  strategy:
    jenkinsPipelineStrategy:
      env:
        - name: PROD_PROJECT
          value: "prod{{ PROJECT_SUFFIX }}"
      jenkinsfilePath: Jenkinsfile.release
    type: JenkinsPipeline
~~~