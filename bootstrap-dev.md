## Setup Development Environment


~~~shell
oc new-project dev{{PROJECT_SUFFIX}} --display-name="Catalog DEV"

# if using Che for oc commmands
# oc policy add-role-to-user admin system:serviceaccount:lab-infra:default -n dev

# Login to Gogs with developer/openshift
# Create a git repository called "catalog"

cd ~
curl -L -o projects.tar.gz https://github.com/openshift-labs/rhsummit18-cloudnative-labs/archive/master.tar.gz
tar xvfz projects.tar.gz
mv rhsummit18-cloudnative-labs-master/* . 
rm -rf rhsummit18-cloudnative-labs-master

cd catalog
git config --global user.name "rhdeveloper"
git config --global user.email "rhdeveloper@me.com"
git init
git add . --all
git commit -m "initial add"
git remote add origin http://{{GIT_USER}}:{{GIT_PASSWORD}}@{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git
git push -u origin master
~~~

Deploy Catalog:

~~~shell
oc new-app -f https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/catalog-template.yml \ 
      -p GIT_URI=http://{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git \
      -p MAVEN_MIRROR_URL=http://nexus.lab-infra.svc:8081/repository/maven-all-public
~~~

Set up development workspace on Eclipse Che

* Go to Che url and then workspace administration page
* Select the "Java RH Summit" stack and click on the create button
* Click on "Open" to open the workspace
* Once workspace is ready, click on "Import Project..."
* Select GIT and paste the Catalog git repo url: http://{{GIT_USER}}:{{GIT_PASSWORD}}@{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git
* Click on "Import", then choose "Maven" project configuration and then "Save"
* In the menues, go to "Profile > Preferences > Git > Committer" and specify your name and email
* Build the project by clicking on the maven commands pallette if you can find the icon or instead "Run > Commands Palette > build" (there are more ways to do this, see if you can find them!)

Create pipline to deploy project

* In the project explorer in the root of catalog, right-click and then New > File and call it "Jenkinsfile"
* Paste the following pipeline into Jenkinsfile

~~~shell
pipeline {
  agent {
      label 'maven'
  }
  stages {
    stage('Build JAR') {
      steps {
        sh "cp .settings.xml ~/.m2/settings.xml"
        sh "mvn package"
      }
    }
    stage('Archive JAR') {
      steps {
        sh "mvn deploy -DskipTests"
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.startBuild("catalog", "--from-file=target/catalog-${readMavenPom().version}.jar", "--wait")
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          openshift.withCluster() {
            def result, dc = openshift.selector("dc", "catalog")
            dc.rollout().latest()
            timeout(10) {
                result = dc.rollout().status("-w")
            }
            if (result.status != 0) {
                error(result.err)
            }
          }
        }
      }
    }
  }
}
~~~

Commit the pipeline to code repo:

* In the project explorer, right click on catalog and then Git > Commit
* Make sure Jenkinsfile is checked
* Add a commit message "dev pipeline added"
* Check the "Push commit changes to..."
* Click on "Commit"


Deploy Jenkins

~~~shell
oc new-app jenkins-persistent
~~~

In the web console go to the Catalog Dev project. Click on "Add to Project" > "Import YAML/JSON" and paste the following 
pipeline definition:

~~~shell
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: catalog-build
spec:
  runPolicy: Serial
  source:
    git:
      ref: master
      uri: "http://{{ GIT_HOSTNAME }}/{{ GIT_USER }}/catalog.git"
    type: Git
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: Jenkinsfile
    type: JenkinsPipeline
  triggers:
    - generic:
        secret: 4LXwMdx9vhQY4WXbLcFR
      type: Generic
~~~

Try the pipeline!

Add a webhook:

* Go to catalog-build pipelien -> Congigurations tab and then copy the Generic Webhook URL
* Go to Gogs > catalog repo > Settings > Webhook and click on Add Webhook > Gogs
* Paste the webhook url in the Payload URL and then click on Add Webhook button
* Test the webhook by clicking on it and then Test Delivery