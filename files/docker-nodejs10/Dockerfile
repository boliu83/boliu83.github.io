FROM oraclelinux:latest
ADD https://boliu83.github.io/files/docker-nodejs10/nodesource-el7.repo /etc/yum.repos.d/nodesource-el7.repo
ADD https://boliu83.github.io/files/docker-nodejs10/NODESOURCE-GPG-SIGNING-KEY-EL /etc/pki/rpm-gpg/NODESOURCE-GPG-SIGNING-KEY-EL
RUN yum install -y nodejs
RUN yum clean all
