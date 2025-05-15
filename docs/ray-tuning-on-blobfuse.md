# Run a Kuberay tuning job on AKS with Azure Blob Storage and Blobfuse

In this article, you tune gpt2-large model on kuberay which would create and write the final model and the checkpoints to an Azure Blob Storage account. 

## Prerequisites
An AKS cluster with 9 Standard_D16d_v5 VMs and 1 Standard_D32d_v5 VM. To create such and AKS cluster run the following commands.

    
    az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 9 --node-vm-size Standard_D16d_v5 --generate-ssh-keys
    az aks nodepool add --resource-group myResourceGroup --cluster-name myAKSCluster --name nodepool2 --node-count 1 --node-vm-size Standard_D32d_v5


## Setup storage for the tuning job
1. Create a Storage Account as described [here](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal).
2. Create a container in that storage account as described [here](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-portal#create-a-container)
3. Enable Azure Blob CSI driver on the AKS cluster as per instructions [here](https://learn.microsoft.com/en-us/azure/aks/azure-blob-csi?tabs=NFS#enable-csi-driver-on-a-new-or-existing-aks-cluster).

## Setup the AKS cluster and your local machine

1. Install [KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/getting-started/kuberay-operator-installation.html#step-2-install-kuberay-operator) operator on your AKS cluster.

    ```helm repo add kuberay https://ray-project.github.io/kuberay-helm/
        helm repo update
        helm install kuberay-operator kuberay/kuberay-operator --version 1.3.0
    ```

2. Modify the pv.yaml under sample-tuning-setup and update the RESOURCE_GROUP_NAME, STORAGE_ACCOUNT_NAME and CONTAINER_NAME.

3. Update the MANAGED_IDENTITY_CLIENT_ID to the Managed Identity that has access to your storage account.

4. Apply all the yaml files under sample-tuning-setup

    ```
    kubectl apply -f sample-tuning-setup/storageclass.yaml
    kubectl apply -f sample-tuning-setup/pv.yaml
    kubectl apply -f sample-tuning-setup/pvc.yaml
    kubectl apply -f sample-tuning-setup/raycluster.yaml
    ```

    You should be able to see the ray pods running on the cluster when you run:

    ```
    kubectl get pods
    ```

5. Open a terminal window and enable port-forwarding for the ray service.

    ```
    kubectl port-forward services/raycluster-gpt2-head-svc 8265:8265
    ```


3. Install Python, Pip and [Ray](https://docs.ray.io/en/latest/ray-overview/installation.html) on your local machine.
   Open a new terminal window and run:
   
    ```
    apt-get update
    apt-install python3
    apt-install python3-pip
    pip install -U "ray[data,train,tune,serve]"
    ```
    
    Set RAY_ADDRESS as environment variable

    ```
    export RAY_ADDRESS=http://127.0.0.1:8265
    ```

## Run the tuning job 
To run the tuning job, run the gpt2_submit.py python scripts as below

    python sample-tuning-setup/gpt2_submit.py
    
You can track the status of the tuning job by running the following command
    
    ray job logs '<JOB_ID>' --follow

After the run is complete, you should be able to see the final model and the checkpoint files under your blob container.



