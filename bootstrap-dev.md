## Setup Development Environment
{:.no_toc}

In this lab you will configure you local development environment using Eclipse Che IDE and create 
a build pipeline that build, tests and deploys a Spring Boot service on OpenShift.

#### Configure Development Workspace in Eclipse Che

You might be familiar with the Eclipse IDE which has been for years, one of the popoular IDEs for Java and other 
programming languages. [Eclipse Che](https://www.eclipse.org/che/){:target="_blank"} is the next-generation Eclipse IDE which is web-based
and gives you a full-features IDE running in the cloud. You have an Eclipse Che instance deployed on your OpenShift cluster 
which you will use during these labs.

Go to the [Eclipse Che url]({{ ECLIPSE_CHE_URL }}){:target="_blank"} in order to configuration your development workspace.

A stack is a template of workspace configuration for example the programming language and tools you want to use
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and configurations 
on-demand. 

For this lab, click on the **Java RH Summit** stack and then on the **Create** button. 

![Eclipse Che Workspace]({% image_path bootstrap-che-create-workspace.png %}){:width="900px"}

Click on **Open** to open the workspace and then on **Start** button to start the workspace for use.

![Eclipse Che Workspace]({% image_path bootstrap-che-start-workspace.png %}){:width="900px"}

You can click on the left arrow icon to switch to the wide view:

![Eclipse Che Workspace]({% image_path bootstrap-che-wide.png %}){:width="600px"}

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
Eclipse Che IDE running in your browser.

![Eclipse Che Workspace]({% image_path bootstrap-che-workspace.png %}){:width="900px"}

Now you can import the Catalog project skeleton from the git repository directly into your workspace and start 
coding on it. 

You should first set the Git name and email so that it marks your commits with your name in the 
source code version history. Go to **Profile** > **Preferences** > **Git** > **Committer**, enter 
your name and email and click on **Save**

![Eclipse Che - Git Config]({% image_path bootstrap-che-git-profile.png %}){:width="600px"}

The source code for the Catalog service is available in the git server running on OpenShift. In the project 
explorer pane, click on **Import Projects...** link and enter the Catalog git repository url:

|`http://{{GIT_USERNAME}}:{{GIT_PASSWORD}}@{{GIT_HOSTNAME}}/{{GIT_USERNAME}}/catalog.git`{: style="color: blue"}


![Eclipse Che - Import Project]({% image_path bootstrap-che-import.png %}){:width="720px"}

Click on **Import**. Make sure you choose the **Java** > **Maven** project configuration so that 
the Maven features get enabled for your project. Click on **Save**

![Eclipse Che - Import Maven]({% image_path bootstrap-che-maven.png %}){:width="720px"}

The catalog project is imported now into your workspace and is visible in the project explorer.

![Eclipse Che - Project]({% image_path bootstrap-che-project.png %}){:width="900px"}

Build the Catalog project by clicking on the maven commands pallette if you can find the toolbar icon or alternatively
click on **Run** > **Commands Palette** > **Build** (there are more ways to do this, see if you can find them!)

![Eclipse Che - Project]({% image_path bootstrap-che-build-palette.png %}){:width="600px"}


#### Create Dev Environment

You will use OpenShift CLI and OpenShift Web Console for interacting with OpenShift during these labs. Open the
[OpenShift Web Console]({{ OPENSHIFT_MASTER_URL }}){:target="_blank"}.

Login with the following credentials:

* Username: ``{{ OPENSHIFT_USERNAME }}``
* Password: ``{{ OPENSHIFT_PASSWORD }}``

You can also use the Eclipse Che **Terminal** panel to run OpenShift CLI commands that is what you will
use in the following labs whenever it's instructed to run an OpenShift CLI command.

Use the OpenShift CLI to login into OpenShift with the following credentials:

* Username: ``{{ OPENSHIFT_USERNAME }}``
* Password: ``{{ OPENSHIFT_PASSWORD }}``

~~~shell
oc login {{ OPENSHIFT_MASTER_URL }}
~~~

![OpenShift CLI Login]({% image_path bootstrap-ocp-login.png %}){:width="900px"}

Create a project for the Dev environment: 

~~~shell
oc new-project dev{{PROJECT_SUFFIX}} --display-name="Catalog DEV"
~~~

Now that the project is created, you can navigate to it using the OpenShift Web Console by clicking on the name on the left
side of the web console (you may need to click **View All** if the **Catalog DEV** project isn't listed). You can also directly
access it [here]({{ OPENSHIFT_MASTER_URL }}/console/project/dev{{PROJECT_SUFFIX}}){:target="_blank"}. It's currently empty, but that's about to change.

#### Deploy Catalog in DEV Environment

OpenShift [Source-to-Image (S2I)]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/builds_and_image_streams.html#source-build){:target="_blank"}
capability can be used to build a container image from the source code. OpenShift S2I uses the supported 
OpenJDK container image to build the final container image of the Catalog service using the Spring Boot uber-jar laid over the 
certified OpenJDK container image that comes with the OpenShift platform.

Run the following in Eclipse Che **Terminal** in order to build the Catalog container image using S2I:

~~~sh
oc new-build redhat-openjdk18-openshift:1.2~http://{{GIT_HOSTNAME}}/{{GIT_USERNAME}}/catalog.git \
    -e MAVEN_MIRROR_URL=http://nexus.lab-infra.svc:8081/repository/maven-all-public
~~~

The `base-image#source-repo` expression in the above command instructs OpenShift to pull the 
application source code from the specified source code repository, build it using the build tool that 
is suitable for the application (nothing says Maven louder than a `pom.xml`!), and then build the container 
image for the application by layering the application binaries on the `base-image`. Since Catalog service 
is based on Spring Boot, we use the certified OpenJDK image that is available in OpenShift 
aka `redhat-openjdk18-openshift:1.2`.

Go to **Builds** > **Builds** to see the Catalog image build running. You can also see the build logs by 
clicking on the build.

![Catalog Build]({% image_path bootstrap-catalog-build.png %}){:width="900px"}

[OpenShift Templates]({{OPENSHIFT_DOCS_BASE}}/dev_guide/templates.html){:target="_blank"} allow composing applications
from multiple containers and deploy them at once. The Catalog service needs a PostgreSQL database and 
therefore you can use a template that is already created to deploy the Catalog skeleton project and 
a PostgreSQL database on OpenShift.


When the Catalog image build is complete and you have the image ready, use the 
[Catalog template](https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/catalog-template.yml){:target="_blank"}
to deploy the image in the **Catalog DEV** project.

Run the following in Eclipse Che **Terminal**:

~~~shell
oc new-app -f https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/catalog-template.yml 
~~~

![Deploy Catalog]({% image_path bootstrap-deploy-catalog.png %}){:width="900px"}

Click on **Overview** in the left sidebar menu to see the development project overview. You 
can see that the Catalog pod and a PostgreSQL database pod which were declared in the template 
that you deployed are up and ready.

![Catalog Service]({% image_path bootstrap-catalog-overview.png %}){:width="900px"}

The Catalog service currently doesn't have much code in it except an example endpoint. [Try
the endpoint in your browser](http://catalog-dev{{ PROJECT_SUFFIX }}.{{ APPS_HOSTNAME_SUFFIX }}/hello){:target="_blank"} to make sure it is deployed correctly.

If you see a `Hello, World!` response back in your browser, then it's working. If you don't see `Hello, World!` the application may not be fully started
yet. You can run `oc rollout status -w dc/catalog` in your Terminal to wait for it to be ready, then try the URL again.

In the next labs, you will write some code to build a few REST endpoints in the Catalog service.

#### Continuous Integration Pipeline

In order to automate the build and test process every time someone changes source code of the Catalog 
service, create a CI pipeline on Jenkins which checks the code on every commit and builds and tests it. 
The CI pipeline enables fast feedback to developers and makes sure everyone knows and can react if someone 
breaks the code.

OpenShift has built-in support for CI/CD pipelines by allowing developers to define a 
[Jenkins pipeline](https://jenkins.io/solutions/pipeline/){:target="_blank"} for execution by a Jenkins
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
        sh "mvn deploy -Popenshift -DskipTests"
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

Go to Eclipse Che **Terminal** and run the following to deploy a Jenkins container using the certified Jenkins image that 
comes with OpenShift.

~~~shell
oc new-app jenkins-persistent
~~~

![Deploy Jenkins]({% image_path bootstrap-jenkins-deploy.png %}){:width="900px"}

Since the pipeline will control the build and deployment flow, you should disable 
[the automatic deployment triggers]({{OPENSHIFT_DOCS_BASE}}/dev_guide/deployments/basic_deployment_operations.html#triggers){:target="_blank"}
that OpenShift uses to automate the deployment process:

~~~shell
oc set triggers dc/catalog --manual -n dev{{PROJECT_SUFFIX}}
~~~

You can now create an OpenShift Pipeline that uses the `Jenkinsfile` definition from the git repository 
to create a pipeline. In the OpenShift Web Console go to the **Catalog DEV** project. Click 
on **Add to Project** > **Import YAML/JSON** and paste the following pipeline definition:

~~~shell
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: catalog-build
spec:
  runPolicy: SerialLatestOnly
  source:
    git:
      ref: master
      uri: "http://{{ GIT_HOSTNAME }}/{{ GIT_USERNAME }}/catalog.git"
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


![Import YAML/JSON - Create Pipeline]({% image_path bootstrap-create-pipeline.png %}){:width="700px"}

Click on **Create**. Try the pipeline!

#### Build and Test on Every Code Change

Manually triggering the deployment pipeline to run is useful but the real goal is to be able 
to build and test the catalog service on every change in code or configuration and deploy it into 
the development environment.

In order to automate triggering the pipeline, you can define a 
[webhook](https://developer.github.com/webhooks/){:target="_blank"} on your Git repository to 
notify OpenShift on every commit that is made to the Git repository and trigger a pipeline execution.

In the OpenShift Web Console go to **Builds** > **Pipelines** and then click on the `catalog-build` pipeline. 
On the **Configurations** tab copy the **Generic Webhook URL**:

![Pipeline Webhook]({% image_path bootstrap-pipeline-webhook.png %}){:width="900px"}


Now go to the [Git server web console](http://{{GIT_HOSTNAME}}){:target="_blank"} in your browser and login with your Git credentials:

* Git Username: `{{GIT_USERNAME}}`
* Git Password: `{{GIT_PASSWORD}}`

Click on the `catalog` repository and then on **Settings**

![Git Repository Settings]({% image_path bootstrap-gogs-settings.png %}){:width="900px"}

Click on **Webhooks** and then on **Add Webhook** > **Gogs**. Paste the pipeline generic webhook url 
you copied into the **Payload URL** textbox and then click on **Add Webhook**.

![Add Webhook]({% image_path bootstrap-webhook.png %}){:width="600px"}

The webhook is added now and will trigger the pipeline to run on every push to the `catalog` git 
repository.

You can test the webhook by clicking on the webhook and then on **Test Delivery** button.

![Webhook Test Delivery]({% image_path bootstrap-webhook-testdelivery.png %}){:width="600px"}

Well done! You are now ready to proceed to the next lab and start writing some code.