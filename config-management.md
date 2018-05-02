## Configuration Management with Spring Boot

So far our catalog service has been using an in-memory H2 database. Although H2 is a
convenient database to run locally on your laptop, it's in no ways appropriate for production
or even integration tests. Since it's strongly recommended to use the same technology stack
(operating system, JVM, middleware, database, etc) that is used in production across all environments,
we will modify the Catalog service to use a PostgreSQL database instead of the H2 in-memory database.

Fortunately, OpenShift supports stateful applications such as databases which required access to a
persistent storage that survives the container itself. You can deploy databases on OpenShift and
regardless of what happens to the container itself, the data is safe and can be used by the next
database container.

As part of this lab, a PostgreSQL database has already been deployed, which you can see by running 
the following in the Eclipse Che **Terminal**:

~~~shell
oc get pods -l app=catalog
~~~

You should see something like:

~~~shell
NAME                         READY     STATUS    RESTARTS   AGE
catalog-2-cb9lk              1/1       Running   0          6m
catalog-postgresql-1-c4rsm   1/1       Running   0          41m
~~~

Our Spring Boot Catalog application configuration is provided via a properties filed called `application-default.properties`
and can be overriden and [overlayed via multiple mechanisms](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html){:target="_blank"}.

In this scenario, you will configure the Catalog service which is based on Spring Boot to override the default
H2 database configuration using alternative configuration values backed by an [OpenShift ConfigMap]({{ OPENSHIFT_DOCS_BASE }}/dev_guide/configmaps.html){:target="_blank"}.

The ConfigMap object in OpenShift and Kubernetes provides mechanisms to inject containers with configuration 
data while keeping containers agnostic of OpenShift and Kubernetes. A ConfigMap can be used to store fine-grained 
information like individual properties or coarse-grained information like entire configuration files or JSON blobs.

A ConfigMap has been created for you already, which supplies the necessary PostgreSQL configuration to be
able to connect to the PostgreSQL database. Run the following in Eclipse Che **Terminal**:

~~~shell
oc describe cm catalog
~~~

You should see something like the following:

~~~shell
Name:         catalog
Namespace:    dev
Labels:       app=catalog
Annotations:  openshift.io/generated-by=OpenShiftNewApp

Data
====
application.properties:
----
server.port=8080
spring.application.name=catalog
feign.hystrix.enabled=false
spring.datasource.url=jdbc:postgresql://catalog-postgresql:5432/catalog
spring.datasource.username=user2av
spring.datasource.password=2Y3ieayR
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=create
Events:  <none>
~~~

The username and password was generated when you first deployed the catalog template, and the ConfigMap shown
above was also created at that time. This ConfigMap consists of one config item, as you can see above, which is 
a file called `application.properties` which the content the follows it after `----`.

### Use ConfigMap with Spring Boot

Let's modify our application to use it. We will use the [Spring Cloud Kubernetes](https://github.com/fabric8io/spring-cloud-kubernetes#configmap-propertysource){:target="_blank"}
project. Using this dependency, Spring Boot will search for a ConfigMap (by default with the same name as
the application) to use as the source of application configuration during application bootstrapping and
if enabled, triggers hot reloading of beans or Spring context when changes are detected on the ConfigMap.
Add the following dependency to your `pom.xml` beneath the existing
dependencies (look for the `<!-- add additional dependencies -->` comment):

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
~~~

Although Spring Cloud Kubernetes tries to discover ConfigMaps, due to security reasons containers
by default are not allowed to snoop around OpenShift clusters and discover objects. Security comes first,
and discovery is a privilege that needs to be granted to containers in each project.

Since you do want Spring Boot to discover the ConfigMaps inside the **dev** project, you
need to grant permission to the Spring Boot service account to access the OpenShift REST API and find the
ConfigMaps. To grant this permission, run the following in Eclipse Che **Terminal**:

~~~bash
oc policy add-role-to-user view -n dev{{PROJECT_SUFFIX}} -z default
~~~

Let's re-run our tests just to make sure our new additions don't cause breakage:

Click on **View command palette** button up to the right.

![View command palette button]({% image_path service-to-service-view-cmd-palette.png %}){:width="340px"}

Then double click on the **test** command.

![View command palette test button]({% image_path service-to-service-cmd-palette.png %}){:width="340px"}

At the end of the output you should see the following which says all tests have been successfully.

~~~shell
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0

...

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 16.273 s
[INFO] Finished at: 2018-05-02T16:54:53+00:00
[INFO] Final Memory: 28M/48M
[INFO] ------------------------------------------------------------------------
~~~

If your test fails go back and check previous steps before moving to the next section.

### Push Code Changes to Git Repository

Commit the `pom.xml` changes to code repository:

* In the project explorer, right click on catalog and then Git > Commit
* Make sure `pom.xml` is checked
* Add a commit message `externalized database configuration`
* Check the **Push commit changes to...**
* Click on **Commit**

This will trigger the pipeline build.

Open the [OpenShift Web Console]({{OPENSHIFT_MASTER_URL}}){:target="_blank"} and click on **Builds** > **Pipelines** to watch the pipeline build and deploy the updated catalog code:

![Build Pipeline]({% image_path create-catalog-pipeline.png %}){:width="700px"}

|**NOTE:** If your pipeline is not triggered, you may have forgotten to *Push* the changes after commit. In that case, 
simply use the **Git -> Remotes -> Push...** menu to push the committed changes.

Wait for it to complete. At this point the application
should fetch the configuration and start using PostgreSQL instead of H2.

### Verify Configuration

Let's verify that PostgreSQL is being used. First, let's dump the database contents by
running the `psql` utility command from within the PostgreSQL container to verify that the seed data is loaded:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"select itemId, name, price from catalog\""
~~~

You should see the seed data gets listed.

~~~
 itemid |          name          | price
--------+------------------------+-------
 329299 | Red Fedora             | 34.99
 329199 | Forge Laptop Sticker   |   8.5
 165613 | Solid Performance Polo |  17.8
 165614 | Ogio Caliber Polo      | 28.75
 165954 | 16 oz. Vortex Tumbler  |     6
 444434 | Pebble Smart Watch     |    24
 444435 | Oculus Rift            |   106
 444436 | Lytro Camera           |  44.3
(8 rows)
~~~

Now let's do a quick update of the price of the _Lytro Camera_ product to make it cost 100.00 by referring to its
itemid of `444436`:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"update catalog set price=100.0 where itemid='444436'\""
~~~

You will see a returned message of `UPDATE 1` indicating that the price was updated in Postgres. Let's re-fetch the catalog
and verify the new price:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"select itemId, name, price from catalog\""
~~~

The _Lytro Camera_ product should now cost `100`:

~~~
 itemid |          name          | price
--------+------------------------+-------
 329299 | Red Fedora             | 34.99
 329199 | Forge Laptop Sticker   |   8.5
 165613 | Solid Performance Polo |  17.8
 165614 | Ogio Caliber Polo      | 28.75
 165954 | 16 oz. Vortex Tumbler  |     6
 444434 | Pebble Smart Watch     |    24
 444435 | Oculus Rift            |   106
 444436 | Lytro Camera           |   100  <----- New price!
~~~

|**CAUTION:** The SQL UPDATE command is very powerful and can wreak havoc on production databases and unsuspecting applications not designed to deal with changing data at this level. There may also be caches in place preventing low level updates from having any effect, so use sparingly and with caution!

### Congratulations!

You've now got a quick way to alter service configuration without redeploying! As the application moves
through different environments (test, staging, production), it will pick up its configuration via a 
ConfigMap within each environment, rather than being re-compiled with the new configuration each time.
This mechanism can also be used to alter business logic in addition to infrastructure (database, etc)
configuration.