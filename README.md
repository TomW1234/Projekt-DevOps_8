# Nginx + ECR + ECS + Terraform + GitHub Actions

Tento projekt vytváří kompletní deployment pro vlastní nginx image s `index.html`, publikuje image do Amazon ECR a následně nasazuje infrastrukturu do AWS pomocí Terraformu a GitHub Actions. Původní přiložené Terraform soubory už obsahují VPC, public/private subnety, NAT gateway, ALB, ECS cluster, ECS service, security groups a ECS task execution roli, takže nově bylo potřeba hlavně doplnit ECR repository a změnit task definition z veřejného `nginx:alpine` na image z ECR [file:9][file:1][file:2][file:5].

## Struktura projektu

```text
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

## Co dělají Terraform soubory

- `providers.tf` nastavuje AWS provider a zachovává S3 backend, který už byl v přiloženém řešení použit pro ukládání Terraform state [file:10][file:4].
- `variables.tf` vychází z původních proměnných pro region, project name a CIDR rozsahy a přidává `container_image_tag`, aby šel image tag řídit přes Terraform [file:8].
- `vpc.tf` zachovává původní VPC architekturu: dvě public subnety, dvě private subnety, internet gateway, NAT gateway a routovací tabulky [file:9].
- `securitygroups.tf` ponechává ALB přístup z internetu na port 80 a ECS taskům povoluje port 80 jen z ALB security group [file:6].
- `alb.tf` vytváří veřejný Application Load Balancer, target group typu `ip` a HTTP listener na portu 80 s health checkem na `/` [file:1].
- `iam.tf` používá ECS task execution roli s managed policy `AmazonECSTaskExecutionRolePolicy`, což odpovídá původnímu zadání pro execution role [file:5].
- `ecr.tf` nově vytváří Amazon ECR repository, kam GitHub Actions pushne sestavený image.
- `ecs-service.tf` mění task definition tak, aby image bral z `aws_ecr_repository.nginx.repository_url` místo `nginx:alpine`; execution role zůstává napojená na ECS task execution roli [file:2][file:5].
- `outputs.tf` rozšiřuje výstupy o ECR URL a ALB DNS; původní soubory už vracely ALB DNS, VPC ID a subnet IDs [file:3].

## Co dělají Docker soubory

- `docker/Dockerfile` staví image na `nginx:alpine`, což je stejné nginxové jádro jako v původní task definition, jen nyní s vlastním obsahem [file:2].
- `docker/index.html` je vlastní statická stránka, která se kopíruje do `/usr/share/nginx/html/index.html`.

## GitHub Actions workflow

Workflow v `.github/workflows/deploy.yml` provádí přesně pořadí požadované zadáním: nejdřív `terraform apply -target=aws_ecr_repository.nginx`, potom build Docker image, push do ECR a nakonec plný `terraform apply`, aby ECS service už startovala nad existujícím image v registru [file:2]. Toto pořadí je důležité, protože bez existující ECR repository a nahraného image by task definition s odkazem na ECR image neměla co spustit [file:2].

## Co nastavit v AWS před prvním spuštěním

### 1. Vytvoř GitHub OIDC provider v AWS

V AWS IAM vytvoř OIDC provider pro GitHub Actions:

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 2. Vytvoř IAM roli pro GitHub Actions

Vytvoř IAM roli, kterou bude GitHub Actions assumovat přes OIDC. Trust policy je připravená v `scripts/github-oidc-trust-policy.json`; uprav v ní:

- `<AWS_ACCOUNT_ID>`
- `<GITHUB_OWNER>`
- `<GITHUB_REPO>`

### 3. Připoj oprávnění k této GitHub roli

Na GitHub Actions roli připoj práva minimálně pro:

- ECR: create/list/push image
- ECS: create/update/read service a task definition
- IAM: read pass role pro ECS execution role
- EC2/ELB: vytváření síťových a load balancer resources
- S3 + DynamoDB, pokud backend/state locking používáš mimo ukázku
- Terraform resources podle použité infrastruktury

Prakticky na cvičení bývá nejrychlejší dočasně použít širší deployment policy, ale produkčně je lepší least privilege.

## Co nastavit v GitHub repozitáři

V repozitáři v **Settings -> Secrets and variables -> Actions** vytvoř secret:

- `AWS_GITHUB_ACTIONS_ROLE_ARN` = ARN role, kterou GitHub Actions assumuje přes OIDC

Pokud chceš, můžeš region a project name přesunout z workflow do repository variables, ale v této ukázce jsou nastavené přímo v YAML.

## Přesný postup nasazení přes GitHub CI/CD

### Varianta A: první automatický deployment z branch `main`

1. Nahraj celou strukturu projektu do GitHub repozitáře.
2. Uprav `scripts/github-oidc-trust-policy.json` a vytvoř podle něj IAM roli v AWS.
3. Ulož ARN role do GitHub secret `AWS_GITHUB_ACTIONS_ROLE_ARN`.
4. Zkontroluj, že S3 backend v `terraform/providers.tf` odpovídá existujícímu bucketu; původní řešení už S3 backend používalo [file:10].
5. Commitni a pushni změny do `main`.
6. GitHub Actions spustí workflow `Deploy nginx to AWS ECS via Terraform`.
7. Workflow zavolá `terraform init`.
8. Workflow vytvoří pouze ECR přes `terraform apply -target=aws_ecr_repository.nginx`.
9. Workflow si vezme `ecr_repository_url` z Terraform output.
10. Workflow provede login do ECR.
11. Workflow sestaví image z `docker/Dockerfile` a `docker/index.html`.
12. Workflow otaguje image jako `<ecr_repository_url>:latest`.
13. Workflow image pushne do ECR.
14. Workflow spustí plný `terraform apply`.
15. Terraform vytvoří nebo aktualizuje zbytek infrastruktury: ECS task definition, ECS service, ALB a síťové resources podle konfigurace [file:1][file:2][file:6][file:9].
16. Na konci workflow vypíše `alb_dns_name`; ten pak otevřeš v prohlížeči [file:3].

### Varianta B: ruční spuštění z GitHub Actions

Workflow obsahuje i `workflow_dispatch`, takže ho můžeš spustit ručně z karty **Actions** bez nového commitu. To se hodí při testování IAM nebo při opakovaném deployi stejného commitu.

## Jak funguje nasazení interně

Architektura je tato:

- Docker image s vlastním `index.html` se uloží do ECR.
- ECS task definition odkazuje na ECR URL místo veřejného Docker Hub image [file:2].
- ECS service běží na Fargate ve private subnetech bez public IP [file:2][file:9].
- Internetový provoz jde přes veřejný ALB v public subnetech [file:1][file:9].
- Security group dovolí provoz z internetu jen do ALB a z ALB dál jen na ECS tasky na portu 80 [file:6].

## Doporučené úpravy před odevzdáním

- Změň `project_name`, aby resources měly vlastní jednoznačný prefix [file:8].
- Zkontroluj bucket a key v Terraform backendu, protože v přiloženém řešení jsou natvrdo nastavené na konkrétní S3 bucket [file:10].
- Pokud vyučující chce přesně jeden task, změň `desired_count` na 1; v přiloženém souboru je nyní 2, zatímco README předchozí lekce popisuje 1, takže si to sjednoť podle zadání [file:2][file:7].
- Doporučuje se přidat `.gitignore` pro `.terraform/`, `terraform.tfstate*` a případné lokální soubory.

## Ověření po deployi

Po úspěšném běhu workflow ověř:

- v ECR existuje repository a obsahuje image tag `latest`,
- v ECS běží service i tasky,
- target group v ALB má healthy targety,
- `terraform output alb_dns_name` vrací veřejnou adresu load balanceru [file:3],
- po otevření ALB DNS se zobrazí tvůj vlastní `index.html`.
