# Using Azure Files share with GitHub Actions and Kubernetes Tutorial

On this tutorial, you can learn how to use Azure File share to enable caching and dynamic volume creation in your GitHub pipelines when using Self Hosted Agents running on Kubernetes. We are using GitHub ARC - Actions Runner Controller and ARC Runner Scale sets on AKS - Azure Kubernetes Services to manage and scale your self-hosted github runners.

![Solution Architecture](./docs/images/Tutorial_Solution_Architecture_Azure_Files.png)

Learn more here about [GitHub ARC](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/about-actions-runner-controller) and [Azure File share](https://learn.microsoft.com/en-us/azure/aks/azure-files-csi) running on Kubernetes.

On this solution we are using [Metadata caching for premium SMB file shares](https://learn.microsoft.com/en-us/azure/storage/files/smb-performance#metadata-caching-for-premium-smb-file-shares). It's an enhancement for SMB Azure premium file shares aimed to reduce metadata latency, increase available IOPS, and boost network throughput. This preview feature improves the following metadata APIs and can be used from both Windows and Linux clients: Create, Open, Close and Delete.
To onboard, [sign up for the public preview](https://aka.ms/PremiumFilesMetadataCachingPreview) and they will provide you with additional details. Currently this preview feature is only available for premium SMB file shares (file shares in the FileStorage storage account kind). There are no additional costs associated with using this feature.

## Pre-requisites

We are using 3 CLI tools: Azure CLI, Kubectl and Helm. If you are running in CloudShell, these tools are already available there for you.

* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)
* Install [Helm](https://helm.sh/docs/intro/install/)

## Register Azure resource providers

This tutorial provisions AKS, a storage account, virtual networking and user-assigned managed identities. The corresponding Azure resource providers must be registered in your subscription before you begin. On a new subscription these are frequently `NotRegistered`, which causes the deployment commands to fail with misleading errors (for example, `az storage account create` returning `SubscriptionNotFound`).

Register them once per subscription:

```bash
for provider in \
    Microsoft.ContainerService \
    Microsoft.Storage \
    Microsoft.Network \
    Microsoft.Compute \
    Microsoft.ManagedIdentity; do
  az provider register --namespace "${provider}"
done
```

Registration runs asynchronously and can take a few minutes. Confirm every provider reports `Registered` before continuing:

```bash
for provider in \
    Microsoft.ContainerService \
    Microsoft.Storage \
    Microsoft.Network \
    Microsoft.Compute \
    Microsoft.ManagedIdentity; do
  echo "${provider}: $(az provider show --namespace "${provider}" --query registrationState -o tsv)"
done
```

> The keyless (managed identity) Azure Files SMB mount used by this tutorial requires the Azure Files CSI driver **v1.34.0 or later** (included with current AKS releases) and a recent Azure CLI. Run `az upgrade` if the `--enable-smb-oauth` flag is not recognised.

## Defining parameters

Make sure to replace the following mandatory placeholders :

* `AKS_AND_STORAGE_ACCOUNT_RG` with the name of the resource group used by storage account and AKS cluster
* `AKS_CLUSTER_NAME` with the name of the AKS - Azure Kubernetes Services cluster to be created
* `STORAGE_ACCOUNT_NAME` with the name of the storage account to be created
* `AKS_STORAGE_ACCOUNT_LOCATION` with the name of the region to create the resources in. we are deploying on the same region as the AKS cluster nodes to facilitate performance and cost.
* `GITHUB_CONFIG_URL` the URL to GitHub organisation or repository

and these are optional, please keep these default values if possible :

* `NAMESPACE_ARC_CONTROLLER` the name of Kubernetes namespace to run Arc runners scaleset controller
* `ARC_CONTROLLER_NAME` the name of Arc runners scaleset controller
* `NAMESPACE_ARC_RUNNERS` the name of Kubernetes namespace to run Arc self-hosted runners
* `ARC_RUNNER_SCALESET_NAME` the name of Arc runners scaleset
* `ARC_RUNNER_GITHUB_SECRET_NAME` the name of GITHUB secret

```bash
# please fill these env variables with your details
AKS_AND_STORAGE_ACCOUNT_RG="aks-files-actions"
AKS_CLUSTER_NAME="aks-actions"
STORAGE_ACCOUNT_NAME="metadatacaching11"
AKS_STORAGE_ACCOUNT_LOCATION="westus3"
GITHUB_CONFIG_URL="https://github.com/jorgearteiro/azurefiles-actions-aks"

# optional, changes maybe require aditional chnages on ./install/*.yaml files
NAMESPACE_ARC_CONTROLLER="arc-systems"
ARC_CONTROLLER_NAME="arc-controller"
NAMESPACE_ARC_RUNNERS="arc-runners"
ARC_RUNNER_SCALESET_NAME="arc-runner-set"
ARC_RUNNER_GITHUB_SECRET_NAME="arc-runner-github-secret"

# user-assigned managed identities for keyless (identity-based) storage access
CONTROLPLANE_IDENTITY_NAME="aks-files-controlplane-mi"
KUBELET_IDENTITY_NAME="aks-files-kubelet-mi"
```

## Create the resource group

Please follow the [Quickstart: Deploy an Azure Kubernetes Services (AKS) cluster using Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) to create the required Azure Kubernetes Services.

Create the resource group that will hold the AKS cluster, the storage account and the managed identities:

```bash
# Create Resource Group used by AKS and Storage account
az group create --name "${AKS_AND_STORAGE_ACCOUNT_RG}" --location "${AKS_STORAGE_ACCOUNT_LOCATION}"
```

## Create user-assigned managed identities

Instead of authenticating to Azure Storage with a storage account key, this solution uses a **user-assigned managed identity** that is attached to the AKS node pool (the kubelet identity). The Azure Files CSI driver uses this identity to mount SMB shares — both the static NuGet cache share and the dynamically-provisioned ephemeral shares — with no keys or Kubernetes secrets involved.

We create two identities: one for the AKS **control plane** and one for the **kubelet** (nodes). Assigning our own kubelet identity at cluster creation is the most robust option — it survives AKS reconciliation and the CSI driver uses it automatically.

```bash
# Create the control-plane and kubelet (node) managed identities
az identity create -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${CONTROLPLANE_IDENTITY_NAME}"
az identity create -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${KUBELET_IDENTITY_NAME}"

# Capture their resource IDs and principal IDs
CONTROLPLANE_IDENTITY_ID=$(az identity show -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${CONTROLPLANE_IDENTITY_NAME}" --query id -o tsv)
CONTROLPLANE_IDENTITY_PRINCIPAL_ID=$(az identity show -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${CONTROLPLANE_IDENTITY_NAME}" --query principalId -o tsv)
KUBELET_IDENTITY_ID=$(az identity show -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${KUBELET_IDENTITY_NAME}" --query id -o tsv)
KUBELET_IDENTITY_PRINCIPAL_ID=$(az identity show -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${KUBELET_IDENTITY_NAME}" --query principalId -o tsv)

# The control-plane identity must be able to operate (assign) the kubelet identity
az role assignment create \
   --assignee-object-id "${CONTROLPLANE_IDENTITY_PRINCIPAL_ID}" \
   --assignee-principal-type ServicePrincipal \
   --role "Managed Identity Operator" \
   --scope "${KUBELET_IDENTITY_ID}"
```

## Create AKS - Azure Kubernetes Services Cluster

Run the following command to create your AKS Cluster, assigning the managed identities created above:

```bash
# Create AKS Cluster
az aks create -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${AKS_CLUSTER_NAME}" \
       --os-sku AzureLinux \
       --node-count 1 \
       --enable-cluster-autoscaler \
       --min-count 1 \
       --max-count 3 \       
       --node-vm-size standard_d4s_v5 \
       --max-pods=100 \
       --network-plugin azure \
       --network-plugin-mode overlay \
       --enable-managed-identity \
       --assign-identity "${CONTROLPLANE_IDENTITY_ID}" \
       --assign-kubelet-identity "${KUBELET_IDENTITY_ID}" \
       --generate-ssh-keys
```

To manage a Kubernetes cluster, use the Kubernetes command-line client, [kubectl][kubectl]. `kubectl` is already installed if you use Azure Cloud Shell. To install `kubectl` locally, use the `az aks install-cli` command.

1. Configure `kubectl` to connect to your Kubernetes cluster using the `az aks get-credentials` command. This command downloads credentials and configures the Kubernetes CLI to use them.

    ```bash
    az aks get-credentials -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${AKS_CLUSTER_NAME}"
    ```

1. Verify the connection to your cluster using the `kubectl get` command. This command returns a list of the cluster nodes.

    ```bash
    kubectl get nodes
    ```

## Create an Azure file share

Before you can use an Azure Files file share as a Kubernetes volume, you must create an Azure Storage account and the file share. We are using Azure file share Premium SMB with support for metadata caching. The minimal is 100 Gb for each share you create.

This solution uses **identity-based (keyless) authentication**, so the storage account is created with shared key access **disabled** and SMB OAuth **enabled**. Both the static NuGet cache share and the dynamically-provisioned per-job shares are created and mounted using Azure RBAC and the managed identity — no storage account keys are ever used.

1. Create a storage account using the `az storage account create` command. Shared key access is disabled up front so keys can never be used.

    ```bash
    az storage account create -n "${STORAGE_ACCOUNT_NAME}" -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -l "${AKS_STORAGE_ACCOUNT_LOCATION}" \
        --sku Premium_LRS --kind FileStorage \
        --allow-shared-key-access false
    ```

2. Enable SMB OAuth on the storage account so the managed identity can obtain a Kerberos ticket to mount the share. (This requires a recent Azure CLI; upgrade with `az upgrade` if the flag is not recognised.)

    ```bash
    az storage account update -n "${STORAGE_ACCOUNT_NAME}" -g "${AKS_AND_STORAGE_ACCOUNT_RG}" --enable-smb-oauth true
    ```

3. Create the 100Gb premium file share using the management-plane `az storage share-rm create` command, which uses Azure RBAC instead of a storage key. We are using `metadatacaching` as the share name. If you change this name, you also have to change the `arc-runners-set-pv-pvc.yaml` file to reflect this change.

    ```bash
    az storage share-rm create --storage-account "${STORAGE_ACCOUNT_NAME}" -g "${AKS_AND_STORAGE_ACCOUNT_RG}" --name metadatacaching --quota 100
    ```

4. Grant the **kubelet** managed identity the `Storage File Data SMB MI Admin` role on the storage account. This is the role required for managed-identity SMB mounts — other Azure Files data roles are not sufficient.

    ```bash
    STORAGE_ACCOUNT_ID=$(az storage account show -n "${STORAGE_ACCOUNT_NAME}" -g "${AKS_AND_STORAGE_ACCOUNT_RG}" --query id -o tsv)

    az role assignment create \
       --assignee-object-id "${KUBELET_IDENTITY_PRINCIPAL_ID}" \
       --assignee-principal-type ServicePrincipal \
       --role "Storage File Data SMB MI Admin" \
       --scope "${STORAGE_ACCOUNT_ID}"
    ```

5. Grant the AKS **control-plane** managed identity the `Storage Account Contributor` role on the storage account. The Azure Files CSI driver uses the control-plane identity to create the dynamically-provisioned ephemeral shares (see the `github-azurefile` / `github-azurefile-premium` storage classes), and this role lets it do so without storage keys.

    ```bash
    az role assignment create \
       --assignee-object-id "${CONTROLPLANE_IDENTITY_PRINCIPAL_ID}" \
       --assignee-principal-type ServicePrincipal \
       --role "Storage Account Contributor" \
       --scope "${STORAGE_ACCOUNT_ID}"
    ```

## Installing ARC Runners Scaleset Controler

```bash
helm install "${ARC_CONTROLLER_NAME}" \
    --namespace "${NAMESPACE_ARC_CONTROLLER}" \
    --create-namespace \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

## Creating Kubernetes Secrets

### Create the runners namespace

No storage secret is required — Azure Files shares are mounted using the kubelet managed identity (see the storage account steps above). Create the runners namespace that the manifests and GitHub App secret will use:

```bash
kubectl create namespace "${NAMESPACE_ARC_RUNNERS}"
```

### GitHub App Secret

Create a GitHub App to allow the self-hosted runner to access your GitHub organisation or repository. Please follow the instructions [here](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)

These are the parameters provide by GitHub App creation process:

* `GITHUB_APP_ID` Github App ID created
* `GITHUB_APP_INSTALLATION_ID` GitHub App Installation ID created
* `github_app_private_key` replace the whole '-----BEGIN RSA PRIVATE KEY----- section with your Private key

```bash
GITHUB_APP_ID=856120
GITHUB_APP_INSTALLATION_ID=48447618

kubectl create secret generic ${ARC_RUNNER_GITHUB_SECRET_NAME} \
   --namespace=${NAMESPACE_ARC_RUNNERS} \
   --from-literal=github_app_id=${GITHUB_APP_ID} \
   --from-literal=github_app_installation_id=${GITHUB_APP_INSTALLATION_ID} \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA86Cfc3qBK0EiLtFMGVaTGydZc9NuBSir0I1G6iqRXV5bp40N
1ya3v/PMWWnriq8uX2ThZodBBTbD9A8CA/GTuYdUVhWGluACMjJHiQXBB77okwWT
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
s9uqYckJaMLIY6J2lRmodK9ybknmIJt/ji5R1ugBqF9hlW429tSnJg==
-----END RSA PRIVATE KEY-----
'
```

## Azure File share configurations

Azure Files fileshare can be mounted in multiple pods at same time. We can use this capability called AcccessMode: ReadWriteMany to mount the same fileshare in all pods created by the Arc kubernetes replicateset.

We are going to use Azure Files share in 2 different ways:

1. As a persistent SMB File share to cache Nuget packages used by our .NET example Application. The [`arc-runners-set-pv-pvc.yaml`](./install/arc-runners-set-pv.yaml) file will create the required PV and PVC for this File Share. We recomend Azure File Premium for this first option.
Please customize `volumeAttributes` and any `namespaces` parameter on both PV - Persistent Volume and PVC - Persistent volume claim manifests as showed here:

    ```yaml
    volumeAttributes:
      resourceGroup: aks-files-actions  # optional, only set this when storage account is not in the same RG group as node
      storageAccount: metadatacaching11 # storage account name; must match the account created above
      shareName: metadatacaching
      mountWithManagedIdentity: "true"   # keyless SMB mount using the kubelet managed identity (no secret required)
    ```

    ```bash
    # Create PV and PVC on your cluster
    kubectl apply -f ./install/arc-runners-set-pv-pvc.yaml --namespace "${NAMESPACE_ARC_RUNNERS}" --wait
    ```

2. As Ephemeral volume for the GitHub Runners _work folder. We are also going to create 2 storage classes - Azure Files Standard called `github-azurefile` and Azure File Premium called `github-azurefile-premium`. These classes will allow volumes to be created and deleted on demand. When a GitHub Jobs runs, a new runner pod will be created on Kubernetes and a new Azure File share will be created and mounted. The volume will live only during the job run. Standard class allows any volume size and Premium allows a minimum of 100Gb volume. Only one will be used and the decision is yours. Premium will give you a better performance.
The [`arc-runners-storage-class-files.yaml`](./install/arc-runners-storage-class-files.yaml) file can be customized, but not required.

    ```bash
    kubectl apply -f ./install/arc-runners-storage-class-files.yaml --wait
    ```

## Installing ARC Runner Scale Set

Install ARC Runner Scale Set using the official GitHub Helm chart and manually mount your Azure Files share on Kubernetes

This is a code snippet from the [`arc-runners-set-values.yaml`](./install/arc-runners-set-values.yaml) file on the Install folder that can be customized before installing the Runner set Helm Chart. 

We are using a customized version the `Kubernetes` containerMode, to include Azure File share volume mountings to Nuget packages and to ephemeral _work folder volume.

The only not mandantory changes are:

* `storageClassName` choose between "github-azurefile-premium" and "github-azurefile"
* `storage` choose the size of the storage. 100Gb minimum to premium.

The other helm parameters will be set on the helm install command using --set option.

```yaml
containerMode:
  type: "kubernetes"  ## type can be set to dind or kubernetes
  ## the following is required when containerMode.type=kubernetes
  kubernetesModeWorkVolumeClaim:
    accessModes: ["ReadWriteMany"]
    storageClassName: "github-azurefile-premium" # or "github-azurefile" for Standard_LRS
    resources:
      requests:
        storage: 100Gi # 100Gi minimum to premium or any size when using Standard_LRS "github-azurefile" storage class

template:
  spec:
  securityContext:
    fsGroup: 123 # Group used by GitHub default agent image
  containers:
  - name: runner
    image: ghcr.io/actions/actions-runner:latest
    command: ["/home/runner/run.sh"]
    env:
      - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
        value: "false"
      - name: ACTIONS_RUNNER_CONTAINER_HOOK_TEMPLATE
        value: "/home/runner/container-config/container-podspec.yaml"
    volumeMounts:
      - name: "container-podspec-volume"
        mountPath: "/home/runner/container-config"
      - name: azurefile
        mountPath: /home/runner/.nuget/             
  volumes:
    - name: "container-podspec-volume"
      configMap:
        name: hook-extension
    - name: azurefile
      persistentVolumeClaim:
        claimName: azurefile
```

For compatibility with GitHub Workflow container feature that allows you to run containers inside your pipeline, we are mounting a `container-podspec-volume` with the pod spec for the workflow pod created by ARC when running workflows with the container feature. This pod spec is mounted from a config map created on `arc-runners-set-container-pod-spec.yaml` file on the install folder. No changes are required.

```bash
kubectl apply -f ./install/arc-runners-set-container-pod-spec.yaml
```

### ARC Runner Scaleset Helm Chart Parameters

kubectl apply -f ./install/arc-runners-set-container-pod-spec.yaml

The Arc Runner Scaleset Helm Chart provides a few parameters, these are the most important ones to install a Scaleset with Azure File share volume mount on AKS - Azure Kubernetes Services.

These are the parameters:

* `githubConfigUrl` your Github Organisation or repository. We are using repository on our example.
* `githubConfigSecret` the GitHub App Secret to access Github from the self-hosted runner
* `minRunners` Minimal number of runners on the scale set, waiting for new jobs from GitHub
* `maxRunners` Maximal number of runners, running jobs or waiting for new jobs from GitHub
* `runnerGroup` Github runner Group supported used by the Arc runnerset.

Arc runner set helm chart is available to download here `oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set`

To instal the Helm Chart on AKS, please run "helm install" command on your AKS Cluster

```bash
helm install "${ARC_RUNNER_SCALESET_NAME}" \
    --namespace "${NAMESPACE_ARC_RUNNERS}" \
    --create-namespace \
    --values ./install/arc-runners-set-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=3 \
    --set runnerGroup=default \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

### Upgrading a Runner scale set installation

If you want to upgrade any configuration on the Arc Runner Scaleset, re-run the last helm install command onyl replacing the firt line to "helm upgrade --install".

```bash
helm upgrade --install "${ARC_RUNNER_SCALESET_NAME}" \
    --namespace "${NAMESPACE_ARC_RUNNERS}" \
    --create-namespace \
    --values ./install/arc-runners-set-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=3 \
    --set runnerGroup=default \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

## Running your Workflows - GitHub Actions

I have created 3 workflows on this repository, under the default GitHub workflow folder `.github/workflows` for you to test the self-hosted ARC runners created on AKS.

* `.NET Build using containers` install .NET SDK and restore/build/publish application on the runner itself. File name is `dotnet-using-container.yml`
* `.NET Build without containers` use workflow container feature to run a .NET SDK container and build inside the application inside the container. NUGET Caching is mounted by default on this container. File name is `dotnet-wihout-container.yml`
* `Container and Service Test` testing workflows also using containers feature to create a ubuntu container and a redis service. Both containers run on the same AKS Pod. NUGET Caching is also mounted by default on this container. File name is `container-service-test.yml`

All 3 workflows have an input parameter for the Arc runner name to be used on the `runs-on:` field of your workflow. This is the `ARC_RUNNER_SCALESET_NAME="arc-runner-set"` variable defined before, called `arc-runner-set`. To facilitate testing, we are using `workflow_dispatch:` option on the 3 workflows to only run those workflows when it is requested manually. On GitHub Actions tab of your repository, select one of the workflows and click on `Run worflow` button.

Once the workflow is running, it will request a runner to ARC running on AKS cluster. Once this runner, a pod on Kubernetes, is allocated for the job, the workflow will run in there to completion. As we are using the Ephemeral runner approach, the pod running your workflow will be destroyed at the end and a new one will be created for your next workflow run.

## Removing resources

If you want to remove all resources created on your AKS - Azure Kubernetes Cluster, run these commands.

```bash
# Deleting ARC Runner Scalesets
helm delete "${ARC_RUNNER_SCALESET_NAME}" -n "${NAMESPACE_ARC_RUNNERS}" --wait

# Deleting ARC Runners Scaleset Controler
helm delete "${ARC_CONTROLLER_NAME}" -n "${NAMESPACE_ARC_CONTROLLER}" --wait

# Delete Azure File share configurations
kubectl delete -f ./install/arc-runners-set-pv-pvc.yaml --wait
kubectl delete -f ./install/arc-runners-storage-class-files.yaml --wait

# Delete secrets
kubectl delete secret ${ARC_RUNNER_GITHUB_SECRET_NAME} -n arc-runners --wait

# Delete container runner configmap pod spec
kubectl delete -f ./install/arc-runners-set-container-pod-spec.yaml --wait

# Deleting Namespaces
kubectl delete namespace ${NAMESPACE_ARC_RUNNERS}
kubectl delete namespace ${NAMESPACE_ARC_CONTROLLER}
```

If you don't plan on going through this guideline, clean up unnecessary resources to avoid Azure charges. Remove the resource group, AKS container service, Azure File share and all related resources.
