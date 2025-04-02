---
title: BYO Waypoint in Istio Ambient Mode - Expanding Your Service Mesh Options
date: 2025-04-02
categories: [Kubernetes, Networking]
tags: [istio, ambient, waypoint, envoy, service mesh, kubernetes]
---

## Introducing BYO Waypoint in Istio Ambient Mode

Istio [Ambient](https://istio.io/latest/docs/ambient/overview/)'s architecture continues to evolve with powerful new capabilities that enhance flexibility and integration options. One significant advancement is the "Bring Your Own" (BYO) waypoint functionality, enabled through Ambient's innovative "sandwich mode" deployment model. This approach allows any Kubernetes Gateway API conformant proxy to be deployed as a waypoint, opening up new possibilities for extending your service mesh.

Istio Ambient already supports BYO ingress functionality for north-south traffic (as covered in this [detailed guide](https://ambientmesh.io/docs/traffic/third-party-gateways/)). In this post, we'll explore how the same flexibility extends to east-west traffic through BYO waypoints, enabling enhanced service-to-service communication within the mesh.

## Understanding the Waypoint in Ambient Architecture

Before diving into BYO waypoint capabilities, let's revisit very briefly how waypoints function in the standard Ambient deployment model.

![Standard Ambient Architecture](/assets/lib/ambient-standard-diagram.png)
_Standard Istio Ambient architecture with ztunnels and waypoint_

In this standard configuration:

1. Client Pod's(app container's) traffic gets redirected to the local ztunnel proxy running in its own network namespace.
2. The ztunnel sends traffic via HBONE (HTTP-Based Overlay Network Environment) to the waypoint for L7 processing
3. After processing, the waypoint sends traffic via HBONE to the destination ztunnel
4. Finally, the destination ztunnel(running inside server pod's network namespace) delivers traffic to the app container.

The ztunnel DaemonSet on each node provisions proxies within pod network namespaces but isn't directly in the traffic path. All HBONE connections are secured with mTLS.
Rather than revisiting the fundamentals of Ambient mode that are well-documented in the official Istio documentation and existing community resources, this post focuses specifically on the "sandwich mode" architecture and its implications for extending service mesh capabilities.


## The "Sandwich Mode"

The "sandwich mode" deployment represents a significant architectural innovation. Instead of using the [standard `istio-waypoint` class](https://istio.io/latest/docs/ambient/usage/waypoint/#deploy-a-waypoint-proxy) for waypoint deployment, any [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) conformant proxy, such as [EnvoyGateway](https://gateway.envoyproxy.io/) can be used as a waypoint:

![Sandwich Mode Architecture](/assets/lib/ambient-sandwich-diagram.png)
_Sandwich Mode: Using Gateway Class for Waypoint_

Key aspects of this architecture:

1. The waypoint is deployed using ingress gateway class `istio` unlike istio-managed waypoint where `istio-waypoint` is used
2. It's labeled with `istio.io/dataplane-mode: ambient` to enable traffic capture, unlike istio-managed waypoint where waypoint traffic is not intercepted by ztunnel
3. Server pods specify this non-istio gateway/waypoint as their waypoint using the usual `use-waypoint` label
4. Traffic to this waypoint is intercepted by ztunnel, which terminates the HBONE connection and uses plaintext within the waypoint's network namespace. Please note because of ztunnel interception, we dont need to configure any mtls or HBONE at custom waypoint proxy.
5. The waypoint proxy can be configured using HTTPRoute to apply L7 processing before routing to the server pod
6. The outgoing waypoint traffic also gets intercepted by ztunnel(within the same network namespace) and forwarded via HBONE to the server pod, where another ztunnel terminates the connection.

Let's look at the zoomed-in view of how this works within the waypoint node:

![Waypoint Internals](/assets/lib/ambient-sandwitch-zoomin.png)
_Detailed view of Gateway-class Waypoint in Sandwich Mode_

This diagram shows how traffic flows through the waypoint:
1. HBONE traffic arrives at the waypoint's ztunnel proxy (port 15008)
2. The ztunnel terminates HBONE and forwards to the Gateway proxy via plaintext (within the same network namespace)
3. After L7 processing, which could be configured using any non-istio control plane,the Gateway proxy returns traffic to the ztunnel
4. The ztunnel forwards traffic via HBONE to the destination

## Bringing Your Own Waypoint: The EnvoyGateway Example

The most powerful aspect of this architecture is the ability to use alternative Gateway API conformant proxies as waypoints. This means you can leverage specialized proxies with advanced features that may not be available in Istio's standard waypoint implementation.

For example, you can use EnvoyGateway as your waypoint control plane while still benefiting from Istio's service mesh foundation:

![BYO Waypoint with EnvoyGateway](/assets/lib/ambient-waypoint-zoom.png)
_BYO Waypoint with EnvoyGateway Control Plane_

This architecture provides several significant advantages:
- Istio still manages service discovery, certificates and ztunnel configuration
- The alternative control plane (like EnvoyGateway) manages the waypoint proxy configurations
- Both systems coexist within the same mesh architecture

### Advantages of Using Alternative Gateway API Conformant Proxies

1. **Leverage existing investments**: Use proxies your team already knows and has deployed
2. **Best-of-breed approach**: Select specialized proxies with capabilities tailored to your needs
3. **Advanced features**: Access capabilities like sophisticated rate limiting, custom filters, and authentication methods that may not be available in Istio
4. **Operational efficiency**: Maintain familiar debugging workflows and monitoring tools
5. **Ecosystem integration**: Better align with your existing API gateway and ingress solutions

### Use Cases Where This Flexibility Matters

- **Advanced traffic management**: When you need sophisticated features such as rate limiting or AI gateway capabilities
- **Security requirements**: For specialized authentication protocols or custom security filters
- **Multi-team environments**: Different teams can use their preferred proxies while maintaining mesh cohesion
- **Gradual migration**: Adopt Ambient mode while preserving existing gateway investments
- **Feature requirements**: Access specialized proxy capabilities not yet available in Istio

Important: The integration between EnvoyGateway and Istio Ambient's waypoint functionality is currently under active development. Keep an eye on the EnvoyGateway project for updates as this feature progresses.

## Implementation Details

To deploy a waypoint in "sandwich mode", you need to:

1. Deploy a gateway using a standard gateway class (not `istio-waypoint`)
2. Label it with `istio.io/dataplane-mode: ambient`
3. Configure your workloads to use this gateway as their waypoint with appropriate labels
4. Configure HTTPRoute or other gateway resources to define L7 routing and policies

```yaml
# Example Gateway in "sandwich mode"
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: byo-waypoint
  namespace: my-ns
  labels:
    istio.io/dataplane-mode: ambient
spec:
  gatewayClassName: istio # Using standard gateway class
  listeners:
  - allowedRoutes:
      namespaces:
        from: Same
    hostname: my-svc.my-ns.svc.cluster.local
    name: captured-fqdn
    port: 80
    protocol: HTTP
  - allowedRoutes:
      namespaces:
        from: Same
    name: fake-hbone-port
    port: 15008
    protocol: TCP
---
# Example HTTPRoute for L7 routing
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: example-route
  namespace: my-ns
spec:
  hostnames:
  - my-svc.my-ns.svc.cluster.local
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: byo-waypoint
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - group: ""
        kind: Service
        name: my-svc
        port: 80
```
If you are a hands-on kind of person like me, then trying out istio integration test [TestSimpleHTTPSandwich](https://github.com/istio/istio/blob/release-1.25/tests/integration/ambient/waypoint_test.go#L217) can be a good starting point.

## Conclusion

BYO Waypoint functionality in Istio Ambient mode represents a significant advancement in service mesh architecture. It embraces the Kubernetes Gateway API ecosystem while maintaining the security and connectivity benefits of the Ambient mesh.

This approach gives operators unprecedented flexibility to:
- Leverage specialized gateway implementations
- Access advanced traffic management features
- Integrate with existing infrastructure
- Gradually adopt Ambient mode

As service mesh technology continues to evolve, this kind of architectural flexibility will be crucial for accommodating diverse workloads and infrastructure requirements while maintaining a consistent security and observability foundation.

I welcome your thoughts and questions on this complex topic in the comments section below, or reach out to me directly to continue the conversation!
