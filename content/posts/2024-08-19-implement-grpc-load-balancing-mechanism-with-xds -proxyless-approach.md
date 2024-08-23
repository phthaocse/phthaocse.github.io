---
date:  2024-08-19T08:00:00+07:00
layout: post
title:  "Implement gRPC load balancing mechanism with xDS - Proxyless approach"
summary: As we know, when we use gRPC-based service, it will have a problem about load balancing if we use a L4 load balancing (connection-based load balancing) approach (gRPC/HTTP2 Load Balancing Problem).  To solve this problem, we need a different way to achieve load balancing goal by using L7 load balancing (application/request based load balancing). There are a couple of ways to implement L7 load balancing, such as using a headless service, or using proxy, service mesh like Envoy, Istio, Linkerd. Beside these approaches, we have another mechanism which is officially support by gRPC team called xDS-Based Global Load Balancing (https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md). .
tags:  ["IT", "gRPC", "LoadBalancing"]
draft: false
author: "Thao Phan"
---
Abstract

As we know, when we use gRPC-based service, it will have a problem about load balancing if we use a L4 load balancing (connection-based load balancing) approach (https://giaohangnhanh.atlassian.net/wiki/x/PIDpCg).  To solve this problem, we need a different way to achieve load balancing goal by using L7 load balancing (application/request based load balancing). There are a couple of ways to implement L7 load balancing, such as using a headless service, or using proxy, service mesh like Envoy, Istio, Linkerd. Beside these approaches, we have another mechanism which is officially support by gRPC team called xDS-Based Global Load Balancing (https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md).

# 1. What is xDS ?

The xDS stands for “x Discovery Service”, which is a suite of APIs for service discovery proposed and developed by the Envoy team, where “x” can take on various values. We can find the official spec at https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol, it has a dozen of “x” like, but in the context of gRPC, we only need to pay attention to the following four types:

- **Listener Discovery Service (LDS)**: Returns [Listener](https://github.com/envoyproxy/envoy/blob/main/api/envoy/api/v2/listener/listener_components.proto) resources. Used basically as a convenient root for the gRPC client's configuration. Points to the RouteConfiguration.

- **Route Discovery Service (RDS)**: Returns [RouteConfiguration](https://github.com/envoyproxy/envoy/blob/main/api/envoy/api/v2/route/route_components.proto) resources. Provides data used to populate the gRPC [service](https://github.com/grpc/grpc/blob/master/doc/service_config.md) config. Points to the Cluster.

- **Cluster Discovery Service (CDS)**: Returns [Cluster](https://github.com/envoyproxy/envoy/blob/main/api/envoy/api/v2/cluster.proto) resources. Configures things like load balancing policy and load reporting. Points to the ClusterLoadAssignment.

- **Endpoint Discovery Service (EDS)**: Returns [ClusterLoadAssignment](https://github.com/envoyproxy/envoy/blob/main/api/envoy/api/v2/endpoint.proto) resources. Configures the set of endpoints (backend servers) to load balance across and may tell the client to drop requests.

# 2.Why using xDS ?

For several reasons, we can utilize xDS for gRPC services

- It is officially supported by gRPC.

- Due to this support, it is integrated into the gRPC client and server implementations, so we don’t need to change or modify our code to handle load balancing. The only thing we need is to provide additional configuration.

- There is no additional latency compared to the proxy approach, which increases latency for all requests.

# 3. How to implement xDS ?

xDS is considered a protocol, so we need to implement all the APIs that gRPC clients require. This requires a deep knowledge of xDS and the requirements of gRPC clients, therefore, it would take a significant amount of effort, which cannot be easily estimated, if built from scratch . Fortunately, we have some tools that already implement the xDS such as go-control-plane and Istio.

In this article, I will use the go-control-plane as a xDS server or a Discovery Service. Originally, it was built for Envoy proxy client, but in the official the author also has mentioned that it was built to provides components that is shared by multiple different implementation. Thus we can leverage its API Server implementation, the task we need to do is provide data of my services through the Cache that go-control-plane provided.

You can checkout my project at here: https://github.com/phthaocse/grpc-xds-demo

### 3.1 Overall Architect

![xds_architect_overview.drawio.png](/image/2024-08-19-implement-grpc-load-balancing-mechanism-with-xds-proxyless-approach/xds_architect_overview.drawio.png)


In the beginning, when the xDS server receives a request for the first time, it initiates a process by calling the Kubernetes API to fetch information about upstream services based on the configuration we've provided. It then creates the necessary resources, stores a snapshot of them in the cache, and finally responds to the gRPC client.

After receiving all the resources, a gRPC client will cache them for future requests until the xDS server sends a new updated version of the resources, which means that a gRPC client only calls the xDS server once at the beginning. In this implementation, for simplicity, I simply fetch and store new resources at regular intervals. Note that whenever the resources are updated on the xDS server, they will be sent to the gRPC client through the ADS stream.

### 3.2 Discovery Flow

![xds_discovery_flow.svg](/image/2024-08-19-implement-grpc-load-balancing-mechanism-with-xds-proxyless-approach/xds_discovery_flow.svg)

#### 3.2.1  Establish ADS stream

As we often see, to facilitate communication between a client and a server, we need a place to provide the client with information about the server. Similarly, in this case, we will use a bootstrap file and the location of this is determined via the GRPC_XDS_BOOTSTRAP environment variable. The file is in JSON form, and its contents look like this:
```json
{
  "xds_servers": [
    {
      "server_uri": "xds-server.default.svc.cluster.local:18000",
      "channel_creds": [
        {
          "type": "insecure"
        }
      ]
    }
  ],

  "node": {
    "id": "25386353-c3e2-42f5-ad65-2b003c3386f5",
    "metadata": {
      "TEST_PROJECT_ID": "We45523"
    },
    "locality": {
      "zone": "my-zone"
    }
  }
}

```



#### 3.2.2 LDS

From the official doc “The gRPC client will typically start by sending an LDS request for a Listener resource whose name matches the server name from the target URI used to create the gRPC channel (including port suffix if present)“. By following this instruction, we need to do two things:

Create a listener resource for a given listeners name on the xDS server. To do this, we need keep the information of upstream services (aka listeners) and create a snapshot for all of them by using Cache component of go-control-plane.

Provide the listener name for gRPC through target uri  on the client side when establish a connection.

#### 3.2.3 RDS

With RDS Resource, we don't need to do much more than implementing according to the requirements from the documentation “In our initial implementation, the only field in the VirtualHost proto that the gRPC client needs to look at is the list of routes. The client will look only at the last route in the list (the default route), whose match field must contain a prefix field whose value is the empty string and whose route field must be set. Inside that route message, the cluster field will indicate which cluster to send the request to.”. You can view the implementation here.

#### 3.2.4 CDS

Cluster Discovery Resource require several information (implementation [here](https://github.com/phthaocse/grpc-xds-demo/blob/307cd44be0d275fb01fa1460c7908ab05411780d/xds-server/internal/app/resources.go#L122C1-L137C1))

- The type field must be set to EDS.

- In the eds_cluster_config field, the eds_config field must be set to indicate to use EDS (and the ConfigSource proto for the EDS server must indicate to use ADS). If the service_name field is set, that value will be used for the EDS request instead of the cluster name.

- The lb_policy field must be set to ROUND_ROBIN.

- If the lrs_server field is set, it must have its self field set, in which case the client should use LRS for load reporting. Otherwise (the lrs_server field is not set), LRS load reporting will be disabled.

#### 3.2.5 EDS

The xDS server needs to provide the ClusterLoadAssignment, which contains the endpoints field. For each entry in the endpoints field, we need to follow the official documentation for implementation. We use data obtained from the Kubernetes API to complete this. (Implementation [here](https://github.com/phthaocse/grpc-xds-demo/blob/307cd44be0d275fb01fa1460c7908ab05411780d/xds-server/internal/app/resources.go#L82)) 

# 4. Next Steps

- As mentioned before, in this implementation, I simply pull resource information from Kubernetes at regular intervals. This isn't efficient in practice, as the resource could change before the next trigger, which would affect availability. Conversely, when there are no changes, the pulling mechanism can cause unnecessary overhead. To resolve this, we can consider using the [Kubernetes API watcher](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes).

- Besides the client side, servers are also important for connecting to xDS in order to enable TLS communication, authorization, fault injection, rate limiting, and other miscellaneous features. To integrate the features that xDS provides on the server side, we can further investigate the [A36 proposal](https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md).

- Due to the initial design, which has only one global instance of the xDS server, the unavailability of the xDS server can lead to brief downtime for other dependent services. The gRPC team has proposed a solution to this problem in [A71: xDS Fallback](https://github.com/grpc/proposal/blob/master/A71-xds-fallback.md).

Related articles

https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md

https://www.youtube.com/watch?v=cGJXkZ7jiDk

https://www.youtube.com/watch?v=IbcJ8kNmsrE&t=349s

 

