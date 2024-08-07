---
date:  2024-08-02T08:00:00+07:00
layout: post
title:  "gRPC/HTTP2 Load Balancing Problem"
summary: In this post, we will look at the load balancing problem on gRPC/HTTP2.
tags:  ["IT", "gRPC", "LoadBalancing"]
draft: false
author: "Thao Phan"
---

# gRPC/HTTP2 Load Balancing Problem
## 1. How k8s Service actually handle request load balancing ?

Corresponding to k8s official document:

> The Service API, part of Kubernetes, is an abstraction to help you expose groups of Pods over a network. Each Service object defines a logical set of endpoints (usually these endpoints are Pods) along with a policy about how to make those pods accessible

The Service is an abstraction that mean it doesn’t contain anything, the IP address is allocated for the service actually stored in etcd and is handled by control plane. Beside, the IP address of service is used by kube-proxy component.

The kube-proxy component is responsible for implementing a virtual IP mechanism for Services of type other than ExternalName

For each Service, kube-proxy calls appropriate APIs (depending on the kube-proxy mode) to configure the node to capture traffic to the Service's clusterIP and port, and redirect that traffic to one of the Service's endpoints (usually a Pod, but possibly an arbitrary user-provided IP address)

By default, Kubernetes uses the iptables proxy mode to redirect traffic from clients to backend pods. When a client connects to the Service's virtual IP (ClusterIP), a backend pod is chosen randomly. Packets are delivered to this pod using iptables with the NAT method without rewriting the client IP address. This means that when the client initiates to the service (using TCP connection), it is actually connecting to the specified pod through that TCP connections.

This mechanism is known as L4 load balancer (L4 mean Layer 4 of OSI Network model), L4 load balancer typically operates only at the level of the L4 TCP/UDP connection/session. (For more detail, please read https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236 )
![K8S-l4-load-balancer.drawio (1).png](../hugo-theme-console/static/hugo-theme-console/image/2024-08-02-grpc-load-balancing-problem/K8S-l4-load-balancer.drawio.png)
This L4 LB work well with HTTP1.1 for some reasons:

With HTTP/1.1, because only one request can be handled at a time on a single TCP connection, this leads to the need to open multiple TCP connections running in parallel. Taking advantage of this, request distribution will work efficiently based on L4 load balancing.

Additionally, long-lived HTTP/1.1 connections typically expire after some time, and are torn down by the client (or server).

## 2. gRPC/HTTP2 load balancing problem with L4 LB

As we can see above, with the L4 load balancer, any multiplexing, kept-alive protocol will encounter load balancing problems. Assume we have a client that wants to request the backend through a Kubernetes service using HTTP/2 or gRPC. The connection is initiated between the client and pod A, then all requests will go through this connection until it is terminated, which is not what we desire. Our expectation is that it will work the same as HTTP/1.1, where requests are distributed across the pods.
![request-load-balancer.drawio.png](../hugo-theme-console/static/hugo-theme-console/image/2024-08-02-grpc-load-balancing-problem/request-load-balancer.drawio.png)
### Reference

https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/

https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236 