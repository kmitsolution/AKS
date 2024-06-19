# Access Secrets from Azure Key Vault in Azure Kubernetes Service
![image](https://github.com/kmitsolution/AKS/assets/84008107/7a54cce9-99c4-4f8c-927a-440ce7dedb3e)
Managing secrets and using secrets in the Azure Kubernetes environment is a very important security aspect. While working with Kubernetes and secrets in particular you will know that they are not secure by default. In fact, they are just base64 encoded strings which can be easily decoded. You can use public and private keys to keep your secrets safe and then allow Kubernetes to access them but that will be an extra overhead to manage them.

Azure supports the possibility to get secrets into an Azure key Vault, from AKS, by using the Secret Store CSI (Container Storage Interface) Driver. AKS clusters may need to store and retrieve secrets, keys, and certificates. The Secrets Store CSI Driver provides cluster support to integrate with Key Vault. When enabled and configured secrets, keys, and certificates can be securely accessed from a pod. In this article, we will discuss a secure way of using secrets that are stored in Azure Key Vault into your Azure Kubernetes Cluster(AKS).

## Create Azure Key-Vault and Secret (Role will be keyvault Administrator for the user)
To create an Azure Key Vault and add a secret you need to run the following commands.

```
#To Create KeyVault
az keyvault create --name aksdemocluster-kv321 --resource-group aksgroup --location eastus

# To Create Secret inside the keyVault
az keyvault secret set --vault-name aksdemocluster-kv321 --name mysql-password --value Test123
```

## Enable Secrets Store CSI Driver support
To upgrade an existing AKS cluster with Azure Key Vault Provider for Secrets Store CSI Driver capability run the following command:
```
az aks enable-addons --addons azure-keyvault-secrets-provider --name akscluster --resource-group aksgroup
```
## Verify the Secrets Store CSI Driver installation
Verify that the installation is finished by listing all pods that have the ```secrets-store-csi-driver``` and ```secrets-store-provider-azure``` labels in the kube-system namespace, and ensure that your output looks similar to the output shown here:

```
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure

```

## Verify system-assigned identity on VMs
Enable the ```identity``` in the vmss(select vmss of aks cluster-->Identity-->System Assigned and Trun it on, it will show the principalid . Verify that your virtual machine scale set or availability set nodes have their own system-assigned identity:

```
az vmss identity show -g MC_akscluster-rg_aksdcluster_eastus -n aks-agentpool-32528728-vmss -o yaml

```

## Assigns Permissions
To grant your identity permissions that enable it to read your key vault and view its contents, run the following commands:
```
Azure Conosole Goto KeyVault --> aksdemocluster-kv321 --> Add Access Policies --> Give permission to VMSS principal id(The one which is created in a step above)
# set policy to access secrets in your key vault 
az keyvault set-policy -n <keyvault-name> — secret-permissions get — spn <identity-principal-id>
```

## Create Secret Provider Class
Now it’s time to actually get secrets from the Key Vault. We have to be explicit define and create a Secret Provider Class, where we define all the secrets we need to import from the Key Vault.
```
# This is a SecretProviderClass example using system-assigned identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-system-msi
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"    # Set to true for using managed identity
    userAssignedIdentityID: ""      # If empty, then defaults to use the system assigned identity on the VM
    keyvaultName: aksdemocluster-kv3214
    cloudName: "AzurePublicCloud"                   # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: mysql-password
          objectType: secret        # object types: secret, key, or cert
          objectVersion: ""         # [OPTIONAL] object versions, default to latest if empty
    tenantId: XXXXXXXXXXXX          # The Directory ID of the key vault
```
```
kubectl get secretproviderclass
```

## Create POD with Secrets
By deploying the Secret Provider Class, the secrets will not be created in Kubernetes yet. This will only happen with the first pod, which mounts a volume utilizing CSI and referencing our Secret Provider Class. Also, the pod must make use of the selector that we specified in the Azure Identity Binding.

```
# This is a sample pod definition for using SecretProviderClass and system-assigned identity to access your key vault
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline-system-msi
spec:
  containers:
    - name: nginx-secret
      image: nginx
     
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname-system-msi"
```

## Validate the secrets
After the pod starts, the mounted content at the volume path that you specified in your deployment YAML is available.

```
## show secrets held in secrets-store 
kubectl exec busybox-secrets-store-inline -- ls /mnt/secrets-store/ 

## print a test secret 'ExampleSecret' held in secrets-store 
kubectl exec busybox-secrets-store-inline -- cat /mnt/secrets-store/mysql-password
``` 
As you can see, it is a recommended way to manage secrets in AKS Cluster and all the values will be stored in a key vault (key, secret or certificate). The benefit here is that you can provide access to a specific secret in a specific key vault. And all of this, without managing the public and private keys.


