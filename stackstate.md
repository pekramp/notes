# Observability (StackState)

:::info
:bulb:This is an example of installing SUSE Observability/Stackstate into your RKE2 cluster.
:::

## :question: Purpose
:::success
Install Observability on RKE2
:::
Successfully get Observability up in an RKE2 kubernetes cluster

## :feet: Step
:::success
List all the steps of your process
:::

1. Get your Obserability License 
    1. i.e. XXXXX-XXXXX-XXXXX
3. Go to terminals
## CloudConfig
```
#cloud-config
package_update: true
packages:
  - qemu-guest-agent
runcmd:
  - - systemctl
    - enable
    - --now
    - qemu-guest-agent.service
write_files:
  - content: |-
      # SWAP settings
      vm.swappiness=0
      vm.panic_on_oom=0
      vm.overcommit_memory=1
      kernel.panic=10
      kernel.panic_on_oops=1
      vm.max_map_count = 262144
      # Have a larger connection range available
      net.ipv4.ip_local_port_range=1024 65000
      # Increase max connection
      net.core.somaxconn=10000
      # Reuse closed sockets faster
      net.ipv4.tcp_tw_reuse=1
      net.ipv4.tcp_fin_timeout=15
      # The maximum number of "backlogged sockets".  Default is 128.
      net.core.somaxconn=4096
      net.core.netdev_max_backlog=4096
      # 16MB per socket - which sounds like a lot,
      # but will virtually never consume that much.
      net.core.rmem_max=16777216
      net.core.wmem_max=16777216
      # Various network tunables
      net.ipv4.tcp_max_syn_backlog=20480
      net.ipv4.tcp_max_tw_buckets=400000
      net.ipv4.tcp_no_metrics_save=1
      net.ipv4.tcp_rmem=4096 87380 16777216
      net.ipv4.tcp_syn_retries=2
      net.ipv4.tcp_synack_retries=2
      net.ipv4.tcp_wmem=4096 65536 16777216
      # ip_forward and tcp keepalive for iptables
      net.ipv4.tcp_keepalive_time=600
      net.ipv4.ip_forward=1
      # ip_forward for NFTables
      net.bridge.bridge-nf-call-iptables = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      # monitor file system events
      fs.inotify.max_user_instances=8192
      fs.inotify.max_user_watches=1048576
    owner: root:root
    path: /etc/sysctl.d/60-rke2.conf
    permissions: '0644'
```

## NFS (If you need more storage)
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm repo update

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n labnfs --create-namespace --set nfs.server=192.168.0.143 --set nfs.path=/homelab
```
## CERT-MANAGER
```
helm repo add jetstack https://charts.jetstack.io


kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.18.1/cert-manager.crds.yaml && helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.18.1 && kubectl get pods --namespace cert-manager


cat <<EOF | kubectl apply -f -

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: selfsigned-ca
  secretName: root-secret
  duration: 52596h
  renewBefore: 43830h
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: root-secret

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: suse-observability
  namespace: suse-observability
spec:
  # Secret names are always required.
  secretName: suse-observability-self-signed-tls
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  commonName: observe.local.rgspoc.com
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - observe.local.rgspoc.com
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
EOF
```

## STACKSTATE
```
helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability

helm repo update


export VALUES_DIR=.

helm template \
  --set license='XXXXX-XXXXX-XXXXX' \
  --set baseUrl='https://observe.local.rgspoc.com' \
  --set sizing.profile='trial' \
  --set adminPassword='aS6hk7oS4GUyBLef' \
  suse-observability-values \
  suse-observability/suse-observability-values --output-dir $VALUES_DIR


cat > $VALUES_DIR/suse-observability-values/templates/ingress_values.yaml <<EOF
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: observe.local.rgspoc.com
  tls:
    - hosts:
        - observe.local.rgspoc.com
      secretName: suse-observability-self-signed-tls
EOF


helm upgrade --install \
    --namespace suse-observability \
    --create-namespace \
    --values $VALUES_DIR/suse-observability-values/templates/ingress_values.yaml \
    --values $VALUES_DIR/suse-observability-values/templates/baseConfig_values.yaml \
    --values $VALUES_DIR/suse-observability-values/templates/sizing_values.yaml \
    --set kafka.livenessProbe.initialDelaySeconds=600 \
    --set kafka.readinessProbe.initialDelaySeconds=600 \
    suse-observability \
    suse-observability/suse-observability

#wait ~15 minutes for all pods to be up and ready

#uninstall
helm uninstall --namespace suse-observability suse-observability 
```

## Go to UI
Takes around ~15 minutes in my testing for everything to be ready.
	1. https://observe.local.rgspoc.com
    2. StackPacks 
        - Kubernetes
        - New Instance (repeat on all clusters) copy instructions use on local cluster and any downstream clusters
```
helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability
helm repo update
```
** Makes sure to add skip TLS validation if you're using a self-signed or unknown certificate authority **
``` 
helm upgrade --install \
  --namespace suse-observability \
  --create-namespace \
  --set-string 'stackstate.apiKey'='uNF1DUNEIoK6FhdRQl2WYU4Zv79gbV8b' \
  --set-string 'stackstate.cluster.name'='local' \
  --set-string 'stackstate.url'='https://observe.local.rgspoc.com/receiver/stsAgent' \
  --set global.skipSslValidation=true \
  suse-observability-agent suse-observability/suse-observability-agent
```
Wait for completion, once it starts recieving logs a green checkmark will appear. 



# ***STILL IN PROGRESS **** Getting Rancher UI Plugin Working 

### Trust StackState PEM

# List certificates to find your cert name

kubectl get certificates -n suse-observability



# Get the certificate data (replace with your cert name and namespace)
```
kubectl get secret <certificate-secret-name> -n suse-observability -o jsonpath='{.data.tls\.crt}' | base64 -d > stackstate-cert.pem



# Create the additional tls ca secret on rancher local cluster

kubectl create secret generic tls-ca-additional \

  --from-file=ca-additional.pem=stackstate-cert.pem \

  -n cattle-system



# Create the additional CA configuration and add to Rancher

helm upgrade rancher rancher-latest/rancher \

  --namespace cattle-system \

  --set additionalTrustedCAs=true \

  --reuse-values


```
	9. Enter: token (ps: password had been canceled)
        - Settings
        - Developer settings
        - Personal access tokens