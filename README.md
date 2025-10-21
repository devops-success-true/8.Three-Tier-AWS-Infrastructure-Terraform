# Three-Tier AWS Infrastructure (Terraform)

**Frontend in public subnets, backend + database in private subnets, spread across two Availability Zones with public and internal load balancers.**

## What this repo contains (and how pieces relate)

```
.
├─ modules/                      # Reusable Terraform modules consumed by root stacks
├─ networking.tf.tf              # VPC, subnets (public/private, 2x AZ), IGW, NAT, routes
├─ webTier.tf.tf                 # Web tier (public) + Public ALB; SG rules to app tier
├─ appTier.tf.tf                 # App tier (private) + Internal LB; SG rules to DB tier
├─ dataTier.tf.tf                # DB tier (private, Multi-AZ RDS optional); SG hardening
├─ variables.tf.tf               # Inputs shared across stacks (CIDRs, AZs, sizes, names)
├─ outputs.tf.tf                 # Key outputs (ALB DNS, subnets, SG IDs, endpoints)
├─ provider.tf.tf                # AWS provider + region/account config
├─ terraform.tfvars              # Your env defaults (edit here)
├─ web.sh.sh, app.sh.sh          # Bootstrap/user-data style scripts for tiers
└─ README.md                     # You’re reading it (rename from README.md.md if needed)
```

### How root files drive `modules/`
- **`networking.tf.tf`** instantiates the **networking module(s)** from `modules/` to build the **VPC foundation** (VPC, 2× public + 2× private subnets, route tables, IGW, NAT). Everything else depends on these IDs.  
- **`webTier.tf.tf`** calls the **web module** to create **public ALB → target group → EC2/ASG (or LT)** in **public subnets**. Security groups allow **80/443 from the internet** and **egress to app tier**.  
- **`appTier.tf.tf`** calls the **app module** to place **private compute** (EC2/ASG or containers) behind an **internal LB** in **private subnets**. SGs allow **only web tier** to reach app ports (e.g., 8080/3000).  
- **`dataTier.tf.tf`** calls the **data module** for **RDS** (or other data store) in **private subnets** with **Multi-AZ** option. SGs allow **only app tier** to reach DB port (e.g., 5432/3306).  
- **`variables.tf.tf` / `terraform.tfvars`** supply **CIDRs, AZs, instance sizes, node counts**, names, and toggles (e.g., multi-AZ).  
- **`outputs.tf.tf`** returns **ALB DNS names, subnet IDs, SG IDs, DB endpoint** so you can connect apps/tests.  
- **`provider.tf.tf`** pins **AWS provider + region**.

---

## End-to-end architecture

```
Internet
   │
   ▼
[Public ALB] ──► Web Tier (EC2/ASG)  [Public Subnets, AZ-a + AZ-b]
                     │
                     ▼
          [Internal NLB/ALB] ──► App Tier (EC2/ASG/Service)
                     │
                     ▼
                 RDS (Multi-AZ)
         [Private Subnets, AZ-a + AZ-b]
```

**Availability:** Every tier is placed across **two AZs**. ALBs/NLBs are **multi-AZ**. DB is **Multi-AZ** for failover. Public traffic only hits the **public ALB**. App and DB have **no public IPs**.

**Connectivity & communication (SG policy):**
- **Internet → Public ALB (80/443)**  
- **Public ALB → Web instances (web ports, e.g., 80/443)**  
- **Web SG → App SG (app port, e.g., 8080/3000)**  
- **App SG → DB SG (db port, e.g., 5432/3306)**  
- **Egress** locked down to what tiers actually need (NAT for outbound from private subnets).

**Routing:**
- **Public subnets**: default route → **IGW**  
- **Private subnets**: default route → **NAT Gateway(s)**  
- App/DB never traverse the internet directly; outbound goes via NAT.

**Application flow:**
1. Client hits **public ALB DNS** → forwards to **web tier** targets.  
2. Web tier calls **internal LB** → dispatch to **app tier** targets.  
3. App tier connects to **RDS endpoint** within private subnets.  
4. Responses goes back the same path.

---

## Modules (intent and relationships)

| Module | What it creates | Consumed by | Outputs used by |
|---|---|---|---|
| **networking** | VPC, 2×(public+private) subnets, route tables, IGW, NAT | `networking.tf.tf` | web, app, data (subnet IDs, VPC ID, RTs) |
| **web** | Public ALB, TG, listeners, SG; EC2/ASG/LaunchTemplate; user-data (`web.sh.sh`) | `webTier.tf.tf` | app (web SG id), outputs (ALB DNS) |
| **app** | Internal NLB/ALB, SG; EC2/ASG/LaunchTemplate; user-data (`app.sh.sh`) | `appTier.tf.tf` | data (app SG id), outputs (internal LB DNS) |
| **data** | RDS (Multi-AZ), subnet group, SG, parameter group, backups | `dataTier.tf.tf` | outputs (DB endpoint, SG id) |

---

## High availability (real-world checklist)

- **Multi-AZ everywhere:**  
  - Subnets in **AZ-a** and **AZ-b**.  
  - **ALB/NLB** spans both AZs.  
  - **RDS Multi-AZ** for automatic failover.  
- **Stateless web/app tiers** with **ASG** and **health checks** on target groups.  
- **No single NAT**: prefer **one NAT per AZ** to avoid cross-AZ data charges/failure blast radius.  
- **Cross-AZ SG & route correctness** verified by health/status of target groups.

---

## Security model

- **Public exposure only on ALB (layer 7)**; instances remain shielded.  
- **Tier-to-tier allow-list:** web→app only on app port; app→db only on DB port.  
- **No public IPs** on app/db.  
- **IAM** for instances (S3/SSM etc.).  
- **DB backups + minor version auto-upgrade** recommended.  
- **TLS**: terminate at ALB; to be added ACM cert for HTTPS.

---

## Inputs & Outputs

- **Inputs** (`variables.tf.tf` + `terraform.tfvars`): region, AZs, CIDRs, instance types/counts, ALB/NLB flags, RDS engine/class/multi-az, names/tags.  
- **Outputs** (`outputs.tf.tf`): public ALB DNS, internal LB DNS, VPC/subnet IDs, SG IDs, DB endpoint to wire your app quickly.

---

## How to run

```bash
# 1) Clone
git clone https://github.com/devops-success-true/8.Three-Tier-AWS-Infrastructure-Terraform.git
cd 8.Three-Tier-AWS-Infrastructure-Terraform

# 2) Configure AWS auth (one of)
aws configure
# or export AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_DEFAULT_REGION

# 3) Review terraform.tfvars (CIDRs, AZs, sizes, counts, RDS settings)
# 4) Init/Plan/Apply
terraform init
terraform plan
terraform apply
```

When apply completes, grab the **public ALB DNS** from outputs and paste/hit it in a browser. The app path should transit **public ALB → web → internal LB → app → DB** as designed.

---

## Ops notes (what DevOps teams actually do)

- **Parameterize**: keep everything env-driven via `terraform.tfvars` (or `*.auto.tfvars`).  
- **State**: use **S3 + DynamoDB** backend for locking in real deployments.  
- **Pipelines**: run `terraform fmt/validate/tflint/checkov` in CI, then plan/apply per workspace.  
- **Logs/metrics**: enable **ALB/NLB access logs**, **RDS enhanced monitoring**, **CloudWatch alarms** for health.  
- **Cost**: when doing a demo, scale instance sizes down and disable Multi-AZ DB.

---

## Author

**Kastro** – DevOps & Cloud Engineer  
Focus: AWS, Terraform, Kubernetes, CI/CD, Cloud Infrastructure.  
GitHub: [devops-success-true](https://github.com/devops-success-true)
