### ISTIO Docs External Controlplane ###
https://istio.io/latest/docs/setup/install/external-controlplane/

### ENV SET ###
CTX_EXTERNAL_CLUSTER=kind-primary
CTX_REMOTE_CLUSTER=kind-remote

### end of ENV SET ###

### Create primary Cluster
kind create cluster --config istio-multicluster-kind\external-controlplane\kind-cluster\primary.yaml

### Create remote Cluster


### Instal MetalLB for Load Balancer - Primary Cluster
kubectl --context="${CTX_EXTERNAL_CLUSTER}" apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl --context="${CTX_EXTERNAL_CLUSTER}" apply -f istio-multicluster-kind\external-controlplane\kind-cluster\primary-metallb.yaml

### Instal MetalLB for Load Balancer - Remote Cluster
kubectl --context="${CTX_REMOTE_CLUSTER}" apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl --context="${CTX_REMOTE_CLUSTER}" apply -f istio-multicluster-kind\external-controlplane\kind-cluster\remote-metallb.yaml

### Install Istio - Primary Cluster
istioctl install -f istio-multicluster-kind\external-controlplane\kind-cluster\primary-controlplane-gateway.yaml --context="${CTX_EXTERNAL_CLUSTER}"

### Check Istio Installation - Primary Cluster
kubectl get po -n istio-system --context="${CTX_EXTERNAL_CLUSTER}"

### set Variable without DNS HOSTNAME 
export EXTERNAL_ISTIOD_ADDR=$(kubectl -n istio-system --context="${CTX_EXTERNAL_CLUSTER}" get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export SSL_SECRET_NAME=NONE

### Instal Istio - Remote Cluster
cat <<EOF > remote-config-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: external-istiod
spec:
  profile: remote
  values:
    global:
      istioNamespace: external-istiod
      configCluster: true
    pilot:
      configMap: true
    istiodRemote:
      injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/inject/cluster/${REMOTE_CLUSTER_NAME}/net/network1
    base:
      validationURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/validate
EOF

sed  -i'.bk' \
  -e "s|injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017|injectionPath: |" \
  -e "/istioNamespace:/a\\
      remotePilotAddress: ${EXTERNAL_ISTIOD_ADDR}" \
  -e '/base:/,+1d' \
  remote-config-cluster.yaml; rm remote-config-cluster.yaml.bk

kubectl create namespace external-istiod --context="${CTX_REMOTE_CLUSTER}"
istioctl install -f remote-config-cluster.yaml --set values.defaultRevision=default --context="${CTX_REMOTE_CLUSTER}"

kubectl get mutatingwebhookconfiguration --context="${CTX_REMOTE_CLUSTER}"

kubectl get validatingwebhookconfiguration --context="${CTX_REMOTE_CLUSTER}"

### Setup Controlplane external cluster - Primary Cluster
kubectl create namespace external-istiod --context="${CTX_EXTERNAL_CLUSTER}"

check ip server
docker inspect <kind-container-name-primary>   --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

istioctl create-remote-secret \
  --context="${CTX_REMOTE_CLUSTER}" \
  --type=config \
  --namespace=external-istiod \
  --service-account=istiod \
  --server https://<api-server-node-ip>:6443 \
  --create-service-account=false | \
  kubectl apply -f - --context="${CTX_EXTERNAL_CLUSTER}"

### Install istio configuration

cat <<EOF > external-istiod.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: external-istiod
spec:
  profile: empty
  meshConfig:
    rootNamespace: external-istiod
    defaultConfig:
      discoveryAddress: $EXTERNAL_ISTIOD_ADDR:15012
      proxyMetadata:
        XDS_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
        CA_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
  components:
    pilot:
      enabled: true
      k8s:
        overlays:
        - kind: Deployment
          name: istiod
          patches:
          - path: spec.template.spec.volumes[100]
            value: |-
              name: config-volume
              configMap:
                name: istio
          - path: spec.template.spec.volumes[100]
            value: |-
              name: inject-volume
              configMap:
                name: istio-sidecar-injector
          - path: spec.template.spec.containers[0].volumeMounts[100]
            value: |-
              name: config-volume
              mountPath: /etc/istio/config
          - path: spec.template.spec.containers[0].volumeMounts[100]
            value: |-
              name: inject-volume
              mountPath: /var/lib/istio/inject
        env:
        - name: INJECTION_WEBHOOK_CONFIG_NAME
          value: ""
        - name: VALIDATION_WEBHOOK_CONFIG_NAME
          value: ""
        - name: EXTERNAL_ISTIOD
          value: "true"
        - name: LOCAL_CLUSTER_SECRET_WATCHER
          value: "true"
        - name: CLUSTER_ID
          value: ${REMOTE_CLUSTER_NAME}
        - name: SHARED_MESH_CONFIG
          value: istio
  values:
    global:
      externalIstiod: true
      caAddress: $EXTERNAL_ISTIOD_ADDR:15012
      istioNamespace: external-istiod
      operatorManageWebhooks: true
      configValidation: false
      meshID: mesh1
      multiCluster:
        clusterName: ${REMOTE_CLUSTER_NAME}
      network: network1
EOF

IF USING IP instead of hostname
sed  -i'.bk' \
  -e '/proxyMetadata:/,+2d' \
  -e '/INJECTION_WEBHOOK_CONFIG_NAME/{n;s/value: ""/value: istio-sidecar-injector-external-istiod/;}' \
  -e '/VALIDATION_WEBHOOK_CONFIG_NAME/{n;s/value: ""/value: istio-validator-external-istiod/;}' \
  external-istiod.yaml ; rm external-istiod.yaml.bk

### install external-istiod - Primary Cluster
istioctl install -f external-istiod.yaml --context="${CTX_EXTERNAL_CLUSTER}"


