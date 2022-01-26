# Multi-cloud Managed Kubernetes

This project was created to build managed Kubernetes `Composite Resource` (XR) with 
compositions across multiple cloud providers. You can use it as a foundation to understand, build
and operate managed Kubernetes Platform in the Cloud.
This repository uses Upbound Universal Crossplane ([UXP](https://www.upbound.io/products/uxp)), 
which is downstream distribution of Crossplane (XP)
Compositions use native Crossplane Providers for AWS, Azure and GCP.

* [AWS Provider](https://doc.crds.dev/github.com/crossplane/provider-aws)
* [GCP Provider](https://doc.crds.dev/github.com/crossplane/provider-gcp)
* [Azure Provider](https://doc.crds.dev/github.com/crossplane/provider-azure)

## Prerequisites

1. Install Kubernetes Cluster
1. Configure UXP
   - Install UXP cli
   - Install UXP
   - Configure profile
   - Connect to Upbound Cloud
1. Install cloud providers
   ```console
   kubectl apply -f providers/providers.yaml
   kubectl get providers.pkg
   ```
1. Setup Cloud Credentials
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

### Configure Providers

To be able to provision cloud resources using XP we have to create and configure cloud provider resource in Crossplane. This resource stores the cloud information and is used by XP to interact with cloud provider.

We need to set up two environment variables:
- base64 encoded cloud credentials
- Name of the Namespace, for UXP: `upbound-system` for XP: `crossplane-system` 

#### AWS Provider

```console
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(base64 aws-cred.conf | tr -d "\n")
PROVIDER_SECRET_NAMESPACE=upbound-system
eval "echo \"$(cat providers/aws-provider.yaml)\"" | kubectl apply -f -
kubectl get providerconfig aws-provider # or kubectl get providerconfig
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
kubectl apply -f config/definition.yaml
kubectl apply -f config/eks-composition.yaml 
kubectl apply -f config/aks-composition.yaml 
kubectl apply -f config/gke-composition.yaml 
``` 

#### Consuming the infrastructure by Dev team

App team provisions infrastructure by creating claim objects for the XRDs defined by Ops team.

```console
kubectl create ns managed
kubectl apply -f claims/eks-claim.yaml 
kubectl apply -f claims/aks-claim.yaml 
kubectl apply -f claims/gke-claim.yaml 
``` 

#### Verifying Infrastructure status

Dev team provisions Claims which either generate new Composite Resource (XR) or assign existing ones.

We can check progress using:

```
kubectl get managedcluster -n managed
# Example Output
NAME     CLUSTERNAME   CONTROLPLANE   NODEPOOL    FARGATEPROFILE   READY   CONNECTION-SECRET   AGE
uxpaks   uxpaks        Succeeded      Succeeded   NA4-uxpaks       True    uxpaks              29m
uxpeks   uxpeks        ACTIVE         ACTIVE      ACTIVE           True    uxpeks              79m
uxpgke   uxpgke        RUNNING        RUNNING     NA4-uxpgke       True    uxpgke              52m
```

### Cleanup & Uninstall

#### Delete Claims

Deleting claims will take care of clean-up of all managed resources created to satisfy the claim.

```console
kubectl delete managedcluster -n managed uxpeks 
kubectl delete managedcluster -n managed uxpaks 
kubectl delete managedcluster -n managed uxpgke 
``` 

#### Delete Cloud Configuration

```console
kubectl delete providerconfig aws-provider
kubectl delete providerconfig.azure.crossplane.io azure-provider
kubectl delete providerconfig.gcp.crossplane.io gcp-provider
``` 

#### Delete Cloud Secrets

```console
kubectl delete secret -n upbound-system aws-account-creds
kubectl delete secret -n upbound-system azure-account-creds
kubectl delete secret -n upbound-system gcp-account-creds
``` 

#### Uninstall Provider

```console
kubectl delete provider.pkg provider-aws
kubectl delete provider.pkg provider-azure
kubectl delete provider.pkg provider-gcp
``` 

## APIs in this Configuration

* `config/` - Composite Resource Definition (XRD) with satisfying Compositions
  * [xmanagedcluster XRD](config/definition.yaml)
  * [eks composition](config/eks-composition.yaml) includes :
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
  * [aks composition](config/aks-composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
    * `AKSCluster`
  * [gke composition](config/gke-composition.yaml) includes:
    * `Network`
    * `Subnetwork`
    * `Cluster`
    * `NodePool`
* `providers/` - Provider Installation and Configuration
  * [Setup](providers/providers.yaml) 
  * [AWS Provider Config](providers/aws-provider.yaml)    
  * [Azure Provider Config](providers/azure-provider.yaml)    
  * [GCP Provider Config](providers/gcp-provider.yaml)
* `claims/` - Examples to consume defined by Ops XRDs
  * [EKS Claim](claims/eks-claim.yaml)    
  * [AKS Claim](claims/aks-claim.yaml)    
  * [GKE Claim](claims/gke-claim.yaml)     

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
