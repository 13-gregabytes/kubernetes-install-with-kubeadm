# Installing Helm
> https://helm.sh/docs/intro/install/

## Download your desired version
    https://github.com/helm/helm/releases

    curl https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz -o helm-v3.2.4-linux-amd64.tar.gz

## Unpack it

    tar -zxvf helm-v3.2.4-linux-amd64.tar.gz

## Add it to a path directory
Find the helm binary in the unpacked directory, and move it to its desired destination

    mkdir /opt/helm
    mv linux-amd64/ /opt/helm/
    ln -s /opt/helm/linux-amd64/helm /usr/local/bin/helm

## Initialize a Helm Chart Repository
Official Helm stable charts

    helm repo add stable https://kubernetes-charts.storage.googleapis.com/ (no longer available)
    
    helm repo add stable https://charts.helm.sh/stable

## List the charts you can install

    helm search repo stable

## Add the artifactory helm charts repository
  
    helm repo add experience-ai https://muartprd01.opentext.net/artifactory/experience-ai-helm/

v0.0.1