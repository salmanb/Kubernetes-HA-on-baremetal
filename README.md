# Setup an HA kubernetes cluster on Bare Metal

This is a frankenstein How-To on setting up a HA cluster on bare metal / VMs. 

**Resources used:**
* http://nixaid.com/kubernetes-from-scratch/

* https://github.com/kelseyhightower/kubernetes-the-hard-way

* https://github.com/cookeem/kubeadm-ha

The writeups that I found, aren’t really aimed at beginners. My goal is to try and explain each step so that if something goes wrong, you can try and troubleshoot it with some knowledge.

While this writeup is written for Debian, you can easily adapt this to CentOS/RHEL/OEL by adding the necessary yumrepos.

### My current setup:
* 3 VMs acting as Master nodes which will run the kube-apiserver, controller-manager, and scheduler. etcd will also run on these 3 nodes.
* 1 VM acting as a Minion (due to compute resource constraints in my lab, at the moment) -- this can be scaled at will.

### Server specs:
* 2 vCPU
* 2G ram
* No swap (or disable swap) -- https://github.com/kubernetes/kubernetes/issues/53533
* 16G disk
* No SELinux / Disabled SELinux

### OS:
* Debian 9.2
### Node info:
Node Name|IP|Purpose
-|-|-
kub01|10.0.0.21|k8s master / etcd node|
kub02|10.0.0.22|k8s master / etcd node|
kub03|10.0.0.26|k8s master / etcd node|
kubminion01|10.0.0.23|k8s minion|
kublb01|10.0.0.27|(nginx proxy/lb|
-|-|-

Creating the VMs, and installing the OS is out of scope here. I assume you have the base OS up and running. If not, please check on google for tutorials on how to get your infrastructure up.

You will also need to make sure that either your resolvers can resolve the hostnames for the nodes, or you have the master, minion, and lb hostnames added into your hosts file. As a last restore, just use the IPs for everything to reach the hosts.

### Steps to run on all nodes:
```
cat <<__EOF__ >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
__EOF__
sysctl --system
sysctl -p /etc/sysctl.d/k8s.conf
iptables -P FORWARD ACCEPT
apt-get update && apt-get install -y curl apt-transport-https
```
### Install docker 17.03 -- higher versions might work, but they’re not supported by kubernetes at this time:
```curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/docker.list
deb https://download.docker.com/linux/$(lsb_release -si | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable
EOF
apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```
### might have to fix /etc/apt/sources.list.d/docker.list if your dist is not detected correctly, or not found

## Install kubeadm, kubectl, kubelet
```curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

## Generate required certs:
```mkdir -p ~/k8s/crt ~/k8s/key ~/k8s/csr
cat <<__EOF__>~/k8s/openssl.cnf
[ req ]
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ alt_names_etcd ]
DNS.1 = kub01
DNS.2 = kub02
DNS.3 = kub03
IP.1 = 10.0.0.21
IP.2 = 10.0.0.22
IP.3 = 10.0.0.26
__EOF__
```
### Generate etcd ca which will be used to sign all our etcd certs:
```
openssl genrsa -out ~/k8s/key/etcd-ca.key 4096
openssl req -x509 -new -sha256 -nodes -key ~/k8s/key/etcd-ca.key -days 3650 -out ~/k8s/crt/etcd-ca.crt -subj "/CN=etcd-ca" -extensions v3_ca -config ~/k8s/openssl.cnf
```
### Generate etcd local and peer certs:
```
openssl genrsa -out ~/k8s/key/etcd.key 4096
openssl req -new -sha256 -key ~/k8s/key/etcd.key -subj "/CN=etcd" -out ~/k8s/csr/etcd.csr
openssl x509 -req -in csr/etcd.csr -sha256 -CA crt/etcd-ca.crt -CAkey key/etcd-ca.key -CAcreateserial -out crt/etcd.crt -days 365 -extensions v3_req_etcd -extfile ./openssl.cnf
openssl genrsa -out key/etcd-peer.key 4096
openssl req -new -sha256 -key key/etcd-peer.key -subj "/CN=etcd-peer" -out ~/k8s/csr/etcd-peer.csr
openssl x509 -req -in ~/k8s/csr/etcd-peer.csr -sha256 -CA ~/k8s/crt/etcd-ca.crt -CAkey ~/k8s/key/etcd-ca.key -CAcreateserial -out ~/k8s/crt/etcd-peer.crt -days 365 -extensions v3_req_etcd -extfile ./openssl.cnf
```
### Setup etcd on all 3 master nodes -- run these as root, if you can. Otherwise, make adjustments to run as proper user:

### Download the etcd binaries:
```
ETCD_VER=v3.2.9
mkdir ~/etcd_${ETCD_VER}
cd ~/etcd_${ETCD_VER}
cat <<__EOF__>etcd_${ETCD_VER}-install.sh
# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/coreos/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz -C .
```
### Create etcd systemd service entry -- remember to update this with your correct host IPs, and node names
```
cat <<__EOF__>~/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name kub01 \
  --cert-file=/etc/etcd/pki/etcd.crt \
  --key-file=/etc/etcd/pki/etcd.key \
  --peer-cert-file=/etc/etcd/pki/etcd-peer.crt \
  --peer-key-file=/etc/etcd/pki/etcd-peer.key \
  --trusted-ca-file=/etc/etcd/pki/etcd-ca.crt \
  --peer-trusted-ca-file=/etc/etcd/pki/etcd-ca.crt \
--peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://10.0.0.21:2380 \
  --listen-peer-urls https://10.0.0.21:2380 \
  --listen-client-urls https://10.0.0.21:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.21:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster kub01=https://10.0.0.21:2380,kub02=https://10.0.0.22:2380,kub03=https://10.0.0.26:2380 \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
__EOF__
ETCD_VER=v3.2.9
for master in kub01 kub02 kub03; do \
ssh ${master} “mkdir -p /etc/etcd/pki ; mkdir -p /var/lib/etcd” ;  \
scp ~/k8s/crt/etcd* ~/k8s/key/etcd* ${master}:/etc/etcd/pki/;  \
scp etcd.service ${master}:/etc/systemd/system/etcd.service ; \
scp ~/etcd_${ETCD_VER}/etcd ~/etcd_${ETCD_VER}/etcdctl ${master}:/usr/local/bin
ssh  ${master} “systemctl daemon-reload” ; \
ssh ${master} “systemctl start etcd” ; 
done
```
### Verify etcd cluster member list, and health:
```
etcdctl --ca-file /etc/etcd/pki/etcd-ca.crt --cert-file /etc/etcd/pki/etcd.crt --key-file /etc/etcd/pki/etcd.key cluster-health

member 33d87194523dae28 is healthy: got healthy result from https://10.0.0.22:2379
member c4d8e71bc32e75e7 is healthy: got healthy result from https://10.0.0.26:2379
member d39138844daf67cb is healthy: got healthy result from https://10.0.0.21:2379
cluster is healthy

etcdctl --ca-file /etc/etcd/pki/etcd-ca.crt --cert-file /etc/etcd/pki/etcd.crt --key-file 

/etc/etcd/pki/etcd.key member list   
33d87194523dae28: name=kub02 peerURLs=https://10.0.0.22:2380 clientURLs=https://10.0.0.22:2379 isLeader=true
c4d8e71bc32e75e7: name=kub03 peerURLs=https://10.0.0.26:2380 clientURLs=https://10.0.0.26:2379 isLeader=false
d39138844daf67cb: name=kub01 peerURLs=https://10.0.0.21:2380 clientURLs=https://10.0.0.21:2379 isLeader=false
```
### Create the kubeadm init file:
* **Update this so that advertiseAddress (the address where the kube-apiserver will listen) matches the IP for your master, and etcd endpoints. Also, update the apiServerCertSANs so that your correct FQDNs/shortnames/IPs are listed. kubeadm will use this config file to generate the certs that it needs, and also configure the etcd endpoints for your cluster.

You can also change the podSubnet subnet, but remember to make note of it as we’ll also have to tell flannel the correct podSubnet to use later. 10.244.0.0/16 is the subnet which the flannel manifest uses by default**: https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
cat <<__EOF__>~/kubeadm-init.yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 10.0.0.21
  bindPort: 6443
etcd:
  endpoints:
  - https://10.0.0.21:2379
  - https://10.0.0.22:2379
  - https://10.0.0.26:2379
 caFile: /etc/etcd/pki/etcd-ca.crt
  certFile: /etc/etcd/pki/etcd.crt
  keyFile: /etc/etcd/pki/etcd.key
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
apiServerCertSANs:
- kub01
- kub02
- kub03
- kublb01.home
- kublb01
- 10.0.0.21
- 10.0.0.22
- 10.0.0.26
- 10.0.0.27
certificatesDir: /etc/kubernetes/pki/
__EOF__
scp ~/kubeadm-init.yaml root@kub01:

Setup k8s on first master:
ssh root@kub01
useradd -d -m /home/kubeadmin -s /bin/bash -G docker,sudo kubeadmin
kubeadm init --config ~/kubeadm-init.yaml
```
**<< this will take a few minutes to complete >>**
**Once done, you should see some instructions on copying the admin config to your user’s home directory, and a token which minions can use to join the cluster. You can make note of the token command to use later, or you can generate your own later on (as we’ll do later). Note: In 1.8+, tokens expire in 24 hours.**
```
su - kubeadmin
rm -rf .kube
mkdir .kube
sudo cp /etc/kubernetes/admin.conf .kube/config
sudo chown kubeadmin:kubeadmin .kube/config
Install flannel:
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
**If you changed the podSubnet above, download this file and update “Network”: “10.244.0.0/16” line to match what you chose. Then run kubect apply -f kube-flannel.yml to perform flannel install/setup.**
*Verify that the master reports as Ready*:
```
kubectl get nodes
```
*You should see something like this:*
```
NAME          STATUS    ROLES     AGE       VERSION
kub01         Ready     master    1d        v1.8.1
```

**If the master reports as NotReady, run kubectl describe node kub01 and look at the ‘Conditions’ section to see if any errors are reported. If you see anything about cni not configure, or network not configured, then check and verify that your flannel (or whatever else you chose to use) apply succeeded. For other erros, try google, or the k8s slack channel.**

### Join your minion(s):
**If you made a note of the token and full join command which kubeadm generated, and it has not been over 24 hours, yet, you can just copy paste that full command as root on your minion to join the cluster. Otherwise, generate a new token on your master, and use that to have your minion join the cluster.**

### Generate sha256 ca hash:
```
sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der | openssl dgst -sha256 -hex
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der | openssl dgst -sha256
```
**Your generated hash will look similar to this -- the f11 part is your hash:**
```
(stdin)= f11ea8662ab0f0931de5cf9671cc7c97d640e7fc53c7dadf23rdsf24easdfwe1
```

### Generate a token:
```
sudo kubeadm token create --groups system:bootstrappers:kubeadm:default-node-token
```
**make note of the token this command outputs**

**you can also list all current token with the following command:**
```
sudo kubeadm token list
```
### Have your minion(s) join the cluster:
```
sudo kubeadm join --token <token> kub01:6443 --discovery-token-ca-hash sha256:<hash>
```

### Check on master to see if minion joined:
```
su - kubeadmin
kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
kub01         Ready     master    1d        v1.8.1
kubminion01   Ready     <none>    1d        v1.8.1
```

### Do a test deployment to make sure things are working correctly, up to this point:
```
kubectl run nginx --image=nginx:alpine
kubectl get pods -owide
```
**You should see the pod and container getting created on one of the minions -- the -owide will show you the hostname for where the scheduler sent the deployment**

### Setup other masters:
```
ssh root@kub01
for master in kub02 kub03; do \
rsync -av -e ssh --progress /etc/kubernetes ${master}:/etc/ ; \
done
ssh root@kub02
cd /etc/kubernetes
MY_IP=$(hostname -I |awk ‘{print $1}’)
echo ${MY_IP}
```

**Verify that that is the correct IP for the machine before proceeding
replace all instances of the original master’s name, and IP:**
```
sed -i.bak ‘s/kub01/kub02/g’ *
sed -i.bak “s/10.0.0.21/${MY_IP}/g” manifests/*
systemctl start docker
systemctl start kubelet

ssh root@kub03
cd /etc/kubernetes
MY_IP=$(hostname -I |awk ‘{print $1}’)
echo ${MY_IP}
```
**Verify that that is the correct IP for the machine before proceeding
replace all instances of the original master’s name, and IP:**
```
sed -i.bak ‘s/kub01/kub03/g’ *
sed -i.bak “s/10.0.0.21/${MY_IP}/g” manifests/*
systemctl start docker
systemctl start kubelet
```

### Check if the two new masters joined the cluster 
**on kub01**:
```
watch kubectl get nodes
```
 **It should show something like this, eventually:**
```
NAME          STATUS    ROLES     AGE       VERSION
kub01         Ready     master    1d        v1.8.1
kub02         Ready     <none>    1d        v1.8.1
kub03         Ready     <none>    1d        v1.8.1
kubminion01   Ready     <none>    
### Mark the two new masters as a master node:
```
kubectl patch node kub02 -p ‘{"metadata":{"labels":{"node-role.kubernetes.io/master":""}},"spec":{"taints":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master","timeAdded":null}]}}’

kubectl patch node kub03 -p ‘{"metadata":{"labels":{"node-role.kubernetes.io/master":""}},"spec":{"taints":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master","timeAdded":null}]}}’
```

**Run kubectl get nodes again to verify that all master nodes show as master**

### Setup your nginx prox/lb
```
apt-get install nginx nginx-extras
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
cat <<__EOF__>/etc/nginx/nginx.conf
worker_processes  1;
include /etc/nginx/modules-enabled/*.conf;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
stream {
        upstream apiserver {
            server 10.0.0.21:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 10.0.0.22:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 10.0.0.26:6443 weight=5 max_fails=3 fail_timeout=30s;
            #server ${HOST_IP}:6443 weight=5 max_fails=3 fail_timeout=30s;
            #server ${HOST_IP}:6443 weight=5 max_fails=3 fail_timeout=30s;
        }

    server {
        listen 6443;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass apiserver;
    }
}
__EOF__
```
**make sure to update the IPs in the config to match your env.**

### Update the kube-proxy config so that requests go thorugh your lb:
```
kubectl edit configmap kube-proxy -nkube-system
```
**look for the line starting with “server:” and update the IP/hostname to match your lb. save the file once done.**

### Verify communication:
**On each master**:

```
kubectl get pods --all-namespaces -owide
```
**You should see out similar to this**:
```
```
NAME                            READY     STATUS    RESTARTS   AGE       IP           NODE
kube-apiserver-kub01            1/1       Running   1          1d        10.0.0.21    kub01
kube-apiserver-kub02            1/1       Running   1          1d        10.0.0.22    kub02
kube-apiserver-kub03            1/1       Running   1          1d        10.0.0.26    kub03
kube-controller-manager-kub01   1/1       Running   2          1d        10.0.0.21    kub01
kube-controller-manager-kub02   1/1       Running   1          1d        10.0.0.22    kub02
kube-controller-manager-kub03   1/1       Running   1          1d        10.0.0.26    kub03
kube-dns-545bc4bfd4-mll6z       3/3       Running   3          1d        10.244.0.6   kub01
kube-flannel-ds-46r9v           1/1       Running   1          1d        10.0.0.21    kub01
kube-flannel-ds-5ck84           1/1       Running   4          1d        10.0.0.26    kub03
kube-flannel-ds-f75hz           1/1       Running   2          1d        10.0.0.22    kub02
kube-flannel-ds-kjzdw           1/1       Running   0          1d        10.0.0.23    kubminion01
kube-proxy-6hxlq                1/1       Running   1          1d        10.0.0.26    kub03
kube-proxy-llzd6                1/1       Running   2          1d        10.0.0.21    kub01
kube-proxy-lpx67                1/1       Running   0          1d        10.0.0.23    kubminion01
kube-proxy-s754k                1/1       Running   1          1d        10.0.0.22    kub02
kube-scheduler-kub01            1/1       Running   2          1d        10.0.0.21    kub01
kube-scheduler-kub02            1/1       Running   1          1d        10.0.0.22    kub02
kube-scheduler-kub03            1/1       Running   1          1d        10.0.0.26    kub03
```
