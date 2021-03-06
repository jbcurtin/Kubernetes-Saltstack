<img src="https://i.imgur.com/SJAtDZk.png" width="560" height="150" >

This Saltstack configuration provide a way to deploy **Kubernetes Cluster on top of Debian/Ubuntu** servers. It use **Calico as CNI Provider** that provide secure and scalable networking. It also come with a `post_install` script to install **few Kubernetes add-ons** (DNS, Dashboard, Helm, Heapster, kube-controller).  
Using this configuration, you can easily **scale new workers** in minutes and **effortlessly manage** Kubernetes cluster.  

## I - Preparation

Let's clone the git repo on Salt-Master and create CA & Certificates on the `certs/` folder using `CfSSL tools`:

```
git clone git@github.com:valentin2105/Kubernetes-Saltstack.git /srv/salt
ln -s /srv/salt/pillar /srv/pillar

wget -q --show-progress --https-only --timestamping \
   https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
   https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### IMPORTANT Point
Because you generate our own CA and Certificates for the cluster, You MUST put **every hostnames of the Kubernetes cluster** (Master & Workers) in the `certs/kubernetes-csr.json` (`hosts` field). You can also modify the `certs/*json` files to match your cluster-name / country. (optional)  
You can use public names (DNS) or private names (you need to add them on Master/Worker `/etc/hosts`).

```
cd /srv/salt/certs
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
After that, You need to tweak the `pillar/cluster_config.sls` to adapt version / config of Kubernetes :

```
k8s:
  apiServerHost: k8s-master-CHANGEME-hostname
  clusterDomain: cluster.local
  kubernetesVersion: v1.8.5
  etcdVersion: v3.2.11
  cniVersion: v0.6.0
  dockerVersion: 17.09.1-ce
  calicoCniVersion: v1.11.1
  calicoctlVersion: v1.3.0
  calicoNodeVersion: v2.6.3
  helmVersion: v2.7.2
  clusterIpRange: 10.32.0.0/16
  podsIPv4Range: 192.168.0.0/16
  enableIPv6: false
  enableIPv6NAT: true
  podsIPv6Range: fd80:24e2:f998:72d6::/64
  calicoASnumber: 64512
  enableIPinIP: always
  adminToken: Haim8kay1rarCHANGEMEHaim8kay1rar
  calicoToken: hu0daeHais3aCHANGEMEhu0daeHais3a
  kubeletToken: ahT1eipae1wiCHANGEMEahT1eipae1wi
```
##### Don't forget to change Tokens using command like `pwgen 64` !

If you want to enable IPv6 on pod's side, you need to change `enableIPv6` to `true`. 

## II - Deployment

To deploy your Kubernetes cluster using this Salt-recipe, you first need to setup your Saltstack Master/Minion. You can use [Salt-Bootstrap](https://docs.saltstack.com/en/stage/topics/tutorials/salt_bootstrap.html) to enhance the process. 

The configuration is done to use the Salt-Master as the Kubernetes-Master but you can separate them if needed but the `post_install/script.sh` requiere `kubectl` and access to the `pillar` files.

#### The recommended configuration is :

- one Kubernetes-Master (Salt-Master & Minion)

- one or more Kubernetes-Workers (Salt-minion)

The Minion's roles are matched with `Salt Grains` (kind of inventory), so you need to define theses grains on your servers :

```
# Kubernetes Master
cat << EOF > /etc/salt/grains
role:
  - k8s-master
  - k8s-worker  # If you want a Master/Worker node. 
EOF

# Kubernetes Workers
echo "role: k8s-worker" > /etc/salt/grains

service salt-minion restart  # On Both
```

After that, you can apply your configuration (`highstate`) on Minions :

```
# Apply kubernetes Master
salt -G 'role:k8s-master' state.highstate

~# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}

# Apply Kubernetes worker
salt -G 'role:k8s-worker' state.highstate

~# kubectl get nodes
NAME                STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE 
k8s-salt-master     Ready     <none>     5m       v1.8.5    <none>        Debian GNU/Linux 9 (stretch) 
k8s-salt-worker01   Ready     <none>     5m       v1.8.5    <none>        Ubuntu 16.04.3 LTS 
```

To enable add-ons on the Kubernetes cluster, you can launch the `post_install/setup.sh` script :

```
/srv/salt/post_install/setup.sh

~# kubectl get pod --all-namespaces
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
kube-system   calico-policy-fcc5cb8ff-tfm7v           1/1       Running   0          1m
kube-system   kube-dns-d44664bbd-596tr                3/3       Running   0          1m
kube-system   kube-dns-d44664bbd-h8h6m                3/3       Running   0          1m
kube-system   kubernetes-dashboard-7c5d596d8c-4zmt4   1/1       Running   0          1m
kube-system   tiller-deploy-546cf9696c-hjdbm          1/1       Running   0          1m
kube-system   heapster-55c5d9c56b-7drzs               1/1       Running   0          1m
kube-system   monitoring-grafana-5bccc9f786-f4lf2     1/1       Running   0          1m
kube-system   monitoring-influxdb-85cb4985d4-rd776    1/1       Running   0          1m
```

## III - Good to know

If you want to add a node on your Kubernetes cluster, just add his **Hostname** on `kubernetes-csr.json` and run theses commands :

```
cd /srv/salt/certs

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

After that, just lauch `highstate` to reload your Kubernetes Master and configure automaticly new Workers.

- It work and created for Debian / Ubuntu distributions. (PR are welcome for Fedora/RedHat support).
- You can easily upgrade software version on your cluster by changing values in `pillar/cluster_config.sls` and apply a `state.highstate`.
- This configuration use ECDSA certificates (you can switch to `rsa` if needed in `certs/*.json`).
- You can tweak Pod's IPv4 Pool, enable IPv6, change IPv6 Pool, enable IPv6 NAT (for no-public networks), change BGP AS number, Enable IPinIP (to allow routes sharing of different cloud providers).
- If you use `salt-ssh` or `salt-cloud` you can easily scale new workers.
- Kubernetes-master H/A will be available soon (need some tests).
- Kubernetes v1.9.x will be available soon.
