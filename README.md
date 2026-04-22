# Terraform-Demo
Terraform Demo

# Setting up Environment
# Terraform Installation for windows using CMD
winget -e --id Hashicorp.Terraform --version 1.12.2

# Setting up GCP credentials for terraform
 Create a service account in GCP
 Add & store the Key

# Setting GITHUB PAT & Adding to ENV variable
set GITHUB_TOKEN=your_token_here # CMD 
curl -H "Authorization: Bearer %GITHUB_TOKEN%" https://api.github.com/user # Verify


## Interview Talking Points

### 1. Why modules over monolithic Terraform?
> Modules enforce the **single-responsibility principle** — each module owns one domain (network, compute, GKE). This means: independent `terraform apply -target`, isolated state for blast radius control, reuse across environments/projects, and testability with tools like Terratest.

### 2. Why remote state in GCS?
> Local state breaks team workflows — concurrent applies cause state corruption. GCS backend gives you **state locking via Cloud Spanner** (built into the GCS backend), versioning for rollback, and encryption at rest. The state bucket is bootstrapped manually to avoid the chicken-and-egg problem.

### 3. Why Cloud NAT instead of public IPs?
> Public IPs on every VM are an attack surface. Cloud NAT lets private instances reach the internet for outbound traffic (package installs, image pulls) while being completely unreachable from the internet inbound. Combined with IAP for SSH, zero resources need a public IP.

### 4. Why Workload Identity over SA key files?
> SA key files are long-lived credentials that can leak via code, logs, or storage. Workload Identity federates Kubernetes ServiceAccounts with GCP IAM at the API level — no JSON key files exist. Access is automatically rotated via short-lived tokens issued by the GKE metadata server.

### 5. Why a regional GKE cluster?
> Zonal clusters have a single-point-of-failure on the control plane. A regional cluster replicates the control plane across 3 zones. Node pools span all zones by default. For prod, this means no downtime during GCP zonal maintenance windows.

### 6. How do you handle secrets rotation?
> SA keys stored in Vault can be rotated by: (1) generating a new key, (2) `vault kv put` to update the secret, (3) CI re-reads on next pipeline run. The old key can be revoked in IAM. Workload Identity tokens rotate automatically every hour — no manual rotation needed for pod credentials.

### 7. What is the blast radius of this setup?
> Each component is isolated in its own module with its own service account. If a storage bucket is misconfigured, it cannot affect the GKE cluster's IAM. Module-level `depends_on` controls creation order. Targeted destroys (`-target`) let you tear down one component without touching others.

---

## Common Interview Questions & Answers

**Q: What happens if two engineers run `terraform apply` simultaneously?**
> The GCS backend uses **state locking**. The second apply will fail with a lock error and show the lock ID + the identity that holds it. You can force-unlock with `terraform force-unlock <lock-id>` if the holder crashed — but always verify the other apply completed first.

**Q: How do you manage Terraform across multiple environments without code duplication?**
> Environment-specific `terraform.tfvars` files override root variable defaults. The modules are environment-agnostic; they accept `var.environment` and use it in naming/labels. For larger orgs, Terragrunt adds DRY wrappers and remote state config per environment.

**Q: What is VPC-native GKE and why does it matter?**
> VPC-native (alias IP) clusters assign pod IPs from a subnet secondary range. This means pods are first-class VPC citizens — you can create firewall rules targeting pod IP ranges directly, route traffic to pods from on-prem via VPN/Interconnect, and avoid double-NAT. The alternative (routes-based) clusters have a limit of 1000 routes per VPC and can't use Network Policy.

**Q: How would you debug a GKE pod that can't reach Cloud Storage?**
> Checklist: (1) Is `private_ip_google_access = true` on the subnet? (2) Does the pod's GCP SA (`sa-app-dev`) have `roles/storage.objectUser`? (3) Is the Workload Identity annotation on the Kubernetes SA correct? (4) `kubectl exec` into the pod and test `gcloud storage ls` with the workload identity credential.

**Q: What's the difference between `terraform taint` and `terraform apply -replace`?**
> `taint` was deprecated in Terraform 0.15.2. The modern equivalent is `terraform apply -replace=<resource>`. Both mark a resource for destruction and recreation in the next apply. Use case: force-recreate a VM after a startup script change that doesn't trigger a diff in Terraform.

**Q: How does Pub/Sub guarantee delivery?**
> Pub/Sub is an at-least-once delivery system. Messages are persisted until acknowledged. The subscription's `ack_deadline_seconds` (60s here) gives consumers time to process before re-delivery. The dead-letter policy catches messages that fail 5 times, preventing poison-pill messages from blocking the subscription.

**Q: How would you add a new environment (e.g., staging)?**
> 1. Create `environments/staging/terraform.tfvars` with staging-specific values.
> 2. Update the GCS backend `prefix` to `terraform/staging` (or use a workspace).
> 3. Run `terraform init` with the new backend config.
> 4. `terraform apply -var-file=environments/staging/terraform.tfvars`.
> No module changes needed — the modules are environment-agnostic.

**Q: What would you add to make this truly production-grade?**
> - **Cloud Armor** WAF in front of the GKE ingress
> - **Binary Authorization** to enforce only signed container images
> - **VPC Service Controls** to prevent data exfiltration from GCS/BigQuery
> - **Cloud Interconnect or HA VPN** for on-prem connectivity
> - **Artifact Registry** for private container images
> - **Cloud Monitoring alerts** with PagerDuty/OpsGenie integration
> - **Terraform Cloud/Spacelift** for enterprise-grade state management and policy enforcement (Sentinel/OPA)
> - **Customer-Managed Encryption Keys (CMEK)** for GCS, GKE, and Pub/Sub

---

*Built with ❤️ for interview preparation — House of Hyderabad Biryani Infrastructure*
#

  
