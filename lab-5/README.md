# Lab 5 - Traffic management with Istio

Service routes and version subsets should be in place given the destination rules applied in [Lab 3](../lab-3). If they are not present, re-apply the destination rules:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
    - name: v1
      labels:
        version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v2-mysql
      labels:
        version: v2-mysql
    - name: v2-mysql-vm
      labels:
        version: v2-mysql-vm
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---

```

## 5.1 Start managing traffic routes

Some basics to get us started.

## 5.1.1 Send all requests to Productpage v1

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
```

_Using Meshery, apply custom configuration._

Refresh productpage.

## 5.1.2 Send all requests to Productpage 2

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v2
```

_Using Meshery, apply custom configuration._

Refresh productpage.

## 5.2 Traffic routing based on user or user-agent type

### 5.2.1 Redirect requests from mobile devices

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"
  gateways:
    - bookinfo-gateway
  http:
    - match:
        - headers:
            User-Agent:
              regex: ^.*Mobile.*$
      route:
        - headers:
            request:
              set:
                x-response: "Success with Mobile"
          destination:
            host: productpage
            port:
              number: 9080
    - route:
        - destination:
            host: product
            subset: random
```

Set your browser to mimic a mobile device. Enable Developer tools, if need.Refresh productpage.

### 5.2.2 Redirect requests based on Cookie data

Example of using user information from a cookie to redirect requests.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: naruto
      route:
        - destination:
            host: reviews
            subset: v2
    - match:
        - headers:
            end-user:
              exact: sasuke
      route:
        - destination:
            host: reviews
            subset: v3
    - route:
        - destination:
            host: reviews
            subset: v1
```

## 5.3 Traffic Mirroring with Istio

You will need to generate load on BookInfo's productpage service. See [Lab 4](../lab-4/README.md] for instructions for running a performance test.

### 5.3.1

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 100
      # mirror:
      #   host: reviews
      #   subset: v2
      # mirror_percent: 100
```

5.3

Incrmentally increase the traffic split percentage to the higher version service #.
Start at 20% traffic split

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 80
        - destination:
            host: reviews
            subset: v3
          weight: 20
```

Move to 50%. Observe in Meshery.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
```

## 5.4 Injecting Latency

Acknowledging the fallacy that the network is always reliable, you will intentionally cause a little chaos.

## 5.4.1 Exploring timeouts with Reviews service

Note Istio Proxy's default timeout settings.

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: naruto
      fault:
        delay:
          percentage:
            value: 100.0
          fixedDelay: 11s
      route:
        - destination:
            host: reviews
            subset: v1
    - route:
        - destination:
            host: reviews
            subset: v2
```

### 5.4.2 Exploring timeouts with Ratings service

Note

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - match:
        - headers:
            end-user:
              exact: naruto
      fault:
        delay:
          percentage:
            value: 100.0
          fixedDelay: 5s
      route:
        - destination:
            host: ratings
            subset: v1
    - route:
        - destination:
            host: ratings
            subset: v1
```

## 5.5 Retries

Overcoming the latency issue.

### 5.5.1 Set 5 retry attempts and a 3 second timeout

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
      retries:
        attempts: 5
        perTryTimeout: 3s
```

<h2>
  <a href="../lab-6/README.md">
  <img src="../img/go.svg" width="32" height="32" align="left" />
  Continue to Lab 6</a>: Exploring security capabilities in Istio
</h2>

<br />
<hr />

Alternative, manual installation steps are provided for reference below. No need to execute these if you have performed the steps above.

<hr />

## <a name="appendix"></a> Appendix - Alternative Manual Steps

### 5 Traffic management with Istio

If you haven't forked or cloned this repository, please do so now:

```sh
git clone https://github.com/layer5io/advanced-istio-service-mesh-workshop
```

### 5.1.1

```sh
kubectl apply -f route-v1-v2/v1.yaml
```

### 5.1.2

```sh
kubectl apply -f route-v1-v2/v2.yaml
```

## 5.2 Traffic routing based on user or user-agent type

### 5.2.1 Redirect requests from mobile devices

```sh
kubectl apply -f route-header/device.yaml
```

### 5.2.2 Redirect requests based on Cookie data

```sh
kubectl apply -f route-header/user.yaml
```
