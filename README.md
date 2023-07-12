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
