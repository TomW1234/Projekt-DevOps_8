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


- v ECR existuje repository a obsahuje image tag `latest`,
- v ECS běží service i tasky,
- target group v ALB má healthy targety,
- `terraform output alb_dns_name` vrací veřejnou adresu load balanceru [file:3],
- po otevření ALB DNS se zobrazí tvůj vlastní `index.html`.
