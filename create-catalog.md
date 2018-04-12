## Create a Catalog Service

As a developer you have been tasked with creating a catalog service that exposes a HTTP API to read from the product catalog database. After talking to the UI interface team you have decided to create two endpoints. One that returns a list of all products as a JSON string and another one that based on product Id returns a single product.

After the initial sprint planing the team has decided to use Spring Boot, Spring Data and PostgreSQL to implement this. However, since the team wants to be able to test and run this locally without having to install a database the you will also use H2 (an in-memory database) for local development.

### Creating the database model

Your first task is to create a database model that will represent the product data. :

* **ItemId** - a string representing a unique id for the product
* **name** - a string between 1-255 characters that contains the name of the product
* **description** - a string between 0-2500 characters that contains the a details description about the product.
* **price** - a double value that represent the list price of a product.

Based on that you decide to create a database table called `CATALOG` that looks like this:

| ITEMID |    NAME     |                                       DESCRIPTION                                       | PRICE |
|--------|-------------|-----------------------------------------------------------------------------------------|-------|
| 444435 | Oculus Rift | The world of gaming has also undergone some very unique and compelling tech advances... | 106.0 |


Since you will be using hibernate you decide to create a file called `schema.sql` in `src/main/resources` with the following content:

~~~sql
DROP TABLE IF EXISTS CATALOG;

CREATE TABLE CATALOG (
  ITEMID VARCHAR(256) NOT NULL PRIMARY KEY,
  NAME VARCHAR(256),
  DESCRIPTION VARCHAR(2560),
  PRICE DOUBLE PRECISION
);
~~~

The UI team has also been friendly enough to provide you with some test data, that you also add to the `schema.sql` file

~~~sql
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('329299', 'Red Fedora', 'Official Red Hat Fedora', 34.99);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('329199', 'Forge Laptop Sticker', 'JBoss Community Forge Project Sticker', 8.50);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('165613', 'Solid Performance Polo', 'Moisture-wicking, antimicrobial 100% polyester design wicks for life of garment. No-curl, rib-knit collar; special collar band maintains crisp fold; three-button placket with dyed-to-match buttons; hemmed sleeves; even bottom with side vents; Import. Embroidery. Red Pepper.',17.80);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('165614', 'Ogio Caliber Polo', 'Moisture-wicking 100% polyester. Rib-knit collar and cuffs; Ogio jacquard tape inside neck; bar-tacked three-button placket with Ogio dyed-to-match buttons; side vents; tagless; Ogio badge on left sleeve. Import. Embroidery. Black.', 28.75);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('165954', '16 oz. Vortex Tumbler', 'Double-wall insulated, BPA-free, acrylic cup. Push-on lid with thumb-slide closure; for hot and cold beverages. Holds 16 oz. Hand wash only. Imprint. Clear.', 6.00);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('444434', 'Pebble Smart Watch', 'Smart glasses and smart watches are perhaps two of the most exciting developments in recent years.', 24.00);
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('444435', 'Oculus Rift', 'The world of gaming has also undergone some very unique and compelling tech advances in recent years. Virtual reality, the concept of complete immersion into a digital universe through a special headset, has been the white whale of gaming and digital technology ever since Geekstakes Oculus Rift GiveawayNintendo marketed its Virtual Boy gaming system in 1995.Lytro',106.00 );
insert into CATALOG (ITEMID, NAME, DESCRIPTION, PRICE) values ('444436', 'Lytro Camera', 'Consumers who want to up their photography game are looking at newfangled cameras like the Lytro Field camera, designed to take photos with infinite focus, so you can decide later exactly where you want the focus of each image to be.', 44.30);
~~~

Now, if you feel up to it please go and create the schema.sql based on the text above, or if you need more input please see the ste-by-step guide [here](step-by-step/lab2-database-model.md) 

  
### Create the Java Object Model

Next step is to create a java model that represents the product data. So you create a class named `Product` and is located in `com.redhat.coolstore.model` like this:

| MODIFIER |    NAME     |  TYPE  |
|----------|-------------|--------|
| public   | itemId      | String |
| public   | name        | String |
| public   | description | String |
| public   | price       | double |

For detailed instructions see [here]().

### Create a service to access data.

In Domain Driven Design (DDD) a *Repository* is definied as *"a mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects"*. So to access data from the database we will implement a `ProductRepository` class using `JdbcTemplate` based on KISS principle. Keep It Simple Stupid ;-)


The full implementation looks like this:

~~~java
package com.redhat.coolstore.service;

import java.util.List;

import com.redhat.coolstore.model.Product;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

@Repository
public class ProductRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;


    private RowMapper<Product> rowMapper = (rs, rowNum) -> new Product(
            rs.getString("ITEMID"),
            rs.getString("NAME"),
            rs.getString("DESCRIPTION"),
            rs.getDouble("PRICE"));


    public List<Product> readAll() {
        return jdbcTemplate.query("SELECT * FROM CATALOG", rowMapper);
    }

    public Product findById(String id) {
        return jdbcTemplate.queryForObject("SELECT * FROM CATALOG WHERE ITEMID = '" + id + "'", rowMapper);
    }

}
~~~

So either create this class or look at the details instructions [here]()

### Test the Product Repository

After creating these classes you should now be eger to test and verify that what you have created works. :-)

For that you will create a unit test called `ProductRepositoryTest` in `src/test/java` catalog that looks like this

~~~java
package com.redhat.coolstore.service;

import java.util.List;
import java.util.stream.Collectors;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import static org.assertj.core.api.Assertions.assertThat;

import com.redhat.coolstore.model.Product;


@RunWith(SpringRunner.class)
@SpringBootTest()
public class ProductRepositoryTest {

    @Autowired
    ProductRepository repository;

    @Test
    public void test_readOne() {
        Product product = repository.findById("444434");
        assertThat(product).isNotNull();
        assertThat(product.name).as("Verify product name").isEqualTo("Pebble Smart Watch");
        assertThat(product.price).as("Price should match ").isEqualTo(24.0);
    }

    @Test
    public void test_readAll() {
        List<Product> productList = repository.readAll();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> names = productList.stream().map(p -> p.name).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift","Solid Performance Polo","16 oz. Vortex Tumbler","Pebble Smart Watch","Lytro Camera");
    }

}
~~~

Now, go ahead and run the unit test. For detailed instructions see [here]().

### Creating a Product Endpoint
Since we now have confirmed that we can successfully get products from a database we are now ready to create our endpoint. 

The semantics our our endpoint looks like this

~~~java
package com.redhat.coolstore.service;

@Controller
@RequestMapping("/services")
public class ProductEndpoint {

}
~~~    

To access the `ProductRepository` we need to auto inject it as a field like this:

~~~java
@Autowired
private ProductRepository productRepository;
~~~

Then we need to expose the endpoint that list all products via the `/services/products` URI for GET calls like this:

~~~java
@ResponseBody
@GetMapping("/products")
public ResponseEntity<List<Product>> readAll() {
    return new ResponseEntity<List<Product>>(productRepository.readAll(),HttpStatus.OK);
}
~~~

And finally we create the endpoint to find a particular product, by providing the `itemId` like this:

~~~java
@ResponseBody
@GetMapping("/product/{id}")
public ResponseEntity<Product> readOne(@PathVariable("id") String id) {
    return new ResponseEntity<Product>(productRepository.findById(id),HttpStatus.OK);
}
~~~

For detailed instructions and full code of the ProductRepository see [here]()

### Testing the endpoint

We are now ready to test the endpoint and for that we could implement a unit test, but we will save that for a bit later. Instead we are going to run the application, using the Run command in the Eclipse Che.

![Git Server - Create Repo]({% image_path create-catalog-run-command.png %}){:width="700px"} 

When the RUN output panel reports "initialization complete" click on the link next to **Preview:** to open a preview browser to the application.

|**NOTE:** This command will run `mvn spring-boot:run`, but is similar to running `java -jar target/<app>.jar` directly from a terminal window.

|**NOTE:** The Preview URL is unique and will be different from the screenshot above

![Git Server - Create Repo]({% image_path create-catalog-preview-link.png %}){:width="700px"}

Don't worry if you see a **Application Not Available** page. Just reload the page using using F5 (or CTRL+R or CMD+R if on mac).

![Git Server - Create Repo]({% image_path create-catalog-application-not-available.png %}){:width="700px"}

You should now see a page that list the content of the product catalog like this:

![Git Server - Create Repo]({% image_path create-catalog-success.png %}){:width="700px"}


|**INFO:** Even if you application is actually running in Eclipse Che on OpenShift we are actually not running a proper OpenShift managed application yet. This development flow is similar as if you where developing locally on you developer machine and testing locally before deploying to "real" server. This is a great way for you as a developer to test your own code changes before committing and pushing it to a shared repository.

|**NOTE:** The current version of the application does not provide a inventory quantity number. We will look at that in the next section.


### Commit and push the first version of the application

Commit all the changes and push all the changes by selecting **Git > Commit** from the menu.

![Git Server - Create Repo]({% image_path create-catalog-commit-1.png %}){:width="200px"}

Select all the changed or new files and check the box next to **Push committed changes to:** and then push **Commit**

![Git Server - Create Repo]({% image_path create-catalog-commit-2.png %}){:width="600px"}

Open the OpenShift [webconsole]({{OPENSHIFT_MASTER_URL}}/console/project/dev/browse/pipelines){:target="_blank"} and watch the pipeline build

![Git Server - Create Repo]({% image_path create-catalog-pipeline.png %}){:width="700px"}

### Test on OpenShift

TODO: Describe how to test on OpenShift

|**NOTE:** The current version of the application does not make use of the PostgreSQL server. Instead it currently uses the H2 database. We will come back to that when we look at module `Externalize Configuration`

### Summary
Congratulations, you have managed to create the first version of our microservices. Pad yourself on the back and reflect a bit on how easy it was to do this using this integrated development environment and how easy it was to deploy it to OpenShift. In the next module we will look at how to do a service-to-service call.
