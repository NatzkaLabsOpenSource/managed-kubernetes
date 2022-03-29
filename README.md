# What's New

1. Support for Upstream Crossplane
1. Support for [Regions](docs/cloud-regions.MD) in Composite Resource
1. Postprovisioning using Helm and Kubernetes Providers

# Multi-cloud Managed Kubernetes

This project was created to build managed Kubernetes `Composite Resource` (XR) with 
compositions across multiple cloud providers. You can use it as a foundation to understand, build
and operate managed Kubernetes Platform in the Cloud.
This repository uses both upstream ([Crossplane (XP)](https://crossplane.io/)) and its downstream version ([Upbound Universal Crossplane (UXP)](https://www.upbound.io/products/uxp)).
Compositions use for cloud provisioning Native/Classic Crossplane Providers for AWS, Azure and GCP and for the post provisioning Helm and Kubernetes:

* [AWS Provider](https://doc.crds.dev/github.com/crossplane/provider-aws)
* [GCP Provider](https://doc.crds.dev/github.com/crossplane/provider-gcp)
* [Azure Provider](https://doc.crds.dev/github.com/crossplane/provider-azure)

* [Helm](https://doc.crds.dev/github.com/crossplane-contrib/provider-helm)
* [Kubernetes](https://doc.crds.dev/github.com/crossplane-contrib/provider-kubernetes)

To demonstrate usage for both post provisioning resources I created following examples:
* Crossplane Provisioning using Helm Chart
+ Production Namespace Provisioning using Kubernetes Manifest

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
```
- Verify status
```
helm list -n crossplane-system

kubectl get all -n crossplane-system
```

#### Configure UXP
- Install UXP cli
- Install UXP
- Configure profile
- Connect to Upbound Cloud

### Install providers
   ```console
   # UXP                                                  XP                                   
   kubectl apply -f providers/providers.yaml              kubectl apply -f providers/xp-providers.yaml
   
   kubectl get providers.pkg
   ```
### Setup Cloud Credentials
   - [IAM User](https://crossplane.io/docs/v1.6/cloud-providers/aws/aws-provider.html) for AWS
   - [Service Principal](https://crossplane.io/docs/v1.6/cloud-providers/azure/azure-provider.html) for Azure
   - [Service Account](https://crossplane.io/docs/v1.6/cloud-providers/gcp/gcp-provider.html) for GCP  

As an output of last step you should get three credentials files with following content.
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
To retrive credential files from Azure KV you can use following:

```console
az keyvault secret show --name uxpAzureCred --vault-name KeyVault --query value -o tsv | jq > azure-cred.json
az keyvault secret show --name uxpAwsCred --vault-name KeyVault --query value -o tsv | sed -r 's@ aws@\naws@g' > aws-cred.conf
az keyvault secret show --name uxpGcpCred --vault-name KeyVault --query value -o tsv | jq > gcp-cred.json
```

## Quick Start

### Configure Cloud Providers

To be able to provision cloud resources using Crossplane we have to create and configure cloud provider resource in Crossplane. This resource stores the cloud information and is used by XP to interact with cloud provider.

We need to set up two environment variables:
- base64 encoded cloud credentials
- Name of the Namespace, for UXP: `upbound-system` for XP: `crossplane-system` 

#### AWS Provider

```console
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 aws-cred.conf | tr -d "\n")
PROVIDER_SECRET_NAMESPACE=upbound-system
eval "echo \"$(cat providers/aws-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.aws.crossplane.io
```

#### Azure Provider

```console
BASE64ENCODED_AZURE_ACCOUNT_CREDS=$(base64 azure-cred.json | tr -d "\n")
PROVIDER_SECRET_NAMESPACE=upbound-system
eval "echo \"$(cat providers/azure-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.azure.crossplane.io
```

#### GCP Provider

For GCP we need additionally third environment variable: project ID.

```console
PROJECT_ID=$(gcloud projects list --filter='NAME:<Project Name>' --format="value(PROJECT_ID.scope())")
BASE64ENCODED_GCP_PROVIDER_CREDS=$(base64 gcp-cred.json | tr -d "\n")
PROVIDER_SECRET_NAMESPACE=upbound-system
eval "echo \"$(cat providers/gcp-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig.gcp.crossplane.io
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
# UXP: CONFIG=config, XP: CONFIG=config-xp                                           
kubectl apply -f $CONFIG/definition.yaml
kubectl apply -f $CONFIG/eks-composition.yaml
kubectl apply -f $CONFIG/aks-composition.yaml
kubectl apply -f $CONFIG/gke-composition.yaml
``` 

#### Consuming the infrastructure by Dev team

App team provisions infrastructure by creating claim objects for the XRDs defined by Ops team.

```console
kubectl create ns managed
# UXP: CLAIMS=claims, XP: CLAIMS=claims-xp
kubectl apply -f $CLAIMS/eks-claim.yaml 
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
kubectl -n managed get secret xpaks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
kubectl -n managed get secret xpeks --output jsonpath="{.data.kubeconfig}" | base64 -d | tee kubeconfig
# To get credentials for GKE use cloud APIs. 
export KUBECONFIG=$PWD/kubeconfig

# Using Cloud APIs
gcloud container clusters get-credentials uxpgke --zone us-west2 --project <project name>
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
kubectl delete providerconfig aws-provider
kubectl delete providerconfig.azure.crossplane.io azure-provider
kubectl delete providerconfig.gcp.crossplane.io gcp-provider
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
kubectl delete provider.pkg aws-provider
kubectl delete provider.pkg azure-provider
kubectl delete provider.pkg gcp-provider
kubectl delete provider.pkg helm-provider
kubectl delete provider.pkg kubernetes-provider
```

## Troubleshooting

### Compositions

```
kubectl get xmanagedcluster

NAME          CLUSTERNAME   CONTROLPLANE   NODEPOOL    FARGATEPROFILE   READY   COMPOSITION   AGE
xpaks-72lq5   xpaks         Succeeded      Succeeded   NA4-xpaks        True    aks           25h
xpeks-5zxn6   xpeks         ACTIVE         ACTIVE      ACTIVE           True    eks           24h
xpgke-w29h6   xpgke         RUNNING        RUNNING     NA4-xpgke        True    gke           23h

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

### AWS

```console
kubectl get Role.iam.aws.crossplane.io,\
RolePolicyAttachment.iam.aws.crossplane.io,\
VPC.ec2.aws.crossplane.io,\
SecurityGroup.ec2.aws.crossplane.io,\
Subnet.ec2.aws.crossplane.io,\
InternetGateway.ec2.aws.crossplane.io,\
RouteTable.ec2.aws.crossplane.io,\
Cluster.eks.aws.crossplane.io,\
NodeGroup.eks.aws.crossplane.io,\
FargateProfile.eks.aws.crossplane.io,\
ProviderConfig.helm.crossplane.io,\
Release.helm.crossplane.io,\
ProviderConfig.kubernetes.crossplane.io,\
Object.kubernetes.crossplane.io
```

### Azure

```console
kubectl get ResourceGroup.azure.crossplane.io,\
VirtualNetwork.network.azure.crossplane.io,\
Subnet.network.azure.crossplane.io,\
AKSCluster.compute.azure.crossplane.io,\
ProviderConfig.helm.crossplane.io,\
Release.helm.crossplane.io,\
ProviderConfig.kubernetes.crossplane.io,\
Object.kubernetes.crossplane.io
```

### GCP

```console
kubectl get Network.compute.gcp.crossplane.io,\
Subnetwork.compute.gcp.crossplane.io,\
Cluster.container.gcp.crossplane.io,\
NodePool.container.gcp.crossplane.io,\
ProviderConfig.helm.crossplane.io,\
Release.helm.crossplane.io,\
ProviderConfig.kubernetes.crossplane.io,\
Object.kubernetes.crossplane.io
```

## Supported Kubernetes Cluster properties

1. Cluster ID
1. Kubernetes Version
1. Node Size
1. Node Count
1. Region (Cross Cloud [Abstraction](docs/cloud-regions.MD)
1. FargateProfile Namespace (valid for EKS)

## APIs in this Configuration

* `config-xp/` - Composite Resource Definition (XRD) with satisfying Compositions (`config/` for UXP)
  * [xmanagedcluster XRD](config-xp/definition.yaml)
  * [eks composition](config-xp/eks-composition.yaml) includes :
    * `Role`
    * `RolePolicyAttachment`
    * `VPC`
    * `SecurityGroup`
    * `Subnet`
    * `InternetGateway`
    * `RouteTable`
    * `Cluster`
    * `NodeGroup`
    * `FargateProfile`
    * `Relase`
    * `Object`
  * [aks composition](config-xp/aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `AKSCluster`
    * `Relase`
    * `Object`   
  * [gke composition](config-xp/gke-composition.yaml) includes:
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
* `claim-xp/` - Examples to consume defined by Ops XRDs (`claims/` for UXP)
  * [EKS Claim](claims-xp/eks-claim.yaml)    
  * [AKS Claim](claims-xp/aks-claim.yaml)    
  * [GKE Claim](claims-xp/gke-claim.yaml)     

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
