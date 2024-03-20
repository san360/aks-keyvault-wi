# What's implemented
- AKS cluster should exist
- Enable OIDC issuer and workload identity (WI deploys pod in kube-system namespace)
- Enable secret store provider (Creates pod in kube-system namespace)
- Create UAMI for accessing Key Vault
- Create namespace in Kubernetes
- Create federation with Kubernetes namespace and service account (Refer to service YAML file)
- Create Key Vault and assign role to created UAMI
- Create dummy secret
- Deploy service account YAML
- Deploy secret provider YAML
- Deploy pods
- Execute in pod and validate the mounted credentials or environment variable secrets
- Volume mount is mandatory in pod for Kubernetes secrets to be created

The approach would be to have Key Vault for each application deployed in Kubernetes and store their respective secrets in Key Vault, for example, MySQL DB connection string required for PHP app. This approach is being **used only for PHP based as there is no native Azure SDK available as of today**. For other languages for e.g python, nodejs, we can use native workload identity to access Azure services. To test follow the below steps in order

### Set variables
```
resourceGroup="aks-cluster"
clusterName="san360-vnet"
location="northeurope"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
az account set --subscription "${SUBSCRIPTION}"
```

### Update the AKS cluster to enable OIDC
```
az aks update --resource-group $resourceGroup --name $clusterName --enable-oidc-issuer --enable-workload-identity
```

### Enable secret store csi driver
```
az aks enable-addons --addons azure-keyvault-secrets-provider --name $clusterName --resource-group $resourceGroup
```

### Export oidc issuer url 
```
export AKS_OIDC_ISSUER="$(az aks show -n $clusterName -g $resourceGroup --query "oidcIssuerProfile.issuerUrl" -o tsv)"
```

### Create a user assigned managed identity
```
az identity create --name keyvaultaccessapp1 --resource-group $resourceGroup --location $location --subscription "${SUBSCRIPTION}"
az identity create --name keyvaultaccessapp2 --resource-group $resourceGroup --location $location --subscription "${SUBSCRIPTION}"
```

### Validate the identity
```
az identity show --resource-group $resourceGroup --name keyvaultaccessapp1 --query 'clientId' -o tsv
az identity show --resource-group $resourceGroup --name keyvaultaccessapp2 --query 'clientId' -o tsv
```

### Create namespace
```
kubectl create ns accessapp1
kubectl create ns accessapp2
```

### Create managed identity
```
az identity federated-credential create --name accessapp1 --identity-name keyvaultaccessapp1 --resource-group $resourceGroup --issuer "${AKS_OIDC_ISSUER}" --subject system:serviceaccount:accessapp1:accessapp1 --audience api://AzureADTokenExchange

az identity federated-credential create --name accessapp2 --identity-name keyvaultaccessapp2 --resource-group $resourceGroup --issuer "${AKS_OIDC_ISSUER}" --subject system:serviceaccount:accessapp2:accessapp2 --audience api://AzureADTokenExchange
```

### Create a new Azure key vault
```
az keyvault create -n accessapp1vault -g  $resourceGroup -l $location --enable-rbac-authorization
az keyvault create -n accessapp2vault -g  $resourceGroup -l $location --enable-rbac-authorization
```

### Create role assignment for keyvault
```
export accessapp1uami="$(az identity show --resource-group $resourceGroup --name keyvaultaccessapp1 --query 'clientId' -o tsv)"
export accesapp1keyvault="$(az keyvault show --name accessapp1vault --query id -o tsv)"
az role assignment create --role "Key Vault Secrets User" --assignee $accessapp1uami --scope $accesapp1keyvault --subscription "${SUBSCRIPTION}"

export accessapp2uami="$(az identity show --resource-group $resourceGroup --name keyvaultaccessapp2 --query 'clientId' -o tsv)"
export accesapp2keyvault="$(az keyvault show --name accessapp2vault --query id -o tsv)"
az role assignment create --role "Key Vault Secrets User" --assignee $accessapp2uami --scope $accesapp2keyvault --subscription "${SUBSCRIPTION}"
```

### Set dummy secret (Assign yourself as admin to keyvault)
```
az keyvault secret set --vault-name accessapp1vault -n ExampleSecret --value MyAKSExampleSecret
az keyvault secret set --vault-name accessapp2vault -n ExampleSecret --value MyAKSExampleSecret
```

### Deploy Kubernetes service account yaml
```
kubectl apply -f kube-service-account.yaml
```

### Deploy Kubernetes secret provider yaml
```
kubectl apply -f secret-prodvider.yaml
```

### Deploy Kubernetes pod yaml
```
 kubectl apply -f kube-pods.yaml
```