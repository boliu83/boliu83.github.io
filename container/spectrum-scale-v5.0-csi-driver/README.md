# IBM Spectrum Scale Container Interface Driver setup

## Objective
> Use Spectrum Scale as persistent volume in Kubernetes

## Official documents

https://github.com/IBM/ibm-spectrum-scale-csi-driver

https://github.com/IBM/ibm-spectrum-scale-csi-operator

## Steps

### Spectrum Scale and GUI installation

1. Prepare all worker nodes 

   ```
   # install Spectrum scale dependencies on all worker nodes
   yum install -y gcc kernel-devel kernel-headers \
    kernel m4 gcc-c++ ksh numactl net-tools \
    openldap-clients python-xml
   ```

   create /etc/sysctl.d/99-spectrum.conf with content below.  [Reference](https://www.ibm.com/support/knowledgecenter/STXKQY_5.0.4/com.ibm.spectrum.scale.v5r04.doc/bl1ins_netperf.htm)

   ```
   # increase Linux TCP buffer limits
   net.core.rmem_max = 8388608
   net.core.wmem_max = 8388608
   # increase default and maximum Linux TCP buffer sizes
   net.ipv4.tcp_rmem = 4096 262144 8388608
   net.ipv4.tcp_wmem = 4096 262144 8388608
   ```

2. Add all worker nodes to spectrum scale cluster

   ```
   # run all commands from Spectrum Scale admin node
   # /usr/lpp/mmfs/<version>/installer
   
   ./spectrumscale node add worker01.homelab.net
   
   ./spectrumscale install
   ```

3. Create new GUI user for CSI driver and assign it to `CsiAdmin` group

   ```
   [root@scale01 ~]# /usr/lpp/mmfs/gui/cli/mkuser csi-admin -g CsiAdmin
   ```

4. Download Spectrum Scale CSI driver and operator image for x86 on all nodes

   ```bash
   # download images on all worker nodes
   docker pull boliu83/ibm-spectrum-scale-csi:v1.0.0
   docker tag boliu83/ibm-spectrum-scale-csi:v1.0.0 ibm-spectrum-scale-csi:v1.0.0
   
   docker pull boliu83/ibm-spectrum-scale-csi-operator:v0.9.1
   ```

5. Deploy operator

   ```bash
   # master node
   [root@master01 ~]# git clone https://github.com/IBM/ibm-spectrum-scale-csi-operator.git
   
   [root@master01 ~]# cd ./ibm-spectrum-scale-csi-operator/stable/ibm-spectrum-scale-csi-operator-bundle/operators/ibm-spectrum-scale-csi-operator/
   
   # Update your deployment to point at the operator image
   [root@master01 ibm-spectrum-scale-csi-operator]# ./hacks/change_deploy_image.py -i boliu83/ibm-spectrum-scale-csi-operator:v0.9.1
   
   # run commands below to manually deploy operator
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/namespace.yaml
   namespace/ibm-spectrum-scale-csi-driver created
   
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/service_account.yaml
   serviceaccount/ibm-spectrum-scale-csi-operator created
   serviceaccount/ibm-spectrum-scale-csi-attacher created
   serviceaccount/ibm-spectrum-scale-csi-node created
   serviceaccount/ibm-spectrum-scale-csi-provisioner created
   
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/role.yaml
   role.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-operator created
   clusterrole.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-operator created
   clusterrole.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-node created
   clusterrole.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-attacher created
   clusterrole.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-provisioner created
   
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/role_binding.yaml
   rolebinding.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-operator created
   clusterrolebinding.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-operator created
   clusterrolebinding.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-node created
   clusterrolebinding.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-provisioner created
   clusterrolebinding.rbac.authorization.k8s.io/ibm-spectrum-scale-csi-attacher created
   
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/crds/ibm-spectrum-scale-csi-operator-crd.yaml
   customresourcedefinition.apiextensions.k8s.io/csiscaleoperators.csi.ibm.com created
   
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f deploy/operator.yaml
   deployment.apps/ibm-spectrum-scale-csi-operator created
   
   ```

6. Creating secrets for CSI driver
   create `secrets.yaml` file. Use username and password created in previous step. 

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: guisecret
     labels:
       product: ibm-spectrum-scale-csi
   data:
     username: <base64_encoded_username>
     password: <base64_encoded_password>
   ```

   create secrets

   ```bash
   [root@master01 ibm-spectrum-scale-csi-operator]# kubectl apply -f secrets.yaml -n ibm-spectrum-scale-csi-driver
   secret/guisecret created
   ```

7. Obtaining Spectrum Scale cluster id

   ```
   [root@scale01 ~]# mmlscluster
   
   GPFS cluster information
   ========================
     GPFS cluster name:         c01.homelab.net
     GPFS cluster id:           12175908011412921698
     GPFS UID domain:           c01.homelab.net
     Remote shell command:      /usr/bin/ssh
     Remote file copy command:  /usr/bin/scp
     Repository type:           CCR
   
    Node  Daemon node name      IP address       Admin node name       Designation
   --------------------------------------------------------------------------------
      1   scale01.homelab.net   192.168.122.116  scale01.homelab.net   quorum-manager-perfmon
      2   worker01.homelab.net  192.168.122.80   worker01.homelab.net  perfmon
   ```

   Cluster id is `12175908011412921698`

8. Create parent fileset `csi` for persistent volume through Spectrum Scale GUI. 

9.  On K8S master node, update `deploy/crds/ibm-spectrum-scale-csi-operator-cr.yaml` file with details of your cluster and CSI filesystem and fileset

   ```
     ...
     scaleHostpath: "/gpfs"
     ...
     clusters:
       - id: "12175908011412921698"
         secrets: "guisecret"
         secureSslMode: false
         primary:
           primaryFs: "fs01"
           primaryFset: "csi"
         restApi:
         - guiHost: "192.168.122.116"
   ```

10. Start CSI plugin

    ```
    kubectl apply -f 
    ```

    check progress

    ```bash
    [root@master01 ibm-spectrum-scale-csi-operator]# kubectl get pods -n ibm-spectrum-scale-csi-driver
    NAME                                              READY   STATUS    RESTARTS   AGE
    ibm-spectrum-scale-csi-attacher-0                 1/1     Running   0          44s
    ibm-spectrum-scale-csi-n8b6x                      2/2     Running   0          42s
    ibm-spectrum-scale-csi-operator-fb8489bb9-stds7   2/2     Running   0          6m26s
    ibm-spectrum-scale-csi-provisioner-0              1/1     Running   0          43s
    ```

    you can run `kubectl describe pod/<pod_name> -n ibm-spectrum-scale-csi-driver` for more details if any container fails to start



## Setup Dynamic Provision

1. Create storage class named `ibm-spectrum-scale-csi-lt`

   ```bash
   
   [root@master01 ~]# cat > storageclass.yaml <<EOF
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
      name: ibm-spectrum-scale-csi-lt
   provisioner: spectrumscale.csi.ibm.com
   parameters:
       volBackendFs: "fs01"
       volDirBasePath: "csi"
   reclaimPolicy: Delete
   EOF
   
   [root@master01 ~]# kubectl apply -f storageclass.yaml
   storageclass.storage.k8s.io/ibm-spectrum-scale-csi-lt created
   ```

2. Create PVC

   ```bash
   [root@master01 ~]# cat > pvc.yaml <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: scale-lt-pvc
   spec:
     accessModes:
     - ReadWriteMany
     resources:
       requests:
         storage: 1Gi
     storageClassName: ibm-spectrum-scale-csi-lt
   EOF
     
   [root@master01 ~]# kubectl apply -f pvc.yaml
   
   # verify PVC is created and in bound status
   [root@master01 ~]# kubectl get pvc
   NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
   scale-lt-pvc   Bound    pvc-d7a273e8-15af-4970-a253-8cc3a30868f1   1Gi        RWX            ibm-spectrum-scale-csi-lt   10m
   
   # verify sub directory is created under 'csi' fileset
   [root@scale01 csi]# ls -l /gpfs/fs01/csi/
   total 1
   drwxrwx--x. 2 root root 4096 Jan 13 09:40 pvc-d7a273e8-15af-4970-a253-8cc3a30868f1
   ```

   

