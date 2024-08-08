Para criar um cenário no Azure usando o AKS (Azure Kubernetes Service) e testar as habilidades de troubleshooting de um candidato, você pode seguir os passos abaixo. A ideia é reproduzir o ambiente em que uma conta de serviço do Kubernetes (ServiceAccount) está configurada com permissões incorretas para acessar um recurso do Azure (como o Blob Storage), e o candidato precisa identificar e corrigir o problema.

### Passos para Configurar o Cenário no Azure

#### 1. **Criar um Cluster AKS**

Primeiro, vamos criar um cluster AKS. Você pode usar o Azure CLI para isso.

```bash
# Definir variáveis
RESOURCE_GROUP="MyResourceGroup"
CLUSTER_NAME="MyAKSCluster"
LOCATION="eastus"

# Criar um grupo de recursos
az group create --name $RESOURCE_GROUP --location $LOCATION

# Criar um cluster AKS
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --enable-managed-identity \
    --generate-ssh-keys
```

#### 2. **Configurar o Acesso ao Cluster AKS**

Depois que o cluster for criado, você pode configurar o kubectl para acessar o cluster.

```bash
# Obter credenciais do cluster AKS
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

#### 3. **Criar uma ServiceAccount e Associar uma Identidade Gerenciada do Azure**

Em vez de uma IAM Role (como na AWS), no Azure, você usará uma Identidade Gerenciada (Managed Identity) que será associada à ServiceAccount no AKS. Vamos criar uma identidade gerenciada e uma conta de serviço Kubernetes, e então associá-los.

```bash
# Criar uma Identidade Gerenciada
IDENTITY_NAME="MyManagedIdentity"
az identity create --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP

# Obter o ID da Identidade Gerenciada
IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query 'clientId' --output tsv)
```

No Kubernetes, crie a ServiceAccount e associe-a à Identidade Gerenciada:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<IDENTITY_CLIENT_ID>"
```

**Observação:** O valor `<IDENTITY_CLIENT_ID>` deve ser substituído pelo ID da Identidade Gerenciada que você criou anteriormente.

#### 4. **Configurar o Acesso ao Blob Storage (Análogo ao S3)**

Agora, configure o acesso ao Azure Blob Storage (equivalente ao S3). Primeiro, crie uma conta de armazenamento e um container:

```bash
STORAGE_ACCOUNT_NAME="myaksstorage$(date +%s)"
CONTAINER_NAME="mycontainer"

# Criar uma conta de armazenamento
az storage account create --name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP --location $LOCATION --sku Standard_LRS

# Criar um container de blob
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
```

Agora, associe a Identidade Gerenciada à Role "Storage Blob Data Reader", mas de propósito configure-a com o escopo errado ou uma política inadequada, para simular um erro.

```bash
# De propósito, use um escopo de recursos incorreto ou omita as permissões necessárias
az role assignment create --role "Storage Blob Data Reader" --assignee $IDENTITY_CLIENT_ID --scope /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP
```

#### 5. **Deploy de um Pod com a ServiceAccount**

Por fim, crie um deployment no AKS que tenta acessar o Blob Storage usando a ServiceAccount configurada.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-reader
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blob-reader
  template:
    metadata:
      labels:
        app: blob-reader
    spec:
      serviceAccountName: my-service-account
      containers:
      - name: blob-reader
        image: mcr.microsoft.com/azure-cli
        command: ["/bin/sh", "-c"]
        args:
        - |
          az storage blob list --container-name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
```

#### 6. **Objetivo do Candidato**

O candidato deverá:
- Identificar por que o pod falha ao acessar o Blob Storage (devido à permissão incorreta).
- Corrigir a associação da Identidade Gerenciada com o escopo ou permissão correta.
- Verificar se o pod consegue acessar o Blob Storage com sucesso após a correção.

Esse cenário não só testa a habilidade do candidato em Kubernetes e Azure, mas também em troubleshooting de permissões e integração entre serviços de nuvem.

Referencias:

https://learn.microsoft.com/pt-br/azure/aks/learn/quick-kubernetes-deploy-cli?source=recommendations

https://learn.microsoft.com/pt-br/entra/identity/managed-identities-azure-resources/overview

https://learn.microsoft.com/pt-br/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations

https://learn.microsoft.com/pt-br/azure/aks/use-managed-identity

https://learn.microsoft.com/pt-br/azure/storage/blobs/assign-azure-role-data-access?tabs=portal





