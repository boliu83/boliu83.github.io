
# CentOS 7 K8S installation



## Prepare System

```bash
setenforce 0
sed -i 's/SELINUX/SELINUX=permissive/' /etc/selinux/config

systemctl disable --now firewalld

swapoff -a

yum install -y device-mapper-persistent-data lvm2

```


## Install Docker stable

```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io 



cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF


mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload
systemctl enable --now docker
```

## Install Kubeadm - common

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
EOF

sysctl --system

cat <<EOF > /etc/yum.repos.d/kubernetes.repo  

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum -y install kubeadm kubelet kubectl


# enable without starting kubelet
systemctl enable kubelet
```

## Master node configuration

```shell
kubeadm init  --pod-network-cidr=10.244.0.0/16

# take node of the join command similar to the one below
# kubeadm join 192.168.122.44:6443 --token b2rqqk.9uqbmqausz43edgn \
#    --discovery-token-ca-cert-hash sha256:2f2d5bce2edd3f26ed6275887355595795e2144e1f31c86ca7421ff17b1f7c00


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify
```shell
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES    AGE     VERSION
master   NotReady   master   4m51s   v1.18.5
```

### Network Configuration - flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Verify
```shell
[root@master ~]# kubectl get pods -A
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-kl4db         1/1     Running   0          7m25s
kube-system   coredns-66bff467f8-r88h2         1/1     Running   0          7m25s
kube-system   etcd-master                      1/1     Running   0          7m35s
kube-system   kube-apiserver-master            1/1     Running   0          7m35s
kube-system   kube-controller-manager-master   1/1     Running   0          7m34s
kube-system   kube-flannel-ds-amd64-jgqw6      1/1     Running   0          40s
kube-system   kube-proxy-fdgnk                 1/1     Running   0          7m25s
kube-system   kube-scheduler-master            1/1     Running   0          7m35s
```

## Worker node configuration

Perform Kubeadm - common tasks

Run `kubeadm join` command saved from master node

```bash
# in case you need to get join command again
# run command below on master node
kubeadm token create --print-join-command
```

Verify
```shell
# on master node
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   18m   v1.18.5
worker   Ready    <none>   18s   v1.18.5
```


## Have some fun

```shell

# create a test deployment
[root@master ~]# kubectl create deployment --image nginx test-nginx
deployment.apps/test-nginx created

[root@master ~]# kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
test-nginx   1/1     1            1           7s

[root@master ~]# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
test-nginx-7b7d9954bd-bjv8l   1/1     Running   0          87s

# expose service using ClusterIP as service
# Type for service: ClusterIP, NodePort, LoadBalancer, or ExternalName. Default is 'ClusterIP'

kubectl expose deploy/test-nginx --port 80 --type NodePort


[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        32m
test-nginx   NodePort    10.109.198.159   <none>        80:32169/TCP   5m36s

# test
[root@master ~]# curl worker:32169
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
....
```

## Install MetalLB
[https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Create layer 2 config map `config.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
        - 192.168.122.30-192.168.122.49
```

Apply config map

```shell
kubectl apply -f config.yaml
```

Verify

```
# now change the svc test-nginx type to LoadBalancer
[root@master ~]# kubectl edit svc/test-nginx
service/test-nginx edited

# pay attention to EXTERNAL-IP 
[root@master ~]# kubectl get svc
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1        <none>           443/TCP        39m
test-nginx   LoadBalancer   10.109.198.159   192.168.122.30   80:32169/TCP   12m

# test from host outside of this cluster
[bo@nuc ~]$ curl 192.168.122.30
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...
```
