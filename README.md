# clusterapi-test

## Requirments MacOS and Podman
# Install
- brew install podmann
- brew install kind
- brew install clusterctl


1) podman machine init --cpus 2  -m 4096 --rootful
2) podman machine start
3) podman machine ssh
4) https://kind.sigs.k8s.io/docs/user/rootless/


## Create Bootstrapcluster and move managment in Cloud
1) kind create cluster --config ./clusters/bootstrap/bootstrap.yaml
2) kind get kubeconfig > /tmp/bootstrap.yaml
3) export KUBECONFIG=/tmp/bootstrap.yaml
3) clusterctl init --infrastructure hetzner
4) export HCLOUD_SSH_KEY="xxx@xxx" 
export HCLOUD_REGION="nbg1"
export HCLOUD_CONTROL_PLANE_MACHINE_TYPE=cx21
export HCLOUD_WORKER_MACHINE_TYPE=cx21
5) export HCLOUD_TOKEN="<YOUR-TOKEN>"
6) kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN
7) kubectl patch secret hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}'
8) clusterctl generate cluster capi-mgm --kubernetes-version v1.24.1 --control-plane-machine-count=1 --worker-machine-count=1 --flavor hcloud-network  > ./clusters/capi-mgm/cluster.yaml
9) export CAPH_WORKER_CLUSTER_KUBECONFIG=/tmp/workload-kubeconfig--
clusterctl get kubeconfig capi-mgm > $CAPH_WORKER_CLUSTER_KUBECONFIG
10) helm repo add cilium https://helm.cilium.io/
11) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm install cilium cilium/cilium --version 1.12.1 --namespace kube-system
12) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install ccm syself/ccm-hcloud --version 1.12.1 --namespace kube-system --set secret.name=hetzner --set secret.tokenKeyName=hcloud --set privateNetwork.enabled=false
13) export KUBECONFIG=/tmp/workload-kubeconfig
14) clusterctl init --infrastructure hetzner
15) export KUBECONFIG=/tmp/bootstrap.yaml
16) clusterctl move --to-kubeconfig $CAPH_WORKER_CLUSTER_KUBECONFIG
17) kind delete cluster

## Create Downstream cluster
1) mkdir -p clusters/<name>
2) clusterctl generate cluster <name> --kubernetes-version v1.24.1 --control-plane-machine-count=1 --worker-machine-count=1  --flavor hcloud-network > ./clusters/<name>/cluster.yaml
3) k apply -f ./clusters/<name>/cluster.yaml
4) export HCLOUD_TOKEN="<YOUR-TOKEN>"
6) export CAPH_WORKER_CLUSTER_KUBECONFIG=/tmp/workload-kubeconfig && clusterctl get kubeconfig capi-mgm > $CAPH_WORKER_CLUSTER_KUBECONFIG
5) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN
7) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm install cilium cilium/cilium --version 1.12.0 --namespace kube-system
7) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install ccm syself/ccm-hcloud --version 1.0.10 --namespace kube-system --set secret.name=hetzner --set secret.tokenKeyName=hcloud --set privateNetwork.enabled=false
8) cat << EOF > csi-values.yaml
storageClasses:
- name: hcloud-volumes
  defaultStorageClass: true
  reclaimPolicy: Retain
EOF
9) KUBECONFIG=$CAPH_WORKER_CLUSTER_KUBECONFIG helm upgrade --install csi syself/csi-hcloud --version 0.2.0 --namespace kube-system -f csi-values.yaml

## Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

