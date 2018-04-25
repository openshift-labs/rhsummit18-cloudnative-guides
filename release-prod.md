#### Push a Fix to Prod

## Investigate reported issue

1. Go to the catalog page and see the timeout
2. Look at a Jeager tracing to see the calls happen in serial mode

## Use parallel calls to the inventory

1. Change the ProductEndpoint to do the calls to inventory in parallelStream
2. Update the version tag in the catalog pom.xml to 2.0.0-SNAPSHOT
3. Deploy the code to dev.
4. Test the dev with a delay of 1 second in dev.
5. Trigger the release pipeline - The release pipeline should now create a release version of the JAR called 2.0.0 and create a new deployment config that uses the release tag as a version label pointing to the new image for the catalog pod.

## Role out the release

1. Deploy a istio routing rule that looks at the query string parameter named coolstore-version and if that matches "v2", then route traffic to the v2 version of catalog.
2. Call the catalog endpoint using query string "?coolstore-version=v2".
3. Check the Jaeger tracing for calls to v2
4. Update the routing rule to use 50% v2 and 50% v1
5. Test that every other call works
6. Update the routing rule to use 100% v2.

# TODO for us:

* Extend the index.html to pass coolstore-version query string on to calls to /services/products
* Inventory delay in production should before this lab be 1000 ms.
* Check that the pipeline will deploy to v2 version
* Inventory delay in dev should before this lab be 0 ms






