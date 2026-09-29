# Monitoring environments with Kustomize

![Kubernetes](https://img.shields.io/badge/Kubernetes-Cluster-326CE5?logo=kubernetes&logoColor=white)
![Kustomize](https://img.shields.io/badge/Kustomize-Overlay%20Management-326CE5?logo=kustomize&logoColor=white)
![Traefik](https://img.shields.io/badge/Ingress-Traefik-24A1C1?logo=traefik&logoColor=white)
![Example](https://img.shields.io/badge/Repo-Example%20Monitoring-181717?logo=github&logoColor=white)


This repository is a sample Kubernetes monitoring stack built as a set of manifests organized with Kustomize overlays. The goal is to keep a common base configuration for the monitoring services and then create environment-specific customizations for local, dev, QA, and production without duplicating the whole stack.

The project contains the base manifests for:
- Prometheus
- Grafana
- Jaeger
- Tempo
- Loki
- OpenTelemetry Collector

Each environment reuses that shared base and only overrides the parts that need to change, such as the Ingress host and routing path.

## Repository structure

```text
.
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── prometheus/
│   ├── grafana/
│   ├── jaeger/
│   ├── tempo/
│   ├── loki/
│   └── otel-collector/
├── environments/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── qa/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── prod/
│       ├── kustomization.yaml
│       └── patches/
├── .gitlab-ci.yml
├── otel-common-config.yaml
└── README.md
```

### Base layer

The `base` directory contains the common YAML resources that are always part of the stack. For example, service deployments, services, ingresses, configs, volumes, and RBAC policies are defined here.

Each service folder includes a local `kustomization.yaml` that declares the resources for that service. Then the root `base/kustomization.yaml` groups all of them together.

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
resources:
  - ../../base/prometheus
  - ../../base/grafana
  - ../../base/jaeger
  - ../../base/tempo
  - ../../base/loki
  - ../../base/otel-collector
```

This is the common foundation that the environment overlays reuse.

## How the environment overlays work

Each environment under `environments/<env>/` is an overlay that imports the shared base resources and applies targeted changes through patches.

Example:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
resources:
  - ../../base/prometheus
  - ../../base/grafana
  - ../../base/jaeger
  - ../../base/tempo
  - ../../base/loki
  - ../../base/otel-collector
patches:
  - path: patches/prometheus-patch.yaml
  - path: patches/grafana-patch.yaml
  - path: patches/jaeger-patch.yaml
  - path: patches/tempo-patch.yaml
  - path: patches/loki-patch.yaml
  - path: patches/otel-collector-patch.yaml
```

This means:
- the base stack is loaded first,
- then each environment redefines only the fields that differ,
- the result is a single final manifest set for that environment.

This pattern avoids duplication and keeps the stack easy to evolve.

## Routing logic

The stack exposes each monitoring service through Traefik Ingress resources. The routing pattern differs depending on the environment.

### Local environment

In the base manifests, all services share the same host:

- `monitoring.localhost`

And each service is routed by path, for example:

- `/prometheus`
- `/grafana`
- `/jaeger`
- `/tempo`
- `/loki`

This pattern is ideal for local development because the same host can be used and the path differentiates the service.

Example:

```yaml
spec:
  rules:
    - host: monitoring.localhost
      http:
        paths:
          - path: /grafana
            backend:
              service:
                name: grafana-service
                port:
                  number: 3000
```

This logic means the same Ingress host serves multiple apps, and Traefik routes based on the URL path.

### Dev and QA

The dev and QA overlays keep the same routing model as local, but they switch the host name:

- Dev: `dev.localhost`
- QA: `qa.localhost`

Then the same paths are used:

- `https://dev.localhost/prometheus`
- `https://dev.localhost/grafana`
- `https://dev.localhost/jaeger`
- `https://dev.localhost/tempo`
- `https://dev.localhost/loki`

and

- `https://qa.localhost/prometheus`
- `https://qa.localhost/grafana`
- `https://qa.localhost/jaeger`
- `https://qa.localhost/tempo`
- `https://qa.localhost/loki`

So dev and QA share the same routing logic as local: same host family, same path-based segmentation, only the host changes.

### Production

Production does not use the shared `host + path` pattern. Instead, each service gets its own dedicated hostname:

- Prometheus: `prome.localhost`
- Grafana: `graf.localhost`
- Loki: `loki.localhost`
- Tempo: `tempo.localhost`
- Jaeger: `jaeger.localhost`

In prod, each patch overrides the source Ingress `host` and sets the path to the root (`""`), so the service is exposed directly on its own FQDN.

Examples:

```yaml
spec:
  rules:
    - host: graf.localhost
      http:
        paths:
          - path: ""
            backend:
              service:
                name: grafana-service
                port:
                  number: 3000
```

And:

```yaml
spec:
  rules:
    - host: prome.localhost
      http:
        paths:
          - path: ""
            backend:
              service:
                name: prometheus-service
                port:
                  number: 9090
```

This makes the production routing model simpler and cleaner for external access, because each service is a standalone endpoint instead of being grouped behind a single host and path prefix.

## How the final manifest is generated

The repository is designed to generate a single final set of manifests for each environment with Kustomize.

The CI job in `.gitlab-ci.yml` performs this:

```yaml
deploy_prod:
  stage: deploy
  image: line/kubectl-kustomize:latest
  script:
    - kubectl kustomize environments/prod > all-in-one.yaml
    - cat all-in-one.yaml
    - envsubst < all-in-one.yaml > applied-manifests.yaml
    - cat applied-manifests.yaml
```

This is the key flow:
1. Kustomize reads the overlay at `environments/prod`.
2. It resolves the base resources and applies the environment-specific patches.
3. It outputs a combined YAML file (`all-in-one.yaml`) containing every resource to be deployed.
4. `envsubst` can replace environment variables before applying the manifest.

So the final manifest is not written manually; it is synthesized from the base + overlay + patches.

## How Kustomize patches work

Kustomize patches are applied by matching a resource that is already part of the build.

The pattern used here is:
- include the base resource in `resources`,
- identify the same resource by `apiVersion`, `kind`, and `metadata.name`,
- apply the patch to override fields.

For example, the Prometheus Ingress in the base is named `prometheus-ingress`. The overlay patch file for `environments/dev/patches/prometheus-patch.yaml` contains the same resource identity:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  rules:
    - host: dev.localhost
      http:
        paths:
          - path: /prometheus
```

Because the overlay `kustomization.yaml` includes the base Prometheus resources and the patch file is declared in the same `patches` list, Kustomize matches the target resource and updates it.

This is the important rule:
- the patch must target a resource that is part of the same Kustomize build,
- and it must be declared in the same `kustomization.yaml` that includes the base resources.

If the patch is not part of that same build context, or if the resource is not matched by its identity, the patch will not be applied.

In other words, this works because the patch is not a standalone file; it is a transformation of an already-defined resource inside the same Kustomize graph.

## Example of the local/overlay model

The repo follows a very practical pattern:

- `base/` holds all the common manifests.
- `environments/<env>/` holds the differences for a given environment.
- `patches/*.yaml` redefine only the necessary fields, such as `host` or `path`.
- `kustomization.yaml` is the entry point that assembles everything.

This is a good example of how to structure a Kubernetes deployment repository for multiple environments without duplicating YAML files.

## Summary

This repository demonstrates a clean way to manage multiple deployment environments with Kustomize:

- local: one shared host with path-based routing,
- dev and QA: same model, different hostname,
- production: one host per service, root route per app,
- patches: environment-specific customization applied to the same resource identity inside the same Kustomize overlay.

This keeps the stack maintainable, reusable, and easy to deploy across different environments while preserving a single source of truth for the base monitoring services.
