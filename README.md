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
Install Harbor in the cluster (skip this step if a regestry exists) 

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
# Add environement variables
export DOMAIN=tmclocal.lab.local
export EMAIL_ADDRESS=ogelbrich@vmware.com
#

