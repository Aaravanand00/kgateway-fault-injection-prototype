# kgateway Fault Injection Prototype

> **Note**: This is a design prototype for proposal and discussion purposes. It contains comprehensive documentation but no production-ready implementation.

## Overview

This prototype proposes adding declarative fault injection capabilities to [kgateway](https://github.com/kgateway-dev/kgateway), enabling users to test service resilience through Gateway API policies. By leveraging Envoy's HTTP fault filters, this feature would allow platform engineers and SREs to inject controlled failures (delays, aborts) into traffic flows without modifying application code.

## Problem Statement

Currently, kgateway users lack a standardized way to:
- Test service behavior under degraded network conditions
- Validate retry logic and circuit breakers
- Perform controlled chaos testing in staging environments
- Build confidence in production failure handling

Existing solutions require external tools, manual service modifications, or complex infrastructure changes.

## What This Prototype Includes

This repository contains a complete design specification organized as follows:

| Document | Purpose |
|----------|---------|
| **[design.md](./design.md)** | Problem analysis, proposed solution, and trade-offs |
| **[architecture.md](./architecture.md)** | System flow, component diagrams, and integration points |
| **[api-prototype.yaml](./api-prototype.yaml)** | User-facing API examples (7 realistic scenarios) |
| **[envoy-mapping.md](./envoy-mapping.md)** | Translation from Gateway Policy to Envoy filter configuration |
| **[demo-flow.md](./demo-flow.md)** | End-to-end validation scenarios and testing methodology |

## Intentionally Out of Scope

To maintain focus, this prototype **does not include**:

- ❌ Actual Go implementation or controller code
- ❌ Working Kubernetes cluster or deployment manifests
- ❌ Production-ready CRD definitions
- ❌ Unit tests, integration tests, or CI pipelines
- ❌ Performance benchmarks or real-world metrics

This is a **design artifact** intended to gather feedback before implementation begins.

## How to Review This Prototype

**Recommended reading order** for reviewers:

1. **Start here** → `README.md` (you are here)
2. **Understand the why** → [`design.md`](./design.md) - Read problem statement and proposed solution
3. **See the API** → [`api-prototype.yaml`](./api-prototype.yaml) - Browse 2-3 examples to understand user experience
4. **Grasp the architecture** → [`architecture.md`](./architecture.md) - Skim component diagrams and flow
5. **Dive into details** _(optional)_ → [`envoy-mapping.md`](./envoy-mapping.md) - For implementation-level questions
6. **Validate completeness** _(optional)_ → [`demo-flow.md`](./demo-flow.md) - Check if testing strategy is sound

**Estimated review time**: 15-20 minutes for core documents (steps 1-4)

## Key Design Decisions

1. **Policy attachment**: Uses Gateway API `targetRef` to attach to `Gateway` or `HTTPRoute` resources
2. **Envoy integration**: Leverages existing `envoy.filters.http.fault` filter (proven, stable)
3. **Percentage-based faults**: Supports probabilistic fault injection (e.g., "fail 10% of requests")
4. **Separation of concerns**: User-facing API is clean; Envoy complexity hidden in controller translation

## Alignment with kgateway

This proposal aligns with kgateway's mission to provide:
- ✅ Enterprise-grade traffic management
- ✅ Kubernetes-native configuration (CRDs)
- ✅ Envoy-based data plane capabilities
- ✅ Production reliability features

## Feedback Welcome

This prototype is intended to start a conversation. Key questions for reviewers:

- Is the API intuitive for the target users (platform engineers, SREs)?
- Are there missing use cases or edge cases?
- Does the Envoy integration strategy make sense?
- Should this be a core feature or a plugin/extension?

## Next Steps

If this proposal is accepted:
1. Formalize CRD schema definitions
2. Implement controller reconciliation logic
3. Add unit and integration tests
4. Create E2E validation suite
5. Document upgrade/migration path

## Author

Aarav Anand
Proposed as part of LFX Mentorship Program (CNCF Project / Kgateway)
Feb 2026
