## Setup Development Environment


#### Create Dev Environment

Create a project for the Dev environment: 

~~~shell
oc new-project dev{{PROJECT_SUFFIX}} --display-name="Catalog DEV"

# if using Che for oc commmands
# oc policy add-role-to-user admin system:serviceaccount:lab-infra:default -n dev
~~~

#### Create Git Repository

Go to Git server and login with your Git credentials:

* Git server URL: [{{GIT_HOSTNAME}}](http://{{GIT_HOSTNAME}}/user/login){:target="_blank"}
* Username: `{{ GIT_USER }}`
* Password: `{{ GIT_PASSWORD }}`

You need a git repository to store the source code for the Catalog service which you will 
be creating throughout the following labs. Create a new repository by clicking on the plus 
icon on the top navigation bar and then on **New Repository**.

Specify `catalog` as **Repository Name** and click on **Create Repository** button, leaving 
the rest with default values.

![Git Server - Create Repo]({% image_path bootstrap-gogs-new-repo.png %}){:width="700px"}


You will use Spring Boot for building the Catalog service. The skeleton project is available 
on GitHub. You can now download the skeleton project from GitHub and import it into the 
Catalog git repository you just created.

Download the project skeleton from GitHub:

~~~shell
cd ~
curl -L -o projects.tar.gz https://github.com/openshift-labs/rhsummit18-cloudnative-labs/archive/master.tar.gz
tar xvfz projects.tar.gz
mv rhsummit18-cloudnative-labs-master/* . 
rm -rf rhsummit18-cloudnative-labs-master
~~~

Configure your name and email on git so that your commits are marked correctly with your name 
in the version history of the changes on the git repository:

~~~shell
git config --global user.name "rhdeveloper"
git config --global user.email "rhdeveloper@me.com"
~~~

Import the Catalog skeleton into your git repository:

~~~shell
cd catalog
git init
git add . --all
git commit -m "initial add"
git remote add origin http://{{GIT_USER}}:{{GIT_PASSWORD}}@{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git
git push -u origin master
~~~

#### Deploy Catalog in DEV Environment

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build) 
capability can be used to build a container image from the source code. OpenShift S2I uses the supported 
OpenJDK container image to build the final container image of the Catalog service using the Spring Boot uber-jar laid over the 
certified OpenJDK container image that comes with the OpenShift platform.

[Templates]({{OPENSHIFT_DOCS_BASE}}/dev_guide/templates.html) allow composing applications from multiple containers and deploy 
them at once. The Catalog service needs a PostgreSQL database and therefore you can use a template that is already 
created to deploy the Catalog skeleton project and a PostgreSQL database on OpenShift.

Deploy the [Catalog template](https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/catalog-template.yml) 
in the **Catalog DEV** project:

~~~shell
oc new-app -f https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/catalog-template.yml \
      -p GIT_URI=http://{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git \
      -p MAVEN_MIRROR_URL=http://nexus.lab-infra.svc:8081/repository/maven-all-public
~~~

Notice that you can provide parameters to templates to for example customize the source code Git repository.

#### Setup Development Workspace

You might be familiar with the Eclipse IDE which has been for years, one of the popoular IDEs for Java and other 
programming languages. [Eclipse Che](https://www.eclipse.org/che/) is the next-generation Eclipse IDE which is web-based 
and gives you a full-features IDE running in the cloud. You have an Eclipse Che instance deployed on your OpenShift cluster 
which you will use during these labs.

Go to the Eclipse Che url in order to configuration your development workspace: {{ECLIPSE_CHE_URL}}

A stack is a template of workspace configuration for example the programming language and tools you want to use 
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and configurations 
on-demand. 

For this lab, click on the **Java RH Summit** stack and then on the **Create** button. 

![Eclipse Che Workspace]({% image_path bootstrap-che-create-workspace.png %}){:width="900px"}

Click on **Open** to open the workspace and then on **Start** button to start the workspace for use.

![Eclipse Che Workspace]({% image_path bootstrap-che-start-workspace.png %}){:width="900px"}

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
Eclipse Che IDE running in your browser.

![Eclipse Che Workspace]({% image_path bootstrap-che-workspace.png %}){:width="900px"}

Now you can import the Catalog project skeleton from the git repository directly into your workspace and start 
coding on it. 

You should first set the Git name and email so that it marks your commits with your name in the 
source code version history. Go to **Profile** > **Preferences** > **Git** > **Committer**, enter 
your name and email and click on **Save**

![Eclipse Che - Git Config]({% image_path bootstrap-che-git-profile.png %}){:width="600px"}

In the project explorer pane, click on **Import Projects...** link and enter the Catalog git 
repository url: http://{{GIT_USER}}:{{GIT_PASSWORD}}@{{GIT_HOSTNAME}}/{{GIT_USER}}/catalog.git

![Eclipse Che - Import Project]({% image_path bootstrap-che-import.png %}){:width="720px"}

Click on **Import**. Make sure you choose the **Java** > **Maven** project configuration so that 
the Maven features get enabled for your project. Click on **Save**

![Eclipse Che - Import Maven]({% image_path bootstrap-che-maven.png %}){:width="720px"}

The catalog project is imported now into your workspace and is visible in the project explorer.

![Eclipse Che - Project]({% image_path bootstrap-che-project.png %}){:width="900px"}

Build the Catalog project by clicking on the maven commands pallette if you can find the toolbar icon or alternatively  
click on **Run** > **Commands Palette** > **Build** (there are more ways to do this, see if you can find them!)


#### Continuos Integration Pipeline

In order to automate the build and test process every time someone changes source code of the Catalog 
service, create a CI pipeline on Jenkins which checks the code on every commit and builds and tests it. 
The CI pipeline enables fast feedback to developers and makes sure everyone knows and can react if someone 
breaks the code.

OpenShift has built-in support for CI/CD pipelines by allowing developers to define a 
[Jenkins pipeline](https://jenkins.io/solutions/pipeline/) for execution by a Jenkins 
automation engine, which is automatically provisioned on-demand by OpenShift when needed.

The build can get started, monitored, and managed by OpenShift in the same way as any other 
build types e.g. S2I. Pipeline workflows are defined in a `Jenkinsfile`, either embedded 
directly in the build configuration, or supplied in a git repository and referenced by 
the build configuration. 

In order to manage the versions of the pipeline, you should keep it in the same git repository 
as the Catalog service which enables tracking all changes to the pipeline the same way that the 
source code changes are tracked.

In your workspace in the root of catalog directory, right-click and then click on 
**New** > **File** and name it `Jenkinsfile`. Paste the following pipeline definition 
into the `Jenkinsfile`

~~~shell
pipeline {
  agent {
      label 'maven'
  }
  stages {
    stage('Verify') {
      steps {
        sh "cp .settings.xml ~/.m2/settings.xml"
        sh "mvn verify"
      }
    }
    stage('Build JAR') {
      steps {
        sh "cp .settings.xml ~/.m2/settings.xml"
        sh "mvn clean package -Popenshift -DskipTests"
      }
    }
    stage('Archive JAR') {
      steps {
        //sh "mvn deploy -DskipTests"
        echo "TODO: Deploy the file to the local Nexus repo"
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


Commit the `Jenkinsfile` in to the git repository by right-clicking on the catalog in the project 
explorer and then on **Git** > **Commit**.

Make sure `Jenkinsfile` is checked. Enter a commit message to describe your change. Check the 
**Push commit changes to...** to push the commit directly to the git server and then click on **Commit***


![Eclipse Che - Git Commit]({% image_path bootstrap-che-git-commit.png %}){:width="600px"}

Go to your shell terminal and deploy a Jenkins container using the certified Jenkins image that 
comes with OpenShift.

~~~shell
oc new-app jenkins-persistent
~~~

In the web console go to the **Catalog DEV** project. Click on **Add to Project** > **Import YAML/JSON** 
and paste the following pipeline definition:

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