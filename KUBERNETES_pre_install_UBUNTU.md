# VM pre-reqs
## Create user k8s
```
adduser k8s
```
## Add user k8s to sudoers
```
vi /etc/sudoers
```
## Fix /etc/apt/apt.conf
Open /etc/apt/apt.conf and add a semicolon (;) to the end of the line
```
vi /etc/apt/apt.conf
```

## (OPTIONAL) Allow root to ssh into machine
Uncomment `#PermitRootLogin prohibit-password` and set to `PermitRootLogin Yes`
```
vi sshd_config
```
Restart sshd
```
systemctl restart sshd
```
# Install Docker CE
These instructions are for installing on Ubuntu 18.4
> https://kubernetes.io/docs/setup/production-environment/container-runtimes/
## Set up the repository:
### Install packages to allow apt to use a repository over HTTPS
```
apt-get update && \
  apt-get install -y apt-transport-https ca-certificates curl software-properties-common gnupg2
```
### Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```
### Add the Docker apt repository:
```
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
## Install Docker CE
```
apt-get update && \
  apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)
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
  "storage-driver": "overlay2"
}
EOF
```
## Allow non-root users to run docker commands
```
chmod 777 /var/run/docker.sock
```
## Add proxy settings to docker
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
## Add certificates to docker to communicate with custom repository
### Go to user's home directory
```
cd /home/k8s
```
### Copy certs from another VM
```
scp -r mu2cemaialpha:/home/docker/certs .
```
### Change ownership of certs forlder to user
```
chown -R k8s:k8s certs
```
### Configure Docker with certificates
```
mkdir -p /etc/docker/certs.d/mu2cemaidocker.otxlab.net
```
```
cd /etc/docker/certs.d/mu2cemaidocker.otxlab.net
```
```
ln -s /home/k8s/certs/cemai.key ca.key
ln -s /home/k8s/certs/cemai.crt ca.crt
ln -s ca.crt ca.cert
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
sysctl --system
```
## Configure apt-get to use https
```
apt-get update && sudo apt-get install -y apt-transport-https curl
```
## Configure apt-get to see Kubernetes repository
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
apt-get update
```
## install Kubernetes
```
apt-get install -y kubelet kubeadm kubectl
```
```
apt-mark hold kubelet kubeadm kubectl
```
## Restat kubelet
```
systemctl daemon-reload
systemctl restart kubelet
systemctl enable kubelet
```
```
systemctl stop ufw
systemctl disable ufw
```

## Turn off swap
```
swapoff -a
vi /etc/fstab
```

```
reboot
```

## kubectl autocomplete
```
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
kubectl completion bash >>  ~/.bash_completion
```
