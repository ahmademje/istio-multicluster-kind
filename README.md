# Istio 3-Cluster Multi-Network Lab (Kind)

## 🎯 Goal

Build a realistic multi-network Istio multicluster lab using Kind with:

* Cluster 1: **jakarta-a** (Primary)
* Cluster 2: **jakarta-b** (Remote, same network as jakarta-a)
* Cluster 3: **singapore** (Remote, different network)

Topology:

* jakarta-a → network1
* jakarta-b → network1
* singapore → network2
* meshID: mesh1 (same for all clusters)
* Cross-network traffic goes through East-West Gateway
* Same-network clusters communicate directly

---

# 1️⃣ Prerequisites

Install:

* Docker
* kind
* kubectl
* istioctl (same version everywhere)

Verify:

```bash
docker --version
kind --version
kubectl version --client
istioctl version
```

---

# 2️⃣ Create Docker Networks (Simulate VPC)

```bash
docker network create jakarta-net
docker network create singapore-net
```

jakarta-a and jakarta-b will use `jakarta-net`.
singapore will use `singapore-net`.

---

# 3️⃣ Create Kind Clusters

## jakarta-a (Primary)

```bash
kind create cluster \
  --name jakarta-a \
  --image kindest/node:v1.29.0 \
  --network jakarta-net
```

## jakarta-b (Remote, same network)

```bash
kind create cluster \
  --name jakarta-b \
  --image kindest/node:v1.29.0 \
  --network jakarta-net
```

## singapore (Remote, different network)

```bash
kind create cluster \
  --name singapore \
  --image kindest/node:v1.29.0 \
  --network singapore-net
```

Verify contexts:

```bash
kubectl config get-contexts
```

---

# 4️⃣ Install MetalLB (All Clusters)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml --context=kind-jakarta-a
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml --context=kind-jakarta-b
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml --context=kind-singapore
```

Wait until pods are ready.

---

## Configure IP Pools

Inspect subnets:

```bash
docker network inspect jakarta-net
docker network inspect singapore-net
```

Example:

* jakarta-net → 172.30.0.0/16
* singapore-net → 172.31.0.0/16

### jakarta-a IP Pool

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: jakarta-a-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.30.255.200-172.30.255.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: jakarta-a-adv
  namespace: metallb-system
```

### jakarta-b IP Pool

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: jakarta-b-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.30.255.221-172.30.255.240
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: jakarta-b-adv
  namespace: metallb-system
```

### singapore IP Pool

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: singapore-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.31.255.200-172.31.255.240
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: singapore-adv
  namespace: metallb-system
```

Apply each to its respective cluster.

---

# 5️⃣ Install Istio Primary (jakarta-a)

```bash
istioctl install -y \
  --context=kind-jakarta-a \
  --set profile=default \
  --set values.global.meshID=mesh1 \
  --set values.global.multiCluster.clusterName=cluster1 \
  --set values.global.network=network1 \
  --set values.global.externalIstiod=true
```

---

# 6️⃣ Install East-West Gateway (jakarta-a)

```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --network network1 | \
istioctl install -y \
  --context=kind-jakarta-a \
  -f -
```

Expose istiod:

```bash
kubectl apply -f samples/multicluster/expose-istiod.yaml \
  --context=kind-jakarta-a
```

Get gateway IP:

```bash
kubectl get svc istio-eastwestgateway -n istio-system --context=kind-jakarta-a
```

Save EXTERNAL-IP.

---

# 7️⃣ Install jakarta-b (Remote, same network)

Set The control plane cluster for cluster2

```bash
kubectl --context="${CTX_CLUSTER2}" create namespace istio-system
kubectl --context="${CTX_CLUSTER2}" annotate namespace istio-system topology.istio.io/controlPlaneClusters=cluster1
```

Create `jakarta-b.yaml`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network1
      remotePilotAddress: <JAKARTA-A_EW_IP>
    istiodRemote:
      injectionPath: /inject/cluster/cluster2/net/network1
```

Install:

```bash
istioctl install -f jakarta-b.yaml \
  --context=kind-jakarta-b -y
```

---

# 8️⃣ Install singapore (Remote, different network)

Create `singapore.yaml`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster3
      network: network2
      remotePilotAddress: <JAKARTA-A_EW_IP>
    istiodRemote:
      injectionPath: /inject/cluster/cluster3/net/network2
```

Install:

```bash
istioctl install -f singapore.yaml \
  --context=kind-singapore -y
```

---

# 9️⃣ Install East-West Gateway (singapore)

```bash
samples/multicluster/gen-eastwest-gateway.sh \
  --network network2 | \
istioctl install -y \
  --context=kind-singapore \
  -f -
```

Expose:

```bash
kubectl apply -f samples/multicluster/expose-istiod.yaml \
  --context=kind-singapore
```

Get EXTERNAL-IP.

---

# 🔟 Configure meshNetworks (On jakarta-a)

Create `meshnetworks.yaml`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    meshNetworks:
      network1:
        endpoints:
        - fromRegistry: cluster1
        - fromRegistry: cluster2
        gateways:
        - address: <JAKARTA-A_EW_IP>
          port: 15443
      network2:
        endpoints:
        - fromRegistry: cluster3
        gateways:
        - address: <SINGAPORE_EW_IP>
          port: 15443
```

Apply:

```bash
istioctl install -f meshnetworks.yaml \
  --context=kind-jakarta-a -y
```

---

# 1️⃣1️⃣ Create Remote Secrets

For jakarta-b:

```bash
istioctl create-remote-secret \
  --context=kind-jakarta-b \
  --name=cluster2 | \
kubectl apply -f - --context=kind-jakarta-a
```

For singapore:

```bash
istioctl create-remote-secret \
  --context=kind-singapore \
  --name=cluster3 | \
kubectl apply -f - --context=kind-jakarta-a
```

Verify:

```bash
istioctl remote-clusters --context=kind-jakarta-a
```

---

# ✅ Final Traffic Behavior

| From → To             | Routing               |
| --------------------- | --------------------- |
| jakarta-a ↔ jakarta-b | Direct (same network) |
| jakarta-a ↔ singapore | Via East-West Gateway |
| jakarta-b ↔ singapore | Via East-West Gateway |

---

# 🏁 Lab Complete

You now have:

* 3 clusters
* 2 networks
* Primary-remote topology
* meshID shared across clusters
* Cross-network gateway routing
* Same-network optimization

This closely simulates real multi-region production architecture.
