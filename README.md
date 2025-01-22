# Access Azure Files from AKS Pods Using Azure AD Workload Identity

This guide provides step-by-step instructions for setting up Azure Kubernetes Service (AKS) to access Azure Files securely using Azure AD Workload Identity.

## Prerequisites

1. An active Azure subscription.
2. An existing AKS cluster or the ability to create one.
3. Azure CLI installed ([Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)).
4. Kubernetes `kubectl` CLI installed ([Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)).
5. Azure AD permissions to create Managed Identities and assign roles.

---

## Step 1: Enable Azure AD Workload Identity on Your AKS Cluster

1. Update the Azure CLI to the latest version:

   ```bash
   az upgrade
   ```

2. Enable the Azure AD Workload Identity Preview feature:

   ```bash
   az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
   az feature register --namespace "Microsoft.ManagedIdentity" --name "ManagedIdentityExtensionsPreview"
   ```

3. Install the `Microsoft.ContainerService` extension:

   ```bash
   az extension add --name "aks-preview"
   ```

4. Enable the `EnableOIDCIssuerPreview` feature on your AKS cluster:

   ```bash
   az aks update -g <RESOURCE_GROUP> -n <CLUSTER_NAME> --enable-oidc-issuer --enable-workload-identity
   ```

   Replace `<RESOURCE_GROUP>` and `<CLUSTER_NAME>` with your resource group and cluster name.

---

## Step 2: Create a User-Assigned Managed Identity

1. Create the Managed Identity:

   ```bash
   az identity create --name <IDENTITY_NAME> --resource-group <RESOURCE_GROUP>
   ```

   Replace `<IDENTITY_NAME>` with a name for your identity and `<RESOURCE_GROUP>` with your resource group.

2. Note down the `clientId`, `principalId`, and `id` from the command output.

3. Grant the Managed Identity access to the Azure Storage Account:

   ```bash
   az role assignment create --assignee <PRINCIPAL_ID> --role "Storage Blob Data Contributor" --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.Storage/storageAccounts/<STORAGE_ACCOUNT_NAME>"
   ```

   Replace `<PRINCIPAL_ID>`, `<SUBSCRIPTION_ID>`, `<RESOURCE_GROUP>`, and `<STORAGE_ACCOUNT_NAME>` with the respective values.

---

## Step 3: Assign the Managed Identity to Your Pod

1. Create a Kubernetes Service Account:

   ```bash
   kubectl create serviceaccount <SERVICE_ACCOUNT_NAME>
   ```

   Replace `<SERVICE_ACCOUNT_NAME>` with a name for your service account.

2. Annotate the Service Account with the Managed Identity details:

   ```bash
   kubectl annotate serviceaccount <SERVICE_ACCOUNT_NAME> --namespace <NAMESPACE> \
     azure.workload.identity/client-id=<CLIENT_ID>
   ```

   Replace `<NAMESPACE>` with your namespace and `<CLIENT_ID>` with the `clientId` from the Managed Identity.

---

## Step 4: Configure the Azure Files CSI Driver

1. Install the Azure Files CSI driver:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/install-driver.yaml
   ```

2. Create a Storage Class for Azure Files:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: azurefile-csi
   provisioner: file.csi.azure.com
   parameters:
     skuName: Standard_LRS
   ```

   Save this as `azurefile-sc.yaml` and apply it:

   ```bash
   kubectl apply -f azurefile-sc.yaml
   ```

3. Create a Persistent Volume Claim (PVC):

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: azurefile-pvc
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 5Gi
     storageClassName: azurefile-csi
   ```

   Save this as `azurefile-pvc.yaml` and apply it:

   ```bash
   kubectl apply -f azurefile-pvc.yaml
   ```

---

## Step 5: Deploy Your Application

1. Deploy your application with the PVC:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: azurefile-app
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: azurefile-app
     template:
       metadata:
         labels:
           app: azurefile-app
       spec:
         serviceAccountName: <SERVICE_ACCOUNT_NAME>
         containers:
         - name: app-container
           image: nginx
           volumeMounts:
           - name: azurefile-volume
             mountPath: /mnt/azure
         volumes:
         - name: azurefile-volume
           persistentVolumeClaim:
             claimName: azurefile-pvc
   ```

   Save this as `azurefile-app.yaml` and apply it:

   ```bash
   kubectl apply -f azurefile-app.yaml
   ```

---

## Verify the Setup

1. Check the pods are running:

   ```bash
   kubectl get pods
   ```

2. Exec into the pod and verify the Azure Files mount:

   ```bash
   kubectl exec -it <POD_NAME> -- ls /mnt/azure
   ```

   Replace `<POD_NAME>` with your pod name.

You have now successfully configured your AKS cluster to access Azure Files using Azure AD Workload Identity.
