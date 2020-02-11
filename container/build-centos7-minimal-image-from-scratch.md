# Build CentOS 7 docker image from scratch

1. Prepare a clean CentOS 7 build machine

   ```shell
   # enable 'base' and 'updates' repo only
   yum-config-manager --disable \*  --enable base,update
   ```

2. Install Docker and start Docker daemon

   ```
   
   ```

3. Create image folder structure and download minimal packages

   ```shell
   mkdir centos7
   cd centos7
   
   yum install -y --releasever=7 --installroot=$(pwd)/imageroot coreutils
   
   rm -rf $(pwd)/imageroot/var/cache/yum
   ```

4. Create `Dockerfile`

   ```shell
   [root@docker02 centos7]# cat > Dockerfile <<EOF
   FROM scratch
   ADD imageroot /
   EOF
   
   [root@docker02 centos7]# docker build --tag centos:latest-minimal .
   Sending build context to Docker daemon  186.4MB
   Step 1/2 : FROM scratch
    --->
   Step 2/2 : ADD imageroot /
    ---> e65d0f5718f1
   Successfully built e65d0f5718f1
   Successfully tagged centos:latest-minimal
   ```

5. Test

   ```shell
   [root@docker02 centos7]# docker images
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   centos              latest-minimal      e65d0f5718f1        43 seconds ago      182MB
   
   [root@docker02 centos7]# docker run -it --rm centos:latest-minimal /bin/sh
   sh-4.2# cat /etc/centos-release
   CentOS Linux release 7.7.1908 (Core)
   ```

   