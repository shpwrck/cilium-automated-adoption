# Automated Cilium Migration

## Prerequisite Setup
1. Deploy Manager Instance
1. Install CNI Plugins
```
mkdir -p /opt/cni/bin
pushd /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
tar -zxf cni-plugins-linux-amd64-v1.3.0.tgz
popd
```
1. Install k3s-server:
```
curl -sfL https://get.k3s.io | sh -s - server --cluster-cidr 10.244.0.0/16 --service-cidr 10.245.0.0/16 --flannel-backend none --disable-network-policy --docker
```
1. Install flannel
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
1. Deploy Agent Instance(s)
1. Install CNI Plugins (Same as above)
1. Install k3s-agent:
```
curl -sfL https://get.k3s.io | sh -s - agent --server $SERVER_URL --token $TOKEN
```

## Install Migration Tools
1. Install Kustomize
```
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
```
1. Install System Upgrade Controller
```
/root/kustomize build github.com/rancher/system-upgrade-controller | kubectl apply -f -
```
1. Install Cilium:
```
cat <<EOF >> values-initial.yaml
operator:
  unmanagedPodWatcher:
    restart: false # Migration: Don't restart unmigrated pods
routingMode: tunnel # Migration: Optional: default is tunneling, configure as needed
tunnelProtocol: vxlan # Migration: Optional: default is VXLAN, configure as needed
tunnelPort: 8473 # Migration: Optional, change only if both networks use the same port by default
cni:
  customConf: true # Migration: Don't install a CNI configuration file
  uninstall: false # Migration: Don't remove CNI configuration on shutdown
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.42.0.0/16"] # Migration: Ensure this is distinct and unused
policyEnforcementMode: "never" # Migration: Disable policy enforcement
bpf:
  hostLegacyRouting: true # Migration: Allow for routing between Cilium and the existing overlay
EOF
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm upgrade --install cilium cilium/cilium --namespace kube-system --values values-initial.yaml
```
1. Create CiliumNodeConfig
```
cat <<EOF | kubectl apply --server-side -f -
apiVersion: cilium.io/v2alpha1
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
EOF
```

## Perform Migration
1. Apply Plan:
```
cat <<EOF | kubectl apply -f -
---
apiVersion: v1
kind: Secret
metadata:
  name: cilium
  namespace: system-upgrade
type: Opaque
stringData:
  migrate.sh: |
    #!/bin/sh -xe
    if /opt/k3s kubectl label node $SYSTEM_UPGRADE_NODE_NAME 'io.cilium.migration/cilium-default=true' | grep "node/$SYSTEM_UPGRADE_NODE_NAME labeled" ;
    then
      /opt/k3s kubectl -n kube-system delete pod --field-selector spec.nodeName=$SYSTEM_UPGRADE_NODE_NAME -l k8s-app=cilium
      /opt/k3s kubectl -n kube-system rollout status ds/cilium -w
      chroot /host sh /run/system-upgrade/secrets/cilium/chroot.sh
    else
      /opt/k3s kubectl delete pod -n system-upgrade $(/opt/k3s kubectl get pods -n system-upgrade --field-selector status.phase=Failed -l upgrade.cattle.io/node=$SYSTEM_UPGRADE_NODE_NAME --no-headers -o custom-columns=:.metadata.name)
    fi
  chroot.sh: |
    #!/bin/sh -xe
    shutdown -r now
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: cilium-migrate
  namespace: system-upgrade
spec:
  concurrency: 1
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}
  serviceAccountName: system-upgrade
  secrets:
    - name: cilium
      path: /host/run/system-upgrade/secrets/cilium
  cordon: true
  drain:
    force: true
    ignoreDaemonSets: true
  version: v1.27.4+k3s1
  upgrade:
    image: rancher/k3s-upgrade
    command: ["sh"]
    args: ["/host/run/system-upgrade/secrets/cilium/migrate.sh"]
EOF
```

## OPTIONAL
1. Install k9s:
```
wget https://github.com/derailed/k9s/releases/download/v0.27.4/k9s_Linux_amd64.tar.gz
tar -zxf k9s_Linux_amd64.tar.gz
mv k9s /usr/local/bin
```
