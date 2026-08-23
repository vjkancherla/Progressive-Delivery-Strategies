# Progressive Delivery Strategies

Notes and examples for implementing **Progressive Delivery** (Canary and
Blue/Green deployments) in production Kubernetes environments, using Argo Rollouts
and Istio.

## What's inside

- `argo_rollouts_istio_guide.md` — guide to Argo Rollouts + Istio for progressive
  delivery.
- `from_git_commit_to_production.md` — end-to-end flow from a git commit to
  production promotion.
- `IGNORE-comprehensive_argo_istio_guide.md`, `TO-DELETE-*` — older/working guides
  kept for reference.
- `example/` — example application used in the demos.

## Tools covered

- Argo Rollouts (canary / blue-green)
- Istio for traffic management
- Git-driven promotion to production

## How to use

Read the guides to understand the strategy, then use `example/` as a reference for
the underlying manifests.
