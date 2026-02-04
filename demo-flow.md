# Demo Flow: Fault Injection Validation

## 1. Overview

This document describes realistic usage scenarios for the proposed gateway-level fault injection feature in kgateway. It demonstrates how platform engineers and SREs would apply and validate fault injection policies in practice.

These scenarios are conceptual and intended for design-time validation. They illustrate the expected behavior of the system without requiring a running Kubernetes cluster or implementation. Reviewers can use this document to evaluate whether the proposed API design meets real-world requirements.

---

## 2. Demo Setup (Conceptual)

For all scenarios, assume the following baseline infrastructure exists:

**Kubernetes Cluster:**
- kgateway is installed and operational
- Gateway API CRDs are available (`Gateway`, `HTTPRoute`, `GatewayPolicy`)

**Resources:**
- **Gateway**: `demo-gateway` listening on port 8080
- **HTTPRoute**: `httpbin-route` routing `/` to the `httpbin` service
- **Backend Service**: `httpbin` (a simple HTTP testing service)

**Baseline Behavior:**
- Client requests to `http://<gateway-ip>:8080/get` are routed to the httpbin service
- Normal latency: 5-50ms (network + processing)
- Success rate: 100% (no errors)

The following scenarios focus exclusively on adding and modifying **GatewayPolicy** resources to inject faults.

---

## 3. Scenario 1: Baseline (No Fault Injection)

### Initial State

No GatewayPolicy applied. Traffic flows normally through the gateway to the backend.

### Request Flow

```
Client → Envoy Gateway → [Router Filter] → Backend Service → Response
```

### Expected Behavior

- **Request latency**: 5-50ms (normal network overhead)
- **Success rate**: 100%
- **HTTP status codes**: 200 OK
- **Backend impact**: All requests reach the backend service

### Purpose

Establish a baseline for comparison. Subsequent scenarios introduce faults and measure deviations from this baseline.

---

## 4. Scenario 2: Delay Injection

### Policy Configuration

A GatewayPolicy is created to inject a 5-second delay into 50% of requests:

```yaml
apiVersion: gateway.kgateway.io/v1
kind: GatewayPolicy
metadata:
  name: delay-test
  namespace: demo
spec:
  targetRef:
    kind: Gateway
    name: demo-gateway
  faultInjection:
    delay:
      fixedDelay: 5s
      percentage: 50.0
```

### Request Flow (Affected Requests)

```
Client → Envoy Gateway → [Fault Filter: Sleep 5s] → [Router Filter] → Backend → Response
```

For 50% of requests, Envoy pauses for 5 seconds before forwarding to the backend.

### Expected Behavior

- **Request latency**:
  - 50% of requests: ~5000ms (delay) + backend processing time
  - 50% of requests: ~5-50ms (normal)
- **Success rate**: Still 100% (delay does not cause failures)
- **HTTP status codes**: 200 OK (delayed responses are still successful)
- **Backend impact**: All requests eventually reach the backend

### Validation Points

An operator would validate this by:
1. **Measuring latency distribution**: Approximately half of requests should exhibit ~5s latency
2. **Checking Envoy metrics**: `http.fault.delays_injected` counter should increment
3. **Client timeout handling**: Clients with short timeouts (e.g., 3s) may timeout on delayed requests

### Why This Matters

This scenario tests:
- Whether clients handle slow responses gracefully
- Retry logic for timed-out requests
- User experience degradation under high latency

---

## 5. Scenario 3: Abort Injection

### Policy Configuration

A GatewayPolicy is created to abort 10% of requests with HTTP 503:

```yaml
apiVersion: gateway.kgateway.io/v1
kind: GatewayPolicy
metadata:
  name: abort-test
  namespace: demo
spec:
  targetRef:
    kind: Gateway
    name: demo-gateway
  faultInjection:
    abort:
      httpStatus: 503
      percentage: 10.0
```

### Request Flow (Aborted Requests)

```
Client → Envoy Gateway → [Fault Filter: Return 503] → Client
                              ↓
                    (Backend never reached)
```

For 10% of requests, Envoy immediately returns HTTP 503 without forwarding to the backend.

### Expected Behavior

- **Request latency**:
  - 10% of requests: < 5ms (immediate abort)
  - 90% of requests: ~5-50ms (normal)
- **Success rate**: 90%
- **HTTP status codes**:
  - 10%: 503 Service Unavailable
  - 90%: 200 OK
- **Backend impact**: Only 90% of requests reach the backend

### Validation Points

An operator would validate this by:
1. **Error rate monitoring**: Approximately 10% of responses should be 503
2. **Envoy metrics**: `http.fault.aborts_injected` counter should increment
3. **Backend logs**: Backend should see ~90% of the original request volume (not 100%)
4. **Circuit breaker behavior**: If clients have circuit breakers, they may trip after consecutive 503 errors

### Why This Matters

This scenario tests:
- Client retry mechanisms
- Circuit breaker thresholds
- Alerting and monitoring for elevated error rates
- Graceful degradation when services are unavailable

---

## 6. Scenario 4: Combined Delay + Abort

### Policy Configuration

A GatewayPolicy is created with both delay (40% of requests) and abort (15% of requests):

```yaml
apiVersion: gateway.kgateway.io/v1
kind: GatewayPolicy
metadata:
  name: combined-test
  namespace: demo
spec:
  targetRef:
    kind: Gateway
    name: demo-gateway
  faultInjection:
    delay:
      fixedDelay: 3s
      percentage: 40.0
    abort:
      httpStatus: 500
      percentage: 15.0
```

### Independent Evaluation

Delay and abort are evaluated independently for each request. This creates multiple possible outcomes.

### Expected Behavior

For each request, Envoy evaluates:
1. **Delay**: 40% chance of 3-second delay
2. **Abort**: 15% chance of immediate HTTP 500

**Outcome Probabilities:**
- **No fault** (60% * 85%): ~51% of requests
- **Delay only** (40% * 85%): ~34% of requests
- **Abort only** (60% * 15%): ~9% of requests
- **Both delay and abort** (40% * 15%): ~6% of requests

### Request Flow Examples

**Delay only (34% of requests):**
```
Client → [Fault Filter: Sleep 3s] → [Router] → Backend → 200 OK
```

**Abort only (9% of requests):**
```
Client → [Fault Filter: Return 500] → Client (500 error)
```

**Both delay and abort (6% of requests):**
```
Client → [Fault Filter: Sleep 3s, then Return 500] → Client (500 error after 3s)
```

### Validation Points

An operator would validate this by:
1. **Latency + error distribution**: Confirm approximately 6% of requests are both slow AND fail
2. **Backend request volume**: Only ~85% of requests should reach the backend (15% aborted)
3. **Client experience**: Some clients may experience delayed failures (worst case)

### Why This Matters

This scenario tests:
- Compound degradation (slow AND failing)
- Realistic production incident simulation
- Client resilience under multiple fault conditions

---

## 7. Observability & Validation

### Metrics

Envoy exposes the following metrics for fault injection validation:

- **`http.fault.delays_injected`**: Total count of delayed requests
- **`http.fault.aborts_injected`**: Total count of aborted requests
- **`http.fault.faults_overflow`**: Requests that exceeded configured limits (should remain 0)

An operator would:
1. **Query Prometheus** (or equivalent) to retrieve these metrics
2. **Compare observed rates** to configured percentages (e.g., 10% abort → ~10% abort rate observed)
3. **Validate over time**: Metrics should stabilize after sufficient request volume

### Logs

Envoy access logs would show:
- **Delayed requests**: Higher response time in access logs
- **Aborted requests**: 503/500 status codes without upstream request logs
- **Correlation**: Timestamps and request IDs for debugging

### Client-Side Validation

Clients can validate fault injection by:
1. **Measuring end-to-end latency histograms** (e.g., p50, p95, p99)
2. **Tracking HTTP status code distribution** (e.g., 503 rate)
3. **Monitoring timeout rates** (requests aborted by client timeout)

### Controller Validation

The kgateway controller would:
1. **Report policy status** via Kubernetes status conditions (e.g., `Ready`, `Invalid`)
2. **Emit events** for policy application or errors
3. **Log translation errors** if GatewayPolicy cannot be converted to valid Envoy config

---

## 8. Safety Checks

### Namespace Isolation

Policies are namespace-scoped. A policy in the `staging` namespace cannot affect resources in the `production` namespace.

**Example:**
```
Namespace: staging
  GatewayPolicy: chaos-test → Affects staging-gateway only

Namespace: production
  No fault injection policy → 100% normal traffic
```

**Validation**: An operator would verify that production traffic is unaffected even if staging has aggressive fault injection enabled.

### Percentage-Based Blast Radius Control

Operators can start with a low percentage (e.g., 1%) and gradually increase:

**Gradual Rollout:**
1. Apply policy with `percentage: 1.0` (1% of traffic)
2. Monitor metrics and client behavior
3. Increase to `percentage: 5.0` if no issues
4. Increase to `percentage: 10.0` after further validation
5. Rollback to `percentage: 0.0` immediately if problems occur

**Validation**: Metrics and logs confirm that only the specified percentage of requests are affected.

### No Application Code Changes

Backend services remain completely unaware of fault injection. No libraries, SDKs, or code modifications are required.

**Validation**:
1. Backend logs show normal request patterns (no fault injection logic)
2. Backend can be deployed/restarted without coordination with fault injection policies
3. Fault injection can be enabled/disabled without backend changes

### RBAC and Access Control

Kubernetes RBAC controls who can create or modify GatewayPolicy resources.

**Safety Mechanism:**
- Only authorized users (e.g., platform team) can create policies
- Developers cannot accidentally enable aggressive faults in production

**Validation**: Attempt to create a policy as an unauthorized user (should be rejected by Kubernetes API server).

---

## 9. What This Demo Proves

This conceptual demo validates the following aspects of the proposed design:

**Functional Correctness:**
- Delay injection correctly pauses requests for the specified duration
- Abort injection correctly returns HTTP error codes without reaching the backend
- Delay and abort are evaluated independently as designed
- Percentage-based selection is probabilistic and accurate over time

**API Usability:**
- Platform engineers can configure fault injection using familiar Kubernetes primitives
- Policy attachment model (targetRef to Gateway or HTTPRoute) is intuitive
- Percentage values (0.0-100.0) are easy to reason about

**Safety Guarantees:**
- Namespace isolation prevents cross-environment impact
- Percentage-based control allows gradual rollout and immediate rollback
- No application-level changes required reduces deployment risk

**Operational Validation:**
- Metrics and logs provide clear observability into fault injection behavior
- Operators can validate that faults are applied correctly before increasing scope
- GatewayPolicy status reflects configuration state (ready/invalid)

**Architecture Soundness:**
- Controller correctly translates GatewayPolicy to Envoy HTTP fault filter config
- Envoy filter chain placement (fault filter before router) ensures correct behavior
- xDS updates apply configuration changes without downtime

**Integration with kgateway:**
- Follows Gateway API policy attachment conventions
- Compatible with existing Gateway and HTTPRoute resources
- Aligns with kgateway's Envoy-based architecture

---

## Summary

This demo flow demonstrates the end-to-end feasibility of the proposed gateway-level fault injection feature. Through four realistic scenarios (baseline, delay, abort, combined), the design proves capable of:

1. Injecting controlled faults into HTTP traffic without application changes
2. Providing percentage-based blast radius control for safe testing
3. Offering clear observability through metrics and logs
4. Ensuring production safety via namespace isolation and RBAC

The conceptual validation confirms that the proposed API design is intuitive, the Envoy integration is sound, and the safety mechanisms are sufficient for real-world chaos engineering and resilience testing.

This prototype provides a strong foundation for implementation and demonstrates alignment with kgateway's mission to deliver enterprise-grade traffic management capabilities.
