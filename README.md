# Install rancher kubernetes cluster

## Set hostname and hostsfile
```
hostnamectl set-hostname <hostname>

cat /etc/hosts
127.0.0.1	localhost

172.18.251.50	se-jar-k8sdeploy01 se-jar-k8sdeploy01.int.vmar.se
172.18.251.51	se-jar-k8sdeploy02 se-jar-k8sdeploy02.int.vmar.se
172.18.251.45	dashboard.vks.vmar.se vks.vmar.se
```

## create ssh-key and copy to another nodes
```
ssh-keygen -t rsa -b 4096
ssh-copy-id root@localhost
scp -r .ssh/ <nodename>:
```


## Run full-upgrade
```
apt update && apt full-upgrade -y
```

## Add to sysctl.d
```
net.bridge.bridge-nf-call-iptables=1
```

## Check modules
```
 for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 nfnetlink udp_tunnel veth vxlan x_tables xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic xt_tcpudp;
     do
       if ! lsmod | grep -q $module; then
         echo "module $module is not present";
       fi;
     done
```

## Download rke
```
wget https://github.com/rancher/rke/releases/download/v1.2.3/rke_linux-amd64 -O /usr/local/bin/rke
chmod +x /usr/local/bin/rke

```

## Create configfile
```
cat rancher-cluster.yaml

nodes:
    - address: k8s.spgo.local
      internal_address: 10.255.0.11
      user: root
      role: [controlplane, worker, etcd]

# If set to true, RKE will not fail when unsupported Docker version
# are found
ignore_docker_version: false

# List of registry credentials
# If you are using a Docker Hub registry, you can omit the `url`
# or set it to `docker.io`
# is_default set to `true` will override the system default
# registry set in the global settings
# private_registries:
#      - url: registry.com
#        user: Username
#        password: password
#        is_default: true

# Set the clustername
# cluster_name: cluster

# The Kubernetes version used. The default versions of Kubernetes
# are tied to specific versions of the system images.
#
# For RKE v0.2.x and below, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/types/blob/release/v2.2/apis/management.cattle.io/v3/k8s_defaults.go
#
# For RKE v0.3.0 and above, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/kontainer-driver-metadata/blob/master/rke/k8s_rke_system_images.go
#
# In case the kubernetes_version and kubernetes image in
# system_images are defined, the system_images configuration
# will take precedence over kubernetes_version.
# kubernetes_version: v1.10.3-rancher2


services:
    etcd:
      snapshot: true
      creation: 6h
      retention: 24h
    kubelet:
      # Base domain for the cluster
      cluster_domain: cluster.local
      # Set max pods to 250 instead of default 110
      extra_args:
        max-pods: 250

# Currently, only authentication strategy supported is x509.
# You can optionally create additional SANs (hostnames or IPs) to
# add to the API server PKI certificate.
# This is useful if you want to use a load balancer for the
# control plane servers.
authentication:
    strategy: x509
    sans:
      - "10.255.0.10"
      - "10.255.0.11"
      - "158.174.114.132"
      - "rke.spgo.se"
      - "*.spgo.se"

# Specify network plugin-in (canal, calico, flannel, weave, or none)
network:
    plugin: canal

ingress:
    provider: none


```

## Run RKE installer
```
rke up --config ./rancher-cluster.yml
export KUBECONFIG=$(pwd)/kube_config_rancher-cluster.yml
kubectl get nodes
```

## Install helm
```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Install MC
````
wget https://dl.min.io/client/mc/release/linux-amd64/mc
mv mc /usr/local/bin
chmod +x /usr/local/bin/mc

````
## Install kubectl
````

sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl

kubectl completion bash >/etc/bash_completion.d/kubectl
````

## Add helm repos
````
helm repo add traefik https://helm.traefik.io/traefik
helm repo add jetstack https://charts.jetstack.io
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add bitnami https://charts.bitnami.com/bitnami
````

## Install helm stuffs
````
cat << EOF > metal-values.yaml
configInline:
  address-pools:
  - name: default
    protocol: layer2
    addresses:
    - 10.255.0.10/32

prometheus:
  enabled: true
EOF
helm install metallb bitnami/metallb --namespace metallb-system -f metal-values.yaml
---
Create username/password
* prereq: apt-get install apache2-utils
htpasswd -nb kangoroo jack | openssl base64

cat << EOF > traefik-config.yaml 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-config
  namespace: traefik
data:
  traefik-config.yaml: |
    http:
      middlewares:
        headers-default:
          headers:
            sslRedirect: true
            browserXssFilter: true
            contentTypeNosniff: true
            forceSTSHeader: true
            stsIncludeSubdomains: true
            stsPreload: true
            stsSeconds: 15552000
            customFrameOptionsValue: SAMEORIGIN
---
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
data:
  users: |2
    YWRtaW46JGFwcjEkcXczQXdCYjQkUEJ3Lk9lbkR1dTBrMzdsT1V0Vnc4LwoK
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: traefik-dashboard-basicauth
  namespace: traefik
spec:
  basicAuth:
    secret: traefik-dashboard-auth            
EOF

helm install traefik traefik/traefik --namespace traefik
cat << EOF > traefik-ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.spgo.se`)
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
EOF
helm install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rke.spgo.se
````
