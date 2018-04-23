#### Automate PROD Release


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

In your workspace in the root of catalog directory, right-click and then click on 
**New** > **File** and name it `Jenkinsfile.release`. Paste the following pipeline definition 
into `Jenkinsfile.release`:

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

Commit the `Jenkinsfile.release` in to the git repository by right-clicking on the catalog in the project 
explorer and then on **Git** > **Commit**.

Make sure `Jenkinsfile.release` is checked. Enter a commit message to describe your change. Check the 
**Push commit changes to...** to push the commit directly to the git server and then click on **Commit***

![Eclipse Che - Jenkinsfile Commit]({% image_path prod-pipeline-commit.png %}){:width="600px"}

You can now create an OpenShift Pipeline that uses the `Jenkinsfile.release` definition from the git repository 
to create a pipeline. In the OpenShift Web Console go to the **Catalog DEV** project. Click on 
**Add to Project** > **Import YAML/JSON** and paste the following pipeline definition:

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

Click on **Create**. Try the pipeline!