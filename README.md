# Nginx + ECR + ECS + Terraform + GitHub Actions

Tento projekt vytváří kompletní deployment pro vlastní nginx image s `index.html`, publikuje image do Amazon ECR a následně nasazuje infrastrukturu do AWS pomocí Terraformu a GitHub Actions. 

## Struktura projektu

```
nginx-ecr-ecs-github/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docker/
│   ├── Dockerfile
│   └── index.html
├── scripts/
│   └── github-oidc-trust-policy.json
├── terraform/
│   ├── alb.tf
│   ├── ecr.tf
│   ├── ecs-service.tf
│   ├── iam.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── securitygroups.tf
│   ├── terraform.tfvars.example
│   ├── variables.tf
│   └── vpc.tf
└── README.md
```

