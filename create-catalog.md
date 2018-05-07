## Create a Catalog Service

As a developer you have been tasked with creating a catalog service that exposes a REST API to 
read from the product catalog database.

The catalog service is part of a microservices-based application called **CoolStore**.

![CoolStore Microservices Architecture]({% image_path coolstore-arch.png %}){:width="400px"}

<br/>

![CoolStore Microservices]({% image_path create-catalog-coolstore.png %}){:width="900px"}

After talking to the **web** team you have decided to create two endpoints: One that returns a list of all products as a JSON
string and another one that takes a product ID returns a single product.

After the initial sprint planning the team has decided to use Spring Boot, Spring Data and PostgreSQL to implement this.
However, since the team wants to be able to test and run this locally without having to install a database, you will
use H2 (an in-memory database) for local development.

### Create the Domain Object Model

The first step is to create a Java class that represents the product data domain model with the following
fields:

* **`itemId`** - a string representing a unique ID for the product
* **`name`** - a string between 1-255 characters that contains the name of the product
* **`description`** - a string between 0-2500 characters that contains the detailed description of the product.
* **`price`** - a _double_ value that represent the list price of a product.

The database table would look like this:

| ITEMID |    NAME     |                                       DESCRIPTION                                       | PRICE |
|--------|-------------|-----------------------------------------------------------------------------------------|-------|
| 444435 | Oculus Rift | The world of gaming has also undergone some very unique and compelling tech advances... | 106.0 |


Create a Java class named `Product` in `src/main/java` and the `com.redhat.coolstore.model` package with the following code:

~~~java
package com.redhat.coolstore.model;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class Product {
    @Id
    private String itemId;
    private String name;
    @Column(length = 2500)
    private String description;
    private double price;
}
~~~

The `@Entity` annotation marks the class as a data model for Spring Data. You can generate the getter and
setter methods using Eclipse Che. Click on a spot where you want the 
methods to be generated, write `get` or `set` into the editor and then press Ctrl+Space to get the list of proposals for
the code to be generated. Click on `getName()` for example to generate the `getName()` method. Repeat for 
all the getters and setters for each of the above fields.

|**STEP BY STEP:** Creating the Domain Object Model
|![Create a model]({% image_path che-create-model.gif %}){:width="900px"}

### Adding Seed Data

To populate our database we will use the auto import feature of Spring that will pre-populate
our database with content. To do this, create a file called `import.sql` under `src/main/resources` with the following content,
which is automatically imported when the application starts:

~~~sql
insert into product (item_id, name, description, price) values ('329299', 'Red Fedora', 'Official Red Hat Fedora', 34.99);
insert into product (item_id, name, description, price) values ('329199', 'Forge Laptop Sticker', 'JBoss Community Forge Project Sticker', 8.50);
insert into product (item_id, name, description, price) values ('165613', 'Solid Performance Polo', 'Moisture-wicking, antimicrobial 100% polyester design wicks for life of garment. No-curl, rib-knit collar; special collar band maintains crisp fold; three-button placket with dyed-to-match buttons; hemmed sleeves; even bottom with side vents; Import. Embroidery. Red Pepper.',17.80);
insert into product (item_id, name, description, price) values ('165614', 'Ogio Caliber Polo', 'Moisture-wicking 100% polyester. Rib-knit collar and cuffs; Ogio jacquard tape inside neck; bar-tacked three-button placket with Ogio dyed-to-match buttons; side vents; tagless; Ogio badge on left sleeve. Import. Embroidery. Black.', 28.75);
insert into product (item_id, name, description, price) values ('165954', '16 oz. Vortex Tumbler', 'Double-wall insulated, BPA-free, acrylic cup. Push-on lid with thumb-slide closure; for hot and cold beverages. Holds 16 oz. Hand wash only. Imprint. Clear.', 6.00);
insert into product (item_id, name, description, price) values ('444434', 'Pebble Smart Watch', 'Smart glasses and smart watches are perhaps two of the most exciting developments in recent years.', 24.00);
insert into product (item_id, name, description, price) values ('444435', 'Oculus Rift', 'The world of gaming has also undergone some very unique and compelling tech advances in recent years. Virtual reality, the concept of complete immersion into a digital universe through a special headset, has been the white whale of gaming and digital technology ever since Geekstakes Oculus Rift GiveawayNintendo marketed its Virtual Boy gaming system in 1995.Lytro',106.00 );
insert into product (item_id, name, description, price) values ('444436', 'Lytro Camera', 'Consumers who want to up their photography game are looking at newfangled cameras like the Lytro Field camera, designed to take photos with infinite focus, so you can decide later exactly where you want the focus of each image to be.', 44.30);
~~~

### Create a Repository to Access Data

You can use the Spring Data repository abstraction which is a convenient way to access data
models in Spring applications without writing much code.

Create a repository interface called `ProductRepository` in 
`src/main/java/com/redhat/coolstore/service` with the following code:

~~~java
package com.redhat.coolstore.service;

import com.redhat.coolstore.model.Product;
import org.springframework.data.repository.CrudRepository;

public interface ProductRepository extends CrudRepository<Product, String> {
}
~~~

And that's it! You might be wondering why this interface does not have an implementation. `CrudRepository` is a 
marker interface from Spring Data which tells Spring Data to generate the common finder methods (e.g. `findAll`) for your 
repository and domain mode and make it available at runtime.

You have now created a domain model and a repository for our Catalog Service.

### Test the Product Repository

After creating domain model and a repository for our Catalog Service you realize that it would probably be smart to also
 write some test code that can verify that the implementation works as expected.


Create a unit test called `ProductRepositoryTest` under `src/test/java` folder and in the `com.redhat.coolstore.service` 
package with the following code:

~~~java
package com.redhat.coolstore.service;

public class ProductRepositoryTest {

}
~~~

Since this is a unit test that needs to run as a Spring test, add the following annotations to the 
test class:

~~~java
@RunWith(SpringRunner.class)
@SpringBootTest
~~~

You will get an error notice near the annotation which tells you the annotation should be imported in order to be used in 
your Java class. Fortunately Eclipse Che can fix that for you automatically (you *did* mark the project as a Maven project when you
first imported it into Eclipse Che, right?)

Right-click on the annotation (or the error) and
then choose **Quick Fix**. Eclipse Che will show you a list of quick fixes, click on **Import 'RunWith'...** to import it to your 
Java class. 

The above annotations set up the correct Spring context and make required objects available for injection 
and use in the unit tests.

Inject an instance of `ProductRepository` which is the class under test:

~~~java
    @Autowired
    ProductRepository repository;
~~~

Now you are ready to write the first test. First you can test the `findOne` method by collecting one of 
the products that you defined in `import.sql` and verifying that the data is correct:

~~~java
    @Test
    public void test_findOne() {
        Product product = repository.findOne("444434");
        assertThat(product).isNotNull();
        assertThat(product.getName()).as("Verify product name").isEqualTo("Pebble Smart Watch");
        assertThat(product.getPrice()).as("Price should match ").isEqualTo(24.0);
    }
~~~

Add another test for the `findAll` method:

~~~java
    @Test
    public void test_findAll() {
        Iterable<Product> products = repository.findAll();
        assertThat(products).isNotNull();
        assertThat(products).isNotEmpty();
        List<String> names = StreamSupport.stream(products.spliterator(), false).map(p -> p.getName()).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift","Solid Performance Polo","16 oz. Vortex Tumbler","Pebble Smart Watch","Lytro Camera");
    }
~~~

At this point you should have a lot of errors because you might not have imported any classes.  Click on 
**Assistant > Organize Imports** in the top bar menu. Review the list of suggested imports (be sure that
Eclipse Che imports `java.util.List` and not a different `List` implementation, and click **Finish** if prompted.) _Organize imports_ will not import static methods
so add the static import for `assertThat` method to the top of the test class near the other imports:

~~~java
import static org.assertj.core.api.Assertions.assertThat;
~~~

Build the project to make sure everything is in order by click on **Run** > **Commands Palette** > **build**.

Now, go ahead and run the unit test by right-clicking on the the `ProductRepositoryTest` class file 
in the project explorer and then select **Run Test > Run JUnit Test**. 

Verify that the tests are passing green.

|**STEP BY STEP:** Create a Unit test for the Repository
|![Create a repository]({% image_path create-service-step-by-step-run-test.gif %}){:width="640px"}

### Creating a Product Endpoint

Now that you've verified you can successfully fetch products from the database, you are ready to create a REST endpoint
that returns the products data as a JSON response.

Create a class called `ProductEndpoint` in `src/main/java/com/redhat/coolstore/service` with 
the following code:

~~~java
package com.redhat.coolstore.service;

public class ProductEndpoint {

}
~~~

Annotate the class with `@Controller` to tell Spring that this is a controller bean. Also annotate it 
with `@RequestMapping` specifying that this controller will be handling request to `/service/*` like this:

~~~java
@Controller
@RequestMapping("/services")
~~~

To access the `ProductRepository` you also need to auto inject it as a field like this:

~~~java
    @Autowired
    private ProductRepository productRepository;
~~~

Then you need to expose the endpoint that list all products via the `/services/products` URI for GET calls like this:

~~~java
    @ResponseBody
    @GetMapping("/products")
    public ResponseEntity<List<Product>> readAll() {
        Spliterator<Product> iterator = productRepository.findAll().spliterator();
        List<Product> products = StreamSupport.stream(iterator, false).collect(Collectors.toList());
        return new ResponseEntity<>(products, HttpStatus.OK);
    }
~~~

And finally create the endpoint to find a particular product, by providing the item it as part of the path like this:

~~~java
    @ResponseBody
    @GetMapping("/product/{id}")
    public ResponseEntity<Product> readOne(@PathVariable("id") String id) {
        Product product = productRepository.findOne(id);
        return new ResponseEntity<Product>(product,HttpStatus.OK);
    }
~~~

Don't forget to **Assistant > Organize Imports** to import the necessary classes after adding the above code!

|**STEP BY STEP:** Create a REST endpoint for Catalog service 
|![Step by step - create product endpoint]({% image_path create-catalog-step-by-step-product-endpoint.gif %}){:width="900px"}

### Testing the Endpoint

We are now ready to test the endpoint and for that we could implement a unit test, 
but we will save that for a bit later. Instead we are going to run the application, 
using the **Commands Palette** in Eclipse Che. Click on the **Commands Palette** and then on **RUN > run**

When the **run** output panel reports `Started CatalogServiceApplication` click 
on the link next to **Preview:** at the top of the output panel to open a preview in your browser to the application.

Don't worry if you see a **Application Not Available** page, it make take a few seconds to fully spin up. Just reload the
page using using F5 (or CTRL+R or CMD+R if on mac) until you see the catalog.

![Application Not Available]({% image_path create-catalog-application-not-available.png %}){:width="600px"}

You should now see a page that list the content of the product catalog like this:

![Application Preview]({% image_path create-catalog-success.png %}){:width="900px"}

|**NOTE:** This command will run `mvn spring-boot:run`, but is similar to 
running `java -jar target/<app>.jar` directly from a terminal window.

|**NOTE:** The Preview URL is unique and will be different from the screenshot above!

|**STEP BY STEP:** Run the application locally 
|![Step by step - run locally]({% image_path create-catalog-step-by-step-run.gif %}){:width="640px"}

|**INFO:** Even if you application is actually running in Eclipse Che on OpenShift 
it is not running as a proper OpenShift-managed application yet. This development
flow allows you to develop "locally" on your developer machine and test
locally before deploying to "real" server. This is a great way for you as a developer 
to test your own code changes before committing and pushing it to a shared repository.

|**NOTE:** The current version of the application does not provide an inventory _quantity_ number. We will look at that in the next section.

Stop the local test run of Catalog by clicking on the stop icon near **run** in Eclipse Che.

### Push the Changes to Git Repository

Commit and push all the changes by selecting **Git > Commit** from the menu.

![Git Commit]({% image_path create-catalog-commit-1.png %}){:width="200px"}

Select all the changed or new files and check the box next to **Push committed changes to:** and then push **Commit**

![Git Commit]({% image_path create-catalog-commit-2.png %}){:width="600px"}

Open the [OpenShift Web Console]({{OPENSHIFT_MASTER_URL}}/console/project/dev{{PROJECT_SUFFIX}}/browse/pipelines){:target="_blank"} and click on **Builds** > **Pipelines**: to watch the pipeline build and deploy the Catalog code.

![Build Pipeline]({% image_path create-catalog-pipeline.png %}){:width="700px"}

When the pipeline is completed, point your browser to the [Catalog url deployed on OpenShift](http://catalog-dev{{PROJECT_SUFFIX}}.{{APPS_HOSTNAME_SUFFIX}}){:target="_blank"} to access it.

Note that you can also find the Catalog url in the project overview in the OpenShift Web Console.

|**STEP BY STEP:** Commit the changes and run the pipeline
|![Step by step - Commit the changes and run the pipeline]({% image_path create-catalog-step-by-step-pipeline.gif %}){:width="640px"}

|**NOTE:** The current version of the application does not make use of the PostgreSQL server which is deployed in the **Catalog DEV** project.
Instead it currently uses the H2 database. You will fix that in the next lab.

### Summary

Congratulations, you now created the first version of your microservice. Pat yourself on the back and reflect a bit on how easy it was to do
this using this integrated development environment and how easy it was to deploy it to OpenShift. In the next module we will look at how
to externalize the database configuration and in the following lab you'll do a service-to-service call to link the `catalog` with the `inventory` to produce
an aggregated result. On to the next lab!
