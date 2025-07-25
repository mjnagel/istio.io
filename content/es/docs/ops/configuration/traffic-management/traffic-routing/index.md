---
title: Understanding Traffic Routing
linktitle: Traffic Routing
description: How Istio routes traffic through the mesh.
weight: 30
keywords: [traffic-management,proxy]
owner: istio/wg-networking-maintainers
test: n/a
---

One of the goals of Istio is to act as a "transparent proxy" which can be dropped into an existing cluster, allowing traffic to continue to flow as before.
However, there are powerful ways Istio can manage traffic differently than a typical Kubernetes cluster because of the additional features such as request load balancing.
To understand what is happening in your mesh, it is important to understand how Istio routes traffic.

{{< warning >}}
This document describes low level implementation details. For a higher level overview, check out the traffic management [Concepts](/es/docs/concepts/traffic-management/) or [Tasks](/es/docs/tasks/traffic-management/).
{{< /warning >}}

## Frontends and backends

In traffic routing in Istio, there are two primary phases:

* The "frontend" refers to how we match the type of traffic we are handling.
  This is necessary to identify which backend to route traffic to, and which policies to apply.
  For example, we may read the `Host` header of `http.ns.svc.cluster.local` and identify the request is intended for the `http` Service.
  More information on how this matching works can be found below.
* The "backend" refers to where we send traffic once we have matched it.
  Using the example above, after identifying the request as targeting the `http` Service, we would send it to an endpoint in that Service.
  However, this selection is not always so simple; Istio allows customization of this logic, through `VirtualService` routing rules.

Standard Kubernetes networking has these same concepts, too, but they are much simpler and generally hidden.
When a `Service` is created, there is typically an associated frontend -- the automatically created DNS name (such as `http.ns.svc.cluster.local`),
and an automatically created IP address to represent the service (the `ClusterIP`).
Similarly, a backend is also created - the `Endpoints` or `EndpointSlice` - which represents all of the pods selected by the service.

## Protocols

Unlike Kubernetes, Istio has the ability to process application level protocols such as HTTP and TLS.
This allows for different types of [frontend](#frontends-and-backends) matching than is available in Kubernetes.

In general, there are three classes of protocols Istio understands:

* HTTP, which includes HTTP/1.1, HTTP/2, and gRPC. Note that this does not include TLS encrypted traffic (HTTPS).
* TLS, which includes HTTPS.
* Raw TCP bytes.

The [protocol selection](/es/docs/ops/configuration/traffic-management/protocol-selection/) document describes how Istio decides which protocol is used.

The use of "TCP" can be confusing, as in other contexts it is used to distinguish between other L4 protocols, such as UDP.
When referring to the TCP protocol in Istio, this typically means we are treating it as a raw stream of bytes,
and not parsing application level protocols such as TLS or HTTP.

## Traffic Routing

When an Envoy proxy receives a request, it must decide where, if anywhere, to forward it to.
By default, this will be to the original service that was requested, unless [customized](/es/docs/tasks/traffic-management/traffic-shifting/).
How this works depends on the protocol used.

### TCP

When processing TCP traffic, Istio has a very small amount of useful information to route the connection - only the destination IP and Port.
These attributes are used to determine the intended Service; the proxy is configured to listen on each service IP (`<Kubernetes ClusterIP>:<Port>`) pair and forward traffic to the upstream service.

For customizations, a TCP `VirtualService` can be configured, which allows [matching on specific IPs and ports](/es/docs/reference/config/networking/virtual-service/#L4MatchAttributes) and routing it to different upstream services than requested.

### TLS

When processing TLS traffic, Istio has slightly more information available than raw TCP: we can inspect the [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) field presented during the TLS handshake.

For standard Services, the same IP:Port matching is used as for raw TCP.
However, for services that do not have a Service IP defined, such as [ExternalName services](#externalname-services), the SNI field will be used for routing.

Additionally, custom routing can be configured with a TLS `VirtualService` to [match on SNI](/es/docs/reference/config/networking/virtual-service/#TLSMatchAttributes) and route requests to custom destinations.

### HTTP

HTTP allows much richer routing than TCP and TLS. With HTTP, you can route individual HTTP requests, rather than just connections.
In addition, a [number of rich attributes](/es/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) are available, such as host, path, headers, query parameters, etc.

While TCP and TLS traffic generally behave the same with or without Istio (assuming no configuration has been applied to customize the routing), HTTP has significant differences.

* Istio will load balance individual requests. In general, this is highly desirable, especially in scenarios with long-lived connections such as gRPC and HTTP/2, where connection level load balancing is ineffective.
* Requests are routed based on the port and *`Host` header*, rather than port and IP. This means the destination IP address is effectively ignored. For example, `curl 8.8.8.8 -H "Host: productpage.default.svc.cluster.local"`, would be routed to the `productpage` Service.

## Unmatched traffic

If traffic cannot be matched using one of the methods described above, it is treated as [passthrough traffic](/es/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services).
By default, these requests will be forwarded as-is, which ensures that traffic to services that Istio is not aware of (such as external services that do not have `ServiceEntry`s created) continues to function.
Note that when these requests are forwarded, mutual TLS will not be used and telemetry collection is limited.

## Service types

Along with standard `ClusterIP` Services, Istio supports the full range of Kubernetes Services, with some caveats.

### `LoadBalancer` and `NodePort` Services

These Services are supersets of `ClusterIP` Services, and are mostly concerned with allowing access from external clients.
These service types are supported and behave exactly like standard `ClusterIP` Services.

### Headless Services

A [headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) is a Service that does not have a `ClusterIP` assigned.
Instead, the DNS response will contain the IP addresses of each endpoint (i.e. the Pod IP) that is a part of the Service.

In general, Istio does not configure listeners for each Pod IP, as it works at the Service level.
However, to support headless services, listeners are set up for each IP:Port pair in the headless service.
An exception to this is for protocols declared as HTTP, which will match traffic by the `Host` header.

{{< warning >}}
Without Istio, the `ports` field of a headless service is not strictly required because requests go directly to pod IPs, which can accept traffic on all ports.
However, with Istio the port must be declared in the Service, or it will [not be matched](/es/docs/ops/configuration/traffic-management/traffic-routing/#unmatched-traffic).
{{< /warning >}}

### ExternalName Services

An [ExternalName Service](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) is essentially just a DNS alias.

To make things more concrete, consider the following example:

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
  name: alias
spec:
  type: ExternalName
  externalName: concrete.example.com
{{< /text >}}

Because there is no `ClusterIP` nor pod IPs to match on, for TCP traffic there are no changes at all to traffic matching in Istio.
When Istio receives the request, they will see the IP for `concrete.example.com`.
If this is a service Istio knows about, it will be routed as described [above](#tcp).
If not, it will be handled as [unmatched traffic](#unmatched-traffic)

For HTTP and TLS, which match on hostname, things are a bit different.
If the target service (`concrete.example.com`) is a service Istio knows about, then the alias hostname (`alias.default.svc.cluster.local`) will be added
as an _additional_ match to the [TLS](#tls) or [HTTP](#http) matching.
If not, there will be no changes, so it will be handled as [unmatched traffic](#unmatched-traffic).

An `ExternalName` service can never be a [backend](#frontends-and-backends) on its own.
Instead, it is only ever used as additional [frontend](#frontends-and-backends) matches to existing Services.
If one is explicitly used as a backend, such as in a `VirtualService` destination, the same aliasing applies.
That is, if `alias.default.svc.cluster.local` is set as the destination, then requests will go to the `concrete.example.com`.
If that hostname is not known to Istio, the requests will fail; in this case, a `ServiceEntry` for `concrete.example.com` would make this configuration work.

### ServiceEntry

In addition to Kubernetes Services, [Service Entries](/es/docs/reference/config/networking/service-entry/#ServiceEntry) can be created to extend the set of services known to Istio.
This can be useful to ensure that traffic to external services, such as `example.com`, get the functionality of Istio.

A ServiceEntry with `addresses` set will perform routing just like a `ClusterIP` Service.

However, for Service Entries without any `addresses`, all IPs on the port will be matched.
This may prevent [unmatched traffic](#unmatched-traffic) on the same port from being forwarded correctly.
As such, it is best to avoid these where possible, or use dedicated ports when needed.
HTTP and TLS do not share this constraint, as routing is done based on the hostname/SNI.

{{< tip >}}
The `addresses` field and `endpoints` field are often confused.
`addresses` refers to IPs that will be matched against, while endpoints refer to the set of IPs we will send traffic to.

For example, the Service entry below would match traffic for `1.1.1.1`, and send the request to `2.2.2.2` and `3.3.3.3` following the configured load balancing policy:

{{< text yaml >}}
addresses: [1.1.1.1]
resolution: STATIC
endpoints:
- address: 2.2.2.2
- address: 3.3.3.3
{{< /text  >}}

{{< /tip >}}
