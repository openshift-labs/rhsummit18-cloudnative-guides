## Connect the Catalog service to Inventory Service.

We now have a working Catalog Service, but you might have notices that we still don't have inventory information. 

![Inventory Undefined]({% image_path create-catalog-inventory-undefined.png %}){:width="700px"}

The UI team has asked us to include inventory information in the responses when calling the catalog service. However, the inventory information is owned by another team. The inventory team have agreed to expose a REST endpoint that we can use to collect inventory information.

### Deploy the Inventory Service

Our first task is to create the inventory service. The implementation are irrelevant, but we need to test and verify our deployment.

~~~shell
oc process -f https://raw.githubusercontent.com/openshift-labs/rhsummit18-cloudnative-labs/master/openshift/inventory-template.yml | oc create -f -
~~~

Check that inventory service is successfully rolled out.

~~~shell
oc rollout status dc inventory
~~~

Test the inventory endpoint:

|**CAUTION:** Replace `GUID` with the guid provided to you.

~~~shell
curl http://inventory-dev.{{APPS_HOSTNAME_SUFFIX}}/services/inventory/444434
~~~

The response from the curl command should look like this:

~~~
{
  "itemId":"444434",
  "quantity":343
}
~~~

Now that we know what the inventory service looks like and how to call it we can extend our catalog service to use it.

### Creating the Inventory Model

We need a inventory model that matches the return from the inventory service so we start by creating a new Java class under `src/main/java/com/redhat/coolstore/model` called Inventory that looks like this:

~~~java
package com.redhat.coolstore.model;

public class Inventory {

    public String itemId;
    public int quantity;

}
~~~

### Creating the Client

Spring Cloud has a client library called Feign that will save us a lot of boiler plate code. To implement a REST client in Feign all you have to do is to create a interface and annotate it with the necessary clients.

First, we need to add a couple of dependencies. Add the following the `pom.xml` in the dependency section:

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
~~~

Now, we are ready to create our client:

~~~java
package com.redhat.coolstore.client;

import com.redhat.coolstore.model.Inventory;
import org.springframework.http.MediaType;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name="inventory")
public interface InventoryClient {
    @RequestMapping(method = RequestMethod.GET, value = "/services/inventory/{itemId}", consumes = {MediaType.APPLICATION_JSON_VALUE})
    Inventory getInventoryStatus(@PathVariable("itemId") String itemId);
}
~~~

The `Feign` client will use the name to lookup a service from a service registry. However, when we run unit tests or when we test the application locally we do not have a service registry. To solve that we are going to provide a default configuration by adding the following line to `src/test/resources/application-default.properties`

~~~java
ribbon.listOfServers=mock.service.url:9999
~~~

|**IMPORTANT:** Notice that this file is created in `src/test/resources` and not `src/main/resources`

### Implementing a Fallback Strategy

Calling an external service is easy, but we also have to consider what will happen if a service is not responding or if it's slow.  We will look closer at all that in then module `Fault tolerance and Resilience`, but we still have to consider a fallback strategy. What happens if the inventory client doesn't respond or respond with an error. There for we will update the `InventoryClient` to look like this:

~~~java
@FeignClient(name="inventory",fallbackFactory = InventoryClient.InventoryClientFallbackFactory.class)
public interface InventoryClient {
    @RequestMapping(method = RequestMethod.GET, value = "/services/inventory/{itemId}", consumes = {MediaType.APPLICATION_JSON_VALUE})
    Inventory getInventoryStatus(@PathVariable("itemId") String itemId);

    @Component
    static class InventoryClientFallbackFactory implements FallbackFactory<InventoryClient> {
        @Override
        public InventoryClient create(Throwable cause) {
            return new InventoryClient() {
                @Override
                public Inventory getInventoryStatus(@PathVariable("itemId") String itemId) {
                    Inventory fallbackInventory = new Inventory();
                    fallbackInventory.itemId = itemId;
                    fallbackInventory.quantity = -1;
                    return fallbackInventory;
                }
            };
        }
    }
}

Before we put the client in use, let's create a unit test for it. However, to test REST calls we need to be able to mock the actual call. For that we will use a test framework called `Howeverfly`. Let's add the dependency for `Hoverfly` and `AssertJ` to the `pom.xml`

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

Now, we can write a test class for our client. Let's first create a new package under `src/test/java` called `com.redhat.coolstore.client`. Then, add a class called `InventoryClientTest`.

~~~java
package com.redhat.coolstore.client;

import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;
import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static io.specto.hoverfly.junit.dsl.matchers.HoverflyMatchers.startsWith;
import static org.assertj.core.api.Assertions.assertThat;
import io.specto.hoverfly.junit.rule.HoverflyRule;

import org.junit.ClassRule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.redhat.coolstore.model.Inventory;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class InventoryClientTest {

    @Autowired
    InventoryClient inventoryClient;

    private static Inventory mockInventory;

    static {
        mockInventory = new Inventory();
        mockInventory.quantity = 98;
        mockInventory.itemId = "1234";
    }

    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
            service("mock.service.url:9999")
                    .get(startsWith("/services/inventory"))
                    .willReturn(success(json(mockInventory)))));

    @Test
    public void testInventoryClient() {
        Inventory inventory = inventoryClient.getInventoryStatus("1234");
        assertThat(inventory)
                .returns(98,i -> i.quantity)
                .returns("1234",i -> i.itemId);
    }
}
~~~

Notice how we use a `@ClassRule` to create a mock service using `mock.service.url` as the URL and port `9999` matching what we previously specified in `src/test/resources/application-default.properties`. Howeverfly will mock a REST service by intercepting all REST calls. So when run `inventoryClient.getInventoryStatus("1234")` in the test the Feign client will call `http://mock.service.url:9999/services/inventory/98` and Howeverfly should return our `mockInventory` object.

We are now finally ready to run the test. :-) 

Click on the **Command Palette** button up to the right
![View command palette button]({% image_path service-to-service-view-cmd-palette.png %})

Then double click on the **TEST** command:
![View command palette button]({% image_path service-to-service-cmd-palette.png %}) 

If your test fails go back and check previous steps before moving to the next section.

### Using the Inventory Client

Since we now have a working inventory client we are now ready to extend the Catalog Service to use it.

First lets, add a variable to the the `Product` class to hold the quantity from the inventory. The list of variable should look like this:


| MODIFIER |    NAME     |  TYPE  |
|----------|-------------|--------|
| public   | itemId      | String |
| public   | name        | String |
| public   | description | String |
| public   | price       | double |
| public   | quantity    | int    |


Do NOT update the Constructor with the new parameter since that will mess with our `RowMapper`. 

Next, add an injection of an `InventoryClient` to the `ProductEndpoint` like this:

~~~java
@Autowired
private InventoryClient inventoryClient;
~~~

Next, change the implementation of the `readOne` method of the `ProductEndpoint` to look like this:

~~~java
@ResponseBody
@GetMapping("/product/{id}")
public ResponseEntity<Product> readOne(@PathVariable("id") String id) {
    Product product = productRepository.findById(id);
    Inventory inventory = inventoryClient.getInventoryStatus(id);
    product.quantity = inventory.quantity;
    return new ResponseEntity<Product>(product,HttpStatus.OK);
}
~~~

Also, change the implementation of the `readAll` method of the `ProductEndpoint` to look like this:

~~~java
@ResponseBody
@GetMapping("/products")
public ResponseEntity<List<Product>> readAll() {
    List<Product> productList = productRepository.readAll();
    productList.stream()
            .forEach(p -> {
                p.quantity = inventoryClient.getInventoryStatus(p.itemId).quantity;
            });
    return new ResponseEntity<List<Product>>(productList,HttpStatus.OK);
}
~~~

Finally, create a new `Test` to test our `Endpoint` in `src/test/java/com/redhat/coolstore/service` called `ProductEndpointTest` like this:

~~~java
package com.redhat.coolstore.service;

import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.response;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;
import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static io.specto.hoverfly.junit.dsl.matchers.HoverflyMatchers.startsWith;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.stream.Collectors;

import io.specto.hoverfly.junit.rule.HoverflyRule;

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


@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ProductEndpointTest {

    @Autowired
    private TestRestTemplate restTemplate;

    private static Inventory mockFedoraInventory, mockStickersInventory, mockDefaultInventory;

    static {
        mockFedoraInventory = new Inventory();
        mockFedoraInventory.quantity = 123;
        mockFedoraInventory.itemId = "329299";
        
        mockStickersInventory = new Inventory();
        mockStickersInventory.quantity = 98;
        mockStickersInventory.itemId = "329199";
    
        mockDefaultInventory = new Inventory();
        mockDefaultInventory.quantity = 0;
        mockDefaultInventory.itemId = "Undefined";
    }

    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
            service("mock.service.url:9999")
                    .get(startsWith("/services/inventory/329299"))
                        .willReturn(success(json(mockFedoraInventory)))
                    .get(startsWith("/services/inventory/329199"))
                        .willReturn(success(json(mockStickersInventory)))
                    .get(startsWith("/services/inventory/444436"))
                        .willReturn(response().status(503))
                    .get(startsWith("/services/inventory"))
                        .willReturn(success(json(mockDefaultInventory)))
    ));
    
    @Test
    public void test_retriving_one_proudct() {
        ResponseEntity<Product> response
                = restTemplate.getForEntity("/services/product/329199", Product.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        Product product = response.getBody();
        assertThat(product)
                .returns("329199",p -> p.itemId)
                .returns("Forge Laptop Sticker",p -> p.name)
                .returns(98,p -> p.quantity)
                .returns(8.50,p -> p.price);
    }
    
    @Test
    public void check_that_endpoint_returns_a_correct_list() {
        ResponseEntity<List<Product>> rateResponse =
                restTemplate.exchange("/services/products",
                        HttpMethod.GET, null, new ParameterizedTypeReference<List<Product>>() {
                        });

        List<Product> productList = rateResponse.getBody();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> names = productList.stream().map(p -> p.name).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift");   
    }
    
    @Test
    public void test_fallback() {
        ResponseEntity<Product> response
                = restTemplate.getForEntity("/services/product/444436", Product.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        Product product = response.getBody();
        assertThat(product)
                .returns("444436",p -> p.itemId)
                .returns("Lytro Camera",p -> p.name)
                .returns(-1,p -> p.quantity)
                .returns(44.30,p -> p.price);
    }
}
~~~

Now we are ready to test our Inventory integration locally.

Click on **View command palette** button up to the right
![View command palette button]({% image_path service-to-service-view-cmd-palette.png %})

Then double click on the **TEST** command:
![View command palette button]({% image_path service-to-service-cmd-palette.png %}) 

If your test fails go back and check previous steps before moving to the next section.

### Test the Application Locally

If you recall we had to set the list of servers to a hard coded value to run the unit test. If we want to run the test locally we will have to do the same and use the inventory service that we deployed to openshift before. We can do that by adding a environment specific configuration. Create a file called `src/main/resources/application-local.properties` with the following content:

|**CAUTION:** Replace `GUID` with the guid provided to you.

~~~java
ribbon.listOfServers=inventory-dev.{{APPS_HOSTNAME_SUFFIX}}
~~~

### Deploy the Application to OpenShift

Before we deploy the new application we should update the settings for the ConfigMap to include a couple of things.

Go to the OpenShift Web Console and log in.

* OpenShift Web Console: `{{OPENSHIFT_MASTER_URL}}`{: style="color: blue"}
* Username: `{{OPENSHIFT_USERNAME}}`
* Password: `{{OPENSHIFT_PASSWORD}}`

Click on **Catalog Dev** project

TODO: Add screen shot of Developer project

Click on **Resources > Config Maps**

TODO: Add screen short of How to navigate to Config Maps

Click on **catalog**

TODO: Add screen shot of config map list

Click on **Action > Edit**

TODO: Add screen shot of edit action

Add the following lines to properties



### Summary
Congratulations, you have now successfully integrated our Catalog application with the inventory application using the Spring Cloud `Feign` Client. We are now ready to deploy the application to the Production environment.
