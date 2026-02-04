# Design Document: Gateway-Level Fault Injection for kgateway

## 1. Background and Context

### What is kgateway?

[kgateway](https://github.com/kgateway-dev/kgateway) is a Kubernetes-native API gateway built on Envoy Proxy, designed to provide advanced traffic management, security, and observability for cloud-native applications. It implements the Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) standard, offering a vendor-neutral, declarative approach to configuring ingress and routing behavior.

kgateway extends the Gateway API with custom policies that enable enterprise-grade features such as:
- Advanced routing and traffic splitting
- Authentication and authorization
- Rate limiting and throttling
- Transformation and manipulation of requests/responses

### Why Gateway-Level Traffic Controls Matter

Modern distributed systems must be designed for failure. The ability to test system resilience under adverse conditions—network delays, service unavailability, partial failures—is critical for building confidence in production deployments.

**Gateway-level traffic controls** provide a unique vantage point:
- **Centralized control**: Apply resilience testing policies without modifying individual services
- **Environment parity**: Test the same traffic patterns across dev, staging, and production
- **Blast radius control**: Inject faults at ingress to simulate real-world failures
- **Zero application impact**: No code changes required in backend services

Fault injection at the gateway layer enables **proactive chaos engineering** and **continuous resilience validation** as part of CI/CD pipelines.

---

## 2. Problem Analysis

### What is Missing Today in kgateway?

Currently, kgateway does not provide native fault injection capabilities. Developers who want to test service resilience must resort to:

1. **Application-level modifications**: Adding fault injection logic directly into service code
2. **External tools**: Deploying separate chaos engineering platforms (e.g., Chaos Mesh, Litmus)
3. **Manual intervention**: Using proxies or network manipulation tools during testing

These approaches have significant drawbacks:

### Limitations of Existing Approaches

| Approach | Drawbacks |
|----------|-----------|
| **Modifying application code** | • Requires engineering time for each service<br>• Risk of leaving test code in production<br>• Not portable across services<br>• Tight coupling between testing and business logic |
| **External chaos tools** | • Additional infrastructure to manage and maintain<br>• Separate authorization and access control models<br>• Learning curve for operators<br>• May not integrate cleanly with Gateway API workflows |
| **Manual network manipulation** | • Not reproducible or automatable<br>• Difficult to control blast radius<br>• High risk of unintended side effects<br>• Poor integration with CI/CD pipelines |

### User Pain Points

Through community feedback and observations from similar API gateway ecosystems, the following pain points have been identified:

1. **No standardized way to test resilience**: Teams have no consistent, declarative method to inject faults for testing retry logic, timeouts, or circuit breakers.

2. **Difficult to validate production-readiness**: Without controlled fault injection, teams cannot confidently validate that services will behave correctly under real-world failure scenarios.

3. **High toil for testing**: Setting up fault injection requires significant manual effort, reducing the frequency and coverage of resilience testing.

4. **Risk of production incidents**: Without proper testing infrastructure, the first encounter with network failures or degraded services often occurs in production.

---

## 3. Design Goals

This prototype aims to achieve the following goals:

### Primary Goals

1. **Enable declarative fault injection** via Gateway API-compatible policies
   - Developers can define delay and abort behaviors using Kubernetes CRDs
   - Configuration follows Gateway API conventions (e.g., `targetRef` for policy attachment)

2. **Leverage Envoy's native fault injection capabilities**
   - Use Envoy's battle-tested `envoy.filters.http.fault` filter
   - No need to implement custom fault injection logic in the data plane

3. **Support percentage-based probabilistic faults**
   - Allow controlled blast radius (e.g., "inject delay into 10% of requests")
   - Enable gradual rollout of fault injection for safety

4. **Provide clear separation of concerns**
   - User-facing API should be simple and intuitive
   - Complexity of Envoy configuration hidden within controller translation

5. **Support both gateway-wide and route-specific policies**
   - Attach policies to `Gateway` resources (affects all routes)
   - Attach policies to `HTTPRoute` resources (affects only specific routes)

### Secondary Goals

1. **Observability**: Emit metrics and logs for injected faults
2. **Developer experience**: Clear validation errors and status reporting
3. **Safety**: Prevent accidental production outages via namespace isolation and RBAC

---

## 4. Non-Goals

To maintain prototype focus and clarity, the following are explicitly **out of scope**:

### Implementation Non-Goals

- ❌ **Production-ready controller implementation**: This prototype provides design specifications only, not a working Go codebase
- ❌ **Custom Resource Definition (CRD) YAML**: Formal schema definitions are deferred to implementation phase
- ❌ **Unit or integration tests**: Testing infrastructure is conceptually described but not implemented
- ❌ **Performance benchmarks**: Overhead analysis is deferred to implementation validation

### Feature Non-Goals

- ❌ **TCP-level fault injection**: Focus is on HTTP/HTTPS traffic only (Envoy's HTTP fault filter)
- ❌ **Service mesh integration**: This prototype focuses on ingress/gateway-level faults, not east-west (service-to-service) traffic
- ❌ **Advanced fault types**: Features like bandwidth throttling, connection limits, or packet loss are out of scope for the initial design
- ❌ **Dynamic runtime updates**: Percentage or duration values are static in policy; runtime adjustments via Envoy's runtime layer are not covered
- ❌ **Multi-cluster or global fault injection**: Scope is limited to a single Kubernetes cluster

### Ecosystem Non-Goals+

- ❌ **Competing with chaos engineering platforms**: This feature complements (not replaces) tools like Chaos Mesh or Litmus
- ❌ **Universal compatibility**: This design assumes Envoy as the data plane; other proxies (e.g., NGINX, HAProxy) are not considered

---

## 5. High-Level Solution Overview

### Conceptual Approach

The proposed solution introduces a new **Gateway Policy** that allows users to declaratively specify fault injection rules. These policies are reconciled by the kgateway controller, which translates them into Envoy HTTP filter configurations and applies them via the xDS (Discovery Service) protocol.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                         │
│                                                                 │
│  ┌──────────────────┐         ┌─────────────────────────────┐  │
│  │  GatewayPolicy   │         │    kgateway Controller      │  │
│  │  (CRD Instance)  │────────▶│                             │  │
│  │                  │         │  • Watches policies         │  │
│  │  faultInjection: │         │  • Validates config         │  │
│  │    delay: 5s     │         │  • Translates to Envoy      │  │
│  │    abort: 503    │         │  • Pushes xDS updates       │  │
│  └──────────────────┘         └─────────────────────────────┘  │
│                                            │                    │
│                                            ▼                    │
│                               ┌─────────────────────────────┐  │
│                               │    Envoy Proxy (Gateway)    │  │
│                               │                             │  │
│           Client Request      │  HTTP Filter Chain:         │  │
│                ──────────────▶│  1. Fault Injection Filter  │  │
│                               │  2. Router Filter           │  │
│                               └─────────────────────────────┘  │
│                                            │                    │
│                                            ▼                    │
│                               ┌─────────────────────────────┐  │
│                               │    Backend Services         │  │
│                               └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Request Flow with Fault Injection

1. **Developer creates a `GatewayPolicy`** with fault injection configuration (delay, abort, percentage)
2. **kgateway controller watches the policy**, validates it, and translates it to Envoy's `HTTPFault` filter configuration
3. **Envoy applies the filter** to matching HTTP traffic based on the policy's `targetRef` (Gateway or HTTPRoute)
4. **For each incoming request**:
   - Envoy evaluates whether the request should be affected (based on percentage)
   - If a delay is configured and the request is selected, Envoy pauses for the specified duration
   - If an abort is configured and the request is selected, Envoy returns the specified HTTP status without forwarding to the backend
   - Otherwise, the request proceeds normally to the backend service

### Policy Attachment Model

Policies follow the **Gateway API Policy Attachment** pattern:

- **Gateway-level attachment**: Affects all routes under a Gateway
  ```yaml
  targetRef:
    kind: Gateway
    name: my-gateway
  ```

- **Route-level attachment**: Affects only traffic matching a specific HTTPRoute
  ```yaml
  targetRef:
    kind: HTTPRoute
    name: my-route
  ```

---

## 6. Key Design Decisions and Trade-offs

### Decision 1: Gateway-Level vs. Service Mesh Fault Injection

**Choice**: Focus on gateway-level (north-south) fault injection rather than service mesh (east-west).

**Rationale**:
- **Simpler deployment model**: Many users run kgateway without a full service mesh
- **Clear use case**: Ingress-level fault injection is ideal for testing client behavior, retries, and timeouts
- **Avoids complexity**: Service mesh integration (e.g., Istio, Linkerd) introduces significant architectural complexity
- **Prototype scope**: Gateway-level injection provides clear value without requiring service mesh infrastructure

**Trade-off**:
- ✅ **Pro**: Lower barrier to entry; works with any backend regardless of service mesh adoption
- ❌ **Con**: Cannot test inter-service resilience (e.g., Service A → Service B failures)

**Future consideration**: If this feature is successful, a service mesh extension could be explored.

---

### Decision 2: Leverage Envoy's HTTP Fault Filter

**Choice**: Use Envoy's existing `envoy.filters.http.fault` filter instead of building custom logic.

**Rationale**:
- **Battle-tested**: Envoy's fault filter is mature, well-documented, and widely used
- **Feature-complete**: Supports delay injection, abort injection, and percentage-based activation
- **Performance**: Optimized by the Envoy community with minimal overhead
- **Maintenance**: No need to maintain custom fault injection code in kgateway

**Trade-off**:
- ✅ **Pro**: Immediate access to proven functionality; reduces implementation risk
- ❌ **Con**: Limited to Envoy's feature set (e.g., no bandwidth throttling or packet-level faults)

**Why this is acceptable**: Envoy's fault filter covers the vast majority of real-world use cases for HTTP-based chaos testing.

---

### Decision 3: Policy-Based Configuration (Not Annotations)

**Choice**: Introduce a dedicated `GatewayPolicy` CRD rather than using annotations on Gateway or HTTPRoute resources.

**Rationale**:
- **Separation of concerns**: Policies are logically distinct from routing configuration
- **Reusability**: A single policy can be referenced by multiple routes (future enhancement)
- **Extensibility**: Following Gateway API's policy attachment pattern enables future features (e.g., rate limiting, transformation)
- **Maintainability**: Cleaner codebase with dedicated reconciliation logic for policies

**Trade-off**:
- ✅ **Pro**: Aligns with Gateway API best practices; easier to evolve over time
- ❌ **Con**: Slightly more verbose than annotations (requires a separate resource)

**Why this is acceptable**: Kubernetes community has standardized on policy attachments as the idiomatic approach.

---

### Decision 4: Percentage-Based (Not Header-Based) Selection

**Choice**: Use percentage-based fault injection as the primary mechanism rather than header-based matching.

**Rationale**:
- **Safety**: Percentage-based selection allows gradual rollout (e.g., "affect 1% of traffic, then increase to 10%")
- **Simplicity**: Easier for operators to reason about blast radius
- **Common use case**: Most chaos testing scenarios involve probabilistic faults, not deterministic ones

**Trade-off**:
- ✅ **Pro**: Prevents accidental 100% outage; more aligned with production safety
- ❌ **Con**: Header-based injection (e.g., "only inject faults if `X-Chaos: enabled`") may be desired for controlled testing

**Future consideration**: Header-based matching can be added as an optional filter in a later iteration.

---

## 7. Open Questions and Assumptions

This prototype is designed to start a conversation with kgateway maintainers and the community. The following areas would benefit from feedback:

### Open Questions

1. **API naming and structure**:
   - Should the policy be called `GatewayPolicy`, `FaultInjectionPolicy`, or something else?
   - Should fault injection be a standalone policy or part of a broader "ResiliencePolicy"?

2. **Conflict resolution**:
   - What happens if multiple policies target the same Gateway or HTTPRoute?
   - Should we use a "first policy wins" model, merge policies, or reject conflicts?

3. **Default behavior**:
   - Should there be a cluster-wide default policy that applies unless overridden?
   - Or should fault injection be strictly opt-in?

4. **Integration with existing kgateway features**:
   - How should this interact with existing rate limiting or transformation policies?
   - Should there be a defined filter chain order?

5. **Production safety mechanisms**:
   - Should there be a "SafeMode" flag that prevents policies from being applied in production namespaces?
   - Should policies require explicit approval annotations for production use?

6. **Observability**:
   - What metrics and logs should be emitted by default?
   - Should Envoy's fault injection stats be exposed via Prometheus?   

### Assumptions

This design makes the following assumptions, which should be validated during implementation:

1. **Envoy as the data plane**: This design assumes kgateway uses Envoy; alternative proxies are not considered.

2. **Gateway API compliance**: The policy attachment model assumes kgateway follows Gateway API v1 conventions.

3. **Namespace isolation**: Policies are assumed to be namespace-scoped (not cluster-wide).

4. **xDS reconciliation**: The controller is assumed to have a working xDS integration for pushing Envoy configuration updates.

5. **User expertise**: Target users are assumed to be familiar with Kubernetes, Gateway API, and basic chaos engineering concepts.

---

## Summary

This design proposes adding **declarative, gateway-level fault injection** to kgateway by introducing a new Gateway Policy that leverages Envoy's native HTTP fault filter. The solution provides:

- **Low barrier to entry**: No code changes in applications; pure Kubernetes configuration
- **Production-ready foundation**: Built on Envoy's proven fault injection capabilities
- **Safe and controllable**: Percentage-based faults with clear namespace boundaries
- **Extensible architecture**: Aligns with Gateway API policy attachment patterns for future enhancements

**Next step**: Gather feedback from maintainers and the community to refine the API design before proceeding to implementation.
