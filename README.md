# Installing Secret Store CSI Driver in a ARO cluster
Azure Key Vault provider for Secrets Store CSI Driver allows you to get secret contents stored in an Azure Key Vault instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods.

We will install secret store CSI driver in a ARO cluster. After the CSI drivers are installed, we will see:
1. How to sync mounted content with Kubernetes secret
2. Set your Environment Variable to Reference a Kubernetes Secret

**Make sure you have the latest [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=dnf) and [oc command-line tool](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz) available**

**How to install Secrets Store CSI Driver and Azure Key Vault Provider on your clusters.**

1. Run this command to set some environment variables to use throughout
~~~
export KEYVAULT_RESOURCE_GROUP=csi-rg
export KEYVAULT_LOCATION=centralindia
export KEYVAULT_NAME=secret-store-$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10 | head -n 1)
export AZ_TENANT_ID=$(az account show -o tsv --query tenantId)
~~~
2. Create an OpenShift Project to deploy the CSI driver
~~~
oc new-project k8s-secrets-store-csi
~~~
3. Set SecurityContextConstraints to allow the CSI driver to run 
~~~
oc adm policy add-scc-to-user privileged system:serviceaccount:k8s-secrets-store-csi:secrets-store-csi-driver
~~~
4. Add the Secrets Store CSI Driver and Azure to your Helm Repositories
~~~
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
~~~
5. Update your Helm Repositories
~~~
helm repo update
~~~
6. Install the secrets store csi driver
~~~
helm install -n k8s-secrets-store-csi csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --version v1.0.1 --set "linux.providersDir=/var/run/secrets-store-csi-providers" --set "syncSecret.enabled=true" --set "enableSecretRotation=true"
~~~
7. Check that the Daemonsets is running
~~~
oc --namespace=k8s-secrets-store-csi get pods -l "app=secrets-store-csi-driver"
~~~
8. Install the Azure Key Vault CSI provider
~~~
helm install -n k8s-secrets-store-csi azure-csi-provider csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --set linux.privileged=true --set secrets-store-csi-driver.install=false --set "linux.providersDir=/var/run/secrets-store-csi-providers" --version=v1.0.1
~~~
9. Set SecurityContextConstraints to allow the CSI driver to run
~~~
oc adm policy add-scc-to-user privileged system:serviceaccount:k8s-secrets-store-csi:csi-secrets-store-provider-azure
~~~
**Create Keyvault and a Secret**
1. Create a namespace for your application
~~~
oc new-project my-application
~~~
2. Create an Azure Keyvault in your Resource Group that contains ARO
~~~
az keyvault create -n ${KEYVAULT_NAME} -g ${KEYVAULT_RESOURCE_GROUP} --location ${KEYVAULT_LOCATION}
~~~
3. Create a secret in the Keyvault. We are creating two secrets here. We will use one to mount as a volume in a pod. We will use other to sync as a environment variable.
~~~
az keyvault secret set --vault-name ${KEYVAULT_NAME} --name secret1 --value "Hello"
az keyvault secret set --vault-name ${KEYVAULT_NAME} --name username --value "testuser"
~~~
4. Create a Service Principal for the keyvault
~~~
export SERVICE_PRINCIPAL_CLIENT_SECRET="$(az ad sp create-for-rbac --skip-assignment --name http://$KEYVAULT_NAME --query 'password' -otsv)"
export SERVICE_PRINCIPAL_CLIENT_ID="$(az ad sp list --display-name http://$KEYVAULT_NAME --query '[0].appId' -otsv)"
~~~
5. Set an Access Policy for the Service Principal
~~~
az keyvault set-policy -n ${KEYVAULT_NAME} --secret-permissions get --spn ${SERVICE_PRINCIPAL_CLIENT_ID}
~~~
6. Create and label a secret for Kubernetes to use to access the Key Vault
~~~
oc create secret generic secrets-store-creds -n my-application --from-literal clientid=${SERVICE_PRINCIPAL_CLIENT_ID} --from-literal clientsecret=${SERVICE_PRINCIPAL_CLIENT_SECRET}
 
oc -n my-application label secret secrets-store-creds secrets-store.csi.k8s.io/used=true
~~~
**Deploy an Application that uses the CSI**
1. Create a SCC which will allow CSI volumes
~~~
oc apply -f https://raw.githubusercontent.com/sureshgaikwad/SecretStoreCSIDriver/master/Deployment/scc.yml
~~~
2. Create a Secret Provider Class to give access to this secret
~~~
oc apply -f https://raw.githubusercontent.com/sureshgaikwad/SecretStoreCSIDriver/master/Deployment/secretproviderclass.yaml
export Secret_Provider_Class=azure-kvname
~~~
3. Create a deployment that uses the above Secret Provider Class. The below yaml file mounts the secret in /mnt/secrets-store/ directory within the pod. It will also set the environment variable "SECRET_USERNAME" and will fetch the value from the secret created in KeyVault.
~~~
oc apply -f https://raw.githubusercontent.com/sureshgaikwad/SecretStoreCSIDriver/master/Deployment/deployment.yaml
~~~
4. Check the Secret is mounted
~~~
oc  exec <pod_name> -- ls /mnt/secrets-store/
~~~
5. Check if the environment variable is set and it's value
~~~
oc exec <pod_name> -- env |grep SECRET_USERNAME
~~~
Since we have enabled auto sync of secrets, if the value is modified from the KeyVault, it will automatically update the secret in ARO cluster. However, you need to restart the pod to reflect the new value. 
