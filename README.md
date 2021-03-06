# AKS Training Lab Instructions

All labs can be done within the Azure CLI (docker samples can be done on Katacoda). This is by design - to avoid issues with Docker Desktop during the workshop. If you have Docker Desktop, you can run the docker containers and K8S workloads locally, but it is not a dependency per se.  

## Lab 1: Getting started with Containers

1. [Open the Docker playground on KataKoda](https://www.katacoda.com/courses/docker/playground)

2. Run basic docker CLI commands\
   `docker run hello-world`\
   `docker images`\
   `docker run -it  ubuntu`

3. Switch to the NGINX playground. We'll quickly spin up an NGINX Webserver and access a static site from a browser. [Navigate to the docker playgroud](https://www.katacoda.com/courses/docker/create-nginx-static-web-server). Try a few of docker commands:\
`docker exec -it containerId /bin/sh`\
`docker pull debian`\
`docker rmi imageName`
   
## Lab 2: Running containers on Azure Container Instances

1) Start the Azure Cloud Shell.
2) Create a dedicated resource group for the labs in the portal or the CLI. All resources will be housed here.\
   `az group create --name aks-training --location eastus`

3) Create a Container Registry in the Azure Portal using a unique name. Alternatively, use CLI. Make a note of the ACR name. This will be used through the course\
`az acr create --resource-group aks-training --name yourACRName --sku Basic  --admin-enabled`

4) Clone the Lab Git Repo the Azure Cloud Shell and navigate to the root folder of the repo.\
`git clone https://github.com/ManojG1978/aks-training.git`\
`cd aks-training/HelloWorldMVC`

5) Build the Docker Image for the Hello World ASP.NET Core MVC site and push it to the ACR\
`az acr build --registry yourACRName --image helloworld-mvc:1.0 .`

6) Create an Azure Container Instance in the portal using the image built. Select networking type as Public and give it a unique DNS label. A new Public IP is created for the container instance. Once created navigate to the URL and review the printed message (host name of the container)

## Lab 3: Getting Started with AKS

1) Create an AKS cluster in the Portal within the resource group created earlier. Choose Node Count as 1 and Node Size as *Standard_B2s*. Enable Container Insights (We'll look at this later)
2) Navigate to the HelloWorld folder. This contains a simple console app.\
`cd HelloWorld`
3) Setup variables to be used in CLI commands later

```bash
AKS_NAME=aks-training-cluster
ACR_NAME=yourACRName
RG_NAME=aks-training
REGION=eastus
```

4) Push Image to your ACR Repo\
`az acr build --registry $ACR_NAME --image helloworld:1.0 .`

5) Attach the ACR to the AKS Cluster\
`az aks update -n $AKS_NAME -g $RG_NAME --attach-acr $(az acr show -n $ACR_NAME --query "id" -o tsv)`

6) Connect to AKS Cluster\
`az aks get-credentials --resource-group $RG_NAME --name $AKS_NAME`

7) Open k8s-pod.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
8) Deploy HelloWorld Pod from the manifest file\
`kubectl create -f ./k8s-pod.yaml`

9) Try some imperative commands with Kubectl like:\
`kubectl run -i --tty busybox --image=busybox -- sh`

## Lab 4: Deploying K8S Service and Load balancer

1) Navigate to the HelloWorldMVC folder of the repo.
2) Open k8s-deploy.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
3) Deploy the service (an ASP.NET Core Application)\
`kubectl create -f k8s-deploy.yaml`

4) Investigate the service and the deployment resources. Copy the Public IP of the service and open it on the browser. Note that it make take a couple of minutes to provision the public IP\
`kubectl get all`\
`kubectl get svc`

## Lab 5: Deploying a simple two tier app to AKS

1) Navigate to the voting-app-aspdotnet-core folder of the repo. This is the canonical two-tier sample from Microsoft and Docker, but built using ASP.NET Core.
2) Build the Docker Image for the Voting and push it to the ACR\
`az acr build --registry yourACRName --image vote-app:1.0 .`
3) Open k8s-deploy-aks.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
4) Deploy the application (front end ASP.NET Core, backend Redis)\
`kubectl create -f k8s-deploy-aks.yaml`

5) Investigate the service and the deployment resources. Copy the Public IP of the service and open it on the browser with port 5000. Note that it make take a couple of minutes to provision the public IP\
`kubectl get all`\
`kubectl get svc`

## Lab 6: Implementing Health and liveness checks

1) Navigate to the voting-app-aspdotnet-core folder of the repo.
2) Open k8s-deploy-healthchecks.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
3) If you already have another deployment of the voting app from the previous exercise, delete it\
`kubectl delete -f k8s-deploy-aks.yaml`
4) Deploy the version of voting application, which includes health checks included\
`kubectl create -f k8s-deploy-healthchecks.yaml`
5) Investigate the service and the deployment resources. Copy the Public IP of the service and open it on the browser with port 5000. Append /hc to the URL to see the health status. Append /liveness to the URL to check the liveness of the application\
`kubectl get all`\
`kubectl get svc`

## Lab 7: K8S Resource Requests and Limits

1) Navigate to the HelloWorld folder of the repo.
2) Open k8s-pod.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
3) Deploy a pod which stresses memory.\
`kubectl create -f k8s-pod-limits.yaml`
4) Investigate the Pod lifecycle and check that the POD is terminated due to OOM limits.\
`kubectl get pod memory-stress`\
`kubectl describe pod memory-stress`

## Lab 8: K8S Jobs and Cronjob

1) Navigate to the HelloWorld folder of the repo.
2) Create a Job from *k8s-job.yaml*. This is a simple job that sleeps for 25 seconds. 3 such executions complete the overall job\
`kubectl create -f ./k8s-job.yaml`
3) Investigate the jobs, notice parallelism. Play around with commands and notice how the *backoffLimit* takes effect.\
`kubectl get jobs --watch`
4) Open k8s-cronjob.yaml using the built-in code editor of the shell. Update the image name with the ACR name in your resource group.
5) Create the Cronjob from the manifest\
`kubectl create -f ./k8s-cronjob.yaml`
6) Investigate the jobs, they run approximately every minute\
`kubectl get cronjob`\
`kubectl get jobs --watch`

## Lab 9: Implementing Persistent Volumes

1) Setup variables to hold values of your resource names

```bash
AKS_SHARE_NAME=redis
AKS_NAME=aks-training-cluster
ACR_NAME=yourACRName
RG_NAME=aks-training
REGION=eastus
```

2) Create a storage account, which will house the volumes used by the pods\
`az storage account create -n $AKS_STORAGE_ACCOUNT_NAME -g $RG_NAME -l $REGION --sku Standard_LRS`\
`export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n $AKS_STORAGE_ACCOUNT_NAME -g $RG_NAME -o tsv)`

3) Create a file share to house the redis data files used by the voting app\
`az storage share create -n $AKS_SHARE_NAME --connection-string $AZURE_STORAGE_CONNECTION_STRING`

4) Extract the Storage account key and create a secret. This secret will be referenced by the pod volume\
`STORAGE_KEY=$(az storage account keys list --resource-group $RG_NAME --account-name $AKS_STORAGE_ACCOUNT_NAME --query "[0].value" -o tsv)`\
`kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY`

5) Deploy the version of the voting app which has volumes attached to the redis pods. Make sure you update the image corresponding to your ACR. This manifest file creates the underlying Persistent Volume and Persistent Volume Claims (Delete any previous deployment of the voting app if they are already running)\
`kubectl create -f .\k8s-deploy-aks-pv.yaml`

6) Investigate the K8s objects. Check the redis pod and ensure the volume is mounted.\
`kubectl get pvc`\
`kubectl describe pod yourredispodName`

7) Try adding some votes in the app. Delete the redis pod now. K8s will recreate the pod and mount the volume again. However, when you access the application, the previous data would show up. Without Persistent volumes, this data would have been lost permanently.\
`kubectl delete pod yourredispodName`

[For more information, check the AKS Doc sample](https://docs.microsoft.com/en-us/azure/aks/azure-files-volume)

## Lab 10: Leveraging Secrets

1) In this lab, we would set up a secret natively in K8s and access that within a pod. No Key Vault involved here. First create a generic secret containing two values - *user* and *password*\
`kubectl create secret generic k8ssecret --from-literal=user=u123 --from-literal=password=p123`

2) Navigate to Secrets folder in the repo and create the Pod from the manifest file. In this scenario, secrets are injected into the pod as environment variables and the pod basically echoes the values on the console.\
`kubectl create -f .\k8s-secrets-env.yaml`

3) Check the pod logs to see the un-encrypted values printed to the console
`kubectl logs secret-test-pod`

## Lab 11: Integrating with Azure Key Vault

1) In this lab, we'll use secrets stored in the Azure KeyVault (as is usually the case with production apps). The first step is to install the Secrets Store CSI Driver to the AKS Cluster\
`helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts`\
`helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name`
2) Create a new Service Principal and note the client id and secret values\
`az ad sp create-for-rbac --name aksServicePrincipal --skip-assignment`
3) Setup a Key Vault in the portal, ensure you assign Get and List permissions to the newly created Service Principal.
4) Create a new Secret with name *connectionString*. Add a secret value.
5) Create a K8S secret to house the credentials of Service Principal\
`kubectl create secret generic secrets-store-creds --from-literal clientid=yourClientID --from-literal clientsecret=yourClientSecret`
6) Create the *ServiceProviderClass* object. Open the k8s-secretprovider.yaml in the Code editor (Secrets folder) and update  *userAssignedIdentityID* with your Service Principal ID, *keyvaultName* with the name of the vault you created, *subscriptionId* with your subscription ID, and *tenantId* with your tenant ID (run *az account list* for getting tenant ID and subscription ID)\
`kubectl create -f k8s-secretprovider.yaml`
7) Create a pod that mounts secrets from the key vault.\
`kubectl create -f k8s-nginx-secrets.yaml`
8) To display all the secrets that are contained in the pod, run the following command\
`kubectl exec -it nginx-secrets-store-inline -- ls /mnt/secrets-store/`
9) To display the contents of the *connectionString* secret, run the following command:\
`kubectl exec -it nginx-secrets-store-inline -- cat /mnt/secrets-store/connectionString`

[For More information, refer to the Docs sample](https://docs.microsoft.com/en-us/azure/key-vault/general/key-vault-integrate-kubernetes)

## Lab 12: Implementing RBAC in AKS

1) Navigate to the RBAC folder. Create a certificate for user called *bob* using (OpenSSL)

```bash
openssl genrsa -out bob.key 2048
openssl req -new -key bob.key -out bob.csr -subj "/CN=bob/O=dev"\n
cat bob.csr | base64 | tr -d '\n'
```

2) Open *k8s-csr.yaml* and update the request field with the output from the previous command (base64 encoded key). Submit the  Certificate Signing Request to K8s\
`kubectl create -f k8s-csr.yaml`
3) Verify the request is still pending\
`kubectl get csr`
4) As cluster admin, approve the request\
`kubectl certificate approve bob`
5) Get the signed certificate from the cluster as a *.crt* file\
`kubectl get csr bob -o jsonpath='{.status.certificate}' | base64 --decode > bob.crt`
6) The file, bob.crt is the client certificate that’s used to authenticate Bob. With the combination of the private key (bob.key) and the approved certificate (bob.crt) from Kubernetes, you can get authenticated with the cluster. Add Bob to K8s\
`kubectl config set-credentials bob --client-certificate=bob.crt --client-key=bob.key --embed-certs=true`
7) Set the context for Bob\
`kubectl config set-context bob --cluster=aks-training-cluster --user=bob`
8) Create a new K8s namespace called *dev*\
`kubectl create namespace dev`
9) Check if Bob can list pods on the dev namespace\
`kubectl auth can-i list pods --namespace dev --as bob`
9) Create a K8S role called dev, which has full access to the dev namespace\
`kubectl create -f .\k8s-role-dev.yaml`
10) Create the role binding for the dev group\
`kubectl create -f .\k8s-rolebinding-dev.yaml`
11) Switch to Bob's context\
`kubectl config use-context bob`
12) Check if Bob can list pods on the dev namespace now\
`kubectl auth can-i list pods --namespace dev`
13) Try this exercise with another user (say Dave) assigned to the *sre* namespace. 

## Lab 13: Autoscaling Pods and AKS Cluster

1) Auto-scale deployment using the Horizontal pod autoscaler using a CPU metric threshold  
`kubectl autoscale deployment vote-app-deployment --cpu-percent=30 --min=3 --max=6`

2) Manually scale nodes in the cluster using this command:\
`az aks scale --resource-group aks-training --name aks-training-cluster --node-count 2`

3) Setup an autoscaler using the command below (Min 1, Max 2). Alternatively, you could do this in the portal for each of your node pools:

```
az aks update \
  --resource-group aks-training \
  --name aks-training-cluster  \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 2	
```
[For more information, refer docs](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler)

## Lab 14: Navigating the native K8s administration portal

1) Switch to cluster-admin credentials\
`az aks get-credentials -a --resource-group aks-training --name aks-training-cluster`
2) Download the Kubeconfig from the console menu. It is located at /$HOME/.kube/config
3) Run the command to access the K8s Admin portal\
`az aks browse --resource-group aks-training --name aks-training-cluster`
4) When asked for credentials, choose KubeConfig and select the file you downloaded. The portal should now show up.

## Lab 15: Monitoring and Alerts with Container Insights/Azure Monitor

1) When we setup the cluster earlier in the lab, we had enabled Container Insights. If not, you can go to the Insights blade on the portal, for your AKS cluster and enable it. It might take 5-10 minutes for the data to show up.
2) Review the Cluster, Nodes, Controllers, Containers and Deployment tabs respectively.  
3) For select pods (like Stress), select View Container logs from the right pane
4) Play with filters and try reviews log records. You can create your own queries like below:

```
ContainerInventory
| where  Image contains "stress"
```

5) To get familiar with alerts, you can select the "Recommended Alerts (Preview)" link from the menu. For this exercise, enable the alert for "OOM Killed Containers" and create an action group to email yourself. Try recreating the memory-stress pod from Lab 7 and verify you get an email alert.

## Lab 16: eShopOnContainers Microservice Application deployment on AKS (optional)

1) In your Azure shell, run the following command:\
`. <(wget -q -O - https://aka.ms/microservices-aspnet-core-setup)`
2) For more information, checkout the [MS Learn module on Microservices](https://docs.microsoft.com/en-gb/learn/modules/microservices-aspnet-core/2-deploy-application)