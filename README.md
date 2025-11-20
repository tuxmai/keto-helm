# Kubernetes Helm Charts for ORY

![CI](https://github.com/ory/k8s/actions/workflows/ci.yaml/badge.svg)

This repository contains helm charts for Kubernetes. All charts are in
incubation phase and use is at your own risk.

Please go to [k8s.ory.sh/helm](https://k8s.ory.sh/helm/) for a list of helm
charts and their configuration options.

**NOTE**

> All charts present in this repository require Kubernetes 1.18+. Please refer
> to releases [0.18.0](https://github.com/ory/k8s/releases/tag/v0.18.0) and
> older for versions supporting older releases of Kubernetes.

## Development

You can test and develop charts locally using
[Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/).

To test a chart locally without applying it to kubernetes, do:

```sh
$ helm install --debug --dry-run <name> .
```

```sh
$ name=<name>
$ helm install $name .
$ helm upgrade $name .
```

### Ingress

If you wish to test ingress, run:

```bash
$ minikube addons enable ingress
```

Next you need to set up `/etc/hosts` to route traffic from domains - in this
example for ORY Oathkeeper:

- `api.oathkeeper.localhost`
- `proxy.oathkeeper.localhost`

to the ingress IP. You can find the ingress IP using:

```bash
$ kubectl get ingress
NAME                           HOSTS                        ADDRESS        PORTS     AGE
kilted-ibex-oathkeeper-api     api.oathkeeper.localhost     192.168.64.3   80        1d
kilted-ibex-oathkeeper-proxy   proxy.oathkeeper.localhost   192.168.64.3   80        1d
```

Then, append the following entries to your host file (`/etc/hosts`):

```bash
192.168.64.3    api.oathkeeper.localhost
192.168.64.3    proxy.oathkeeper.localhost
```

### Testing

To run helm test, do:

```sh
$ helm lint .
$ helm install <name> .
$ helm test <name>
```

### Remove all releases

To remove **all releases (only in test environments)**, do:

```sh
$ helm del $(helm ls --all --short) --purge
```

## Deployment with Helmfile

This repository includes a `helmfile.yaml` for easier deployment management, especially for the Keto chart.

### Prerequisites

- [Helmfile](https://helmfile.readthedocs.io/) installed
- Kubernetes cluster access configured

### Using Helmfile for Keto

The `helmfile.yaml` at the root of this repository provides environment variable-based configuration for deploying Keto.

#### Basic Usage

```bash
# Set required environment variables
export KETO_IMAGE_REPOSITORY=oryd/keto
export KETO_TAG=v25.4.0
export KETO_PULL_POLICY=IfNotPresent
export KETO_DSN=postgres://keto:keto@postgres-service:5432/keto?sslmode=require

# Optional: Override service type (default is ClusterIP for internal access)
# Only use LoadBalancer if you need external access and have proper network security
# export KETO_READ_SERVICE_TYPE=LoadBalancer
# export KETO_READ_LOAD_BALANCER_IP=your-read-ip-address
# export KETO_READ_EXTERNAL_TRAFFIC_POLICY=Local

# Deploy using helmfile
helmfile apply
```

#### Environment Variables

The helmfile supports the following environment variables:

**Image Configuration:**
- `KETO_IMAGE_REPOSITORY`: Container image repository (default: `oryd/keto`)
- `KETO_TAG`: Container image tag (default: `v25.4.0`)
- `KETO_PULL_POLICY`: Image pull policy (default: `IfNotPresent`)

**Service Configuration - Read Service:**
- `KETO_READ_SERVICE_TYPE`: Service type for read service (default: `ClusterIP` - internal access only, no auth protection)
- `KETO_READ_LOAD_BALANCER_IP`: LoadBalancer IP for read service (only used if type is LoadBalancer)
- `KETO_READ_EXTERNAL_TRAFFIC_POLICY`: External traffic policy (only used if type is LoadBalancer)

**Service Configuration - Write Service:**
- `KETO_WRITE_SERVICE_TYPE`: Service type for write service (default: `ClusterIP` - internal access only, no auth protection)
- `KETO_WRITE_LOAD_BALANCER_IP`: LoadBalancer IP for write service (only used if type is LoadBalancer)
- `KETO_WRITE_EXTERNAL_TRAFFIC_POLICY`: External traffic policy (only used if type is LoadBalancer)

**Note:** Keto services use `ClusterIP` by default for security, as Keto does not have built-in authentication. Services are accessible via k3s internal DNS:
- Read: `http://keto-read.keto.svc.cluster.local:80`
- Write: `http://keto-write.keto.svc.cluster.local:80`

**Database Configuration:**
- `KETO_DSN`: Database connection string (default: `memory`)

**Replica Configuration:**
- `KETO_REPLICA_COUNT`: Number of replicas (default: `2`)

**Autoscaling Configuration:**
- `KETO_AUTOSCALING_ENABLED`: Enable autoscaling (default: `true`)
- `KETO_AUTOSCALING_MIN_REPLICAS`: Minimum replicas (default: `2`)
- `KETO_AUTOSCALING_MAX_REPLICAS`: Maximum replicas (default: `10`)

**Metrics Configuration:**
- `KETO_METRICS_ENABLED`: Enable metrics service (default: `true`)

#### Production Values

Production-specific values are defined in `envs/prod/values.yaml`. These values are automatically loaded when using helmfile.

#### Using Helm Directly

You can still use Helm directly:

```bash
# Install with default values
helm install keto ./helm/charts/keto

# Install with production values
helm install keto ./helm/charts/keto -f envs/prod/values.yaml

# Install with custom values (using ClusterIP for internal access)
helm install keto ./helm/charts/keto \
  --set service.read.type=ClusterIP \
  --set service.write.type=ClusterIP

# Or if you need external access (not recommended without auth)
# helm install keto ./helm/charts/keto \
#   --set service.read.type=LoadBalancer \
#   --set service.read.loadBalancerIP=your-ip \
#   --set service.write.type=LoadBalancer \
#   --set service.write.loadBalancerIP=your-ip
```

#### Upgrading

```bash
# Using helmfile
helmfile apply

# Using helm
helm upgrade keto ./helm/charts/keto -f envs/prod/values.yaml
```

#### Uninstalling

```bash
# Using helmfile
helmfile destroy

# Using helm
helm uninstall keto
```
