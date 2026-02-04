# Envoy Configuration Mapping

## Purpose of This Document

This document describes the deterministic translation from the GatewayPolicy fault injection API to Envoy's HTTP fault filter configuration. It is intended for kgateway maintainers and implementers to ensure consistent behavior and provide clarity for future development.

The mapping is straightforward: user-facing GatewayPolicy fields are translated directly into Envoy's `envoy.extensions.filters.http.fault.v3.HTTPFault` configuration. The controller performs this translation and applies the resulting configuration to Envoy via xDS.

---

## GatewayPolicy → Envoy Mapping Table

| GatewayPolicy Field | Envoy Field | Notes |
|---------------------|-------------|-------|
| `faultInjection.delay.fixedDelay` | `delay.fixed_delay` | Duration string (e.g., "5s") converted to Envoy duration proto. Must be a valid duration format. |
| `faultInjection.delay.percentage` | `delay.percentage.numerator` | Float (0.0-100.0) converted to integer numerator. Denominator is always `HUNDRED`. |
| `faultInjection.abort.httpStatus` | `abort.http_status` | Integer HTTP status code. Must be in range 200-599. |
| `faultInjection.abort.percentage` | `abort.percentage.numerator` | Float (0.0-100.0) converted to integer numerator. Denominator is always `HUNDRED`. |

**Key Translation Rules:**

1. **Percentage Conversion**: User provides float (e.g., `50.0`), Envoy expects `{numerator: 50, denominator: HUNDRED}`.
2. **Duration Format**: User provides duration string (e.g., `"5s"`), Envoy expects duration proto (e.g., `"5s"` remains `"5s"`).
3. **Both Fields Optional**: `delay` and `abort` are independently optional. At least one must be specified.
4. **Independent Evaluation**: Delay and abort are evaluated independently per request (not mutually exclusive).

---

## Example Translation

### Input: GatewayPolicy

```yaml
apiVersion: gateway.kgateway.io/v1
kind: GatewayPolicy
metadata:
  name: example-fault
  namespace: demo
spec:
  targetRef:
    kind: Gateway
    name: my-gateway
  faultInjection:
    delay:
      fixedDelay: 5s
      percentage: 50.0
    abort:
      httpStatus: 503
      percentage: 10.0
```

### Output: Envoy HTTP Fault Filter Configuration

```json
{
  "name": "envoy.filters.http.fault",
  "typed_config": {
    "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault",
    "delay": {
      "fixed_delay": "5s",
      "percentage": {
        "numerator": 50,
        "denominator": "HUNDRED"
      }
    },
    "abort": {
      "http_status": 503,
      "percentage": {
        "numerator": 10,
        "denominator": "HUNDRED"
      }
    }
  }
}
```

**Result**: For each incoming request:
- 50% probability of 5-second delay
- 10% probability of immediate 503 response
- Probabilities are independent (some requests may experience both delay and abort)

---

## Validation and Edge Case Handling

### Independent Evaluation

Delay and abort are evaluated independently for each request. This means:
- A request can be delayed without being aborted
- A request can be aborted without being delayed
- A request can be both delayed and then aborted
- A request may experience neither fault

**Example**: With 50% delay and 10% abort:
- Approximately 45% of requests: No fault
- Approximately 45% of requests: Delay only
- Approximately 5% of requests: Abort only
- Approximately 5% of requests: Both delay and abort

### Percentage Bounds

The controller must validate that percentage values fall within the range `[0.0, 100.0]`:
- `percentage: 0.0` → No fault injection (effectively disabled)
- `percentage: 100.0` → All requests affected
- Values outside this range are rejected during policy validation

### HTTP Status Code Validation

he `httpStatTus` field must be a valid HTTP status code in the range `[200, 599]`:
- Common values: 500, 503, 502, 429
- Invalid values (e.g., 999) are rejected during policy validation

### Duration Format

The `fixedDelay` field must be a valid duration string:
- Valid examples: `"1s"`, `"500ms"`, `"2m"`, `"1h"`
- Invalid examples: `"abc"`, `"5"`, `""` (rejected during validation)

### Unsupported Fields

Any fields in the GatewayPolicy that are not explicitly mapped to Envoy configuration are ignored. This ensures forward compatibility if the API is extended in the future.

### Filter Chain Placement

The generated HTTP fault filter is placed **before** the router filter in Envoy's filter chain:

```
http_filters:
  - envoy.filters.http.fault    # ← Generated from GatewayPolicy
  - envoy.filters.http.router   # ← Must be last
```

This ordering ensures:
1. Faults are injected before routing decisions
2. Aborted requests never reach the backend
3. Delays are observed by the client as part of total request latency

---

## Summary

The mapping from GatewayPolicy to Envoy configuration is intentionally simple and deterministic:

- **User API**: Clean, Kubernetes-native configuration with percentage values as floats and durations as strings
- **Envoy Configuration**: Controller translates to Envoy's native HTTP fault filter format
- **Complexity Isolation**: All translation logic resides in the controller; users and backend services remain unaware of Envoy-specific details
- **Zero Application Impact**: Backend services require no code changes or library integrations

This design ensures that platform engineers can configure fault injection through familiar Kubernetes primitives while maintaining full compatibility with Envoy's proven fault injection capabilities.
