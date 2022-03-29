# Region Abstraction

| AWS Name | AWS | Azure | GCP | Code
|--|--|--|--|--|
| US East (N. Virginia) | us-east-1 | eastus | us-east4 | USE1 |
| US East (Ohio) | us-east-2 | eastus2 | us-east1 | USE2 |
| US West (N. California) | us-west-1 | westus | us-west2 | USW1 |
| US West (Oregon) | us-west-2 | westus2 | us-west1  | USW2 |
| Canada (Central) | ca-central-1 | canadacentral | northamerica-northeast1 | CAC |
|  |  |  |  |  |
| South America (São Paulo) | sa-east-1 | brazilsouth | southamerica-east1 | SAE | 
|  |  |  |  |  |
| Africa (Cape Town) | af-south-1 | southafricanorth | asia-east1 | AFS | 
| Europe (Frankfurt) | eu-central-1 | germanywestcentral | europe-west3 | EUC |
| Europe (Ireland) | eu-west-1 | northeurope | europe-west4 | EUW1 |
| Europe (London) | eu-west-2 | uksouth | europe-west2 | EUW2 |
| Europe (Paris) | eu-west-3 | francecentral | europe-west1 | EUW3 |
| Europe (Milan) | eu-south-1 | switzerlandnorth | europe-west6 | EUS |
| Europe (Stockholm) | eu-north-1 | swedencentral | europe-north1  | EUN |
|  |  |  |  |  |
| Asia Pacific (Hong Kong) | ap-east-1 | eastasia | asia-east2 | APE |
| Asia Pacific (Mumbai) | ap-south-1 | westindia | asia-south1 | APS1 |
| Asia Pacific (Tokyo) | ap-northeast-1 | japaneast | asia-northeast1 | APNE1 |
| Asia Pacific (Seoul) | ap-northeast-2 | koreacentral | asia-northeast3 | APEN2 |
| Asia Pacific (Osaka) | ap-northeast-3 | japanwest | asia-northeast2 | APEN3 |
| Asia Pacific (Singapore) | ap-southeast-1 | southeastasia | asia-southeast1 | APSE1 |
| Asia Pacific (Sydney) | ap-southeast-2 | australiaeast | australia-southeast1 | APSE2 |
| Asia Pacific (Jakarta) | ap-southeast-3 | australiacentral | asia-southeast2 | APSE3 |

# Availability Zones (AZ) in AWS

## Disabled by default

Following AZs have to be enabled in console before usage

```
afs: af-south-1       # default: disabled
eus: eu-south-1       # default: disable`
ape: ap-east-1        # default: disabled
apse3: ap-southeast-3 # default: disable`
```

## AZs in regions

Three Standard AZs for most of the regions with postfix a,b,c: 
```
us-east-2: us-east-2a, us-east-2b, us-east-2c   
```
Exceptions: (Important for EKS compositions: Subnet and Route Tables  )

use1: us-east-1       # a-f, 6 zones
usw2: us-west-2       # a-d, 4 zones
apne2: ap-northeast-2 # a-d, 4 zones

usw1: us-west-1       # ac,  no B zone, in XR Zone B mapped to A
cac: ca-central-1     # abd, no C Zone, in XR Zone C mapped to B (Fargate profile not supported in zone D)
apne1: ap-northeast-1 # acd, no B zone, in XR Zone B mapped to D


