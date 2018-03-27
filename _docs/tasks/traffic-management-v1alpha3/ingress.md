---
title: Control Ingress Traffic
overview: Describes how to configure Istio to expose a service outside of the service mesh.

order: 30

layout: docs
type: markdown
redirect_from: /docs/tasks/ingress.html
---
{% include home.html %}

In a Kubernetes environment, the [Kubernetes Ingress Resource](https://kubernetes.io/docs/concepts/services-networking/ingress/)
allows users to specify services that should be exposed outside the cluster.
For traffic entering an Istio service mesh, however, an Istio-aware [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers)
is needed to allow Istio features, for example, monitoring and route rules, to be applied to traffic entering the cluster.

Istio provides an envoy-based ingress controller that implements very limited support for standard Kubernetes Ingress resources
as well as full support for an alternative specification,
[Istio Gateway]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#Gateway).
Using a Gateway is the recommended approach for configuring ingress traffic for Istio services. 
It is significantly more functional, not to mention the only option for non-Kubernetes environments.

This task describes how to configure Istio to expose a service outside of the service mesh using either specification.

## Before you begin

* Setup Istio by following the instructions in the
  [Installation guide]({{home}}/docs/setup/).
  
* Make sure your current directory is the `istio` directory.
  
* Start the [httpbin](https://github.com/istio/istio/tree/master/samples/httpbin) sample,
  which will be used as the destination service to be exposed externally.

  If you installed the [Istio-Initializer]({{home}}/docs/setup/kubernetes/sidecar-injection.html#automatic-sidecar-injection), do

  ```bash
  kubectl apply -f samples/httpbin/httpbin.yaml
  ```

  Without the Istio-Initializer:

  ```bash
  kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
  ```

* Generate a certificate and key that will be used to demonstrate a TLS-secured gateway

   A private key and certificate can be created for testing using [OpenSSL](https://www.openssl.org/).

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=foo.bar.com"
   ```

## Configuring ingress using an Istio Gateway resource (recommended)

> Note: This is still a WIP and not working yet.

An [Istio Gateway]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#Gateway) is the preferred
model for configuring ingress traffic in Istio.
An ingress Gateway describes a load balancer operating at the edge of the mesh receiving incoming
HTTP/TCP connections. 
It configures exposed ports, protocols, etc.,
but, unlike [Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/),
does not include any traffic routing configuration. Traffic routing for ingress traffic is instead configured
using Istio routing rules, exactly in the same was as for internal service requests.

### Configuring a Gateway

1. Create an Istio Gateway

   ```bash
   cat <<EOF | kubectl create -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: httpbin-gateway
   spec:
     servers:
     - port:
         number: 80
         name: http
     - port:
         number: 443
         name: https
       tls:
         mode: SIMPLE
         serverCertificate: /tmp/tls.crt
         privateKey: /tmp/tls.key
   EOF
   ```
   
   Notice that a single Gateway specification can configure multiple ports, a simple HTTP (port 80) and
   secure HTTPS (port 443) in our case.
   
1. Configure routes for traffic entering via the Gateway

   ```bash
   cat <<EOF | kubectl create -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
     - httpbin
     gateways:
     - httpbin-gateway
     http:
     - match:
         uri:
           prefix: /status
       route:
       - destination:
           port:
             number: 8000
           name: httpbin
     - match:
         uri:
           prefix: /delay
       route:
       - destination:
           port:
             number: 8000
           name: httpbin
   EOF
   ``` 
   
   Here we've created a [VirtualService]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#VirtualService)
   configuration for the `httpbin` service, containing two route rules that allow traffic for paths `/status` and `/delay`.
   The [gateways]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#VirtualService.gateways) list
   specifies that only requests through our `httpbin-gateway` are allowed. 
   All other external requests will be rejected with a 404 response.
   
   Note that in this configuration internal requests from other services in the mesh are not subject to these rules,
   but instead will simply default to round-robin routing. To apply these (or other rules) to internal calls,
   we could add the special value `mesh` to the list of `gateways`.
        
### Verifying a Gateway
   
The proxy instances implementing a particular `Gateway` configuration can be specified using a
[selector]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#Gateway.selector) field.
If not specified, as in our case, the `Gateway` will be implemented by the default `istio-ingress` controller.
Therefore, to test our Gateway we will send requests to the `istio-ingress` service.

1. Get the ingress controller pod's hostIP:

   ```bash
   kubectl -n istio-system get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
   ```

   ```bash
   169.47.243.100
   ```

1. Get the istio-ingress service's nodePorts for port 80 and 443:

   ```bash
   kubectl -n istio-system get svc istio-ingress
   ```

   ```bash
   NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
   istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
   ```

   ```bash
   export INGRESS_HOST=169.47.243.100:31486
   export SECURE_INGRESS_HOST=169.47.243.100:32254
   ```

1. Access the httpbin service with either HTTP or HTTPS using _curl_:

   ```bash
   curl -I http://$INGRESS_HOST/status/200
   ```

   ```
   HTTP/1.1 200 OK
   server: envoy
   date: Mon, 29 Jan 2018 04:45:49 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 48
   ```

   ```bash
   curl -I -k https://$SECURE_INGRESS_HOST/status/200
   ```

   ```
   HTTP/1.1 200 OK
   server: envoy
   date: Mon, 29 Jan 2018 04:45:49 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 96
   ```

1. Access any other URL that has not been explicitly exposed. You should
   see a HTTP 404 error:

   ```bash
   curl -I http://$INGRESS_HOST/headers
   ```

   ```
   HTTP/1.1 404 Not Found
   date: Mon, 29 Jan 2018 04:45:49 GMT
   server: envoy
   content-length: 0
   ```

   ```bash
   curl -I https://$SECURE_INGRESS_HOST/headers
   ```

   ```
   HTTP/1.1 404 Not Found
   date: Mon, 29 Jan 2018 04:45:49 GMT
   server: envoy
   content-length: 0
   ```

## Configuring ingress using a Kubernetes Ingress resource

An Istio Ingress specification is based on the standard [Kubernetes Ingress Resource](https://kubernetes.io/docs/concepts/services-networking/ingress/)
specification, with the following differences:

1. Istio Ingress specification contains a `kubernetes.io/ingress.class: istio` annotation.

2. All other annotations are ignored.

The following are known limitations of Istio Ingress:

1. Regular expressions in paths are not supported.
2. Fault injection at the Ingress is not supported.

The `servicePort` field in the Ingress specification can take a port number
(integer) or a name. The port name must follow the Istio port naming
conventions (e.g., `grpc-*`, `http2-*`, `http-*`, etc.) in order to
function properly. The name used must match the port name in the backend
service declaration.

### Configuring simple Ingress

1. Create a basic Ingress specification for the httpbin service

   ```bash
   cat <<EOF | kubectl create -f -
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: simple-ingress
     annotations:
       kubernetes.io/ingress.class: istio
   spec:
     rules:
     - http:
         paths:
         - path: /status/.*
           backend:
             serviceName: httpbin
             servicePort: 8000
         - path: /delay/.*
           backend:
             serviceName: httpbin
             servicePort: 8000
   EOF
   ```
 
   `/.*` is a special Istio notation that is used to indicate a prefix
   match, specifically a
   [rule match configuration]({{home}}/docs/reference/config/istio.networking.v1alpha3.html#HTTPMatchRequest)
   of the form (`prefix: /`).

### Verifying simple Ingress
   
1. Determine the ingress URL:

   * If your cluster is running in an environment that supports external load balancers,
     use the ingress' external address:

     ```bash
     kubectl get ingress simple-ingress -o wide
     ```
   
     ```bash
     NAME             HOSTS     ADDRESS                 PORTS     AGE
     simple-ingress   *         130.211.10.121          80        1d
     ```

     ```bash
     export INGRESS_HOST=130.211.10.121
     ```

   * If load balancers are not supported, use the ingress controller pod's hostIP:
   
     ```bash
     kubectl -n istio-system get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
     ```

     ```bash
     169.47.243.100
     ```

     along with the istio-ingress service's nodePort for port 80:
   
     ```bash
     kubectl -n istio-system get svc istio-ingress
     ```
   
     ```bash
     NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
     istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
     ```
   
     ```bash
     export INGRESS_HOST=169.47.243.100:31486
     ```
   
1. Access the httpbin service using _curl_:

   ```bash
   curl -I http://$INGRESS_HOST/status/200
   ```

   ```
   HTTP/1.1 200 OK
   server: envoy
   date: Mon, 29 Jan 2018 04:45:49 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 48
   ```

1. Access any other URL that has not been explicitly exposed. You should
   see a HTTP 404 error

   ```bash
   curl -I http://$INGRESS_HOST/headers
   ```

   ```
   HTTP/1.1 404 Not Found
   date: Mon, 29 Jan 2018 04:45:49 GMT
   server: envoy
   content-length: 0
   ```

### Configuring secure Ingress (HTTPS)

1. Create a Kubernetes Secret to hold the key/cert

   Create the secret `istio-ingress-certs` in namespace `istio-system` using `kubectl`. The Istio ingress controller
   will automatically load the secret.

   > Note: the secret MUST be called `istio-ingress-certs` in the `istio-system` namespace, or it will not
     be mounted and available to the Istio ingress controller.

   ```bash
   kubectl create -n istio-system secret tls istio-ingress-certs --key /tmp/tls.key --cert /tmp/tls.crt
   ```

   Note that by default all service accounts in the `istio-system` namespace can access this ingress key/cert,
   which risks leaking the key/cert. You can change the Role-Based Access Control (RBAC) rules to protect them.
   See (Link TBD) for details.
   
1. Create the Ingress specification for the httpbin service

   ```bash
   cat <<EOF | kubectl create -f -
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: secure-ingress
     annotations:
       kubernetes.io/ingress.class: istio
   spec:
     tls:
       - secretName: istio-ingress-certs # currently ignored
     rules:
     - http:
         paths:
         - path: /status/.*
           backend:
             serviceName: httpbin
             servicePort: 8000
         - path: /delay/.*
           backend:
             serviceName: httpbin
             servicePort: 8000
   EOF
   ```

   > Note: Because SNI is not yet supported, Envoy currently only allows a single TLS secret in the ingress.
   > That means the secretName field in ingress resource is not used.

### Verifying secure Ingress

1. Determine the ingress URL:

   * If your cluster is running in an environment that supports external load balancers,
     use the ingress' external address:

     ```bash
     kubectl get ingress secure-ingress -o wide
     ```

     ```bash
     NAME             HOSTS     ADDRESS                 PORTS     AGE
     secure-ingress   *         130.211.10.121          80        1d
     ```

     ```bash
     export SECURE_INGRESS_HOST=130.211.10.121
     ```

   * If load balancers are not supported, use the ingress controller pod's hostIP:

     ```bash
     kubectl -n istio-system get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
     ```

     ```bash
     169.47.243.100
     ```

     along with the istio-ingress service's nodePort for port 443:

     ```bash
     kubectl -n istio-system get svc istio-ingress
     ```

     ```bash
     NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
     istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
     ```

     ```bash
     export SECURE_INGRESS_HOST=169.47.243.100:32254
     ```

1. Access the httpbin service using _curl_:

   ```bash
   curl -I -k https://$SECURE_INGRESS_HOST/status/200
   ```

   ```
   HTTP/1.1 200 OK
   server: envoy
   date: Mon, 29 Jan 2018 04:45:49 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 96
   ```

1. Access any other URL that has not been explicitly exposed. You should
   see a HTTP 404 error

   ```bash
   curl -I -k https://$SECURE_INGRESS_HOST/headers
   ```

   ```
   HTTP/1.1 404 Not Found
   date: Mon, 29 Jan 2018 04:45:49 GMT
   server: envoy
   content-length: 0
   ```

## Using Istio routing rules to control ingress traffic

Istio's routing rules can be used to achieve a greater degree of control
when routing requests to backend services. For example, the following
command adds a 4s timeout to requests to the httpbin service's
/delay URL.

```bash
cat <<EOF | kubectl replace -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin
  gateways:
  - httpbin-gateway
  http:
  - match:
      uri:
        prefix: /status
    route:
    - destination:
        port:
          number: 8000
        name: httpbin
  - match:
      uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        name: httpbin
    timeout: 4s
EOF
```

If you were to make a call to the Ingress/Gateway controller with the URL
`http://$INGRESS_HOST/delay/10`, you will find that the call returns in 4s
instead of the expected 10s delay.

You can use other features of the route rules such as redirects, rewrites,
routing to multiple versions, regular expression based match in HTTP
headers, websocket upgrades, timeouts, retries, etc. Please refer to
[routing rules]({{home}}/docs/reference/config/istio.networking.v1alpha1.html)
for more details.

> Note 1: Fault injection does not currently work for ingress traffic

> Note 2: When matching requests in a routing rule to paths specified in a Kubernetes
  Ingress configuration, use the same exact path or prefix as the one used in
  the Ingress specification.

## Understanding what happened

Gateway or Ingress configuration resources allow external traffic to enter the
Istio service mesh and make the traffic management and policy features of Istio
available for edge services.

In the preceding steps we created a service inside the Istio service mesh
and showed how to expose both HTTP and HTTPS endpoints of the service to
external traffic. We also showed how to control the ingress traffic using
an Istio route rule.

## Cleanup

1. Remove the Gateway configuration.
    
   ```bash
   kubectl delete gateway httpbin-gateway
   ```

1. Remove the Ingress configuration.
    
   ```bash
   kubectl delete ingress simple-ingress secure-ingress 
   ```

1. Remove the routing rule and secret.
    
   ```bash
   istioctl delete virtualservice httpbin
   kubectl delete -n istio-system secret istio-ingress-certs
   ```

1. Shutdown the [httpbin](https://github.com/istio/istio/tree/master/samples/httpbin) service.

   ```bash
   kubectl delete -f samples/httpbin/httpbin.yaml
   ```

## What's next

* Learn more about [Ingress Control](https://kubernetes.io/docs/concepts/services-networking/ingress/).

* Learn more about [Traffic Routing]({{home}}/docs/reference/config/istio.networking.v1alpha3.html).