# Kubernetes Permission Manager
Kubernetes uses certificates to manager user permissions. 

To simplify this we'll use permission manager to create our users and assign them permissions.
https://github.com/sighupio/permission-manager/blob/master/docs/installation.md

## Create a namespace for permissions-manager
```
kubectl create namespace permission-manager
```

## Install
To install permission-manager we'll be using the helm chart in this repo.
Download this chart to your local computer. 
Make any necessary changes to for your specific environment. Any changes for your environment should be made there.
```
helm install -n permission-manager permission-manager -f values .
```

## Upgrade
Should there be changes to the values file after the chart has been installed, we can use helm upgrade to update what we have previously installed.
```
helm upgrade -n permission-manager permission-manager -f values .
```
