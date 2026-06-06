# Estable.io - DevOps Technical Assessment

**Christian La Piana** | DevOps Engineer

## Structure

- [Part 1 – Architecture & Design](docs/part1-architecture-design.md)
- [Part 2 – Incident Response](docs/part2-incident-response.md)
- [Part 3 – Security & Compliance](docs/part3-security-compliance.md)

## How to Run and Verify

The Terraform IaC for this architecture would follow these steps:

1. Authenticate: `az login && az account set --subscription "<ID>"`
2. Init: `cd iac/terraform && terraform init`
3. Plan: `terraform plan -var-file="environments/prod.tfvars" -out=tfplan`
4. Apply: `terraform apply tfplan`
5. Connect to AKS: `az aks get-credentials --resource-group rg-estable-prod --name aks-estable-prod`
6. Verify: `kubectl get nodes && kubectl get pods -n wallet && kubectl get pods -n payment-gateway`
