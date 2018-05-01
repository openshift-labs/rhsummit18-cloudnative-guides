## The challenge with distributed computing

As we transition our applications towards a distributed architecture with microservices deployed across a distributed
network, Many new challenges await us.

Technologies like containers and container orchestration platforms like OpenShift solve the deployment of our distributed
applications quite well, but are still catching up to addressing the service communication necessary to fully take advantage
of distributed applications, such as dealing with:

* Unpredictable failure modes
* Verifying end-to-end application correctness
* Unexpected system degradation
* Continuous topology changes
* The use of elastic/ephemeral/transient resources

Today, developers are responsible for taking into account these challenges, and do things like:

* Circuit breaking and Bulkheading (e.g. with Netfix Hystrix)
* Timeouts/retries
* Service discovery (e.g. with Eureka)
* Client-side load balancing (e.g. with Netfix Ribbon)

Another challenge is each runtime and language addresses these with different libraries and frameworks, and in
some cases there may be no implementation of a particular library for your chosen language or runtime.

## Improving the `inventory` service

In this scenario we'll explore how to use a new project called _Istio_ to solve many of the challenges of modern
distributed applications. In particular, our coolstore application suffers from problems when the load is high - although
you improved the production environment in the last step using Jaeger tracing and improving the performance of the
`catalog` service, the `inventory` service still suffers
from occasional scaling problems when the load is high. Let's use Istio and its fault tolerance and resiliency
features to gracefully handle the high load.

### Introduce bad behavior

In the **Coolstore PROD** project, let's scale up the `inventory` service to 2 pods:

~~~sh
oc scale dc/inventory --replicas=2
oc get pods -l app=inventory
~~~

Wait for both pods to be in the `2/2 Ready` state:

~~~
oc get pods -l app=inventory

NAME                READY     STATUS    RESTARTS   AGE
inventory-1-fw9d9   2/2       Running   0          10m
inventory-1-p9c7h   2/2       Running   0          13s
~~~

Now, let's make the 2nd pod misbehave (return `HTTP 503` errors when accessed). We'll use
a special internal endpoint on the running pod to inject this fault.

|**NOTE**: Replace the name of pod in the below `oc` command with the name of one of your pods!

~~~sh
oc rsh -c inventory inventory-2-75pxp curl http://localhost:8080/misbehave
~~~

At this point, you'll have two `inventory` pods, only one of which works. We should see this when
we access the `inventory` service.

### Observe Behavior

At this point, due to the default load balancer, half of all access to the inventory service will go to the working
pod and half to the broken pod. Let's access the `catalog` service directly (which in turn calls the `inventory` service). Let's
call it 4 times, and output the inventory values from each call. 2 of the accesses should return value inventory values, and 2
of them should fail (thanks to the misbehaving pod):

|**CAUTION:** Replace `GUID` with the guid provided to you.

~~~sh
for i in $(seq 4); do
    echo "Fetching Catalog attempt #${i}"
    curl -s http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products | jq '.[] | [.quantity, .name] | @text'
    echo ---------------------
    echo
done
~~~

You should see output similar to this:

~~~
Fetching Catalog attempt #1
"[78,\"Red Fedora\"]"
"[123,\"Forge Laptop Sticker\"]"
"[303,\"Solid Performance Polo\"]"
"[54,\"Ogio Caliber Polo\"]"
"[407,\"16 oz. Vortex Tumbler\"]"
"[343,\"Pebble Smart Watch\"]"
"[85,\"Oculus Rift\"]"
"[245,\"Lytro Camera\"]"
--------------

Fetching Catalog attempt #2
"[-1,\"Red Fedora\"]"
"[-1,\"Forge Laptop Sticker\"]"
"[-1,\"Solid Performance Polo\"]"
"[-1,\"Ogio Caliber Polo\"]"
"[-1,\"16 oz. Vortex Tumbler\"]"
"[-1,\"Pebble Smart Watch\"]"
"[-1,\"Oculus Rift\"]"
"[-1,\"Lytro Camera\"]"
--------------

Fetching Catalog attempt #3
"[78,\"Red Fedora\"]"
"[123,\"Forge Laptop Sticker\"]"
"[303,\"Solid Performance Polo\"]"
"[54,\"Ogio Caliber Polo\"]"
"[407,\"16 oz. Vortex Tumbler\"]"
"[343,\"Pebble Smart Watch\"]"
"[85,\"Oculus Rift\"]"
"[245,\"Lytro Camera\"]"
--------------

Fetching Catalog attempt #4
"[-1,\"Red Fedora\"]"
"[-1,\"Forge Laptop Sticker\"]"
"[-1,\"Solid Performance Polo\"]"
"[-1,\"Ogio Caliber Polo\"]"
"[-1,\"16 oz. Vortex Tumbler\"]"
"[-1,\"Pebble Smart Watch\"]"
"[-1,\"Oculus Rift\"]"
"[-1,\"Lytro Camera\"]"
--------------
~~~

Every other request to the inventory service failed and returned error, and our code in the `catalog` service replaced the inventory
values with `-1`, as expected, due to the misbehaving pod returning `HTTP 503`.

### Add a Circuit Breaker

Suppose that in a production system these errors were caused by too many concurrent
requests to the same instance/pod. We don’t want multiple requests getting queued or
making the instance/pod even slower. So we’ll add an [istio circuit breaker](https://istio.io/docs/reference/config/istio.routing.v1alpha1.html#CircuitBreaker.SimpleCircuitBreakerPolicy)
that will _open_ whenever we have more than 1 request being handled by any instance/pod. When the
circuit is broken for a given pod, we want to _eject_ the pod out of the load balancing pool.

Pool ejection or outlier detection is a resilience strategy that takes place whenever
we have a pool of instances/pods to serve a client request. If the request is forwarded
to a certain instance and it fails (e.g. returns a 50x error code), then Istio will eject
this instance from the pool for a certain sleep window. In our example the sleep window
is configured to be 15s. This increases the overall availability by making sure that
only healthy pods participate in the pool of instances.

Let's add a circuit breaker with pool ejection action. To create Istio circuit breakers you'll need
admin privileges, so login as the admin user:

* OpenShift Admin User: `{{OPENSHIFT_ADMIN_USERNAME}}`
* OpenShift Admin Password: `{{OPENSHIFT_ADMIN_PASSWORD}}`

~~~sh
oc login -u {{OPENSHIFT_ADMIN_USERNAME }} -p {{OPENSHIFT_ADMIN_PASSWORD}}
oc project prod{{PROJECT_SUFFIX}}
~~~

Istio circuit breakers operate by setting policies on _routes_ to specific backends. For this to work,
we need to first create an Istio _RouteRule_ that sends 100% of the traffic to our `inventory` backend:

~~~sh
cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: inventory-v1-route
spec:
  destination:
    name: inventory
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 100
EOF
~~~

Now, add the circuit breaker by creating an Istio [Destination Policy](https://istio.io/docs/concepts/traffic-management/rules-configuration.html#destination-policies) which
is used to create circuit breakers:

~~~sh
cat <<EOF | oc create -f -
apiVersion: config.istio.io/v1alpha2
kind: DestinationPolicy
metadata:
  name: inventory-poolejector
spec:
  destination:
    name: inventory
    labels:
      version: v1
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

### Test behavior with failing instance

Throw some requests at the catalog endpoint, this time only outputting the quantity values:

|**CAUTION:** Replace `GUID` with the guid provided to you.

~~~sh
while true; do
    echo "Fetching Catalog..."
    curl -s http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products | jq '.[] | [.quantity, .name] | @text'
    echo ---------------------
    echo
    sleep 5
done
~~~

Press `CTRL-C` after a few seconds to stop the loop.

You will notice that there is still 1 failure in the above (inventory=`-1`). This is because
the misbehaving pod is accessed once, fails, and is immediately ejected from the load balancing
pool for the specified time (15 seconds). If you immediately run the above `while` loop again,
you'll not see any errors. If you wait out the pool ejection timeout period (15 seconds) and run
it again, you'll again see a single failure followed by success. This is the circuit breaker in action!

You can verify that the failing pod was indeed ejected by looking at the internal Istio proxy (Envoy)
statistics for the `catalog` pod. First, get the pod name:

~~~sh
oc get pods -l app=catalog

NAME                         READY     STATUS    RESTARTS   AGE
catalog-2-p5q4d              2/2       Running   0          57m
~~~

Replace `[CATALOG_POD_NAME]` with your catalog's pod name in the below command:

~~~sh
~  % oc rsh -c catalog [CATALOG_POD_NAME] curl localhost:15000/stats|grep outlier | sed 's/^.*outlier_detection.//g'
ejections_active: 1
ejections_consecutive_5xx: 14
ejections_detected_consecutive_5xx: 14
ejections_detected_consecutive_gateway_failure: 0
ejections_detected_success_rate: 0
ejections_enforced_consecutive_5xx: 14
ejections_enforced_consecutive_gateway_failure: 0
ejections_enforced_success_rate: 0
ejections_enforced_total: 14
ejections_overflow: 0
ejections_success_rate: 0
ejections_total: 14
~~~

You can see that Envoy is reporting `1` active ejection (and if you run the `while` loop enough, the value
of `ejections_total` and other values will go up as Envoy is tracking the # of total lifetime failures).

### Ultimate resilience with retries, circuit breaker, and pool ejection

Even with pool ejection your application doesn’t look that resilient. That’s
because we’re still letting some errors (`quantity = -1`) to be propagated to our clients. But we can
improve this. If we have enough instances and/or versions of a specific service
running into our system, we can combine multiple Istio capabilities to achieve
the ultimate backend resilience:

* Circuit Breaker to avoid multiple concurrent requests to an instance
* Pool Ejection to remove failing instances from the pool of responding instances
* Retries to forward the request to another instance just in case we get an open circuit breaker and/or pool ejection;

By simply adding a retry configuration to Istio, we’ll be able to get
rid completely of our `503`s requests and `inventory = -1` failures. This means that whenever we receive a failed
request from an ejected instance, Istio will forward the request to another supposedly
healthy instance.

Put the retry rule into effect by replacing the default route created earlier with an updated route which contains
a retry policy:

~~~sh
cat <<EOF | oc replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: inventory-v1-route
spec:
  destination:
    name: inventory
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 100
  httpReqRetries:
    simpleRetry:
      perTryTimeout: 1s
      attempts: 3
EOF
~~~

And repeat the quantity test:

|**CAUTION:** Replace `GUID` with the guid provided to you.

~~~sh
while true; do
    echo "Fetching Catalog..."
    curl -s http://catalog-prod.{{APPS_HOSTNAME_SUFFIX}}/services/products | jq '.[] | [.quantity, .name] | @text'
    echo ---------------------
    echo
    sleep 5
done
~~~

You won’t receive any -1 inventories anymore, no matter how long you let it run.
When our misbehaving pod returns a failure, Istio kicks it out of the load balancing pool and retries,
which the working pods are able to handle thanks to pool ejection and retry. Your application is now much
more resiliant in the face of overwhelming load and misbehaving infrastructure!

You can verify the retry is happening by once again looking at the Envoy statistics:

~~~sh
oc rsh -c catalog catalog-2-p5q4d curl localhost:15000/stats|grep inventory|grep upstream_rq_retry

cluster.out.inventory.prod.svc.cluster.local|http|version=v1.upstream_rq_retry: 1
cluster.out.inventory.prod.svc.cluster.local|http|version=v1.upstream_rq_retry_overflow: 0
cluster.out.inventory.prod.svc.cluster.local|http|version=v1.upstream_rq_retry_success: 1
~~~

Notice the values of `upstream_rq_retry` and `upstream_rq_retry_success` indicate that one or more retries were
attempted, and they were successful.

### Clean up

~~~sh
oc delete routerule --all
oc delete destinationpolicy --all
~~~

## Congratulations!

In this section you made your microservices more resilient and able to gracefully
handle high workloads without a loss in quality. This is important in any cloud native
application architecture due to the variable nature of cloud environments and
unpredictable nature of client and end users.