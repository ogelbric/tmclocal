# TMC Self-Managed (local)
TMC local / Self-Managed is the on-prem version of TMC (SAS)

# Documentation
https://docs.vmware.com/en/VMware-Tanzu-Mission-Control/1.0/tanzumc-sm-install/prepare-cluster.html

# Step 1: 
Create cluster in TKGs

```
kubectl apply -f https://github.com/ogelbric/tmclocal/raw/main/tmclocalcluster.yaml 
cluster.cluster.x-k8s.io/tmclocalcluster created
```

#Step 2:
Install Harbor in the cluster (skip this step if a registry exists) 

```
mkdir harbor-install && cd $_
# log onto cluster
kubectl vsphere login --server 192.168.2.100 --vsphere-username administrator@vsphere.local --tanzu-kubernetes-cluster-namespace  namespace1000 --tanzu-kubernetes-cluster-name tmclocalcluster --insecure-skip-tls-verify
# Switch context
kubectl config use-context tmclocalcluster
# Create new namespace
kubectl create namespace projectcontour
# I for some reason on this server did not have Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
# Open up the cluster
kubectl apply -f https://github.com/ogelbric/YAML/raw/master/authorize-psp-for-gc-service-accounts.yaml
#
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install ingress bitnami/contour -n projectcontour
# Watch in a different window
watch kubectl get pods -n projectcontour
# Get the IP
kubectl get svc ingress-contour-envoy --namespace projectcontour -w
kubectl describe svc ingress-contour-envoy --namespace projectcontour | grep Ingress | awk '{print $3}'
# Update DNS with IP
# Add a new subdomain tmclocal.lab.local and apply star A record to the above found IP.
```
![GitHub](DNStmclocal.png)
```
# Add environement variables for cert manager
export DOMAIN=tmclocal.lab.local
export EMAIL_ADDRESS=ogelbrich@vmware.com
#
kubectl create namespace cert-manager
helm search repo bitnami | grep cert
helm install cert-manager bitnami/cert-manager --namespace cert-manager  --set installCRDs=true
# Make sure they are running
watch kubectl get pods -n cert-manager
# Generate Certs
# Generate CA files (.crt and .pem)
openssl genrsa -out servercakey.pem
openssl req -new -x509 -key servercakey.pem -out serverca.crt
# Create private key and public key
openssl genrsa -out server.key
openssl req -new -key server.key -out server_reqout.txt
openssl x509 -req -in server_reqout.txt -days 3650 -sha256 -CAcreateserial -CA serverca.crt -CAkey servercakey.pem -out server.crt
cp serverca.crt tls.crt
cp servercakey.pem tls.key
export tlscrt=`cat tls.crt | base64 -w 0`
export tlskey=`cat tls.key | base64 -w 0`
wget https://github.com/ogelbric/tmclocal/raw/main/clusterissuer.yaml.orig
sed "s/changetlscrt/$tlscrt/g" clusterissuer.yaml.orig | sed "s/changetlskey/$tlskey/g" > clusterissuer.yaml
# Apply the cert(s)
kubectl apply -f clusterissuer.yaml -n cert-manager
# Check the output
kubectl get clusterissuers.cert-manager.io -n cert-manager
# NAME           READY   AGE
#  local-issuer   True    3m42s
#
# Install Harbor in a POD
kubectl create namespace harbor
# Create harbor values file
cat << EOF > harbor-values.yaml
harborAdminPassword: Password12345

service:
  type: ClusterIP
  tls:
    enabled: true
    existingSecret: harbor-tls-staging
    notaryExistingSecret: notary-tls-staging

ingress:
  enabled: true
  hosts:
    core: registry.$DOMAIN
    notary: notary.$DOMAIN
  annotations:
    cert-manager.io/cluster-issuer: local-issuer         # use llocal-issuer as the cluster issuer for TLS certs
    ingress.kubernetes.io/force-ssl-redirect: "true"     # force https, even if http is requested
    kubernetes.io/ingress.class: contour                 # using Contour for ingress
    kubernetes.io/tls-acme: "true"                       # using ACME certificates for TLS
externalURL: https://registry.$DOMAIN

portal:
  tls:
    existingSecret: harbor-tls-staging
EOF
#
# Install Harbor
helm install harbor bitnami/harbor -f harbor-values.yaml -n harbor



# Randon Trouble shooting items
# Delete cert manager
helm del  cert-manager
# List bitnami repo
helm search repo bitnami 

