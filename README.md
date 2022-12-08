# Scope
This project was created to build managed Kubernetes `Composite Resource` (XR) with three
compositions supporting three main cloud providers.
Each XR can have one or more compositions. Each composition describes how XR should be created
by defining provider package and list of resources (Managed Resources) which build it.
This allows a Composition to act as a class of service.
You can use it as a foundation to understand, build and operate managed Kubernetes Platform in the Cloud.
This repository uses two crossplane distributions:
* [XP  - Crossplane](https://crossplane.io/) - upstream project
* [UXP - Upbound Universal Crossplane](https://docs.upbound.io/uxp/) - downstream distribution of crossplane maintained by Upbound

Thought it is possible to mix crossplane distributions with providers I will use following combinations in this repo:
* XP + Native providers
* UXP + Official providers

## What's new in Version 3.2
Added support for official providers maintained by upbound.

# Providers APIs
Providers are Crossplane packages allowing provision the respective infrastructure.
They differ between themselves by number of supported cloud resources (CRDs), written programming language
and maintenance model.
At that moment we can use two different cloud providers to build the composition:
- Native (Classic) - maintained by XP community, the fastest one, written in Go with limited resource coverage.
- Official - maintained by Upbound, the newest one based on Upjet, with coverage between Native and Jet-preview.

There is another provider available, but it was [deprecated](https://github.com/crossplane/terrajet/issues/308)
 and has been replaced by Official one:
- Jet - maintained by XP community, based on Terrajet, available in two packages one with similar coverage as
  classic provider and one (with -preview suffix) with full resource coverage.
I will keep it for a time being in this project but it is not longer maintained and will be release in the future releases.


To give you an idea about current coverage state, based on AWS provider:
* [native 171 CRDs](https://doc.crds.dev/github.com/crossplane/provider-aws@v0.33.0)
* [official 364 CRDs](https://doc.crds.dev/github.com/upbound/provider-aws@v0.18.0)
* [jet 99 CRDs](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws@v0.5.0)
* [jet-preview 780 CRDs](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws@v0.5.0-preview)

## Native Providers
* [AWS Native](https://doc.crds.dev/github.com/crossplane/provider-aws)
* [Azure Native](https://doc.crds.dev/github.com/crossplane/provider-azure)
* [GCP Native](https://doc.crds.dev/github.com/crossplane/provider-gcp)

## Official providers
* [AWS Official Doc](https://doc.crds.dev/github.com/upbound/provider-aws) or [AWS Marketplace](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.20.0/crds)
* [Azure Official](https://doc.crds.dev/github.com/upbound/provider-azure) or [Azure Marketplace](https://marketplace.upbound.io/providers/upbound/provider-azure/v0.19.0/crds)
* [GCP Official](https://doc.crds.dev/github.com/upbound/provider-gcp) or [GCP Marketplace](https://marketplace.upbound.io/providers/upbound/provider-gcp/v0.18.0/crds)

## Jet Providers - deprecated
All resources which are needed to provision managed Kubernetes cluster are defined in smaller
classic-jet provider, so no need to install much bigger with -preview suffix.

* [AWS Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws)
* [Azure Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-azure)
* [GCP Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-gcp)

## Other Providers
Post Provisioning use Helm and Kubernetes Providers.

* [Helm](https://doc.crds.dev/github.com/crossplane-contrib/provider-helm)
* [Kubernetes](https://doc.crds.dev/github.com/crossplane-contrib/provider-kubernetes)

To demonstrate usage for both post provisioning resources I created following examples:
* (Universal) Crossplane Provisioning using Helm Chart
* Production Namespace Provisioning using Kubernetes Manifest

# Quick Start

Install Kubernetes Cluster. I recommend to use [Rancher Desktop](https://rancherdesktop.io/) for local cluster.

## Install crossplane

### Configure Upstream XP

```
# Install XP cli
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh

# Install XP
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait
    # --set nodeSelector."agentpool"=xpjetaks2

# Verify status
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

### Configure Downstream UXP

```
# Install UP Command-Line
brew install upbound/tap/up

# Install UXP
up uxp install

# Verify status
kubectl get pods -n upbound-system`
```

## Setup Cloud Credentials
   - [Prepare cloud credentials](https://crossplane.io/docs/v1.10/reference/configure.html)

As an output of above setup you should get three credentials files with following content.
- aws-cred.conf
```yaml
[default]
aws_access_key_id = XXXXXXXXXX
aws_secret_access_key = WFhYWFhYWFhYWA==
```
- azure-cred.json
```yaml
{
  "clientId": "XXXXXXXXXX",
  "clientSecret": "WFhYWFhYWFhYWA==",
  "subscriptionId": "XXXXXXXXXX",
  "tenantId": "XXXXXXXXXX",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```
- gcp-cred.json
```yaml
{
  "type": "service_account",
  "project_id": "XXXXXXXXXX",
  "private_key_id": "XXXXXXXXXX",
  "private_key": "-----BEGIN PRIVATE KEY-----\nWFhYWFhYWFhYWA==\n-----END PRIVATE KEY-----\n",
  "client_email": "XXXXXXXXXX",
  "client_id": "XXXXXXXXXX",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "XXXXXXXXXX"
}
```
You can store them in cloud Key Vault (KV) or any other Secret Store. Below there is example how to get them from Azure Key Vault.
To retrieve credential files from Azure KV you can use following:

```console
KEYVAULT=<Key Vault Name>
az keyvault secret show --name uxpAwsCred --vault-name $KEYVAULT --query value -o tsv | sed -r 's@ aws@\naws@g' > aws-cred.conf
az keyvault secret show --name uxpAzureCred --vault-name $KEYVAULT --query value -o tsv | jq > azure-cred.json
az keyvault secret show --name uxpGcpCred --vault-name $KEYVAULT --query value -o tsv | jq > gcp-cred.json
```

## Install and configure providers

To be able to provision cloud resources using Crossplane we have to create and configure cloud provider resource. This resource stores the cloud information and is used by XP to interact with cloud provider.

We need to provide two environment variables:
- base64 encoded cloud credentials
- name of the namespace, for UXP: `upbound-system` for XP: `crossplane-system`

### Native Providers

```console          
# XP Native
kubectl apply -f providers/native/xp-providers.yaml

# Service providers
kubectl apply -f providers/service-providers.yaml

# Verification
kubectl get provider
```

#### Native AWS Provider

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 -i aws-cred.conf | tr -d "\n")

eval "echo \"$(cat providers/secret-aws-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/native/xp-aws-providerconfig.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.aws.crossplane.io

# Verification
kubectl apply -f validation/native/xp-aws-bucket.yaml
kubectl get Bucket.s3.aws.crossplane.io

# AWS Validation using cmd (alternatively console UI)
aws s3 ls --output table

# Clean-up
kubectl delete Bucket.s3.aws.crossplane.io xp-aws-bucket
```

#### Native Azure Provider

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_AZURE_ACCOUNT_CREDS=$(base64 -i azure-cred.json | tr -d "\n")

eval "echo \"$(cat providers/secret-azure-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/native/xp-azure-providerconfig.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.azure.crossplane.io

# Verification
kubectl apply -f validation/native/xp-azure-bucket.yaml
kubectl get Account.storage.azure.crossplane.io

# Azure Validation using cmd (alternatively console UI)
az group show --resource-group xp-azure-rg -o table
az storage account show -g xp-azure-rg -n xpazurebucket007 -o table

# Clean-up
kubectl delete Account.storage.azure.crossplane.io xpazurebucket007
```

#### Native GCP Provider

For GCP we need additionally environment variable: PROJECT_ID.

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_GCP_PROVIDER_CREDS=$(base64 -i gcp-cred.json | tr -d "\n")
PROJECT_ID=$(gcloud projects list --filter='NAME:<Project Name>' --format="value(PROJECT_ID.scope())")

eval "echo \"$(cat providers/secret-gcp-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/native/xp-gcp-providerconfig.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.gcp.crossplane.io

# Verification
kubectl apply -f validation/native/xp-gcp-bucket.yaml
kubectl get Bucket.storage.gcp.crossplane.io -w

# GCP Validation using cmd (alternatively console UI)
gsutil ls -p <PROJECT_ID>

kubectl delete Bucket.storage.gcp.crossplane.io xp-gcp-bucket
```

### Official providers

You can install official providers


* using configuration

```
kubectl apply -f configuration/official.yaml
watch kubectl get pkg
```

* manually by applying manifest files

```console
# UXP                     
kubectl apply -f providers/official/uxp-providers.yaml              
# Service providers
kubectl apply -f providers/service-providers.yaml

kubectl get provider.pkg
```

#### Official AWS Provider

```console
PROVIDER_SECRET_NAMESPACE=upbound-system
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 -i aws-cred.conf | tr -d "\n")

eval "echo \"$(cat providers/secret-aws-provider.yaml)\"" | kubectl apply -f -
kubectl apply -f providers/official/uxp-aws-providerconfig.yaml

kubectl get providerconfig

# Verification
kubectl apply -f validation/official/uxp-aws-bucket.yaml
kubectl get Bucket.s3.aws.upbound.io -w

# AWS Validation using cmd (alternatively console UI)
aws s3 ls --output table

# Clean-up
kubectl delete Bucket.s3.aws.upbound.io uxp-aws-bucket
```

#### Official Azure Provider

```console
PROVIDER_SECRET_NAMESPACE=upbound-system
BASE64ENCODED_AZURE_ACCOUNT_CREDS=$(base64 -i azure-cred.json | tr -d "\n")

eval "echo \"$(cat providers/secret-azure-provider.yaml)\"" | kubectl apply -f -
kubectl apply -f providers/official/uxp-azure-providerconfig.yaml

kubectl get providerconfig

# Verification
kubectl apply -f validation/official/uxp-azure-bucket.yaml
kubectl get account.storage.azure.upbound.io/uxpazurebucket007 -w

# Azure Validation using cmd (alternatively console UI)
az group show --resource-group uxp-azure-rg -o table
az storage account show -g uxp-azure-rg -n uxpazurebucket007 -o table

# Clean-up
kubectl delete -f validation/uxp-azure-bucket.yaml
```

#### Official GCP Provider

For GCP we need additionally third environment variable: PROJECT_ID.

```console
PROVIDER_SECRET_NAMESPACE=upbound-system
BASE64ENCODED_GCP_PROVIDER_CREDS=$(base64 -i gcp-cred.json | tr -d "\n")
PROJECT_ID=$(gcloud projects list --filter='NAME:<Project Name>' --format="value(PROJECT_ID.scope())")

eval "echo \"$(cat providers/secret-gcp-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/official/uxp-gcp-providerconfig.yaml)\"" | kubectl apply -f -

kubectl get providerconfig

# Verification
kubectl apply -f validation/official/uxp-gcp-bucket.yaml
kubectl get Bucket.storage.gcp.upbound.io -w

# GCP Validation using cmd (alternatively console UI)
gsutil ls -p <PROJECT_ID>

kubectl delete Bucket.storage.gcp.upbound.io uxp-gcp-bucket
```

### Jet Providers - deprecated

```console
# XP Jet
kubectl apply -f providers/jet-providers.yaml

# Service providers
kubectl apply -f providers/service-providers.yaml

# Verification
kubectl get provider
```

#### Jet AWS Provider

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 -i aws-cred.conf | tr -d "\n")

eval "echo \"$(cat providers/secret-aws-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/jet-aws-provider.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.aws.jet.crossplane.io
```

#### Jet Azure Provider

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_AZURE_ACCOUNT_CREDS=$(base64 -i azure-cred.json | tr -d "\n")

eval "echo \"$(cat providers/secret-azure-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/jet-azure-provider.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.azure.jet.crossplane.io
```

#### Jet GCP Provider

For GCP we need additionally third environment variable: project ID.

```console
PROVIDER_SECRET_NAMESPACE=crossplane-system
BASE64ENCODED_GCP_PROVIDER_CREDS=$(base64 -i gcp-cred.json | tr -d "\n")
PROJECT_ID=$(gcloud projects list --filter='NAME:<Project Name>' --format="value(PROJECT_ID.scope())")

eval "echo \"$(cat providers/secret-gcp-provider.yaml)\"" | kubectl apply -f -
eval "echo \"$(cat providers/jet-gcp-provider.yaml)\"" | kubectl apply -f -

kubectl get providerconfig.gcp.jet.crossplane.io
```

## Clean-up

```console
unset BASE64ENCODED_AWS_ACCOUNT_CREDS BASE64ENCODED_AZURE_ACCOUNT_CREDS BASE64ENCODED_GCP_PROVIDER_CREDS PROJECT_ID PROVIDER_SECRET_NAMESPACE
rm aws-cred.conf azure-cred.json gcp-cred.json
```

# Provisioning Infrastructure

Crossplane divides responsibility for the infrastructure provisioning as follows:
1. Ops/SRE defines platform and APIs for Dev team
1. Dev consumes the infrastructure defined by Ops team

## Defining Infrastructure by Ops team

Platform team creates compositions and composite resource definitions (XRDs) to define and configure
managed kubernetes services infrastructure in cloud.

### Native provider

```console
# Compositions using Native providers
kubectl apply -f configuration/native/definition.yaml
kubectl apply -f configuration/native/xp-eks-composition.yaml
kubectl apply -f configuration/native/xp-aks-composition.yaml
kubectl apply -f configuration/native/xp-gke-composition.yaml
```

### Official provider

```console
# Using configuration
kubectl apply -f configuration/official.yaml
kubectl get pkg

# Manually
kubectl apply -f configuration/official/definition.yaml
kubectl apply -f configuration/official/uxp-eks-composition.yaml
kubectl apply -f configuration/official/uxp-aks-composition.yaml
kubectl apply -f configuration/official/uxp-gke-composition.yaml
```

### Jet provider - deprecated

```console
# Compositions using Jet providers
kubectl apply -f configuration/jet/definition.yaml
kubectl apply -f configuration/jet/jet-eks-composition.yaml
kubectl apply -f configuration/jet/jet-aks-composition.yaml
kubectl apply -f configuration/jet/jet-gke-composition.yaml
```

## Consuming the infrastructure by Dev team

App team provisions infrastructure by creating claim objects for the XRDs defined by Ops team.

```console
kubectl create ns managed

# Claims using native provider
kubectl apply -f claims/native/xp-eks-claim.yaml
kubectl apply -f claims/native/xp-aks-claim.yaml
kubectl apply -f claims/native/xp-gke-claim.yaml

# Claims using official provider
kubectl apply -f claims/official/uxp-eks-claim.yaml
kubectl apply -f claims/official/uxp-aks-claim.yaml
kubectl apply -f claims/official/uxp-gke-claim.yaml

# Claims using jet provider - deprecated
kubectl apply -f claims/jet/jet-eks-claim.yaml
kubectl apply -f claims/jet/jet-aks-claim.yaml
kubectl apply -f claims/jet/jet-gke-claim.yaml
```

### Verifying Infrastructure status

Dev team provisions Claims which either generate new Composite Resource (XR) or assign existing ones.

We can check progress using:

```
kubectl get managedcluster -n managed
# Example Output
NAME     CLUSTERNAME      CONTROLPLANE   NODEPOOL   FARGATEPROFILE       SYNCED   READY   CONNECTION-SECRET   AGE
xpaks    cluster-xpaks    Succeeded      Succeeded  NA4-cluster-xpaks    True     True    xpaks               7m
xpgke    cluster-xpgke    RUNNING        RUNNING    NA4-cluster-xpgke    True     True    xpgke               9m1s
xpeks    cluster-xpeks    ACTIVE         ACTIVE     ACTIVE               True     True    xpeks               14m
uxpaks   cluster-uxpaks   True           True       NA4-cluster-uxpaks   True     True    uxpaks              5m59s
uxpgke   cluster-uxpgke   True           True       NA4-cluster-uxpgke   True     True    uxpgke              11m
uxpeks   cluster-uxpeks   ACTIVE         ACTIVE     ACTIVE               True     True    uxpeks              22m
```

You can check time for cluster readiness for different managed kubernetes under Age column.

To verify status of Helm Charts and Kubernetes Object:
```
kubectl get Object,Release
NAME                                            SYNCED   READY   AGE
object.kubernetes.crossplane.io/xpaks-ns-prod   True     True    23h

NAME                                          CHART        VERSION   SYNCED   READY   STATE REVISION   DESCRIPTION     AGE
release.helm.crossplane.io/xpaks-crossplane   crossplane   1.6.3     True     True    deployed   1  Install complete   23h
```

### Accessing infrastructure

#### Native providers

```
# Using secrets (eks and aks)
kubectl -n managed get secret xpeks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
kubectl -n managed get secret xpaks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig

export KUBECONFIG=$PWD/kubeconfig

# Using Cloud APIs
export KUBECONFIG=$PWD/kubeconfig
gcloud container clusters get-credentials cluster-xpgke --region europe-west2 --project <project name>
```

#### Official providers

```
# Using secrets (eks and aks)
kubectl -n managed get secret uxpaks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
kubectl -n managed get secret uxpeks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig

export KUBECONFIG=$PWD/kubeconfig

# Using Cloud APIs
export KUBECONFIG=$PWD/kubeconfig
gcloud container clusters get-credentials cluster-uxpgke --region europe-west2 --project <project name>
az aks get-credentials --resource-group rg-uxpaks --name cluster-uxpaks --admin
aws eks update-kubeconfig --region eu-west-1 --name cluster-uxpeks --alias uxpeks
```

## Cleanup & Uninstall

### Delete Claims

Deleting claims will take care of clean-up of all managed resources created to satisfy the claim.

```console
# Native
kubectl delete managedcluster -n managed xpeks
kubectl delete managedcluster -n managed xpaks
kubectl delete managedcluster -n managed xpgke

# Official
kubectl delete managedcluster -n managed uxpeks
kubectl delete managedcluster -n managed uxpaks
kubectl delete managedcluster -n managed uxpgke
```

### Delete Cloud Configuration

```console
kubectl get providerconfig

# Clean-up Native Providers
kubectl delete providerconfig.aws.crossplane.io/aws-xp-provider
kubectl delete providerconfig.azure.crossplane.io/azure-xp-provider
kubectl delete providerconfig.gcp.crossplane.io/gcp-xp-provider

# Clean-up Official Providers
kubectl delete providerconfig.aws.upbound.io/aws-uxp-provider
kubectl delete providerconfig.azure.upbound.io/azure-uxp-provider
kubectl delete providerconfig.gcp.upbound.io/gcp-uxp-provider

# Clean-up Jet Providers
kubectl delete providerconfig.aws.jet.crossplane.io/aws-jet-provider
kubectl delete providerconfig.azure.jet.crossplane.io azure-jet-provider
kubectl delete providerconfig.gcp.jet.crossplane.io gcp-jet-provider
```

### Delete Cloud Secrets

Name of the Namespace, for UXP: `upbound-system` for XP: `crossplane-system`

```console
# Clean-up Crossplane Secrets
kubectl delete secret -n crossplane-system aws-account-creds
kubectl delete secret -n crossplane-system azure-account-creds
kubectl delete secret -n crossplane-system gcp-account-creds

# Clean-up Upbound Secrets
kubectl delete secret -n upbound-system aws-account-creds
kubectl delete secret -n upbound-system azure-account-creds
kubectl delete secret -n upbound-system gcp-account-creds
```

### Uninstall Provider

```console
# native
kubectl delete provider.pkg aws-provider
kubectl delete provider.pkg azure-provider
kubectl delete provider.pkg gcp-provider
## services
kubectl delete provider.pkg provider-helm
kubectl delete provider.pkg provider-kubernetes

# official with manual
kubectl delete provider.pkg aws-uxp-provider
kubectl delete provider.pkg azure-uxp-provider
kubectl delete provider.pkg gcp-uxp-provider
kubectl delete provider.pkg provider-helm
kubectl delete provider.pkg provider-kubernetes

# official with configuration
kubectl delete configuration.pkg.crossplane.io/natzka
kubectl delete provider.pkg upbound-provider-aws
kubectl delete provider.pkg upbound-provider-azure
kubectl delete provider.pkg upbound-provider-gcp
kubectl delete provider.pkg crossplane-contrib-provider-helm
kubectl delete provider.pkg crossplane-contrib-provider-kubernetes

# jet
kubectl delete provider.pkg aws-jet-provider
kubectl delete provider.pkg azure-jet-provider
kubectl delete provider.pkg gcp-jet-provider

# Verification
kubectl get provider.pkg
```

### Uninstall Crossplane

```console
# XP
helm delete crossplane --namespace crossplane-system
kubectl get pods -n crossplane-system

# UXP
up uxp uninstall
kubectl get pods -n upbound-system
```

# Troubleshooting

## Removing resources manually

```
kubectl patch --type json --patch='[ { "op": "remove", "path": "/metadata/finalizers" } ]' subnet.ec2.aws.upbound.io/uxpeks-pub-b
```

## Compositions

```
kubectl get xmanagedcluster

NAME     CLUSTERNAME   CONTROLPLANE   NODEPOOL   FARGATEPROFILE   READY   CONNECTION-SECRET   AGE
uxpeks   uxpeks        ACTIVE         ACTIVE     ACTIVE           True    uxpeks              4d4h
uxpaks   uxpaks        True           True       NA4-uxpaks       True    uxpaks              4d
uxpgke   uxpgke        True           True       NA4-uxpgke       True    uxpgke              2d16h

# To find our which resource have issues within Composite resource:
kubectl describe xmanagedcluster uxpeks-5zxn6

...
 Resource Refs:
    API Version:  iam.aws.crossplane.io/v1beta1
    Kind:         Role
    Name:         xpeks-controlplane
...

# To find out issue with not healthy resource

kubectl get Role.iam.aws.crossplane.io
kubectl describe Role.iam.aws.crossplane.io/xpeks-controlplane

```
## Cloud resources

```
# Native and official providers

kubectl get managed
kubectl get aws
kubectl get azure
kubectl get gcp
or
kubectl get providerconfig | grep aws
```

# Supported Kubernetes Cluster properties

1. Cluster ID
1. Kubernetes Version
1. Node Size
1. Node Count
1. Region (Cross Cloud [Abstraction](docs/cloud-regions.MD)
1. FargateProfile Namespace (valid for EKS)

# APIs in this Configuration

## Native providers

* `configuration/native/` - Composite Resource Definition (XRD) with satisfying Compositions
  * [xmanagedcluster XRD](configuration/native/definition.yaml)
  * [eks composition](configuration/native/eks-composition.yaml) includes:
    * `Role`
    * `RolePolicyAttachment`
    * `VPC`
    * `SecurityGroup`, `SecurityGroupRule`
    * `Subnet`
    * `InternetGateway`
    * `RouteTable`, `Route`, `RouteTableAssociation`
    * `Cluster`
    * `NodeGroup`
    * `FargateProfile`
    * `Relase`
    * `Object`
  * [aks composition](configuration/native/aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `AKSCluster`
    * `Relase`
    * `Object`   
  * [gke composition](configuration/native/gke-composition.yaml) includes:
    * `Network`
    * `Subnetwork`
    * `Cluster`
    * `NodePool`
    * `Relase`
    * `Object`    
* `providers/` - Provider Installation and Configuration
  * [Setup](providers/native/xp-providers.yaml)
  * [AWS Provider Config](providers/native/aws-providerconfig.yaml)    
  * [Azure Provider Config](providers/native/azure-providerconfig.yaml)    
  * [GCP Provider Config](providers/native/gcp-providerconfig.yaml)
* `claims/native/` - Examples to consume defined by Ops XRDs
  * [EKS Claim](claims/native/eks-claim.yaml)    
  * [AKS Claim](claims/native/aks-claim.yaml)    
  * [GKE Claim](claims/native/gke-claim.yaml)       

## Official providers

* `configuration/official/` - Composite Resource Definition (XRD) with satisfying Compositions
  * [xmanagedcluster XRD](configuration/official/definition.yaml)
  * [eks composition](configuration/official/uxp-eks-composition.yaml) includes:
    * `Role`
    * `RolePolicyAttachment`
    * `VPC`
    * `SecurityGroup`, `SecurityGroupRule`
    * `Subnet`
    * `InternetGateway`
    * `RouteTable`, `Route`, `RouteTableAssociation`
    * `Cluster`
    * `NodeGroup`
    * `FargateProfile`
    * `ClusterAuth`
    * `Object`    
    * `Relase`
  * [aks composition](configuration/official/uxp-aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `KubernetesCluster`
    * `KubernetesClusterNodePool`
    * `Relase`
    * `Object`   
  * [gke composition](configuration/official/uxp-gke-composition.yaml) includes:
    * `Network`
    * `Subnetwork`
    * `Cluster`
    * `NodePool`
    * `Relase`
    * `Object`    
* `providers/official/` - Provider Installation and Configuration
  * [Setup](providers/official/uxp-providers.yaml)
  * [AWS Provider Config](providers/official/uxp-aws-providerconfig.yaml)    
  * [Azure Provider Config](providers/official/uxp-azure-providerconfig.yaml)    
  * [GCP Provider Config](providers/official/uxp-gcp-providerconfig.yaml)
* `claims/official/` - Examples to consume defined by Ops XRDs
  * [EKS Claim](claims/official/uxp-eks-claim.yaml)    
  * [AKS Claim](claims/official/uxp-aks-claim.yaml)    
  * [GKE Claim](claims/official/uxp-gke-claim.yaml)     

## Jet providers

* `configuration/jet/` - Composite Resource Definition (XRD) with satisfying Compositions
  * [xmanagedcluster XRD](configuration/jet/definition.yaml)
  * [eks composition](configuration/jet/jet-eks-composition.yaml) includes:
    * `Role`
    * `RolePolicyAttachment`
    * `VPC`
    * `SecurityGroup`, `SecurityGroupRule`
    * `Subnet`
    * `InternetGateway`
    * `RouteTable`, `Route`, `RouteTableAssociation`
    * `Cluster`
    * `NodeGroup`
    * `FargateProfile`
    * `Relase`
    * `Object`
  * [aks composition](configuration/jet/jet-aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `KubernetesCluster`
    * `KubernetesClusterNodePool`
    * `Relase`
    * `Object`   
  * [gke composition](configuration/jet/jet-gke-composition.yaml) includes:
    * `Network`
    * `Subnetwork`
    * `Cluster`
    * `NodePool`
    * `Relase`
    * `Object`    
* `providers/jet/` - Provider Installation and Configuration
  * [Setup](providers/jet/jet-providers.yaml)
  * [AWS Provider Config](providers/jet/jet-aws-providerconfig.yaml)    
  * [Azure Provider Config](providers/jet/jet-azure-providerconfig.yaml)    
  * [GCP Provider Config](providers/jet/jet-gcp-providerconfig.yaml)
* `claims/jet/` - Examples to consume defined by Ops XRDs
  * [EKS Jet Claim](claims/jet/jet-eks-claim.yaml)    
  * [AKS Jet Claim](claims/jet/jet-aks-claim.yaml)    
  * [GKE Jet Claim](claims/jet/jet-gke-claim.yaml)     

# Contributing workflow

Here’s how we suggest you go about proposing a change to this project:

1. [Fork this project][fork] to your account.
2. [Create a branch][branch] for the change you intend to make.
3. Make your changes to your fork.
4. [Send a pull request][pr] from your fork’s branch to our `main` branch.

Using the web-based interface to make changes is fine too, and will help you
by automatically forking the project and prompting to send a pull request too.

[fork]: https://help.github.com/articles/fork-a-repo/
[branch]: https://help.github.com/articles/creating-and-deleting-branches-within-your-repository
[pr]: https://help.github.com/articles/using-pull-requests/

# Issues

## Release v3.1

RouteTable, SecurityGroup and Nodegroup resources have been fixed. InternetGateway has been added to AWS classic jet provider so I was able to use this one over preview.

New issues found out:
1. Duplicate resource for FargateProfile (Impact: FargateProfile is commented out in eks jet composition)
    - Crossplane Jet AWS [220](https://github.com/crossplane-contrib/provider-jet-aws/issues/220)
    - Hashicorp AWS [26021](https://github.com/hashicorp/terraform-provider-aws/issues/26021)
   Status: Open
1. No Kubeconfig secret created by AWS Jet Provider (Impact: Kubernetes and Helm providers are commented in eks jet composition)
    - Crossplane Jet AWS [229](https://github.com/crossplane-contrib/provider-jet-aws/issues/229)
   Status: Open

## Release v3

1. Lack of `InternetGateway` resource in Classic Jet version for AWS.
   (Issue [183](https://github.com/crossplane-contrib/provider-jet-aws/issues/183)).
   Status: Fixed => Resource has been added to Classic Jet provider
2. RouteTable - no sync
   (Issue [184](https://github.com/crossplane-contrib/provider-jet-aws/issues/184), PR [197](https://github.com/crossplane-contrib/provider-jet-aws/pull/197))
   Status: Fixed
3. NodeGroup - unable to create resource
   (Issue [187](https://github.com/crossplane-contrib/provider-jet-aws/issues/187)), PR [201](https://github.com/crossplane-contrib/provider-jet-aws/pull/201)
   Status: Fixed
4. SecurityGroup - no sync
   (Issue [157](https://github.com/crossplane-contrib/provider-jet-aws/issues/157), PR [198](https://github.com/crossplane-contrib/provider-jet-aws/pull/198))
   Status: Fixed
