# Creating a Kubernetes cluster
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Run as root
```
kubeadm init
```
Save the join command which is outputted by this command

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Alternatively, if you are the root user, you can run:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
From a node:
```
rsync -az mu2cemai-k8s-new-mstr:/home/k8s/.kube /home/k8s
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
# Adding the Weave Net pod network 
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Once a Pod network has been installed, you can confirm that it is working by checking that the CoreDNS Pod is Running
```
kubectl get pods --all-namespaces
```
# Join a cluster
As non-root user
```
scp -r 10.145.94.210:/home/k8s/.kube $HOME/
```
As root on the node VM
```
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
NOTE:
If the original join command is very old, the token may have expired. In this case, you have to generate a new token / join command with the following:

    sudo kubeadm --v=5 token create --print-join-command

To see a list of tokens:

    kubeadm token list

# Setup imagePullSecret
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

## Log in to repository
```
docker login [repository url]
```

When prompted, enter your Docker username and password.

The login process creates or updates a config.json file that holds an authorization token.

```
cat ~/.docker/config.json
```

## Create a Secret based on existing Docker credentials
```
kubectl create secret generic regcred \
--from-file=.dockerconfigjson=<path/to/.docker/config.json> \
--type=kubernetes.io/dockerconfigjson
```

## OR
## Create a Secret by providing credentials on the command line
```
kubectl create secret docker-registry regcred \
--docker-server=<your-registry-server> \
--docker-username=<your-name> \
--docker-password=<your-pword> \
--docker-email=<your-email>
```

## Inspecting the Secret
```
kubectl get secret regcred --output=yaml
```

## Create a Pod that uses your Secret
```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

# Install Ingress Controller - Helm method (recommended)
> https://kubernetes.github.io/ingress-nginx/deploy/#using-helm
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install my-ingress ingress-nginx/ingress-nginx --set controller.service.externalIPs[0]=<An IP address of a master/node in your cluster (I use master)>
```
If the following error occurs: (This may occur later when the ingress service is instantiated)
```
Error: Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post https://ingress-ingress-nginx-controller-admission.default.svc:443/networking/v1beta1/ingresses?timeout=10s: Service Unavailable
```
Run this command:
```
kubectl delete -A ValidatingWebhookConfiguration ingress-ingress-nginx-admission
```
# Install Ingress Controller - Manual method
> https://confluence.opentext.com/pages/viewpage.action?spaceKey=CAPHOME&title=Setup+nginx+ingress+controller
##Download the ingress nginx deployment yaml file 
```
curl -o mandatory.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```
## Create ingress-nginx deployment in the cluster
```
kubectl apply -f mandatory.yaml
```
## Download ingress nginx service (nodeport) yaml file
``` 
curl -o service-nodeport.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```
## Create ingress-nginx service in the cluster
```
kubectl apply -f service-nodeport.yaml
```
## Create ingress-nginx-configmap.yaml
```
cat <<EOF> ingress-nginx-configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data: 
  enable-vts-status: "false"
  use-forwarded-headers: "true"
EOF
```

    kubectl apply -f ingress-nginx-configmap.yaml
    
## Apply ingress-nginx-configmap.yaml in the cluster
```
kubectl apply -f ingress-nginx-configmap.yaml
```
## Get nginx service port
```
kubectl get svc --namespace ingress-nginx
```
# Start some test apps with associated services
```
cat <<EOF> test_apple.yaml
apiVersion: v1
kind: Pod
metadata:
  name: apple-app
  labels:
    app: apple
spec:
  containers:
    - name: apple-app
      image: hashicorp/http-echo
      args:
        - "-text=apple"

---

apiVersion: v1
kind: Service
metadata:
  name: banana-service
spec:
  selector:
    app: banana
  ports:
    - port: 5678 # Default port for image
EOF
```

    kubectl apply -f test_apple.yaml

```
cat <<EOF> test_banana.yaml
apiVersion: v1
kind: Pod
metadata:
  name: banana-app
  labels:
    app: banana
spec:
  containers:
    - name: banana-app
      image: hashicorp/http-echo
      args:
        - "-text=banana"

---

apiVersion: v1
kind: Service
metadata:
  name: banana-service
spec:
  selector:
    app: banana
  ports:
    - port: 5678 # Default port for image
EOF
```

    kubectl apply -f test_banana.yaml

# For HTTPS through ingress
## Generate TLS certificates
For dev environments generate a self-signed certificate with openssl. For production use, you should request a trusted, signed certificate through a provider or your own certificate authority (CA).

The following example generates a 2048-bit RSA X509 certificate valid for 365 days named cemai-ingress-tls.crt. The private key file is named cemai-ingress-tls.key. A Kubernetes TLS secret requires both of these files.

Specify your own organizational values for the -subj parameter:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out cemai-ingress-tls.crt \
    -keyout cemia-ingress-tls.key \
    -subj "/CN=cemai-otxlab.net/O=cemai-ingress-tls"
```
## Create Kubernetes secret for the TLS certificate
To allow Kubernetes to use the TLS certificate and private key for the ingress controller, you create and use a Secret. The secret is defined once, and uses the certificate and key file created in the previous step. You then reference this secret when you define ingress routes.

The following example creates a Secret name cemai-ingress-tls:
```
kubectl create secret tls cemai-ingress-tls \
    --key cemai-ingress-tls.key \
    --cert cemai-ingress-tls.crt
```
# Deploy Ingress
> If you are using https within the cluster you will need to add the tls section in the yaml. If you're connecting to the backend via http you can omit the TLS section.
```
cat <<EOF> test_ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
# Omit if not using https #################################
  tls:
  - hosts:
# Add an entry for each host which will use the TLS cert ##
    - cemai-otxlab.net
    secretName: cemai-ingress-tls
###########################################################
  rules:
  - host: cemai-otxlab.net
    http:
      paths:
        - path: /apple
          backend:
            serviceName: apple-service
            servicePort: 5678
        - path: /banana
          backend:
            serviceName: banana-service
            servicePort: 5678
EOF 
```
# Test ingress
## Get ingress service ports
```
kubectl get svc --namespace ingress-nginx

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.96.248.254   <none>        80:32299/TCP,443:31528/TCP   23m
```
## Connect to test apps
```
curl http://localhost:32299/apple -H 'Host: cemai-otxlab.net'

apple
```
```
curl http://localhost:32299/banana -H 'Host: cemai-otxlab.net'

banana
```
## Test connectivity to Ingress Controller
> https://kubernetes.github.io/ingress-nginx/troubleshooting/
```
kubectl create namespace test

kubectl run test --namespace test --image=tutum/curl -- sleep 10000

kubectl exec --namespace test test -- ls /var/run/secrets/kubernetes.io/serviceaccount/

kubectl exec --namespace test test -- curl -k https://10.96.0.1

TOKEN_VALUE=$(kubectl exec --namespace test test -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)

echo $TOKEN_VALUE

kubectl exec --namespace test test -- curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H  "Authorization: Bearer $TOKEN_VALUE" https://10.96.0.1

kubectl exec --namespace test test -- curl -s -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes/api/v1/namespaces/$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)/pods

kubectl get pods --all-namespaces -o wide

kubectl get secrets

kubectl delete secret default-token-5dhvf eais-ingress-nginx-token-jhpnm my-release-ingress-nginx-admission
```
## Setting up Kubernetes Dashboard
### Install
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

In the cluster, run 
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
On your local pc, run
```
kubectl proxy
```
Kubectl will make Dashboard available at  
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

### Authenticate
https://www.replex.io/blog/how-to-install-access-and-add-heapster-metrics-to-the-kubernetes-dashboard

Create the dashboard service account

```
kubectl create serviceaccount dashboard-admin-sa
```

Next bind the dashboard-admin-service-account service account to the cluster-admin role
```
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
```

When we created the dashboard-admin-sa service account Kubernetes also created a secret for it
```
kubectl get secrets

NAME                                           TYPE                                  DATA   AGE
dashboard-admin-sa-token-ddv9c                 kubernetes.io/service-account-token   3      68s
```

Get the access token
```
kubectl describe secret dashboard-admin-sa-token-kw7vn
```

Copy the token and enter it into the token field on the Kubernetes dashboard login page.


v0.0.1