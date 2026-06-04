# Kubernetes Deployment with Ansible

Automates deploying a **FastAPI + Redis** multi-container application on Kubernetes using Ansible roles, Jinja2 templates, ConfigMap-driven configuration, and rolling update automation.

---

## Project Structure

```
ansible-k8s-app/
├── site.yml                          # Main playbook entry point
├── inventory                         # Ansible inventory (localhost)
├── requirements.yml                  # Ansible collections (kubernetes.core)
├── group_vars/
│   └── all.yml                       # All configurable variables
└── roles/
    └── k8s_app/
        ├── defaults/main.yml         # Role default values
        ├── tasks/
        │   ├── main.yml              # Task orchestrator with tags
        │   ├── namespace.yml         # Create K8s namespace
        │   ├── configmap.yml         # ConfigMap deploy + rollout trigger
        │   ├── deploy.yml            # Deployment + Service with error handling
        │   ├── scale.yml             # Dynamic scaling
        │   └── validate.yml          # Pod readiness + assertion checks
        ├── templates/
        │   ├── namespace.yml.j2      # Namespace template
        │   ├── configmap.yml.j2      # ConfigMap template
        │   ├── deployment.yml.j2     # Multi-container Deployment template
        │   └── service.yml.j2        # Service template
        ├── handlers/main.yml
        ├── meta/main.yml
        └── tests/test.yml
```

---

## Prerequisites

- Python 3.9+
- Ansible 2.15+
- `kubernetes.core` collection
- A running Kubernetes cluster (Minikube, kind, or any managed cluster)
- `kubectl` configured with the target cluster context

### Install collection

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Variables

All variables are in [group_vars/all.yml](group_vars/all.yml) and can be overridden at runtime with `--extra-vars`.

| Variable | Default | Description |
|---|---|---|
| `namespace_name` | `demo-app` | Kubernetes namespace |
| `app_name` | `fastapi-demo` | Application and resource name |
| `image_name` | `tiangolo/uvicorn-gunicorn-fastapi:python3.11` | Container image |
| `replica_count` | `2` | Base replica count |
| `container_port` | `80` | App container port |
| `service_port` | `80` | Kubernetes service port |
| `service_type` | `ClusterIP` | K8s service type |
| `environment` | `prod` | App environment (affects dynamic scaling) |
| `scale_enabled` | `true` | Enable dynamic scaling |
| `scale_replicas` | `5` (prod) / `2` (other) | Target replicas when scaling |
| `redis_host` | `localhost` | Redis host injected via ConfigMap |
| `app_version` | `1.0.0` | App version injected via ConfigMap |
| `log_level` | `INFO` | Log level injected via ConfigMap |

---

## Usage

### Full deployment

```bash
ansible-playbook -i inventory site.yml
```

### Deploy only namespace

```bash
ansible-playbook -i inventory site.yml --tags namespace
```

### Deploy/update ConfigMap (triggers rolling restart if changed)

```bash
ansible-playbook -i inventory site.yml --tags configmap
```

### Deploy application

```bash
ansible-playbook -i inventory site.yml --tags deploy
```

### Scale deployment

```bash
ansible-playbook -i inventory site.yml --tags scale
```

### Validate pod status

```bash
ansible-playbook -i inventory site.yml --tags validate
```

### Override variables at runtime

```bash
# Scale to 10 replicas in prod
ansible-playbook -i inventory site.yml --extra-vars "scale_replicas=10 environment=prod"

# Deploy to staging with fewer replicas
ansible-playbook -i inventory site.yml --extra-vars "environment=staging scale_enabled=false replica_count=1"
```

---

## Key Design Decisions

### Multi-container pod
The deployment runs **FastAPI** (application) and **Redis** (cache) as a sidecar in the same pod. This satisfies the "more functional than plain NGINX" requirement and demonstrates multi-container pod configuration.

### ConfigMap-driven configuration
All runtime config (environment, Redis host, log level, app version) is stored in a Kubernetes ConfigMap and injected into the app container via `envFrom`. Updating the ConfigMap and re-running the playbook automatically triggers a rolling restart — only when something changed (idempotency).

### Idempotency
The `kubernetes.core.k8s` module is declarative — re-running the playbook when nothing changed produces zero `changed` tasks. The rolling restart in `configmap.yml` is guarded by `when: configmap_result.changed` so it only fires on actual config updates.

### Dynamic scaling
`scale_replicas` evaluates to `5` when `environment == 'prod'` and `2` otherwise. Combined with the `scale_enabled` flag, scaling can be toggled without editing variables.

### Error handling
The deploy block uses Ansible's `block/rescue/always` pattern — a failed deployment prints diagnostics and halts cleanly rather than leaving the system in an unknown state.

### Tags for selective execution
Every task group has a tag (`namespace`, `configmap`, `deploy`, `scale`, `validate`) enabling targeted runs during CI or debugging.

---

## Rolling Update Flow

1. Run playbook with updated `log_level` or `app_version` in `group_vars/all.yml`
2. `configmap.yml` applies the new ConfigMap — `configmap_result.changed` becomes `true`
3. `k8s_rollout_restart` triggers a rolling restart of the Deployment
4. Ansible polls `k8s_rollout_info` until `updatedReplicas == replicas`
5. Final `debug` task prints the rollout summary

---

## Checking Deployment (manual kubectl)

```bash
kubectl get pods -n demo-app
kubectl get configmap fastapi-demo-config -n demo-app -o yaml
kubectl rollout status deployment/fastapi-demo -n demo-app
kubectl describe deployment fastapi-demo -n demo-app
```
