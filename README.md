# OCI DevOps + CloudOps Sample Project

This repository is a **complete sample project** you can run from **OCI Cloud Shell** to demonstrate a full DevOps + CloudOps workflow on Oracle Cloud Infrastructure (OCI).

It includes:

* Terraform to provision networking, OKE cluster, OCIR, and a backend bucket for Terraform state.
* Dockerfile and a simple Node.js sample app.
* Kubernetes manifests for deployment + service + ingress (NGINX).
* OCI DevOps build spec and deploy pipeline example.
* README with step-by-step Cloud Shell commands.

---

## Folder structure

```
oci-devops-sample/
├── README.md
├── terraform/
│   ├── provider.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── networking.tf
│   ├── oke.tf
│   ├── ocir.tf
│   └── backend.tf
├── app/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── ci/
│   ├── build_spec.yaml
│   └── deploy_spec.yaml
└── scripts/
    ├── push-to-ocir.sh
    └── kubeconfig-oke.sh
```

---

> **Important:** This sample uses **placeholders** for OCIDs, tenancy namespace, and region. Replace values with your own before running.

---

## 1) README.md (usage summary)

````markdown
# OCI DevOps + CloudOps Sample Project

## Prereqs
- OCI account and user with admin or appropriate rights
- OCI Cloud Shell (recommended)
- Docker (Cloud Shell has Docker available in most sessions)
- Terraform v1.4+ (included in Cloud Shell)

## High level steps
1. Configure OCI CLI and confirm `oci` works in Cloud Shell.
2. Edit `terraform/variables.tf` values or provide via `terraform.tfvars`.
3. Run `terraform init && terraform apply` to provision VCN, OKE, OCIR, and backend bucket.
4. Build and push Docker image to OCIR (script included).
5. Apply k8s manifests to OKE.
6. Configure OCI DevOps build & deploy pipelines (or use provided spec files for OCI DevOps).

## Quick Cloud Shell commands (examples)
```bash
# 1. clone
git clone <this-repo> && cd oci-devops-sample

# 2. terraform
cd terraform
terraform init
terraform apply -auto-approve

# 3. set kubeconfig
source ../scripts/kubeconfig-oke.sh

# 4. build and push image
cd ../app
./../scripts/push-to-ocir.sh

# 5. deploy to cluster
kubectl apply -f ../k8s
````

```
```

---

## 2) Terraform (in `terraform/`)

### provider.tf

```hcl
terraform {
  required_providers {
    oci = {
      source  = "hashicorp/oci"
      version = ">= 4.0.0"
    }
  }
}

provider "oci" {
  # Cloud Shell uses the default profile; override with environment variables if needed
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = var.private_key_path
  region           = var.region
}
```

### variables.tf

```hcl
variable "tenancy_ocid" {}
variable "user_ocid" {}
variable "fingerprint" {}
variable "private_key_path" { default = "~/.oci/oci_api_key.pem" }
variable "region" { default = "us-ashburn-1" }
variable "compartment_ocid" {}
variable "ssh_public_key" {}
variable "vcn_cidr" { default = "10.0.0.0/16" }
```

### backend.tf

```hcl
terraform {
  backend "oci" {
    bucket    = var.terraform_state_bucket
    namespace = var.tenancy_namespace
    region    = var.region
    key       = "oci-devops-sample/terraform.tfstate"
  }
}
```

### networking.tf (VCN, subnets, IGW, NAT)

```hcl
resource "oci_core_virtual_network" "vcn" {
  compartment_id = var.compartment_ocid
  cidr_block     = var.vcn_cidr
  display_name   = "devops-vcn"
}

resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_ocid
  display_name   = "devops-igw"
  vcn_id         = oci_core_virtual_network.vcn.id
}

resource "oci_core_subnet" "public_subnet" {
  compartment_id      = var.compartment_ocid
  vcn_id              = oci_core_virtual_network.vcn.id
  cidr_block          = "10.0.1.0/24"
  display_name        = "public-subnet"
  route_table_id      = oci_core_route_table.default.id
  security_list_ids   = [oci_core_security_list.public.id]
  dns_label           = "public"
}

resource "oci_core_route_table" "default" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_virtual_network.vcn.id
  route_rules {
    network_entity_id = oci_core_internet_gateway.igw.id
    destination       = "0.0.0.0/0"
    description       = "internet"
  }
}

resource "oci_core_security_list" "public" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_virtual_network.vcn.id
  ingress_security_rules {
    protocol = "6" # TCP
    source   = "0.0.0.0/0"
    tcp_options {
      "min" = 22
      "max" = 22
    }
  }
  egress_security_rules {
    protocol = "all"
    destination = "0.0.0.0/0"
  }
}
```

### ocir.tf (create OCIR namespace repo is implicit — we will use tenancy namespace variable)

```hcl
# no explicit "create namespace" needed; OCIR is available for tenancy. Optionally create an artifact repository via OCI Registry resources in newer providers.
```

### oke.tf (OKE cluster)

```hcl
resource "oci_containerengine_cluster" "oke_cluster" {
  compartment_id = var.compartment_ocid
  name           = "devops-oke"
  vcn_id         = oci_core_virtual_network.vcn.id
  kubernetes_version = "v1.27.3" # change to supported version in your region
}

resource "oci_containerengine_node_pool" "np" {
  compartment_id = var.compartment_ocid
  cluster_id     = oci_containerengine_cluster.oke_cluster.id
  name           = "devops-nodepool"
  kubernetes_version = oci_containerengine_cluster.oke_cluster.kubernetes_version
  node_config_details {
    size = 3
    placement_configs {
      subnet_id = oci_core_subnet.public_subnet.id
    }
    shape = "VM.Standard2.1"
    ssh_public_key = var.ssh_public_key
  }
}
```

### outputs.tf

```hcl
output "oke_cluster_id" {
  value = oci_containerengine_cluster.oke_cluster.id
}
output "vcn_id" {
  value = oci_core_virtual_network.vcn.id
}
```

---

## 3) Sample App (in `app/`)

### package.json

```json
{
  "name": "oci-devops-sample-app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

### index.js

```js
const express = require('express')
const app = express()
const port = process.env.PORT || 3000

app.get('/', (req, res) => {
  res.send('Hello from OCI DevOps sample app!')
})

app.listen(port, () => console.log(`App listening on ${port}`))
```

### Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package.json .
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

---

## 4) Kubernetes Manifests (in `k8s/`)

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-sample
```

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: devops-sample
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: <region-key>.ocir.io/<tenancy-namespace>/sample-app:latest
        ports:
        - containerPort: 3000
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-svc
  namespace: devops-sample
spec:
  type: LoadBalancer
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 3000
```

### ingress.yaml (optional: if you have ingress controller)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app-ingress
  namespace: devops-sample
annotations:
  kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sample-app-svc
            port:
              number: 80
```

---

## 5) CI/CD pipeline specs (for OCI DevOps)

### ci/build_spec.yaml (OCI DevOps build)

```yaml
version: 0.1
phases:
  pre_build:
    commands:
      - echo Logging in to OCIR
      - echo $OCIR_PASS | docker login -u $OCIR_USER --password-stdin $OCIR_HOST
  build:
    commands:
      - docker build -t sample-app:latest .
      - docker tag sample-app:latest $OCIR_HOST/$OCIR_NAMESPACE/sample-app:latest
      - docker push $OCIR_HOST/$OCIR_NAMESPACE/sample-app:latest
artifacts:
  files:
    - '**/*'
```

> In OCI DevOps, you use build pipeline with environment variables for `OCIR_HOST` (like `phx.ocir.io` or `<region-key>.ocir.io`), `OCIR_NAMESPACE` (tenancy namespace) and credentials.

### ci/deploy_spec.yaml (deploy via kubectl)

```yaml
version: 0.1
phases:
  build:
    commands:
      - echo "Fetching kubeconfig"
      - oci ce cluster create-kubeconfig --cluster-id $OKE_CLUSTER_ID --file $KUBECONFIG_PATH --region $OCI_REGION --token-version 2.0.0
      - export KUBECONFIG=$KUBECONFIG_PATH
      - kubectl apply -f k8s/namespace.yaml
      - kubectl apply -f k8s/deployment.yaml
      - kubectl apply -f k8s/service.yaml
```

---

## 6) Helper scripts (in `scripts/`)

### scripts/push-to-ocir.sh

```bash
#!/bin/bash
set -e

# replace these with your values or export as env vars before running
OCIR_HOST=${OCIR_HOST:-"<region-key>.ocir.io"}
OCIR_NAMESPACE=${OCIR_NAMESPACE:-"<tenancy-namespace>"}
IMAGE_NAME=sample-app
TAG=latest

# build
docker build -t ${IMAGE_NAME}:${TAG} .
# tag
docker tag ${IMAGE_NAME}:${TAG} ${OCIR_HOST}/${OCIR_NAMESPACE}/${IMAGE_NAME}:${TAG}
# login (Cloud Shell can use oci os or oci artifacts commands to get auth token; using docker login with auth token)
# You can create an auth token in OCI Console > User > Auth Tokens and export it as OCIR_PASS
if [ -z "$OCIR_PASS" ]; then
  echo "Please set OCIR_PASS environment variable with an auth token"
  exit 1
fi

echo $OCIR_PASS | docker login -u $OCIR_USER --password-stdin $OCIR_HOST

# push
docker push ${OCIR_HOST}/${OCIR_NAMESPACE}/${IMAGE_NAME}:${TAG}

echo "Pushed ${OCIR_HOST}/${OCIR_NAMESPACE}/${IMAGE_NAME}:${TAG}"
```

### scripts/kubeconfig-oke.sh

```bash
#!/bin/bash
set -e
# Requires: oci cli configured and terraform outputs present
CLUSTER_ID=$(terraform output -raw oke_cluster_id)
KUBECONFIG_PATH=${KUBECONFIG_PATH:-"$HOME/.kube/config"}
OCI_REGION=${OCI_REGION:-$(terraform output -raw region)}

oci ce cluster create-kubeconfig --cluster-id $CLUSTER_ID --file $KUBECONFIG_PATH --region $OCI_REGION --token-version 2.0.0

export KUBECONFIG=$KUBECONFIG_PATH
kubectl get nodes
```

---

## 7) Notes & best practices

* **State management:** Use the OCI Object Storage backend for Terraform state, enable versioning on the bucket and lock state using workarounds (Terraform Cloud or a locking mechanism) if multiple operators.
* **Secrets:** Use OCI Vault for storing OCIR tokens, DB credentials, etc. Do not store secrets in plaintext or code.
* **CI/CD best practices:** Use ephemeral build agents, sign images, scan images (Clair, Trivy) and enforce policy gates before deploy.
* **Zero-downtime deploys:** Use Kubernetes rolling updates and readiness/liveness probes.
* **Observability:** Install Prometheus + Grafana or forward metrics to OCI Monitoring. Collect logs with Fluentd/OCI Logging.

---

## 8) Next steps & customizations

1. Add Helm charts and use OCI Helm repo or Artifact Registry.
2. Add image scanning stage to CI.
3. Add blue/green or canary deploy stage (Argo Rollouts, Flagger).
4. Hook up OCI Monitoring alarms to send to PagerDuty/Slack via OCI Notifications.

---

*End of sample project.*
