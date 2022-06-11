# SecretStoreCSIDriver
Azure Key Vault provider for Secrets Store CSI Driver allows you to get secret contents stored in an Azure Key Vault instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods.

**How to install Secrets Store CSI Driver and Azure Key Vault Provider on your clusters.**

Run this command to set some environment variables to use throughout
~~~
export KEYVAULT_RESOURCE_GROUP=csi-rg
export KEYVAULT_LOCATION=centralindia
export KEYVAULT_NAME=secret-store-$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10 | head -n 1)
export AZ_TENANT_ID=$(az account show -o tsv --query tenantId)
~~~
We are using kube-system namespace in this example to install the CSI driver

~~~
helm install -n k8s-secrets-store-csi csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --version v1.0.1 --set "linux.providersDir=/var/run/secrets-store-csi-providers" --set "syncSecret.enabled=true" --set "enableSecretRotation=true"
~~~
