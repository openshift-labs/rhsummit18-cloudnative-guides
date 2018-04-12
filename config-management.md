## Configuration management with Spring Boot

So far our catalog service has been using an in-memory H2 database. Although H2 is a
convenient database to run locally on your laptop, it's in no ways appropriate for production
or even integration tests. Since it's strongly recommended to use the same technology stack
(operating system, JVM, middleware, database, etc) that is used in production across all environments,
we will modify the Catalog service to use a PostgreSQL database instead of the H2 in-memory database.

Fortunately, OpenShift supports stateful applications such as databases which required access to a
persistent storage that survives the container itself. You can deploy databases on OpenShift and
regardless of what happens to the container itself, the data is safe and can be used by the next
database container.

As part of this lab, a PostgreSQL database has already been deployed, which you can see:

~~~console
$ oc get pods
NAME                         READY     STATUS    RESTARTS   AGE
catalog-postgresql-1-kbs84   1/1       Running   0          2d
~~~

Our Spring Boot Catalog application configuration is provided via a properties filed called `application-default.properties`
and can be overriden and [overlayed via multiple mechanisms](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html).

In this scenario, you will configure the Catalog service which is based on Spring Boot to override the default
H2 database configuration using alternative configuration values backed by an [OpenShift ConfigMap](https://docs.openshift.org/latest/dev_guide/configmaps.html).

The ConfigMap has been created for you already, which supplies the necessary PostgreSQL configuration to be
able to connect to the PostgreSQL database:

~~~yaml
% oc get configmap/catalog -o yaml
apiVersion: v1
data:
  application.properties: |-
    spring.datasource.url=jdbc:postgresql://catalog-postgresql:5432/catalog
    spring.datasource.username=********
    spring.datasource.password=********
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=create
kind: ConfigMap
metadata:
    ...
~~~

The username and password was generated when you first deployed the catalog template, and the ConfigMap shown
above was also created at that time.

### Use ConfigMap

Let's modify our application to use it. We will use the [Spring Cloud Kubernetes](https://github.com/fabric8io/spring-cloud-kubernetes#configmap-propertysource)
project. Using this dependency, Spring Boot will search for a ConfigMap (by default with the same name as
the application) to use as the source of application configuration during application bootstrapping and
if enabled, triggers hot reloading of beans or Spring context when changes are detected on the ConfigMap.
We'll use auto-reload later, but for now, add the following dependency to your `pom.xml` beneath the existing
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
ConfigMaps. To grant this permission, run:

~~~bash
oc policy add-role-to-user view -n dev -z default
~~~

Next,
Re-run the unit tests to ensure they still pass:

~~~console
$ mvn verify
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 24.662 s
[INFO] Finished at: 2018-04-06T16:17:33-04:00
[INFO] Final Memory: 36M/327M
[INFO] ------------------------------------------------------------------------
~~~

### Commit code changes

Commit the `pom.xml` changes to code repo:

* In the project explorer, right click on catalog and then Git > Commit
* Make sure `pom.xml` is checked
* Add a commit message "externalized database configuration"
* Check the "Push commit changes to..."
* Click on "Commit"

This will trigger the pipeline build. Wait for it to complete. At this point the application
should fetch the configuration and start using PostgreSQL instead of H2.

### Verify configuration

When the Catalog pod is ready, verify that the PostgreSQL database is being
used. Check the Catalog pod logs:

~~~sh
oc logs dc/catalog | grep hibernate.dialect
~~~

You would see the **PostgreSQL94Dialect** is selected by Hibernate in the logs:

~~~console
2017-08-10 21:07:51.670  INFO 1 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.PostgreSQL94Dialect
~~~

You can also connect to the Catalog PostgreSQL database and verify that the seed data is loaded:

~~~sh
oc rsh dc/catalog-postgresql
~~~

Once connected to the PostgreSQL container, run the following:

> Run this command inside the Catalog PostgreSQL container, after opening a remote shell to it.

~~~sh
psql -U $POSTGRESQL_USER -d $POSTGRESQL_DATABASE -c "select itemId, name, price from catalog"
~~~

You should see the seed data gets listed.

~~~
 itemid |            name             | price
----------------------------------------------
 329299 | Red Fedora                  | 34.99
 329199 | Forge Laptop Sticker        |   8.5
 ...
~~~

Exit the container shell.

~~~
exit
~~~

### Test the result on OpenShift

  APPS_HOSTNAME_SUFFIX: "apps.GUID.generic.opentlc.com"

Test the catalog running on OpenShift:

`open http://catalog-dev.{{APPS_HOSTNAME_SUFFIX}}`

Observe all products are present!

### Congratulations!

You've now got a quick way to alter service configuration without redeploying! As the application moves
through different environments (test, staging, production), it will pick up its configuration via a
ConfigMap within each environment, rather than being re-compiled with the new configuration each time.

This mechanism can also be used to alter business logic in addition to infrastructure (database, etc)
configuration. We'll employ this later in in the Fault Tolerance exercise to do a custom feature toggle
for our services.