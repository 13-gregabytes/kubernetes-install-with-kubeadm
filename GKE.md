## Install the Google Cloud SDK
https://cloud.google.com/sdk/docs/install
```
sudo echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
sudo sudo apt-get install apt-transport-https ca-certificates gnupg
sudo curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install -y google-cloud-sdk
gcloud init
```

## Logging into gcloud
```
gcloud auth login
```

## get kubeconfig
Configure kubectl command line access by running the following command:
```
gcloud container clusters get-credentials otl-eng-dx-eais-gke-cluster-1 --zone us-east4-a --project otl-eng-dx-eais
```

## Install Ingress in the cluster
https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer
https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balance-ingress
```
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```

v0.0.1