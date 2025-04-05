# Disconnected OpenShift Installation (No Internet Access)

## ᵀ Introduction
This guide explains how to install OpenShift 4.x in a **disconnected environment** (i.e., an environment with no direct internet access). This type of installation is also known as an **air-gapped** or **restricted network installation**.

In a disconnected setup:
- We mirror required container images on a system **with internet**.
- We **copy** those images to a **local registry** in the disconnected environment.
- OpenShift pulls everything from this **internal registry**.

---

## ✅ Prerequisites

### On Internet-Connected System:
- Linux server with internet
- Tools Installed:
  - `oc` CLI
  - `openshift-install`
  - `podman` or `docker`
  - `skopeo`
  - `htpasswd`, `jq`
- Pull Secret from [Red Hat](https://cloud.redhat.com/openshift/install/pull-secret)

### On Disconnected Environment:
- At least 3 Master (Control plane) nodes
- At least 2 Worker nodes
- DNS and DHCP configured properly
- Access to local registry

---

## ♻ Step-by-Step Setup

### 🔄 Step 1: Create Local Registry (on Internet System)
```bash
podman run -d -p 5000:5000 --name registry registry:2
```

### 🌐 Step 2: Mirror OpenShift Images
```bash
oc adm release mirror \
  --from=quay.io/openshift-release-dev/ocp-release:4.14.0-x86_64 \
  --to=registry.local:5000/ocp4/openshift4 \
  --to-release-image=registry.local:5000/ocp4/openshift4:4.14.0 \
  --insecure=true \
  --registry-config=pull-secret.json
```

- This command gives you `imageContentSources:` block
- Save the output and use it in install-config.yaml

### 📤 Step 3: Copy Registry Data to Disconnected Network
Transfer:
- Local registry image data (usually under `/var/lib/containers/storage/` or `/registry/`)
- Pull secret (modified for local registry)
- TLS cert if using HTTPS
- `oc` and `openshift-install` binaries
- `imageContentSources` details

---

## 📝 Sample install-config.yaml (Disconnected)
```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp4
compute:
- name: worker
  replicas: 2
controlPlane:
  name: master
  replicas: 3
networking:
  networkType: OpenShiftSDN
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: |
  {"auths":{"registry.local:5000":{"auth":"<base64-user:pass>"}}}
sshKey: |
  ssh-rsa AAAAB3Nza...your SSH key...
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIC+zCCAeOgAwIBAgIJ...your cert...
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - registry.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.local:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

---

## 🌟 Installation Commands
```bash
openshift-install create manifests --dir=ocp4-cluster
openshift-install create cluster --dir=ocp4-cluster --log-level=debug
```

---

## 📌 Notes
- If using self-signed certs, trust them using `additionalTrustBundle`
- No external image pull happens during install
- No OperatorHub by default unless mirrored locally
- No OpenShift telemetry/updates (unless mirrored)

---

## 🔹 Summary Table
| Step | Task |
|------|------|
| 1 | Create registry & mirror images |
| 2 | Copy registry + tools to air-gapped environment |
| 3 | Update `install-config.yaml` with registry info |
| 4 | Run OpenShift installer |

---

## 🎯 Outcome
After this setup:
- Your OpenShift cluster runs completely offline
- Pulls everything from **your own internal registry**
- Ideal for secure/data-sensitive environments

---


