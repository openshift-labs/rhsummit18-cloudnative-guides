## Distributed Tracing

Before we start this scenario you will have to issue the following command 

~~~shell
oc env -n prod{{PROJECT_SUFFIX}} dc/inventory SERVICE_DELAY=400 --overwrite 
oc rollout status dc/inventory -n prod{{PROJECT_SUFFIX}}
~~~

|**NOTE:** This command will introduce a delay in our inventory service so that each call will take > 400ms

Our first release are now in production, but shortly after the release we are getting alarms that calls to `/services/products` is takeing over 2 secs to respond to. Our first task will be to investigate why that is. First let's see if we can see the issue.

Navigate to the [Web UI](http://web-ui-prod.{{APPS_HOSTNAME_SUFFIX}}){:target="_blank"}. The application takes quite a bit of time to respond with a product list. Let's verify that this is because of the catalog service by timing a couple of calls to the catalog service like this:

~~~shell
curl -w "status=%{http_code} size=%{size_download} time=%{time_total}\n" -so /dev/null http://catalog-prod{{PROJECT_SUFFIX}}.{{APPS_HOSTNAME_SUFFIX}}/services/products
~~~

The above command should print something like this:

~~~shell
status=200 size=2147 time=3.238312
~~~

Where, the time attribute says how long in seconds it took to do the call.

Since our product environment uses a distributed tracing tool called [Jaeger](https://www.jaegertracing.io){:target="_blank"}. We can investigate further why our catalog service is taking so long to respond. Open the [Jaeger Query Console](http://jaeger-query-istio-system.{{APPS_HOSTNAME_SUFFIX}}){:target="_blank"} and specify the following query:

|Property|Value|
|--------|--------|
|Service |catalog |
|Operation|default-route|
|Min Duration|2s|

Click on **Find Traces** and you should see some result for traces to service catalog where the call took longer than 2s.

![Jaeger query]({% image_path jaeger-query.png %}){:width="900px"}

Click on one of the results and you should see a trace like this:

![Jaeger trace]({% image_path jaeger-trace.png %}){:width="900px"}

The trace shows that our inventory service is pretty slow and take about 400ms to respond. This in turn is causing our services to response slow since it calls the inventory multiple times in sequence.

### Finding a Solution for Slow Inventory

After discussing the issue with the inventory team, they confirmed that calls to `/service/inventory/{itemid}` is slow since it's not cached. They suggest that we instead call `/service/inventory/all` which returns a cached list of all products inventory status.

Let's investigate that API call.

~~~shell
curl -w "\n" -s http://inventory-prod{{PROJECT_SUFFIX}}.{{APPS_HOSTNAME_SUFFIX}}/services/inventory/all
~~~

It should return something like this:

~~~
[{"itemId":"165613","quantity":303},{"itemId":"165614","quantity":54},{"itemId":"165954","quantity":407},{"itemId":"329199","quantity":123},{"itemId":"329299","quantity":78},{"itemId":"444434","quantity":343},{"itemId":"444435","quantity":85},{"itemId":"444436","quantity":245}]
~~~

So, if we could rework the catalog application to retrieve the list of items in one single calls we could cut the response time from 3s to approx 400ms.

### Updating the Inventory Client

First, open the `com.redhat.coolstore.client.InventoryClient` and add a method declaration like this:

~~~java
    @RequestMapping(method = RequestMethod.GET, value = "/services/inventory/all", consumes = {MediaType.APPLICATION_JSON_VALUE})
    List<Inventory> getInventoryStatusForAll();
~~~

Then, open the `com.redhat.coolstore.service.ProductEndpoint` and replace the `readAll()` method with the following implementation:

~~~java
    @ResponseBody
    @GetMapping("/products")
    public ResponseEntity<List<Product>> readAll() {
        Spliterator<Product> iterator = productRepository.findAll().spliterator();
        List<Product> products = StreamSupport.stream(iterator, false).collect(Collectors.toList());

        //Get all the inventory and convert it to a Map.
        Map<String, Integer> inventoryMap = inventoryClient.getInventoryStatusForAll()
                .stream()
                .collect(Collectors.toMap(
                  (Inventory i) -> i.getItemId(), (Inventory i) -> i.getQuantity())
                );

        products.stream().forEach(p -> p.setQuantity(inventoryMap.get(p.getItemId())));
        return new ResponseEntity<List<Product>>(products,HttpStatus.OK);
    }
~~~

|**NOTE:** We are converting the `List` returned from the `InventoryClient` to a `Map` since that will make it much easier to update the productList with the correct quantity.

### Updating the Unit Test

We also need to update the test case to support calls to `/services/inventory/all`.

Open `ProductEndpointTest` and start by adding a `String` with the return value we got from the curl request before like this:

~~~java
    private static final String ALL_INVENTORY="[{\"itemId\":\"165613\",\"quantity\":303},{\"itemId\":\"165614\",\"quantity\":54},{\"itemId\":\"165954\",\"quantity\":407},{\"itemId\":\"329199\",\"quantity\":123},{\"itemId\":\"329299\",\"quantity\":78},{\"itemId\":\"444434\",\"quantity\":343},{\"itemId\":\"444435\",\"quantity\":85},{\"itemId\":\"444436\",\"quantity\":245}]";
~~~

Then add the following to the HoverFly ClassRule declaration:

~~~java
    .get("/services/inventory/all")
      .willReturn(success(ALL_INVENTORY, "application/json"))
~~~

We are now ready to run the tests and verify that our application passes the unit test, but clicking on the command palette and choose **test**.

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

If it for some reason doesn't work, please go back an check the changes that you have done. Otherwise, go a head and commit and push your changes to the git repository.

![Git Commit]({% image_path tracing-inventory-commit.png %}){:width="600px"}


### Deploy the Application in Production

First, make sure that the **catalog-build** pipeline is executed successfully and 
then verify that our changes works in the **Catalog DEV** project. 

After that go to **Builds** > **Pipelines** form the left-side menu and start the **catalog-release** pipeline 
to promote the build to production environment.

![Build and Release Pipeline]({% image_path tracing-pipelines.png %}){:width="800px"}


### Verify the Changes in Production

When the **catalog-release** pipeline is done, execute the following command in the Eclipse Che **Terminal** a couple of times.

~~~shell
curl -w "status=%{http_code} size=%{size_download} time=%{time_total}\n" -so /dev/null http://catalog-prod{{PROJECT_SUFFIX}}.{{APPS_HOSTNAME_SUFFIX}}/services/products
~~~

Check that the response time is now ~400-500ms.

|**NOTE:** The first call after the deployment may take a bit longer

You can also verify that by opening the 
[Jaeger Query Console](http://jaeger-query-istio-system.{{APPS_HOSTNAME_SUFFIX}}){:target="_blank"} and specify the following query:

|Property|Value|
|--------|--------|
|Service |catalog |
|Operation|default-route|

Click on **Find Traces** and you should see some result for traces to service catalog where the calls take ~400-500ms

![Jaeger query]({% image_path tracing-response-improved.png %}){:width="900px"}

Also verify that the web application is now behaving better by [opening it in the browser](http://web-ui-prod.{{APPS_HOSTNAME_SUFFIX}}){:target="_blank"}.

Before we move on please remove the service delay we added in the begining, by running the following command:

~~~shell
oc env -n prod{{PROJECT_SUFFIX}} dc/inventory SERVICE_DELAY=0 --overwrite=true
oc rollout status dc/inventory -n prod{{PROJECT_SUFFIX}}
~~~

## Summary

[Jaeger](https://www.jaegertracing.io){:target="_blank"}, which is part of the developer preview of Istio for OpenShift is a great tool to see how calls are propagated between services. It does that by correlating trace id's that are passed as headers. Spring Boot and most of the other runtimes in [Red Hat OpenShift Application Runtimes](https://www.redhat.com/en/technologies/cloud-computing/openshift/application-runtimes){:target="_blank"} includes client libraries for OpenTracing, and together with Istio side-car proxies they make it possible to trace calls without changing the code of your application. After we have identified the problem updating the application was really easy, and because of our pipelines we could within minutes push the fix all the way to production.




