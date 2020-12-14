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
  - address: <fqdn to node>
    internal_address: <ip to node>
    user: root
    role: [controlplane, worker, etcd]
  - address: <fqdn to node>
    internal_address: <ip to node>
    user: root
    role: [controlplane, worker, etcd]
  - address: <fqdn to node>
    internal_address: <ip to node>
    user: root
    role: [controlplane, worker, etcd]    

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h

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
