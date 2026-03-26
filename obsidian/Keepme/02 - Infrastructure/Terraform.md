# Terraform (keepme-infra)

## Structure

```
infrastructure/eu-west-2/
├── shared/
│   ├── eks/          # EKS cluster provisioning
│   ├── iam-roles/    # IAM roles & policies
│   ├── secrets/      # SOPS + KMS secret management
│   └── ecr/          # Elastic Container Registry
└── production/
    └── terraform/
        ├── keepme-v4-eu-rds/
        ├── keepme-v4-us-rds/
        └── keepme-v4-au-rds/

modules/
├── vpc/              # VPC, subnets, route tables
├── eks/              # EKS control plane & workers
├── iam/              # IAM role/policy creation
└── eks-addons/       # VPC CNI, CoreDNS, kube-proxy
```

## Commands

```bash
cd keepme-infra/infrastructure/eu-west-2/shared/eks
terraform init
terraform plan
terraform apply
```

## AWS Account
- Account ID: `760505282981`
- Redshift IAM role: `arn:aws:iam::760505282981:role/service-role/AmazonRedshift-CommandsAccessRole-20260321T132138`
