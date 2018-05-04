## Connect the Catalog Service to Inventory Service.

We now have a working Catalog Service, but you might have notices that we still don't have inventory information. 

![Inventory Undefined]({% image_path create-catalog-inventory-undefined.png %}){:width="700px"}

The UI team has asked us to include inventory information in the responses when calling the catalog service. However, the inventory information is owned by another team. The inventory application expose a REST endpoint that we can use to collect inventory information the response. 

By calling the inventory service in production like this:

~~~shell
curl -w '\n' http://inventory-prod{{PROJECT_SUFFIX}}.{{APPS_HOSTNAME_SUFFIX}}/services/inventory/444434
~~~

We can see that the response from the curl command looks like this:

~~~
{
  "itemId":"444434",
  "quantity":343
}
~~~

Now that we know what the inventory service looks like and how to call it we can extend our catalog service to use it.

### Creating the Inventory Model

We need a inventory model that matches the return from the inventory service so we start by creating a new Java class under `src/main/java/com/redhat/coolstore/model` called `Inventory` that looks like this:

~~~java
package com.redhat.coolstore.model;

public class Inventory {
    private String itemId;
    private int quantity;
}
~~~

Generate the getter and setter methods using `Ctrl+Space` and writing `get` or `set`.

We also need to extend the `Product` model to contain the quantity information. Add the field to the 
`Product` class and generate the getter and setter methods.

~~~java
    @Transient
    private int quantity;
~~~

Note the `javax.persistence.Transient` annotation which shows that this field should not be persisted in the product table 
and it's rather coming from somewhere else (the inventory table!).

|**STEP BY STEP:** Creating the inventory model
|![New file]({% image_path service-to-service-model.gif %}){:width="900px"}

### Creating the Inventory REST Client

Spring Cloud has a client library called Feign that will save us a lot of boiler plate code. To implement a REST client in Feign all you have to do is to create a interface and annotate it with the necessary clients.

First, we need to add a couple of dependencies. Add the following the `pom.xml` in the dependency section, preferably right above `<!-- Testing -->`:

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
    <version>1.4.4.RELEASE</version>
</dependency>
~~~

|**NOTE:** After changing the `pom.xml` it's recommended that you build the project so that new dependencies are downloaded

Now we are ready to create a client. Using Feign all we have to do to is to declare a interface and annotate it with `@FeignClient`. We will create the interface in the `com.redhat.coolstore.client` package and call it `InventoryClient`

~~~java
package com.redhat.coolstore.client;

@FeignClient(name="inventory")
public interface InventoryClient {

}
~~~

We also have to define a method that returns the inventory object for the given item id and also give Feign 
some meta data about the request to the inventory service using annotations like this:

~~~java
    @RequestMapping(method = RequestMethod.GET, value = "/services/inventory/{itemId}", consumes = {MediaType.APPLICATION_JSON_VALUE})
    Inventory getInventoryStatus(@PathVariable("itemId") String itemId);
~~~

Just to make sure you have the correct imports, here is how they should look like after 
running **Assistant** > **Organize Imports**.

~~~java
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import com.redhat.coolstore.model.Inventory;
~~~

Finally, we also have to tell Spring to look for the `@FeginClient` annotation and automatically create an implementation for it. That is done by enabling Feigh on the Spring application.

Open `com/redhat/coolstore/CatalogServiceApplication.java` and add the class-level `@EnableFeignClients` annotation 
to it.

|**STEP BY STEP:** Creating the client
|![New file]({% image_path service-to-service-client.gif %}){:width="900px"}

### Testing the Inventory REST Client

Before we put the client in use, let's create a unit test for it. However, to test REST calls we need to be able to mock the actual call. For that we will use a test framework called `Howeverfly`. Let's add the dependency for `Hoverfly` and `AssertJ` to the `pom.xml`, preferably below `<!-- Testing -->`

~~~xml
    <dependency>
        <groupId>io.specto</groupId>
        <artifactId>hoverfly-java</artifactId>
        <version>0.9.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.8.0</version>
        <scope>test</scope>
    </dependency>
~~~

|**NOTE:** After changing the `pom.xml` it's recommended that you build the project again so that new dependencies are downloaded

First, create a class called `InventoryClientTest` under `src/test/java/com/redhat/coolstore/client` with the following code:

~~~java
package com.redhat.coolstore.client;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class InventoryClientTest {

}
~~~

Then, inject the `InventoryClient` which is the class to be tested.

~~~java
    @Autowired
    InventoryClient inventoryClient;
~~~

Next, create a mockInventory that Hoverfly simulation can return.

~~~java
    private static Inventory mockInventory;

    static {
        mockInventory = new Inventory();
        mockInventory.setQuantity(98);
        mockInventory.setItemId("1234");
    }
~~~

After that, create a `@ClassRule` that starts the Hoverfly simulator:

~~~java
    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
            service("mock-service.example.com:8080")
                    .get(startsWith("/services/inventory"))
                    .willReturn(success(json(mockInventory)))));
~~~

Finally, create the test itself

~~~java
    @Test
    public void testInventoryClient() {
        Inventory inventory = inventoryClient.getInventoryStatus("1234");
        assertThat(inventory)
                .returns(98,i -> i.getQuantity())
                .returns("1234",i -> i.getItemId());
    }
~~~

After running **Assistant** > **Organize Imports**, add the following static imports to 
`InventoryClientTest` looks like:

~~~java
import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;
import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static io.specto.hoverfly.junit.dsl.matchers.HoverflyMatchers.startsWith;
import static org.assertj.core.api.Assertions.assertThat;
~~~

Take a while and study the test class and notice that we inject a inventory client and then use that in the test to verify that we get a correct response. But how can we test a remote call in a unit test without actually having a service to call? That is where [Hoverfly](http://hoverfly.io){:target="_blank"} comes into play. Hoverfly can simulate API services by registering as a proxy to any outgoing traffic. In the `@ClassRule` we can defines how Hoverfly should respond to different calls.

So how does the Feign client know which URL to call? We will discuss the mechanics of service discovery later, but for now we can set a property called `ribbon.listOfServers`.

For the unit tests configurations we are going to provide the settings via a test application properties files. Create 
a file called `application-default.properties` in `src/test/resources` with the following content:
this file with the following content:

~~~shell
ribbon.listOfServers=mock-service.example.com:8080
~~~

|**IMPORTANT:** Notice that this file is located in `src/test/resources` and not `src/main/resources`

We are now finally ready to run the test. 

### Connecting the Catalog Service with Inventory

Since we now have a working inventory client we are now ready to extend the Catalog endpoint 
use it so that we can provide additional input.

Open `com/redhat/coolstore/service/ProductEndpoint.java`

Add an injection of an `InventoryClient` like this:

~~~java
    @Autowired
    private InventoryClient inventoryClient;
~~~

Next, replace the implementation of the `readOne` method of the `ProductEndpoint` to look like this:

~~~java
    @ResponseBody
    @GetMapping("/product/{id}")
    public ResponseEntity<Product> readOne(@PathVariable("id") String id) {
        Product product = productRepository.findOne(id);
        Inventory inventory = inventoryClient.getInventoryStatus(id);
        product.setQuantity(inventory.getQuantity());
        return new ResponseEntity<Product>(product,HttpStatus.OK);
    }
~~~

Also, replace the implementation of the `readAll` method of the `ProductEndpoint` to look like this:

~~~java
    @ResponseBody
    @GetMapping("/products")
    public ResponseEntity<List<Product>> readAll() {
        Spliterator<Product> iterator = productRepository.findAll().spliterator();
        List<Product> products = StreamSupport.stream(iterator, false).collect(Collectors.toList());
        products.stream().forEach(p -> p.setQuantity(inventoryClient.getInventoryStatus(p.getItemId()).getQuantity()));
        return new ResponseEntity<>(products, HttpStatus.OK);
    }
~~~

### Testing the Catalog REST API

Wouldn't it be nice to also be able to test the endpoint, just like we tested the inventory client. Spring testing provides a nice templating feature where we can call services by injecting a `TestRestTemplate`. Spring will then take care for connecting that TestRestTemplate to the host and port that our application is started on during a unit test. Since the ProductEndpoint also uses the inventory client we need to simulate the inventory service as well. 

Start by creating a class in `src/test/java/com/redhat/coolstore/service` called `ProductEndpointTest` with the following 
code:

~~~java
package com.redhat.coolstore.service;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ProductEndpointTest {

}
~~~

Next, we will inject the `TestRestTemplate` as a class member variable like this

~~~java
    @Autowired
    private TestRestTemplate restTemplate;
~~~

We also need some test data that Hoverfly can return. These can be declared as static member variables like this:

~~~java
{% raw  %}
    private static Inventory mockFedoraInventory, mockStickersInventory, mockDefaultInventory;

    static {
        mockFedoraInventory = new Inventory();
        mockFedoraInventory.setQuantity(123);
        mockFedoraInventory.setItemId("329299");
        
        mockStickersInventory = new Inventory();
        mockStickersInventory.setQuantity(98);
        mockStickersInventory.setItemId("329199");
    
        mockDefaultInventory = new Inventory();
        mockDefaultInventory.setQuantity(0);
        mockDefaultInventory.setItemId("{{ Request.Path.[3] }}");
    }
{% endraw %}
~~~

Next, we define the Hoverfly rule like this:

~~~java
    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
        service("mock-service.example.com:8080")                     
                .get("/services/inventory/329299")
                    .willReturn(success(json(mockFedoraInventory)))
                .get("/services/inventory/329199")
                    .willReturn(success(json(mockStickersInventory)))
                .get("/services/inventory/165613")
                    .willReturn(success(json(mockDefaultInventory)))
                .get("/services/inventory/165614")
                    .willReturn(success(json(mockDefaultInventory)))
                .get("/services/inventory/165954")
                    .willReturn(success(json(mockDefaultInventory)))
                .get("/services/inventory/444434")
                    .willReturn(success(json(mockDefaultInventory)))
                .get("/services/inventory/444435")
                    .willReturn(success(json(mockDefaultInventory)))
                .get("/services/inventory/444436")
                    //.willReturn(serverError().body("Inventory is down"))
                    .willReturn(success(json(mockDefaultInventory)))
    ));
~~~

This takes care of all the setup and configuration of the tests. We can now implement our tests. Let's start with a test that will test retrieving a single product like this:

~~~java
    @Test
    public void test_retrieving_one_product() {
        hoverflyRule.printSimulationData();
        ResponseEntity<Product> response
                = restTemplate.getForEntity("/services/product/329199", Product.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        Product product = response.getBody();
        assertThat(product)
                .returns("329199",p -> p.getItemId())
                .returns("Forge Laptop Sticker",p -> p.getName())
                .returns(98,p -> p.getQuantity())
                .returns(8.50,p -> p.getPrice());
    }
~~~

At this point we also create a test that check that the endpoint also returns a correct list with inventory data in it.

~~~java
    @Test
    public void check_that_endpoint_returns_a_correct_list() {
        ResponseEntity<List<Product>> rateResponse =
                restTemplate.exchange("/services/products",
                        HttpMethod.GET, null, new ParameterizedTypeReference<List<Product>>() {
                        });

        List<Product> productList = rateResponse.getBody();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> names = productList.stream().map(p -> p.getName()).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift");   
    }
~~~

If you haven't already then it's time to organize our imports. However, since there is a lot of static method imports in this test class you can just past in the full list of imports from the below:

~~~java
import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;
import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.serverError;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static org.assertj.core.api.Assertions.assertThat;
import io.specto.hoverfly.junit.rule.HoverflyRule;

import java.util.List;
import java.util.stream.Collectors;

import org.junit.ClassRule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import com.redhat.coolstore.model.Inventory;
import com.redhat.coolstore.model.Product;
~~~

Now we are ready to run our tests.

Click on **View command palette** button up to the right.

![View command palette button]({% image_path service-to-service-view-cmd-palette.png %}){:width="340px"}

Then double click on the **test** command.

![View command palette button]({% image_path service-to-service-cmd-palette.png %}){:width="340px"}

If your test fails go back and check previous steps before moving to the next section.

~~~shell
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
 ...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
~~~

|**STEP BY STEP:** Creating tests for the endpoint

|![New file]({% image_path service-to-service-endpoint-test.gif %}){:width="900px"}

### Fallback Strategies

Testing is good for many reason, but not only to verify our code. For example, what would happen if the inventory service didn't respond? Let's test that. 

Open the `ProductEndpointTest` again and in the `@ClassRule` definition, lets introduce a server error bye un-comment the provided error response for calls to `/services/inventory/444436` (around line 70). Also comment out the successful response.

The code should now look something like this:

~~~java
        .get("/services/inventory/444436")
            .willReturn(serverError().body("Inventory is down"))
            //.willReturn(success(json(mockDefaultInventory)))
~~~

Run the test again.

|**NOTE:** The test are expected to fail.

~~~shell
[INFO] Results:
[INFO] 
[ERROR] Errors: 
[ERROR]   ProductEndpointTest.check_that_endpoint_returns_a_correct_list:92 » HttpMessageNotReadable
[INFO] 
[ERROR] Tests run: 5, Failures: 0, Errors: 1, Skipped: 0
~~~

This time we can see that the `test_retrieving_one_product()` test ends with a nasty errors 
and if we look closer we can see that the call to our catalog service fails. The reason is 
that the catalog service does a number of calls to `/services/inventory/{itemId}` and even 
if 7 out of 8 calls where successful, one failure causes our catalog service 
to fail as well. We need a fallback strategy!

After discussing this with the UI team we have come to the conclusion that if a call to the inventory service fails the catalog service should return a `quantity` of `-1`. The UI will then detect this and display `undefined`.

There are a number of ways of doing fall back using Feign, but at the core Feign 
will throw a `FeignException` that we can catch and react to.

Open the `ProductEndpoint` class and update the `readOne` and `readAll` methods to look like this:

~~~java
    @ResponseBody
    @GetMapping("/products")
    public ResponseEntity<List<Product>> readAll() {
        Spliterator<Product> iterator = productRepository.findAll().spliterator();
        List<Product> products = StreamSupport.stream(iterator, false).collect(Collectors.toList());
        products.stream()
                .forEach(p -> {
                    try {
                        p.setQuantity(inventoryClient.getInventoryStatus(p.getItemId()).getQuantity());
                    } catch (feign.FeignException e) {
                        p.setQuantity(-1);
                    }
                });
        return new ResponseEntity<>(products, HttpStatus.OK);
    }
    
    @ResponseBody
    @GetMapping("/product/{id}")
    public ResponseEntity<Product> readOne(@PathVariable("id") String id) {
        Product product = productRepository.findOne(id);
        try {
            Inventory inventory = inventoryClient.getInventoryStatus(id);
            product.setQuantity(inventory.getQuantity());
        } catch (feign.FeignException e) {
            product.setQuantity(-1);
        }
        return new ResponseEntity<Product>(product,HttpStatus.OK);
    }
~~~

Now, run the tests again and verify that even if we return a server error for inventory item **444436** so will the application still give an acceptable response.

~~~shell
[INFO] Results:
[INFO] 
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
~~~

|**STEP BY STEP:** Implementing a fallback strategy
|![New file]({% image_path service-to-service-fallback.gif %}){:width="640px"}

### Deploy the Application to OpenShift

Before we deploy the a new version of the catalog service our application now depends on the inventory service available. We could use the product version of the inventory, but since that might affect performance of the production system. For that reason the inventory team has provided a mockup of the inventory service as a OpenShift template. The mock service behaves like the real service, but instead of actually calling the back-end system to retrieve the information, it returns a hard coded value of the inventory.

Go to the [OpenShift Web Console]({{ OPENSHIFT_MASTER_URL }}){:target="_blank"}:

* Username: `{{OPENSHIFT_USERNAME}}`
* Password: `{{OPENSHIFT_PASSWORD}}`

Install the inventory-mockup into the **Catalog Dev** project. Click on the search box in the service catalog 
and enter `inventory`. Click on **inventory-mockup** and then **Next** and **Next** leaving the parameters 
with their default values.

|**CAUTION:** Replace GUID with the guid provided to you

|**STEP BY STEP:** Create a mockup service
|![New file]({% image_path service-to-service-mockup.gif %}){:width="640px"}

|**NOTICE:** The inventory-mock template has already been installed in OpenShift since other teams are also depending using this mockup application.

Now, before we deploy our application we need to update the configmap with the service URL. We can of course automate this via for example the pipeline, but for simplicity reasons we are going to use the web console for that.

On the left-side menu click on **Resources** > **Config Maps** and then **catalog** to open the 
configmap that is used by the Catalog service. Click on the **Actions** menu and then **Edit**.

Add the following property as the last line and then click on **Save**.

~~~shell
ribbon.listOfServers=inventory:8080
~~~

|**STEP BY STEP:** Update the configmap
|![New file]({% image_path service-to-service-configmap.gif %}){:width="640px"}

|**NOTICE:** We mentioned before that Feign has a number of different possibilities to use a service repository like Eureka, but when using OpenShift that is not necessary. OpenShift (and Kubernetes) has a concept of a [Service]({{OPENSHIFT_DOCS_BASE}}/architecture/core_concepts/pods_and_services.html#services){:target="_blank"} that provides built-in service discovery 
by automatically be load balanced between multiple instances of the same service with a fixed service name that doesn't change 
across environments. Later one we will also look at how Istio service mesh can provide more advanced routing and fail-over.

Now, that everything is setup we are ready to deploy our new version of the application by commit and push the code changes we have done.

|**STEP BY STEP:** Commit and push the changes
|![New file]({% image_path service-to-service-commit.gif %}){:width="640px"}

After pushing, the pipeline will automatically start (based on the hook we configured in lab 1). Let's go and look at the pipelines progress in the console.

|**STEP BY STEP:** Commit and push the changes
|![New file]({% image_path service-to-service-pipeline.gif %}){:width="640px"}

### Summary

Congratulations, you have now successfully integrated our Catalog application with the inventory application using the Spring Cloud Feign Client. We are now ready to deploy the application to the Production environment, which we will look at in the next lab.
