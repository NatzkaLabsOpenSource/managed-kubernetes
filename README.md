# Scope
This project was created to build managed Kubernetes `Composite Resource` (XR) with three
compositions supporting three main cloud providers.
Each XR can have one or more compositions. Each composition describes how XR should be created
by defining provider package and list of resources (Managed Resources) which build it.
This allows a Composition to act as a class of service.
You can use it as a foundation to understand, build and operate managed Kubernetes Platform in the Cloud.
This repository uses two crossplane distributions:
* [XP  - Crossplane](https://crossplane.io/) - upstream project
* [UXP - Upbound Universal Crossplane](https://docs.upbound.io/uxp/) - downstram distribution of crossplane maintained by Upbound

## What's new in Version 3.2
Added support for official providers maintained by upbound.

# Providers APIs
Providers are Crossplane packages allowing provision the respective infrastructure.
They differ between themselves by number of supported cloud resources (CRDs), written programming language
and maintenance model.
At that moment we can use three different cloud providers to build the composition:
- Classic (Native) - maintained by XP community, the fastest one, written in Go with limited resource coverage.
- Jet - maintained by XP community, based on Terraform, available in two packages one with similar coverage as
  classic provider and one (with -preview suffix) with full resource coverage.
- Official - maintained by Upbound, the newest one based on Upjet, with coverage between Native and Jet-preview.

To give you an idea about current coverage state, based on AWS provider:
* [classic 171 CRDs](https://doc.crds.dev/github.com/crossplane/provider-aws@v0.33.0)
* [jet 99 CRDs](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws@v0.5.0)
* [jet-preview 780 CRDs](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws@v0.5.0-preview)
* [official 364 CRDs](https://doc.crds.dev/github.com/upbound/provider-aws@v0.18.0)

## Classic Providers
* [AWS Classic](https://doc.crds.dev/github.com/crossplane/provider-aws)
* [Azure Classic](https://doc.crds.dev/github.com/crossplane/provider-azure)
* [GCP Classic](https://doc.crds.dev/github.com/crossplane/provider-gcp)

## Jet Providers
All resources which are needed to provision managed Kubernetes cluster are defined in smaller
classic-jet provider, so no need to install much bigger with -preview suffix.

* [AWS Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-aws)
* [Azure Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-azure)
* [GCP Jet](https://doc.crds.dev/github.com/crossplane-contrib/provider-jet-gcp)

## Official providers
* [AWS Official Doc](https://doc.crds.dev/github.com/upbound/provider-aws) or [AWS Marketplace](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.20.0/crds)
* [Azure Official](https://doc.crds.dev/github.com/upbound/provider-azure) or [Azure Marketplace](https://marketplace.upbound.io/providers/upbound/provider-azure/v0.19.0/crds)
* [GCP Official](https://doc.crds.dev/github.com/upbound/provider-gcp) or [GCP Marketplace](https://marketplace.upbound.io/providers/upbound/provider-gcp/v0.18.0/crds)

## Other Providers
Post Provisioning use Helm and Kubernetes Providers.

* [Helm](https://doc.crds.dev/github.com/crossplane-contrib/provider-helm)
* [Kubernetes](https://doc.crds.dev/github.com/crossplane-contrib/provider-kubernetes)

To demonstrate usage for both post provisioning resources I created following examples:
* Crossplane Provisioning using Helm Chart
* Production Namespace Provisioning using Kubernetes Manifest

## Prerequisites

Install Kubernetes Cluster. I recommend to use [Rancher Desktop](https://rancherdesktop.io/) for local cluster.

### Configure Crossplane

#### Configure XP
- Install XP cli
`curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh`
- Install XP
```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait
    # --set nodeSelector."agentpool"=xpjetaks2
```
- Verify status
```
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

#### Configure UXP
- Install UP Command-Line
`brew install upbound/tap/up`
- Install UXP
`up uxp install`
- Verify status
`kubectl get pods -n upbound-system`

### Install providers

   ```console
   # UXP                              
   kubectl apply -f providers/uxp-providers.yaml              
   # XP Native
   kubectl apply -f providers/xp-providers.yaml
   # XP Jet
   kubectl apply -f providers/jet-providers.yaml
   # XP Service
   kubectl apply -f providers/service-providers.yaml

   kubectl get provider
   ```
### Setup Cloud Credentials
   - [Prepare cloud credentials](https://crossplane.io/docs/v1.9/reference/configure.html)

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
KEYVAULT=KeyVault
az keyvault secret show --name uxpAwsCred --vault-name $KEYVAULT --query value -o tsv | sed -r 's@ aws@\naws@g' > aws-cred.conf
az keyvault secret show --name uxpAzureCred --vault-name $KEYVAULT --query value -o tsv | jq > azure-cred.json
az keyvault secret show --name uxpGcpCred --vault-name $KEYVAULT --query value -o tsv | jq > gcp-cred.json
```

## Quick Start

### Configure Cloud Providers

To be able to provision cloud resources using Crossplane we have to create and configure cloud provider resource in Crossplane. This resource stores the cloud information and is used by XP to interact with cloud provider.

We need to set up two environment variables:
- base64 encoded cloud credentials
- Name of the Namespace, for UXP: `upbound-system` for XP: `crossplane-system`

#### AWS Provider

```console
# XP
PROVIDER_SECRET_NAMESPACE=crossplane-system
# UXP
PROVIDER_SECRET_NAMESPACE=upbound-system

BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 aws-cred.conf | tr -d "\n")
eval "echo \"$(cat providers/secret-aws-provider.yaml)\"" | kubectl apply -f -

# Classic Provider
eval "echo \"$(cat providers/aws-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.aws.crossplane.io

# Jet Provider
eval "echo \"$(cat providers/jet-aws-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.aws.jet.crossplane.io

# Official Provider
eval "echo \"$(cat providers/uxp-aws-providerconfig.yaml)\"" | kubectl apply -f -
kubectl get providerconfig
# Validation
kubectl apply -f validation/uxp-aws-bucket.yaml
# UXP validation (Until True/True)
kubectl get Bucket.s3.aws.upbound.io -w
# AWS Validation using cmd (alternatively console UI)
aws s3 ls --profile dev --output table

kubectl delete Bucket.s3.aws.upbound.io uxp-aws-bucket
```

#### Azure Provider

```console
# XP
PROVIDER_SECRET_NAMESPACE=crossplane-system
# UXP
PROVIDER_SECRET_NAMESPACE=upbound-system

BASE64ENCODED_AZURE_ACCOUNT_CREDS=$(base64 azure-cred.json | tr -d "\n")
eval "echo \"$(cat providers/secret-azure-provider.yaml)\"" | kubectl apply -f -

# Classic Provider
eval "echo \"$(cat providers/azure-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.azure.crossplane.io

# Jet Provider
eval "echo \"$(cat providers/jet-azure-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.azure.jet.crossplane.io

# Official Provider
eval "echo \"$(cat providers/uxp-azure-providerconfig.yaml)\"" | kubectl apply -f -
kubectl get providerconfig
# Validation
kubectl apply -f validation/uxp-azure-bucket.yaml
# UXP validation (Until True/True)
kubectl get account.storage.azure.upbound.io/uxpazurebucket007 -w
# Azure Validation using cmd (alternatively console UI)
az group show --resource-group uxp-azure-rg -o table
az storage account show -g uxp-azure-rg -n uxpazurebucket007 -o table

kubectl delete -f validation/uxp-azure-bucket.yaml
```

#### GCP Provider

For GCP we need additionally third environment variable: project ID.

```console
# XP
PROVIDER_SECRET_NAMESPACE=crossplane-system
# UXP
PROVIDER_SECRET_NAMESPACE=upbound-system

PROJECT_ID=$(gcloud projects list --filter='NAME:<Project Name>' --format="value(PROJECT_ID.scope())")
BASE64ENCODED_GCP_PROVIDER_CREDS=$(base64 gcp-cred.json | tr -d "\n")
eval "echo \"$(cat providers/secret-gcp-provider.yaml)\"" | kubectl apply -f -

# Classic Provider
eval "echo \"$(cat providers/gcp-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.gcp.crossplane.io

# Jet Provider
eval "echo \"$(cat providers/jet-gcp-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.gcp.jet.crossplane.io

# Official Provider
eval "echo \"$(cat providers/uxp-gcp-providerconfig.yaml)\"" | kubectl apply -f -
kubectl get providerconfig
# Validation
kubectl apply -f validation/uxp-gcp-bucket.yaml
# UXP validation (Until True/True)
kubectl get Bucket.storage.gcp.upbound.io
# GCP Validation using cmd (alternatively console UI)
gsutil ls -p <PROJECT_ID>

kubectl delete Bucket.storage.gcp.upbound.io uxp-gcp-bucket
```

#### Clean-up

```console
unset BASE64ENCODED_AWS_ACCOUNT_CREDS BASE64ENCODED_AZURE_ACCOUNT_CREDS BASE64ENCODED_GCP_PROVIDER_CREDS PROJECT_ID PROVIDER_SECRET_NAMESPACE
rm aws-cred.conf azure-cred.json gcp-cred.json
```

### Provisioning Infrastructure

Crossplane divides responsibility for the infrastructure provisioning as follows:
1. Ops/SRE defines platform and APIs for Dev team
1. Dev consumes the infrastructure defined by Ops team

#### Defining Infrastructure by Ops team

Platform team creates compositions and composite resource definitions (XRDs) to define and configure
managed kubernetes services infrastructure in cloud.

```console
kubectl apply -f config/definition.yaml

# Compositions using Classic provider
kubectl apply -f config/eks-composition.yaml
kubectl apply -f config/aks-composition.yaml
kubectl apply -f config/gke-composition.yaml

# Compositions using Jet provider
kubectl apply -f config/jet-eks-composition.yaml
kubectl apply -f config/jet-aks-composition.yaml
kubectl apply -f config/jet-gke-composition.yaml

# Compositions using Official provider
kubectl apply -f config/uxp-eks-composition.yaml
kubectl apply -f $CONFIG/jet-aks-composition.yaml
kubectl apply -f $CONFIG/jet-gke-composition.yaml
```

#### Consuming the infrastructure by Dev team

App team provisions infrastructure by creating claim objects for the XRDs defined by Ops team.

```console
kubectl create ns managed

# Claims using classic provider
kubectl apply -f claims/eks-claim.yaml
kubectl apply -f claims/aks-claim.yaml
kubectl apply -f claims/gke-claim.yaml

# Claims using jet provider
kubectl apply -f claims/jet-eks-claim.yaml
kubectl apply -f claims/jet-aks-claim.yaml
kubectl apply -f claims/jet-gke-claim.yaml

# Claims using official provider
kubectl apply -f claims/uxp-eks-claim.yaml
kubectl apply -f $CLAIMS/aks-claim.yaml
kubectl apply -f $CLAIMS/gke-claim.yaml

```

#### Verifying Infrastructure status

Dev team provisions Claims which either generate new Composite Resource (XR) or assign existing ones.

We can check progress using:

```
kubectl get managedcluster -n managed
# Example Output
NAME    CLUSTERNAME   CONTROLPLANE   NODEPOOL    FARGATEPROFILE   READY   CONNECTION-SECRET   AGE
xpaks   xpaks         Succeeded      Succeeded   NA4-xpaks        True    xpaks               24h
xpeks   xpeks         ACTIVE         ACTIVE      ACTIVE           True    xpeks               24h
xpgke   xpgke         RUNNING        RUNNING     NA4-xpgke        True    xpgke               23h
```
To verify status of Helm Charts and Kubernetes Object:
```
kubectl get Object,Release
NAME                                            SYNCED   READY   AGE
object.kubernetes.crossplane.io/xpaks-ns-prod   True     True    23h
object.kubernetes.crossplane.io/xpeks-ns-prod   True     True    23h
object.kubernetes.crossplane.io/xpgke-ns-prod   True     True    22h

NAME                                          CHART        VERSION   SYNCED   READY   STATE REVISION   DESCRIPTION     AGE
release.helm.crossplane.io/xpaks-crossplane   crossplane   1.6.3     True     True    deployed   1  Install complete   23h
release.helm.crossplane.io/xpeks-crossplane   crossplane   1.6.3     True     True    deployed   1  Install complete   23h
release.helm.crossplane.io/xpgke-crossplane   crossplane   1.6.3     True     True    deployed   1  Install complete   22h
```

### Consuming infrastructure

```
# Using secret
kubectl -n managed get secret uxpaks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
kubectl -n managed get secret uxpeks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
# For GKE use Cloud APIs
kubectl -n managed get secret uxpgke --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
# To get credentials for GKE use cloud APIs.
export KUBECONFIG=$PWD/kubeconfig

# Using Cloud APIs
export KUBECONFIG=$PWD/kubeconfig
gcloud container clusters get-credentials cluster-uxpgke --region europe-west2 --project <project name>
az aks get-credentials --resource-group rg-uxpaks --name uxpaks --admin
aws eks --region us-west-1 update-kubeconfig --name uxpeks
```

### Cleanup & Uninstall

#### Delete Claims

Deleting claims will take care of clean-up of all managed resources created to satisfy the claim.

```console
kubectl delete managedcluster -n managed xpeks
kubectl delete managedcluster -n managed xpaks
kubectl delete managedcluster -n managed xpgke
```

#### Delete Cloud Configuration

```console
kubectl get provider
# Clean-up Classic Provider
kubectl delete providerconfig.aws.crossplane.io/aws-provider
kubectl delete providerconfig.azure.crossplane.io azure-provider
kubectl delete providerconfig.gcp.crossplane.io gcp-provider
# Clean-up Jet Provider
kubectl delete providerconfig.aws.jet.crossplane.io/aws-jet-provider
kubectl delete providerconfig.azure.jet.crossplane.io azure-jet-provider
kubectl delete providerconfig.gcp.jet.crossplane.io gcp-jet-provider
```

#### Delete Cloud Secrets

Name of the Namespace, for UXP: `upbound-system` for XP: `crossplane-system`

```console
# UXP: NS=upbound-system, XP: NS=crossplane-system
kubectl delete secret -n $NS aws-account-creds
kubectl delete secret -n $NS azure-account-creds
kubectl delete secret -n $NS gcp-account-creds
```

#### Uninstall Provider

```console
kubectl get provider.pkg
# classic
kubectl delete provider.pkg aws-provider
kubectl delete provider.pkg azure-provider
kubectl delete provider.pkg gcp-provider
# jet
kubectl delete provider.pkg aws-jet-provider
kubectl delete provider.pkg azure-jet-provider
kubectl delete provider.pkg gcp-jet-provider
# Others
kubectl delete provider.pkg helm-provider
kubectl delete provider.pkg kubernetes-provider
```

## Troubleshooting

### Removing resources manually

```
kubectl patch --type json --patch='[ { "op": "remove", "path": "/metadata/finalizers" } ]' subnet.ec2.aws.upbound.io/uxpeks-pub-b
```

### Compositions

```
kubectl get xmanagedcluster

NAME     CLUSTERNAME   CONTROLPLANE   NODEPOOL   FARGATEPROFILE   READY   CONNECTION-SECRET   AGE
uxpeks   uxpeks        ACTIVE         ACTIVE     ACTIVE           True    uxpeks              4d4h
uxpaks   uxpaks        True           True       NA4-uxpaks       True    uxpaks              4d
uxpgke   uxpgke        True           True       NA4-uxpgke       True    uxpgke              2d16h

# To find our which resource have issues within Composite resource:
kubectl describe xmanagedcluster xpeks-5zxn6

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
### Cloud resources

```
kubectl get managed
kubectl get aws
kubectl get azure
kubectl get gcp
```
If above commands do not work you can use below one, examples for Official providers:
```
# AKS
kubectl get ResourceGroup.azure.upbound.io\
,VirtualNetwork.network.azure.upbound.io\
,Subnet.network.azure.upbound.io\
,KubernetesCluster.containerservice.azure.upbound.io\
,KubernetesClusterNodePool.containerservice.azure.upbound.io


# EKS
kubectl get Cluster.eks.aws.upbound.io\
,FargateProfile.eks.aws.upbound.io\
,Nodegroup.eks.aws.upbound.io\
,RouteTableAssociation.ec2.aws.upbound.io\
,Route.ec2.aws.upbound.io\
,Routetable.ec2.aws.upbound.io\
,InternetGateway.ec2.aws.upbound.io\
,Subnet.ec2.aws.upbound.io\
,SecurityGroupRule.ec2.aws.upbound.io\
,SecurityGroup.ec2.aws.upbound.io\
,VPC.ec2.aws.upbound.io\
,RolePolicyAttachment.iam.aws.upbound.io\
,Role.iam.aws.upbound.io\
,ClusterAuth.eks.aws.upbound.io

# GKE
kubectl get Network.compute.gcp.upbound.io\
,Subnetwork.compute.gcp.upbound.io\
,Cluster.container.gcp.upbound.io\
,NodePool.container.gcp.upbound.io
```

## Supported Kubernetes Cluster properties

1. Cluster ID
1. Kubernetes Version
1. Node Size
1. Node Count
1. Region (Cross Cloud [Abstraction](docs/cloud-regions.MD)
1. FargateProfile Namespace (valid for EKS)

## APIs in this Configuration

* `config-xp/`, `config/` - Composite Resource Definition (XRD) with satisfying Compositions
  * [xmanagedcluster XRD](config-xp/definition.yaml)
  * [eks composition](config-xp/eks-composition.yaml),[eks jet composition](config-xp/jet-eks-composition.yaml) includes:
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
  * [aks composition](config-xp/aks-composition.yaml),[aks jet composition](config-xp/jet-aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `AKSCluster` or `KubernetesCluster`
    * `None` or `KubernetesClusterNodePool`
    * `Relase`
    * `Object`   
  * [gke composition](config-xp/gke-composition.yaml),[gke jet composition](config-xp/jet-gke-composition.yaml) includes:
    * `Network`
    * `Subnetwork`
    * `Cluster`
    * `NodePool`
    * `Relase`
    * `Object`    
* `providers/` - Provider Installation and Configuration
  * [Setup](providers/xp-providers.yaml)
  * [AWS Provider Config](providers/aws-provider.yaml)    
  * [Azure Provider Config](providers/azure-provider.yaml)    
  * [GCP Provider Config](providers/gcp-provider.yaml)
  * [Setup Jet](providers/jet-providers.yaml)
  * [AWS Jet Provider Config](providers/jet-aws-provider.yaml)    
  * [Azure Jet Provider Config](providers/jet-azure-provider.yaml)    
  * [GCP Jet Provider Config](providers/jet-gcp-provider.yaml)
* `claim-xp/`, `claims/` - Examples to consume defined by Ops XRDs
  * [EKS Claim](claims-xp/eks-claim.yaml)    
  * [AKS Claim](claims-xp/aks-claim.yaml)    
  * [GKE Claim](claims-xp/gke-claim.yaml)     
  * [EKS Jet Claim](claims-xp/jet-eks-claim.yaml)    
  * [AKS Jet Claim](claims-xp/jet-aks-claim.yaml)    
  * [GKE Jet Claim](claims-xp/jet-gke-claim.yaml)     

## Contributing workflow

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

## Issues

# Release v3.1

RouteTable, SecurityGroup and Nodegroup resources have been fixed. InternetGateway has been added to AWS classic jet provider so I was able to use this one over preview.

New issues found out:
1. Duplicate resource for FargateProfile (Impact: FargateProfile is commented out in eks jet composition)
    - Crossplane Jet AWS [220](https://github.com/crossplane-contrib/provider-jet-aws/issues/220)
    - Hashicorp AWS [26021](https://github.com/hashicorp/terraform-provider-aws/issues/26021)
   Status: Open
1. No Kubeconfig secret created by AWS Jet Provider (Impact: Kubernetes and Helm providers are commented in eks jet composition)
    - Crossplane Jet AWS [229](https://github.com/crossplane-contrib/provider-jet-aws/issues/229)
   Status: Open

### Release v3

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
