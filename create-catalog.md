## Create a Spring Boot Microservice

In this step you are going to create our catalog service as a microservice. The point of this lab is to experience how a developer can work with OpenShift and Red Hat Middleware without any previous knowledge of OpenShift and Cloud native development.

### Fork and checkout the base project

Open Gogs web client and login as developer/developer

TODO: Add a generated link to Gogs

Click on Repositories and the `Catalog` repo

TODO: Add screen shot of gogs repos

Click on **Fork**

TODO: Add a screen shot of fork button

Now, in your forked repository click on **Clone**

Now, go to a terminal and run the following commmand replacing the `<repo-url>` with the copied URL.

    $ git clone <repo-url>

Finally, go into the catalog directory

    $ cd catalog

  
### Test the project locally


    $ mvn clean package
    $ mvn spring-boot:run
    $ curl http://localhost:8080/hello



    





