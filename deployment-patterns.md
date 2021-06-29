[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #sentences #kiali)

# Deployment Patterns

## Learning goal

- Use weighted traffic distribution for canary deployment
- Use a match criteria in a virtual service
- Use header based routing for blue/green deployment
- Mirror traffic for shadow deployment

## Introduction
These exercises will introduce you to the match, weight and mirror fields of the 
HTTPRoute. These fields and their combinations can be used to enable several 
useful deployment patterns.

These exercises build on the [Basic traffic routing](basic-traffic-routing.md) exercises.

## Exercise 1

This exercise is going to introduce you to weighted traffic distribution to 
route traffic in canary deployment pattern.

Canary deployment is a pattern for rolling out releases to a **subset** of users. 
The idea is to test and gather feedback from this subset of users.

> :laughing: Fun fact. The term canary comes from the coal mining industry 
> where canaries were used to alert miners when toxic gases reached dangerous 
> levels. In the same way canary deployments can alert you to issues, bad 
> design or whether features actually give the intended value.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service-route
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service-v1
      weight: 90
    - destination:
        host: my-service-v2
      weight: 10
```

### Overview

- 
- 

> :bulb: Some info

### Step by Step
<details>
    <summary> More Details </summary>

* **Step**

```console
Some command
```

*** **Step****

```console
Some command
```

</details>

## Exercise 2

This exercise is going to introduce you to the match field in the HTTPRoute 
so we can use **header** based routing to implement a Blue/Green deployment 
pattern. This allows the client can actively select the version of the name 
service it will hit.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service-route
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        my-header:
          exact: use-v2
    - route:
        - destination:
            host: my-service
            subset: my-service-v2
```

In the example above we define a HTTPMatchRequest with the field `match`.
> :bulb: The match is evaluated prior to any destination's being applied.

Istio allows to use certain parts of incoming requests and match them to values 
**you** define.

<details>
    <summary> More Info </summary>

* **uri:** Matches the request URI to the specified value.
* **schema:** Matches the request schema. (HTTP, HTTPS, ...)
* **method:** Matches the request method. (GET, POST, ...)
* **authority:** Matches the request authority header.
* **header:** Matches the request headers.
> :bulb: Headers need to be **lower** cased and separated by hyphens. 
> If headers is used uri, schema, method and authority are ignored.

There are three match types.

* **exact:** Exactly matches the **provided** value.
* **prefix:** Only the prefix part of the **provided** value will get matched.
* **regex:** The **provided** value will be matched based on the regex.
> :bulb: Istio regex's use the [re2](https://github.com/google/re2/wiki/Syntax) 
> regular expression syntax.

Match conditions to be satisfied for the rule to be activated. 
All conditions inside a **single** match block have AND semantics, while the 
list of match blocks have OR semantics. The rule is matched if any one of the 
match blocks succeed.

</details>

### Overview

- Deploy the sentences app with the destination rule and virtual service from previous exercises.

- Update the virtual service with a HTTPMatchRequest and apply it.

- Run the `scripts/loop-query.sh` and pass the header `x-test: use-v2` to it.

- Use the version app graph in Kiali to observe the traffic flow.

- Run the `scripts/loop-query.sh` **without** passing the `x-test:` header.

- Use the version app graph in Kiali to observe the traffic flow and understand what happened. 

- Update the virtual service to fix the problem.

> :bulb: Order of precedence in the virtual service still applies. 

### Step by Step
<details>
    <summary> More Details </summary>

* **Deploy the sentences app with the destination rule and virtual service**

```console
kubectl apply -f deploy/deployment-patterns/start/
```

This will deploy two versions of the **name** service along with a destination 
rule and virtual service as defined in a previous exercise.

* **Update the virtual service with a HTTPMatchRequest and apply it**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  gateways:
  - mesh
  http:
  - match:
    - headers:
        x-test:
          exact: use-v2
    route:
      - destination:
          host: name
          subset: name-v2
```

```console
kubectl apply -f deploy/deployment-patterns/start/name-virtual-service.yaml
```

* **Run loop-query.sh with the `x-test` header**

```console
./scripts/loop-query.sh 'x-test: use-v2'
```

* **Observe traffic flow with version app graph in Kiali**

![Header based routing](images/kiali-blue-green.png)

* **Run the `scripts/loop-query.sh` **without** header**

```console
./scripts/loop-query.sh
```

* **Observe traffic flow with version app graph in Kiali**

![Missing default destination](kiali-no-default-destination.png)

The problem we have here is that the match is evaluated first **before** any 
destination's are applied. Since the match was not true the route defined under 
it was not applied. Nor have we provided another route to fall back on when the 
match does not evaluate to true.

* **Update the virtual service**
To fix the problem we need to update the virtual service and give it a **default** 
route.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  gateways:
  - mesh
  http:
  - match:
    - headers:
        x-test:
          exact: use-v2
    route:
    - destination:
        host: name
        subset: name-v2
  - route:
    - destination:
        host: name
        subset: name-v1
```

> :bulb: Notice the indentation of the default route.

</details>

## Exercise 3

This exercise will introduce you to the `mirror` field of of the HTTPRoute 
so mirror traffic to another destination while still forwarding traffic to 
the original destination. 

> :bulb: Mirrored traffic is on a best effort basis. This means that the 
sidecar/gateway will **not** wait for mirrored traffic **before** sending 
a response from the original destination.

```yaml
Some yaml
```

### Overview

- 
- 

> :bulb: Some info

### Step by Step
<details>
    <summary> More Details </summary>

* **Step**

```console
Some command
```

*** **Step****

```console
Some command
```

</details>


## Summary

In exercise 1 you learned XXX

In exercise 2 you saw XXX 

In exercise 3 you saw XXX 

# Cleanup

```console
kubectl delete -f deploy/deployment-patterns/start/
```