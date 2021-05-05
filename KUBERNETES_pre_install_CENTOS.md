# Install Docker CE
These instructions are for installing on CentOS 7
> https://kubernetes.io/docs/setup/production-environment/container-runtimes/
## Set up the repository
### Install required packages
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
### Add the Docker repository
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
## Install Docker CE
```
yum update -y && yum install -y containerd.io-1.2.13 docker-ce-19.03.8 docker-ce-cli-19.03.8
```
## Create /etc/docker
```
mkdir /etc/docker
```
## Set up the Docker daemon
```
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
```
```
mkdir -p /etc/systemd/system/docker.service.d
```
```
cat <<EOF >/etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://mu2-prox01-l001.otxlab.net:3128/"
Environment="HTTPS_PROXY=http://mu2-prox01-l001.otxlab.net:3128/"
Environment="NO_PROXY=localhost,127.0.0.1/8,::1,.otxlab.net,10.0.0.0/8"
EOF
```
## Restart Docker
```
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```
```
systemctl show --property=Environment docker
```

# Install kubeadm
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
## Letting iptables see bridged traffic
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
sudo sysctl --system
```
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
```
setenforce 0
```
```
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
```
yum install -y kubelet kubeadm kubectl
```
```
systemctl daemon-reload
systemctl restart kubelet
systemctl enable kubelet
```
```
systemctl stop firewalld
systemctl disable firewalld
```

## Remove swap from /etc/fstab
```
swapoff -a
vi /etc/fstab
```

```
reboot
```
