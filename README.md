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
  type: Ingress
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
#
helm install harbor bitnami/harbor -f ./harbor-values.yaml -n harbor
#
# Makre sure POD(s) are up
#
watch kubectl get pods -n harbor
#
# Look at the requested certs
#
kubectl -n harbor get certificate
#

# some how the above harbor values file does not generate in my environment the proper ingress service and I had to edit it by hand
# with below output
#
k edit ingress harbor-ingress  -n harbor
k edit ingress harbor-ingress-notary  -n harbor
kubectl get ingress -A
# harbor      harbor-ingress          <none>   registry.tmclocal.lab.local   192.168.2.105   80, 443   21h
# harbor      harbor-ingress-notary   <none>   notary.tmclocal.lab.local     192.168.2.105   80, 443   21h
#
k get svc -A | grep ingress
# projectcontour      ingress-contour                   ClusterIP      198.48.119.233   <none>          8001/TCP                     5d
# projectcontour      ingress-contour-envoy             LoadBalancer   198.63.142.20    192.168.2.105   80:30106/TCP,443:31682/TCP   5d


#
# New try via tanzu to install harbor
#
# Tanzu CLI
https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/using-tkg-22/install-cli.html
#
# Tanzu Reop
#
tanzu package repository add tanzu-standard --url projects.registry.vmware.com/tkg/packages/standard/repo:v2.2.0 --namespace tkg-system
#
# List all the packages
#
tanzu package available list -A
tanzu package available list harbor.tanzu.vmware.com -A
#
#  NAMESPACE   NAME                     VERSION               RELEASED-AT                    
#  tkg-system  harbor.tanzu.vmware.com  2.2.3+vmware.1-tkg.1  2021-07-07 14:00:00 -0400 EDT  
#  tkg-system  harbor.tanzu.vmware.com  2.2.3+vmware.1-tkg.2  2021-07-07 14:00:00 -0400 EDT  
#  tkg-system  harbor.tanzu.vmware.com  2.3.3+vmware.1-tkg.1  2021-09-28 02:05:00 -0400 EDT  
#  tkg-system  harbor.tanzu.vmware.com  2.5.3+vmware.1-tkg.1  2021-09-28 02:05:00 -0400 EDT  
#  tkg-system  harbor.tanzu.vmware.com  2.7.1+vmware.1-tkg.1  2021-09-28 02:05:00 -0400 EDT  
#
# note + changes to _
imgpkg pull -b projects.registry.vmware.com/tkg/packages/standard/harbor:v2.7.1_vmware.1-tkg.1 -o /tmp/harbor-package-v2.7.1_vmware.1-tkg.1
#

cp /tmp/harbor-package-v2.7.1_vmware.1-tkg.1/config/values.yaml harbor-data-values.yaml
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq && chmod +x /usr/bin/yq

bash /tmp/harbor-package-v2.7.1_vmware.1-tkg.1/config/scripts/generate-passwords.sh harbor-data-values.yaml
sed -i 's/hostname: harbor.yourdomain.com/hostname: registry.tmclocal.lab.local/g' harbor-data-values.yam
SE=$(kubectl get sc | grep def | awk '{ print $1 }')
sed -i "s/storageClass: \"\"/storageClass: \"$SE\"/g" harbor-data-values.yaml
yq -i eval '... comments=""' harbor-data-values.yaml

tanzu package install harbor \
--package harbor.tanzu.vmware.com \
--version 2.7.1+vmware.1-tkg.1 \
--values-file harbor-data-values.yaml \
--namespace harbor

# Randon Trouble shooting items
# Delete cert manager
helm del  cert-manager
helm del  harbor -n harbor
# List bitnami repo
helm search repo bitnami 
helm del harbor -n harbor
helm del ingress -n projectcontour  
helm del cert-manager -n cert-manager
 
