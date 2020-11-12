$location = "northeurope"
$rg = "mando"
$acr = "skywalker"
$name = "dartvader"

#Create resource group baned $rg for $location
az group create -l $location -n $rg

#Create ACR registry $acr
az acr create -n $acr -g $rg -l $location --sku Basic

#Setup of AKS cluster
$latestK8sVersion = (az aks get-versions -l $location --query 'orchestrators[-1].orchestratorVersion' -o tsv)
 az aks create -l $location -n $name -g $rg --generate-ssh-keys -k $latestK8sVersion

#Get credentials to interract with cluster
  az aks get-credentials -n $name -g $rg

#Set up jedi namespace
kubectl create namespace jedi
kubectl create clusterrolebinding default-view --clusterrole=view --serviceaccount=jedi:default

#Assign acrpull (from specific resource group) role to aks
$ACR_ID=(az acr show -n $acr -g $rg --query id -o tsv)
az aks update -g $rg -n $name --attach-acr $ACR_ID

#Create service principal for AzureDevops pipeline to be able to push and pull images and charts to AKS
$registryPassword=(az ad sp create-for-rbac -n $acr-push --scopes $ACR_ID --role acrpush --query password -o tsv)

#Create service principal for AzureDevops pipelines to be able to deploy to AKS
$AKS_ID=(az aks show -n $name -g $rg --query id -o tsv)
$aksSpPassword=(az ad sp create-for-rbac -n $name-deploy --scopes $AKS_ID --role "Azure Kubernetes Service Cluster User Role" --query password -o tsv)
