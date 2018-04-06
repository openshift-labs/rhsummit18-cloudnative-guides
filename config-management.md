#### Configuration management with Spring Boot

In this scenario you will learn how to manage application configuration and how to
provide application configuration external from the application itself. This is useful for
separating environment-specific configuration and for supplying runtime changes to the business
logic without redeploying. In this situation, the product catalog has some faulty products in it
that must be removed quickly and temporarily to handle sitations where business can be lost
if catalogs are immediately patched to remove the products.

Add necessary dependencies to `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes</artifactId>
</dependency>
```

Configure Spring Cloud Kubernetes to auto-update configuration when ConfigMap changes. In `src/main/resources/application-default.properties`:

```
# enable configmap auto-reload
spring.cloud.kubernetes.reload.enabled=true
```

Add com.redhat.coolstore.service.StoreConfig:

```java
package com.redhat.coolstore.service;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@ConfigurationProperties (prefix = "store")
public class StoreConfig {

    private List<String> recalledProducts;

    public List<String> getRecalledProducts() {
        return recalledProducts;
    }

    public void setRecalledProducts(List<String> recalledProducts) {
        this.recalledProducts = recalledProducts;
    }

    public boolean isRecalled(String itemId) {
        return recalledProducts.contains(itemId);
    }
}
```

Modify `CatalogService` to filter products on the `recalledProducts` list:

Add field:

```java
    @Autowired
    StoreConfig storeConfig;
```

Modify `readAll()` to filter out recalled products (add the one line):

```java
    public List<Product> readAll() {
        List<Product> productList = repository.readAll();
        // remove recalled products
        productList.removeIf(p -> storeConfig.isRecalled(p.getItemId()));
        productList.parallelStream()
                .forEach(p -> {
                    p.setQuantity(inventoryClient.getInventoryStatus(p.getItemId()).getQuantity());
                });
        return productList;
    }
```

Add unit test to `src/test/java` in class `com.redhat.coolstore.service.RecalledProductsTest`:

```java
package com.redhat.coolstore.service;

import com.redhat.coolstore.model.Inventory;
import com.redhat.coolstore.model.Product;
import io.specto.hoverfly.junit.rule.HoverflyRule;
import org.junit.ClassRule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.List;
import java.util.stream.Collectors;

import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;
import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static org.assertj.core.api.Assertions.assertThat;

@RunWith(SpringRunner.class)
@SpringBootTest()
@TestPropertySource(locations="classpath:test-recalled-products.properties")
public class RecalledProductsTest {

    @Autowired
    CatalogService catalogService;

    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
            service("inventory:8080")
                    .get("/services/inventory/165613")
                        .willReturn(success(json(new Inventory("165613",13))))
                    .get("/services/inventory/165614")
                        .willReturn(success(json(new Inventory("165614",85))))
                    .get("/services/inventory/165954")
                        .willReturn(success(json(new Inventory("165954",78))))
                    .get("/services/inventory/329199")
                        .willReturn(success(json(new Inventory("329199",67))))
                    .get("/services/inventory/329299")
                        .willReturn(success(json(new Inventory("329299",98))))
                    .get("/services/inventory/444434")
                        .willReturn(success(json(new Inventory("444434",73))))
                    .get("/services/inventory/444435")
                        .willReturn(success(json(new Inventory("444435",64))))
                    .get("/services/inventory/444436")
                        .willReturn(success(json(new Inventory("444436",30))))

    ));

    @Test
    public void verify_recalled() throws Exception {
        List<Product> productList = catalogService.readAll();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> itemIds = productList.stream().map(Product::getItemId).collect(Collectors.toList());
        assertThat(itemIds).doesNotContain("329299", "329199");
    }
}
```

Add `src/test/resources/test-recalled-products.properties` to support the unit test:

```
# Comma-separated recalled product itemIds
store.recalledProducts = 329199,329299
```

run `mvn verify`, wait for:

```console
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 24.662 s
[INFO] Finished at: 2018-04-06T16:17:33-04:00
[INFO] Final Memory: 36M/327M
[INFO] ------------------------------------------------------------------------
```

Run locally:

`mvn spring-boot:run -Dstore.recalledProducts=329199`

Test to ensure you don't have any `329199` product:

`open http://localhost:8081`

Commit the pipeline to code repo:

* In the project explorer, right click on catalog and then Git > Commit
* Make sure newly created files, and `pom.xml` is checked
* Add a commit message "externalized recalledProducts configuration"
* Check the "Push commit changes to..."
* Click on "Commit"

This will trigger the pipeline build. Wait for it to complete.

Test the catalog running on OpenShift:

`open http://catalog-dev.[OPENSHIFT_MASTER]`

Observe all products are present.

Add a recalledProduct to the ConfigMap:

```
store.recalledProducts = 329299
```

Thanks to auto-refresh, this will immediately take effect. Observe on the catalog UI that the product is now missing
(the UI auto-refreshes every 2 seconds so it should disappear within 2 seconds):

### Congratulations!

You've now got a quick way to alter service configuration without redeploying! Congratulations!
