
# Envoy and the Network Stack

Let’s say you want to write an HTTP network proxy. There are two obvious ways to approach this: work at the level of HTTP, or work at the level of TCP.

At the HTTP level, you’d read an entire HTTP request off the wire, parse it, look at the headers and the URL, and decide what to do. Then you’d read the entire response from the back end, and send it to the client.
+ the proxy has full knowledge of what exactly the user is trying to accomplish, and it gets to use that knowledge to do very clever things.
- complex and slow – adds latency since it is reading and parsing the entire request before making any decisions.

At the TCP level you'd just read and write bytes, and use IP addresses, TCP port numbers, etc., to make your decisions about how to handle things.
+ very fast, and certain things become very elegant and simple.
- suppose you want to proxy different URLs to different back ends? That’s not possible with the typical L3/4 proxy: higher-level application information isn’t accessible down at these layers.

Envoy deals with the fact that both of these approaches have real limitations by operating at layers 3, 4, and 7 simultaneously. This is extremely powerful, and can be very performant… but you generally pay for it with configuration complexity.



# The Envoy Mesh

The next bit that’s a little surprising about Envoy is that most applications involve two layers of Envoys, not one:

- Edge Envoy (Ingress) - gives the rest of the world a single point of ingress. Incoming connections from outside come here, and the edge Envoy decides where they go internally.
- Envoy-Proxy (Sidecar) - each instance of the service has its own Envoy running alongside it, a separate process next to the service itself. These “service Envoys” keep an eye on their services, and remember what’s running and what’s not.

All of the Envoys form a **mesh**, and share routing information amongst themselves.


# Envoy Configuration Overview

Envoy’s configuration consists primarily of listeners and clusters.
- A listener tells Envoy a TCP port on which it should listen, and a set of filters with which Envoy should process what it hears.
- A cluster tells Envoy about one or more backend hosts to which Envoy can proxy incoming requests.

So pretty simple right?... Nope

Each filter will have it's own configuration, which is uniqe and often complex.

- Envoy can be dynamically configured, we can pass configurations through API.

![](C:/workspace/envoy/2022-02-14-19-36-55.png)

# Envoy tracing

## Overview

Distributed tracing allows developers to obtain visualizations of call flows in large service oriented architectures. It can be invaluable in understanding serialization, parallelism, and sources of latency.
Envoy supports three features related to system wide tracing:

- Request ID generation: Envoy will generate UUIDs when needed and populate the x-request-id HTTP header. Applications can forward the x-request-id header for unified logging as well as tracing. The behavior can be configured on a per HTTP connection manager basis using an extension.

- Client trace ID joining: The x-client-trace-id header can be used to join untrusted request IDs to the trusted internal x-request-id.

- External trace service integration: Envoy supports pluggable external trace visualization providers, that are divided into two subgroups:

  - External tracers which are part of the Envoy code base, like LightStep, Zipkin or any Zipkin compatible backends (e.g. Jaeger), Datadog, SkyWalking and AWS X-Ray.

  - External tracers which come as a third party plugin, like Instana.

## How to initiate a trace

The HTTP connection manager that handles the request must have the tracing object set. There are several ways tracing can be initiated:

- By an external client via the x-client-trace-id header.
- By an internal service via the x-envoy-force-trace header.
- Randomly sampled via the random_sampling runtime setting.


# Refrences:

- Hoot: Observing Envoy - Monitoring, Performance, and Troubleshooting:
Great talk that shows things like monitoring with prometheus
https://www.youtube.com/watch?v=ZthWg-_Bg_c