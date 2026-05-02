---
name: oracle-cloud
description: Manage Oracle Cloud Infrastructure (OCI) resources — compute, networking, storage — via the OCI CLI
---

# Oracle Cloud Infrastructure

Manage OCI resources via the CLI. All credentials and config are already set up locally.

## Key File Locations

All sensitive files live in `~/.oci/` — never committed to git.

| File | What it is |
|---|---|
| `~/.oci/config` | CLI config: tenancy OCID, user OCID, region, fingerprint, key path |
| `~/.oci/oci_api_key.pem` | API private key — RSA, chmod 600, never share |
| `~/.oci/oci_api_key_public.pem` | API public key — registered in Oracle Console under your user → API Keys |
| `~/.oci/actual_budget_ssh` | SSH private key for the actual-budget VM, chmod 600 |
| `~/.oci/actual_budget_ssh.pub` | SSH public key — baked into the VM at launch |
| `~/.oci/actual_budget_password` | Actual Budget server password, chmod 600 |

To inspect config values without printing them:
```bash
grep region ~/.oci/config          # safe to read aloud
grep fingerprint ~/.oci/config     # identifies which key pair is active
# tenancy/user OCIDs are in there too — treat as internal, not secret
```

## Read Live Resource IDs

Rather than hardcoding IDs, look them up by name:

```bash
# Compartment ID (= tenancy for a free account's root compartment)
COMPARTMENT=$(awk -F'=' '/^tenancy/{gsub(/ /,"",$2); print $2}' ~/.oci/config)

# VCN ID
VCN_ID=$(oci network vcn list --compartment-id $COMPARTMENT --output json 2>/dev/null \
  | python3 -c "import json,sys; print([v['id'] for v in json.load(sys.stdin)['data'] if v['display-name']=='actual-budget-vcn'][0])")

# Subnet ID
SUBNET_ID=$(oci network subnet list --compartment-id $COMPARTMENT --vcn-id $VCN_ID --output json 2>/dev/null \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['id'])")

# Security list ID
SL_ID=$(oci network security-list list --compartment-id $COMPARTMENT --vcn-id $VCN_ID --output json 2>/dev/null \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['id'])")

# VM public IP
VM_IP=$(oci compute instance list --compartment-id $COMPARTMENT --display-name actual-budget --output json 2>/dev/null \
  | python3 -c "
import json,sys
iid = json.load(sys.stdin)['data'][0]['id']
print(iid)
" | xargs -I{} oci compute instance list-vnics --instance-id {} --output json 2>/dev/null \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['public-ip'])")

echo "VM IP: $VM_IP"
```

## CLI Basics

```bash
# Verify auth
COMPARTMENT=$(awk -F'=' '/^tenancy/{gsub(/ /,"",$2); print $2}' ~/.oci/config)
USER_OCID=$(awk -F'=' '/^user/{gsub(/ /,"",$2); print $2}' ~/.oci/config)
oci iam user get --user-id $USER_OCID 2>/dev/null

# Suppress harmless SyntaxWarnings from OCI CLI
oci ... 2>/dev/null
```

Region: `uk-london-1`

Availability domains:
- `Zcsc:UK-LONDON-1-AD-1`
- `Zcsc:UK-LONDON-1-AD-2`
- `Zcsc:UK-LONDON-1-AD-3`

## Existing Infrastructure

Named resources — look up IDs dynamically as shown above.

| Resource | Name | Notes |
|---|---|---|
| VCN | `actual-budget-vcn` | CIDR 10.0.0.0/16, internet gateway attached |
| Subnet | `actual-budget-subnet` | CIDR 10.0.0.0/24 |
| Security List | (default for VCN) | Ingress: 22, 80, 443 |
| VM | `actual-budget` | E2.1.Micro, Ubuntu 22.04 x86, AD-2 |

SSH into the VM (IP resolved at runtime):
```bash
VM_IP=$(oci compute instance list --compartment-id $(awk -F'=' '/^tenancy/{gsub(/ /,"",$2); print $2}' ~/.oci/config) \
  --display-name actual-budget --output json 2>/dev/null \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['id'])" \
  | xargs -I{} oci compute instance list-vnics --instance-id {} --output json 2>/dev/null \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['public-ip'])")
ssh -i ~/.oci/actual_budget_ssh ubuntu@$VM_IP
```

## Actual Budget App

Running as a Docker container on the VM at `https://<VM_IP>` (HTTPS via nginx + self-signed cert — click through browser warning on first visit).

```bash
# Check status
ssh -i ~/.oci/actual_budget_ssh ubuntu@$VM_IP "sudo docker ps"

# View logs
ssh -i ~/.oci/actual_budget_ssh ubuntu@$VM_IP "sudo docker logs actual-budget"

# Restart
ssh -i ~/.oci/actual_budget_ssh ubuntu@$VM_IP "sudo docker restart actual-budget"

# Update to latest version
ssh -i ~/.oci/actual_budget_ssh ubuntu@$VM_IP "
  sudo docker pull actualbudget/actual-server:latest &&
  sudo docker stop actual-budget && sudo docker rm actual-budget &&
  sudo docker run -d --name actual-budget --restart unless-stopped \
    -p 5006:5006 -v ~/actual-data:/data actualbudget/actual-server:latest"
```

- Data persisted in `~/actual-data` on the VM
- nginx config: `/etc/nginx/sites-available/actual-budget`
- TLS cert: `/etc/nginx/ssl/actual.{crt,key}` (self-signed, valid 10 years)
- `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers are sent by the actual-budget app itself — nginx does not add them (removing duplicates broke SharedArrayBuffer)

## Always Free Tier Limits

| Resource | Limit |
|---|---|
| VM.Standard.E2.1.Micro (x86) | 2 instances — always available |
| VM.Standard.A1.Flex (ARM) | 4 OCPUs + 24 GB RAM total — frequently "Out of host capacity", retry all 3 ADs |
| Block Volume | 2 volumes, 200 GB total |
| Object Storage | 10 GB |
| Autonomous Database | 2 × 20 GB |

## Common CLI Snippets

```bash
# List all compute instances
oci compute instance list \
  --compartment-id $(awk -F'=' '/^tenancy/{gsub(/ /,"",$2); print $2}' ~/.oci/config) \
  --output json 2>/dev/null \
  | python3 -c "import json,sys; [print(i['display-name'], i['lifecycle-state'], i['shape']) for i in json.load(sys.stdin)['data']]"

# Open a new port in the security list (replace SL_ID and port)
oci network security-list update \
  --security-list-id $SL_ID \
  --ingress-security-rules '[{"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":8080,"max":8080}},"isStateless":false}]' \
  --force 2>/dev/null

# Launch an A1 ARM instance (try all ADs — capacity often exhausted)
for AD in "Zcsc:UK-LONDON-1-AD-1" "Zcsc:UK-LONDON-1-AD-2" "Zcsc:UK-LONDON-1-AD-3"; do
  oci compute instance launch \
    --compartment-id $(awk -F'=' '/^tenancy/{gsub(/ /,"",$2); print $2}' ~/.oci/config) \
    --availability-domain "$AD" \
    --shape "VM.Standard.A1.Flex" \
    --shape-config '{"ocpus":2,"memoryInGBs":4}' \
    --image-id <arm-ubuntu-image-id> \
    --subnet-id $SUBNET_ID \
    --assign-public-ip true \
    --ssh-authorized-keys-file ~/.oci/actual_budget_ssh.pub 2>/dev/null && break
done
```
