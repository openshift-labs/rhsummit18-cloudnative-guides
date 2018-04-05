## Create a Spring Boot Service

The first thing we want to do to ensure that our oc command line tools was installed and successfully added to our path is to log in to the OpenShift environment that has been provided for this Roadshow session. In order to log in, we will use the oc command and then specify the server that we want to authenticate to. Issue the following command:

~~~shell
$ oc login {{OPENSHIFT_MASTER_URL}}
~~~

Enter the username and password provided to you by the instructor:

* Username: {{OPENSHIFT_USERNAME}}
* Password: {{OPENSHIFT_PASSWORD}}

![OpenShift Login]({% image_path ocp-login.png %}){:width="900px"}
