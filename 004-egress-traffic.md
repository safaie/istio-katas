[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #sentences #kiali)

# Getting traffic out of the mesh

## Learning goals

- Understand how to access external services
- Understand Istio gateways (Egress)

## Introduction

These exercises will introduce you to Istio concepts 
and ([CRD's](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) 
for configuring traffic **out** of the service mesh. This is commonly 
called Egress traffic.

You will use two Istio CRD's for this. The [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway) 
and the [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry) 
CRD's. 

You will route traffic **directly** to an external service from an internal 
service with a service entry. Then you will route traffic from an internal 
service through a common egress gateway. 

These exercises build on the [Getting Traffic Into The mesh](003-ingress-traffic.md) exercises.

## Exercise: Egress Traffic With Service Entry

In this exercise you will deploy a **new** version of the **sentences** 
service. This new version will use a new **API** service which does nothing 
more than make a call to an external service ([httpbin](https://httpbin.org/)) 
asking for a delay of 1 second for responses. You will then define a ServiceEntry 
to allow traffic to the external service.

A ServiceEntry allows you to apply Istio traffic management for services 
running **outside** of your mesh. Your service might use an external API 
as an example. Once you have defined a service entry you can configure 
virtual services and destination rules to apply Istio features like 
redirecting or forwarding traffic to external destinations, defining 
retries, timeouts and fault injection policies. 

> :bulb: In our environment we have set the outBoundTrafficPolicy to 
> `REGISTRY_ONLY`.

By default, Istio configures Envoy proxies to **passthrough** requests to 
unknown services. So, technically service entries are not required. But 
without them you can't apply Istio features as the external service will be 
a black hole to the service mesh.

There is also the security aspect to consider. While securely controlling 
ingress traffic is the **highest** priority, it is good policy to securely 
control egress traffic also. Because of this many clusters will have the 
`outBoundTrafficPolicy` set to `REGISTRY_ONLY` as is done in our cluster. 
This will force you to define your external services with a service entry.

<details>
    <summary> More Info </summary>

When you create a service entry it is added to Istio's internal service 
registry and traffic is allowed out of the mesh to the defined destination. 

Istio maintains an internal service registry containing the set of services, 
and their corresponding service endpoints, running in a service mesh. 
Istio uses the service registry to generate Envoy configuration.

> Istio does not provide service discovery, although most services are 
> automatically added to the registry by Pilot adapters that reflect the 
> discovered services of the underlying platform (Kubernetes, Consul, plain DNS). 
> Additional services can also be registered manually using a ServiceEntry 
> configuration.

</details>

An example ServiceEntry CRD looks like this.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - external-api.example.com
  exportTo:
  - "."
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
```

The `hosts` field is used to select matching `hosts` in virtual services 
and destination rules. The `resolution` field is used to determine how the 
proxy will resolve IP addresses of the end points. The `exportTo` field scopes 
the service entry to the namespace where it is defined.

> :bulb: The `exportTo` field is important for this exercise. **Not** 
> scoping the service entry to your namespace will open the external 
> service for **all** attendees. 

### Overview

- Modify `sentences-ingress-gw.yaml` and `sentences-ingress-vs.yaml` files with `<YOUR_NAMESPACE>`

- Deploy sentences application in `004-egress-traffic/start`

- Run the loop query script with the `hosts` entry

- Observe the response of the external service

- Observe the traffic flow with Kiali

- Define a service entry for httpbin.org and apply it

- Observe the traffic flow with Kiali

- Create a virtual service with a timeout of 3 seconds

- Observe the traffic flow with Kiali

### Step by Step
<details>
    <summary> More Details </summary>

- **Modify `sentences-ingress-gw.yaml` and `sentences-ingress-vs.yaml`**

The `sentences-svc.yaml` is using a `ClusterIP` type as in exercise 
[Getting Traffic Into The mesh](003-ingress-traffic.md). Modify the 
`sentences-ingress-gw.yaml` and `sentences-ingress-vs.yaml files with 
**your** namespace. E.g. user1, user2, etc.

```yaml
...
    hosts:
    - "<YOUR_NAMESPACE>.sentences.istio.eficode.academy"
```

Apply the changes.

```console
kubectl apply -f 004-egress-traffic/start/sentences-ingress-gw.yaml
```

- **Deploy sentences application in `004-egress-traffic/start`**

Deploy `v2` of the sentences service along with the age, name and api service.

```console
kubectl apply -f 004-egress-traffic/start/
```

Make sure everything is in ready state.

```console
kubectl get gateway,se,vs,dr,svc,pods -n <YOUR_NAMESPACE>
```

You should now see a gateway, two virtual services and a destination rule along 
with four services and pods running. It should look something like below.

```console
NAME                                    AGE
gateway.networking.istio.io/sentences   91m

NAME                                            GATEWAYS        HOSTS                                       AGE
virtualservice.networking.istio.io/name-route   ["mesh"]        ["name"]                                    91m
virtualservice.networking.istio.io/sentences    ["sentences"]   ["user2.sentences.istio.eficode.academy"]   91m

NAME                                                        HOST   AGE
destinationrule.networking.istio.io/name-destination-rule   name   91m

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/age         ClusterIP   172.20.101.189   <none>        5000/TCP   91m
service/api         ClusterIP   172.20.8.90      <none>        5000/TCP   91m
service/name        ClusterIP   172.20.6.247     <none>        5000/TCP   91m
service/sentences   ClusterIP   172.20.109.183   <none>        5000/TCP   91m

NAME                                READY   STATUS    RESTARTS   AGE
pod/age-v1-7b9f67b7dc-gbftv         2/2     Running   0          91m
pod/api-v1-75f5bd69f8-l7ndk         2/2     Running   0          91m
pod/name-v1-795cf79f69-h4htd        2/2     Running   0          91m
pod/sentences-v2-75c766ff6c-f68bw   2/2     Running   0          91m
```

- **Run the loop query script with the `hosts` entry**

```console
./scripts/loop-query.sh -g <YOUR_NAMESPACE>.sentences.istio.eficode.academy
```

- **Observe the responses for the external service**

Export the pod as an environment variable and tail the logs.

```console
export API_POD=$(kubectl get pod -l app=sentences,mode=api -o jsonpath={.items..metadata.name})
kubectl logs "$API_POD" --tail=20 --follow
```

You should see a response of 502(Bad Gateway) because there exists no service entry for 
the external service httpbin.

```console
INFO:werkzeug:127.0.0.1 - - [10/Aug/2021 12:07:41] "GET / HTTP/1.1" 200 -
WARNING:root:Response was: 502                      <-------------------- Bad Gateway Response
WARNING:root:Operation 'api' took 306.376ms
```

- **Observe the traffic flow with Kiali**

Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu.

![No Service Entry](images/kiali-api-no-se.png)

As there is no service entry all traffic to the external service is blocked 
and it is a **black hole** to the service mesh.

- **Define a service entry for httpbin.org**

Create a service entry called `api-egress-se.yaml`in 
`004-egress-traffic/start/`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: api-egress-se
spec:
  hosts:
  - httpbin.org
  exportTo:
  - "."
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
```

Apply the service entry.

```console
kubectl apply -f 004-egress-traffic/start/api-egress-se.yaml
```

- **Observe the responses for the external service**

Export the pod as an environment variable and tail the logs.

```console
export API_POD=$(kubectl get pod -l app=sentences,mode=api -o jsonpath={.items..metadata.name})
kubectl logs "$API_POD" --tail=20 --follow
```

Now you should be getting a 200(OK) response from the external service. 

```console
INFO:werkzeug:127.0.0.1 - - [10/Aug/2021 12:21:14] "GET / HTTP/1.1" 200 -
WARNING:root:Response was: 200                <-------------------- OK Response
WARNING:root:Operation 'api' took 374.259ms
```

- **Observe the traffic flow with Kiali**

Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu.

![No Service Entry](images/kiali-api-se.png)

Now Kiali recognizes the external service because of the service entry and it 
is no longer a black hole.

- **Create a virtual service with a timeout of 0.5 seconds**

Basically all we have done so far is to add an entry for httpbin to Istio's 
internal service registry. But we can now apply some of the Istio features to 
external service. To demonstrate this you will create a virtual service for 
traffic to httpbin with a timeout tof `0.5s`.

> The api service asks httpbin.org for a response delay of 1 second.

Create a file called `api-egress-vs.yaml` in 
`004-egress-traffic/start/`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-egress-vs
spec:
  hosts:
    - httpbin.org
  exportTo:
  - "."
  http:
  - timeout: 0.5s
    route:
      - destination:
          host: httpbin.org
        weight: 100
```

Apply the virtual service.

```console
kubectl apply -f 004-egress-traffic/start/api-egress-vs.yaml
```

- **Observe the responses for the external service**

Export the pod as an environment variable and tail the logs.

```console
export API_POD=$(kubectl get pod -l app=sentences,mode=api -o jsonpath={.items..metadata.name})
kubectl logs "$API_POD" --tail=20 --follow
```

Now you should be getting a 504(Gateway Timeout) response from the external service. 

```console
INFO:werkzeug:127.0.0.1 - - [10/Aug/2021 13:29:11] "GET / HTTP/1.1" 200 -
WARNING:root:Response was: 504                <-------------------- 504 Gateway Timeout
WARNING:root:Operation 'api' took 504.809ms
```

Change the timeout to something greater than 1 second and ensure that you get 200(OK) responses.

</details>

## Exercise: Egress Traffic With Gateway

In a previous exercise you configured Istio to allow access to an external 
service([httpbin](http://httpbin.org)). You then applied a service entry and 
a simple virtual service with a timeout to prove that Istio traffic management 
features can be applied.

In the above case the traffic was flowing directly from the workloads Envoy 
sidecar to the external service. 

But there are use cases where you need to have traffic leaving the mesh 
routed **via** a dedicated **egress** gateway.

In this exercise we will route outbound traffic for api service through the 
dedicated **egress** gateway(`istio-egressgateway`) provided by Istio in the 
`istio-system` namespace.

<details>
    <summary> More Info </summary>

A Gateway **describes** a load balancer operating at the **edge** of the mesh 
receiving incoming or outgoing **HTTP/TCP** connections. The specification 
describes the ports to be expose, type of protocol, configuration for the 
load balancer, etc.

An Istio **Egress** gateway in a Kubernetes cluster consists, at a minimum, of a 
Deployment and a Service. Istio egress gateways are based on Envoy and have a 
**standalone** Envoy proxy. 

Inspecting our course environment would show something like:

```console
NAME                                       TYPE                                   
istio-egressgateway                        deployment  
istio-egressgateway                        service
istio-egressgateway-8679c48588-2p8vw       pod
```

Inspecting the POD would show something like:

```console
NAME                                    CONTAINERS
istio-egressgateway-8679c48588-2p8vw    istio-proxy
```

</details>

You are going to do this by defining a gateway. 

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-egressgateway
spec:
  selector:
    app: istio-egressgateway
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - myapp.org
```
The fields are the same as for the gateway you defined for the sentences 
service in a previous exercise. The notable difference being that the 
**selectors** are now the labels on the **Egress** POD `istio-egressgateway`, 
which is also running a standalone Envoy proxy just like the ingress gateway.

The gateway defines an **exit point** to be exposed in the `istio-egressgateway`. 
That is it. Nothing else. Just like an ingress entry point it knows nothing 
about how traffic is routed to it. 

In order to route the traffic we, of course, use a virtual service. 

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-api
spec:
  hosts:
  - external-api.example.com
  exportTo:
  - "."
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: external-api.example.com
        port:
          number: 80
      weight: 100
```

> :bulb: Notice that there are **two** gateways and **two** matches/routes 
> defined in the above yaml.

The first route uses the reserved keyword `mesh` implying that this rule 
applies to the sidecars in the mesh. E.g any sidecar wanting to hit the 
external service will be routed to the `istio-egressgateway` in the 
`istio-system` namespace.

The second route is the one directing traffic from the egress gateway to 
the external service.

So, in order to get the outbound traffic **from** the workload to our external 
service you need to define a route to the `istio-egressgateway`, which is 
located in the `istio-system` namespace. Then you need to define a route 
from the egress gateway to the external service. 

 ### Overview

- Make sure the sentence service `v2` with age,name and api services are deployed

- Create an exit point(Gateway) for external service(httpbin.org) traffic

- Modify the `api-egress-vs.yaml` file from previous exercise

### Step by Step
<details>
    <summary> More Details </summary>

- **Make sure the sentence service `v2` with age,name and api services are deployed**

If you have not completed exercise [004]() Apply the yaml for the services if not already deployed.

```console
kubectl apply -f 004-egress-traffic/start/
```

**Create an exit point for external service httpbin traffic**

Create a file called `api-egress-gw.yaml` in 
`004-egress-traffic/start/`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    app: istio-egressgateway
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org
```

**Modify the `api-egress-vs.yaml` from previous exercise**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin-egress
spec:
  hosts:
  - httpbin.org
  exportTo:
  - "."
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local        
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
      weight: 100
```

</details>


# Summary

In these exercises you created a service entry to allow access to an external service. 
This is a pretty common use case. A lot of service meshes will have a `REGISTRY_ONLY` 
policy defined for security reasons. So you should be aware of what a service entry does.

The important takeaway from these exercises is this.

**If traffic is not flowing through the mesh, e.g through the envoy sidecars, 
then you cannot leverage Istio features. Regardless of whether it is ingress or 
egress traffic.**

# Cleanup

```console
kubectl delete -f 003-traffic-in-out-mesh/start/
```