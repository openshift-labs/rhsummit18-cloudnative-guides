## Fault Tolerance and Service Resilience

In previous exercises you used Istio to intelligently route traffic
to and between microservices in the service mesh. In this exercise you
will use the fault tolerance and resiliency features of Istio to gracefully
handle failing services due to overwhelming demand.

### Cause Trouble

In the `prod` project, there are two separate versions of the `inventory` service labelled
`version:v1` and `version:v2`. The `v2` version has a built-in artifical delay of 2
seconds, that we'll use later on.

![V1V2]({% image_path inventory-v1-v2-overview.png %}){:width="900px"}

For now, let's round-robin route calls to either `v1` or `v2` in a 50/50 split:

~~~sh
cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: inventory-v1-v2
spec:
  destination:
    name: inventory
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 50
  - labels:
      version: v2
    weight: 50
EOF
~~~

At this point, half of all access to the inventory service will go to `v1` and half
to `v2`. Let's access the `product` service directly (which in turn calls the `inventory` service
multiple times in parallel) and report the results:

~~~sh
curl http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products | jq
~~~

You should see output similar to this:

~~~json
[
  {
    "itemId": "329299",
    "name": "Red Fedora",
    "desc": "Official Red Hat Fedora",
    "price": 34.99,
    "quantity": 78
  },
  {
    "itemId": "329199",
    "name": "Forge Laptop Sticker",
    "desc": "JBoss Community Forge Project Sticker",
    "price": 8.5,
    "quantity": 123
  },
  {
    "itemId": "165613",
    "name": "Solid Performance Polo",
    "desc": "Moisture-wicking, antimicrobial 100% polyester design wicks for life of garment. No-curl, rib-knit collar; special collar band maintains crisp fold; three-button placket with dyed-to-match buttons; hemmed sleeves; even bottom with side vents; Import. Embroidery. Red Pepper.",
    "price": 17.8,
    "quantity": 303
  },
~~~

All of the requests to the catalog were successful, but it took some time to run the test,
as the `v2` inventory was a slow performer due to the artificial 2 second delay.

### Add Circuit Breaker

Suppose that in a production system this 2s delay was caused by too many concurrent
requests to the same instance/pod. We don’t want multiple requests getting queued or
making the instance/pod even slower. So we’ll add an [istio circuit breaker](https://istio.io/docs/reference/config/istio.routing.v1alpha1.html#CircuitBreaker.SimpleCircuitBreakerPolicy)
that will _open_ whenever we have more than 1 request being handled by any instance/pod.

~~~sh
cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: DestinationPolicy
metadata:
  name: inventory-circuitbreaker
spec:
  destination:
    name: inventory
    labels:
      version: v2
  circuitBreaker:
    simpleCb:
      maxConnections: 1
      httpMaxPendingRequests: 1
      sleepWindow: 2m
      httpDetectionInterval: 1s
      httpMaxEjectionPercent: 100
      httpConsecutiveErrors: 1
      httpMaxRequestsPerConnection: 1
EOF
~~~

### Load test with circuit breaker

Now let’s see what is the behavior is now:

~~~sh
curl http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products | jq
  {
    "itemId": "444434",
    "name": "Pebble Smart Watch",
    "desc": "Smart glasses and smart watches are perhaps two of the most exciting developments in recent years.",
    "price": 24,
    "quantity": 343
  },
  {
    "itemId": "444435",
    "name": "Oculus Rift",
    "desc": "The world of gaming has also undergone some very unique and compelling tech advances in recent years. Virtual reality, the concept of complete immersion into a digital universe through a special headset, has been the white whale of gaming and digital technology ever since Geekstakes Oculus Rift GiveawayNintendo marketed its Virtual Boy gaming system in 1995.Lytro",
    "price": 106,
    "quantity": -1
  },
  {
    "itemId": "444436",
    "name": "Lytro Camera",
    "desc": "Consumers who want to up their photography game are looking at newfangled cameras like the Lytro Field camera, designed to take photos with infinite focus, so you can decide later exactly where you want the focus of each image to be.",
    "price": 44.3,
    "quantity": 245
  }
~~~

You may need to run the command multiple times to trigger the circuit breaker, but
you should see some errors (`inventory = -1`) being displayed in the results. That’s the circuit breaker being
opened whenever Istio detects more than 1 pending request being handled by the
instance/pod. The difference here is that once the circuit is opened, calls to the
service will _fail fast_, so you'll still get errors but much quicker.

Remove the circuit breaker before continuing:

~~~sh
oc delete destinationpolicy inventory-circuitbreaker
~~~

### Pool ejection

Pool ejection or outlier detection is a resilience strategy that takes place whenever
we have a pool of instances/pods to serve a client request. If the request is forwarded
to a certain instance and it fails (e.g. returns a 50x error code), then Istio will eject
this instance from the pool for a certain sleep window. In our example the sleep window
is configured to be 15s. This increases the overall availability by making sure that
only healthy pods participate in the pool of instances.

#### Scale `v2` instances

To test pool ejection, let's add some pods to the `v2` pool:

~~~sh
oc scale dc/inventory-v2 --replicas=2
oc get pods -w
~~~

Wait for all pods to be in the `2/2 Ready` state:

~~~console
oc get pods -l app=inventory

NAME                   READY     STATUS    RESTARTS   AGE
inventory-v1-1-x95c2   2/2       Running   0          9h
inventory-v2-2-75pxp   2/2       Running   0          3m
inventory-v2-2-bfqmc   2/2       Running   0          38s
~~~

We still have 50/50 load balancing between the two different versions of the
inventory service. And within version `v2`, some requests
are handled by one pod and some requests are handled by the other pod.

### Test behavior with failing instance and without pool ejection

Let’s get the name of the pods from inventory v2:

~~~sh
oc get pods -l app=inventory,version=v2
~~~
You should see something like this:

~~~console
NAME                   READY     STATUS    RESTARTS   AGE
inventory-v2-2-75pxp   2/2       Running   0          4m
inventory-v2-2-bfqmc   2/2       Running   0          1m
~~~

Now we’ll get into one the pods and add some erratic behavior on it.
Get one of the pod names from above and replace on the following
command accordingly:

~~~sh
oc rsh -c inventory inventory-v2-2-bfqmc
~~~

You will be inside the application container of your pod (mine is named `inventory-v2-2-bfqmc` but yours will be different).
Now execute:

~~~sh
curl localhost:8080/misbehave
exit
~~~

This is a special endpoint that will make our application return only `503`s.

Throw some requests at the inventory endpoint:

~~~sh
while true ; do
curl http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products
sleep .1
done
~~~

You’ll see that whenever the misbehaving pod receives a request, you get a 503 error
and associated `-1` for the inventory value:

~~~json
  ...
  {
    "itemId": "165954",
    "name": "16 oz. Vortex Tumbler",
    "desc": "Double-wall insulated, BPA-free, acrylic cup. Push-on lid with thumb-slide closure; for hot and cold beverages. Holds 16 oz. Hand wash only. Imprint. Clear.",
    "price": 6,
    "quantity": -1
  },
  ...
~~~

### Test behavior with failing instance and with pool ejection

Now let’s add the pool ejection behavior:

~~~sh
cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: DestinationPolicy
metadata:
  name: inventory-poolejector-v2
spec:
  destination:
    name: inventory
    labels:
      version: v2
  loadBalancing:
    name: RANDOM
  circuitBreaker:
    simpleCb:
      httpConsecutiveErrors: 1
      sleepWindow: 15s
      httpDetectionInterval: 5s
      httpMaxEjectionPercent: 100
EOF
~~~

Throw some requests at the customer endpoint, this time only outputting the quantity values:

~~~sh
while true; do
  curl -s http://catalog-prod.apps.127.0.0.1.nip.io/services/products | jq '.[].quantity'
  sleep .1
  echo "---------"
  echo
done
~~~

~~~console
---------

78
123
303
54
407
343
85
245
---------

78
123
-1
54
407
343
85
245
---------
~~~


You will see that whenever you get a failing request with 503 from the failing pod
it gets ejected from the pool, and it doesn’t receive any more requests until the
sleep window expires - which takes at least 15s. Watch closely, every 15s you'll see a
`-1` quantity indicating that the sleep window expired and the failing pod was placed
back in service, and then immediately fail and be re-ejected!

### Ultimate resilience with retries, circuit breaker, and pool ejection

Even with pool ejection your application doesn’t look that resilient. That’s
because we’re still letting some errors (`quantity = -1`) to be propagated to our clients. But we can
improve this. If we have enough instances and/or versions of a specific service
running into our system, we can combine multiple Istio capabilities to achieve
the ultimate backend resilience: - Circuit Breaker to avoid multiple concurrent
requests to an instance; - Pool Ejection to remove failing instances from the pool
of responding instances; - Retries to forward the request to another instance just
in case we get an open circuit breaker and/or pool ejection;

By simply adding a retry configuration to our current routerule, we’ll be able to get
rid completely of our `503`s requests and `inventory = -1` failures. This means that whenever we receive a failed
request from an ejected instance, Istio will forward the request to another supposedly
healthy instance.

Put the retry rule into effect:

~~~sh
cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: inventory-v1-v2
spec:
  destination:
    name: inventory
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 50
  - labels:
      version: v2
    weight: 50
  httpReqRetries:
    simpleRetry:
      perTryTimeout: 1s
      attempts: 3
EOF
~~~

And repeat the quantity test:

~~~sh
while true; do
  curl -s http://catalog-prod.apps.127.0.0.1.nip.io/services/products | jq '.[].quantity'
  sleep .1
  echo "---------"
  echo
done
~~~

You won’t receive any -1 inventories anymore. But the requests from `inventory:v2` are still taking more time to get a response:

When our misbehaving pod returns a failure, Istio kicks it out of the load balancing pool and retries,
which the working pods are able to handle thanks to pool ejection and retry!

### Clean up

~~~sh
oc delete dc/recommendation-v2
oc delete routerule --all
oc delete destinationpolicy --all
~~~

## Congratulations!

In this section you made your microservices more resilient and able to gracefully
handle high workloads without a loss in quality. This is important in any cloud native
application architecture due to the variable nature of cloud environments and
unpredictable nature of client and end users.