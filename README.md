# Using Azure Files with GitHub Actions and Kubernetes
Learn how to install GitHub ARC - Actions Runner Controller and ARC Runner Scale sets on AKS - Azure Kubernetes Services to manage and scale your self-hosted github runners.

## Before you begin

Please follow the [Quickstart: Deploy an Azure Kubernetes Services (AKS) cluster using Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) to create the required Azure Kubernetes Services. We are using 3 CLI tools: az CLI, Kubectl and Helm

## Defining parameters

Make sure to replace the following mandatory placeholders :

* `STORAGE_ACCOUNT_RG` with the name of the resource group that the storage account should be created
* `STORAGE_ACCOUNT_NAME` with the name of the storage account
* `STORAGE_ACCOUNT_LOCATION` with the name of the region to create the resource in. It should be the same region as the AKS cluster nodes to facilitate performance and cost.
* `GITHUB_CONFIG_URL` the URL to GitHub organisation or repository

```bash
STORAGE_ACCOUNT_RG="metadata-agroves"
STORAGE_ACCOUNT_NAME="metadatacaching11"
STORAGE_ACCOUNT_LOCATION="uswest3"
GITHUB_CONFIG_URL="https://github.com/jorgearteiro/azurefiles-actions-aks"
```

and these are optional, please keep these default values if possible :

* `NAMESPACE_ARC_CONTROLLER` the name of Kubernetes namespace to run Arc runners scaleset controller
* `ARC_CONTROLLER_NAME` the name of Arc runners scaleset controller
* `NAMESPACE_ARC_RUNNERS` the name of Kubernetes namespace to run Arc self-hosted runners
* `ARC_RUNNER_SCALESET_NAME` the name of Arc runners scaleset
* `ARC_RUNNER_GITHUB_SECRET_NAME` the name of GITHUB secret

```bash
NAMESPACE_ARC_CONTROLLER="arc-systems"
ARC_CONTROLLER_NAME="arc-controler"
NAMESPACE_ARC_RUNNERS="arc-runners"
ARC_RUNNER_SCALESET_NAME="arc-runner-set"
ARC_RUNNER_GITHUB_SECRET_NAME="arc-runner-github-secret"
```

## Create an Azure file share

Before you can use an Azure Files file share as a Kubernetes volume, you must create an Azure Storage account and the file share. We are using Azure file share Premium SMB with support for metadata caching. The minimal is 100 Gb for each share you create.

1. Create a storage account using the `az storage account create` command with the `--sku` parameter. The following command creates a storage account using the `Premium_LRS` SKU.

    ```bash
    az storage account create -n "${STORAGE_ACCOUNT_NAME}" -g "${STORAGE_ACCOUNT_RG}" -l "${STORAGE_ACCOUNT_LOCATION}" --sku Premium_LRS
    ```

2. Export the connection string as an environment variable using the following command, which you use to create the file share.

    ```bash
    export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n storageAccountName -g resourceGroupName -o tsv)
    ```

3. Create the premium file share using the `az storage share create` command. We are using `metadatacaching` as share name. If you change this name, you also have to change `arc-runners-set-pv.yaml` file to reflect this change.

    ```bash
    az storage share create -n metadatacaching --connection-string $AZURE_STORAGE_CONNECTION_STRING
    ```

## Installing ARC Runners Scaleset Controler

```bash
helm install "${ARC_CONTROLLER_NAME}" \
    --namespace "${NAMESPACE_ARC_CONTROLLER}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

## Creating Kubernetes Secrets


### Azure File Share Storage Key Secret
```bash
STORAGE_KEY=$(az storage account keys list --resource-group ${STORAGE_ACCOUNT_RG} --account-name ${STORAGE_ACCOUNT_NAME} --query "[0].value" -o tsv)

kubectl create namespace ${NAMESPACE_ARC_RUNNERS}

kubectl create secret generic azure-storage-secret \
   --namespace ${NAMESPACE_ARC_RUNNERS} \
   --from-literal=azurestorageaccountname=${STORAGE_ACCOUNT_NAME} \
   --from-literal=azurestorageaccountkey=${STORAGE_KEY} 
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

## Create Azure Files PV - Presistent Volume and PVC - Persistent Volume Claim

Azure Files fileshare can be mounted in multiple pods, we can use this capability called AcccessMode: ReadWriteMany to mount the same fileshare in all pods created by the Arc kubernetes replicateset.

Please manually customize [`arc-runners-set-pv.yaml`](./install/arc-runners-set-pv.yaml) and [`arc-runners-set-pvc.yaml`](./install/arc-runners-set-pvc.yaml) on the install folder, before running these kubectl apply commands. The required customizations are `volumeAttributes` as described here and also any `namespaces` parameter on both PV and PVC files.
```yaml
volumeAttributes:
  resourceGroup: metadata-agroves  # optional, only set this when storage account is not in the same RG group as node
  shareName: metadatacaching
```

After [`arc-runners-set-pv.yaml`](./install/arc-runners-set-pv.yaml) and [`arc-runners-set-pvc.yaml`](./install/arc-runners-set-pvc.yaml) files customzations are complete, please apply the manifests:

```bash
kubectl apply -f ./install/arc-runners-set-pv.yaml --wait
kubectl apply -f ./install/arc-runners-set-pvc.yaml --wait
```

## Installing ARC Runner Scale Set

Install ARC Runner Scale Set using the official GitHub Helm chart and manually mounting your Azure Files share on Kubernetes

This is a code snippet from the [`arc-runners-set-values.yaml`](./install/arc-runners-set-values.yaml) file on the Install folder. We are using a customized version the `Kubernetes` containerMode, to include AzureFile volume mounting. The other helm parameters will be set on the helm install command using --set option.

```yaml
template:
  spec:
    securityContext:
      FSGroup: 1001
    containers:
    - name: runner
      image: ghcr.io/actions/actions-runner:latest
      command: ["/home/runner/run.sh"]
      env:
        - name: ACTIONS_RUNNER_CONTAINER_HOOKS
          value: /home/runner/k8s/index.js
        - name: ACTIONS_RUNNER_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
          value: "false"         
      volumeMounts:
        - name: work
          mountPath: /home/runner/_work
        - name: azurefile
          mountPath: /home/runner/.nuget/             
    volumes:
      - name: work
        emptyDir: {}
      - name: azurefile
        persistentVolumeClaim:
          claimName: azurefile                     
```

### ARC Runner Scaleset Helm Chart Parameters

The Arc Runner Scaleset Helm Chart provides a few parameters, these are the most important ones to install a Scaleset with AzureFile volume mount on AKS - Azure Kubernetes Services.

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
    --values arc-runners-set-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=5 \
    --set runnerGroup=default \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

```

If you want to upgrade any configuration on the Arc Runner Scaleset, re-run the last helm install command onyl replacing the firt line to "helm upgrade --install".

```bash
helm upgrade --install "${ARC_RUNNER_SCALESET_NAME}" \
    --namespace "${NAMESPACE_ARC_RUNNERS}" \
    --create-namespace \
    --values arc-runners-set-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=5 \
    --set runnerGroup=default \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## Removing resources

### Deleting ARC Runner Scalesets created on your Kubernetes Cluster

```bash
helm delete "${ARC_RUNNER_SCALESET_NAME}" -n "${NAMESPACE_ARC_RUNNERS}" --wait
```

### Deleting ARC Runners Scaleset Controler

```bash
helm delete "${ARC_CONTROLLER_NAME}" -n "${NAMESPACE_ARC_CONTROLLER}" --wait
```

### Deleting PV and PVC

```bash
kubectl delete -f ./install/arc-runners-set-pv.yaml --wait
kubectl delete -f ./install/arc-runners-set-pvc.yaml --wait
```

### Deleting Secrets

```bash
kubectl delete secret generic azure-storage-secret 
```

### Deleting Namespaces

```bash
kubectl delete namespace ${NAMESPACE_ARC_CONTROLLER}
kubectl delete namespace ${NAMESPACE_ARC_RUNNERS}
```

### Deleting Github Actions App Secret

```bash
kubectl delete secret generic ${ARC_RUNNER_GITHUB_SECRET_NAME} 
```
