# FDA client networking (fdagcp)

Live hub and Shared Tools subnets in `{prefix}-shared-shared-net` (`fdagcp-shared-shared-net`), region `us-east4`. Use these names and CIDRs in factory YAML (`host_subnets`, `network_subnet_users`, NVA NIC attach). Do not invent overlapping ranges in `10.80.0.0/22` hub space or `10.87.0.0/16` Shared Tools spoke space.

| Subnet | Region | VPC network | IP address range |
| --- | --- | --- | --- |
| `external-default` | us-east4 | `fdagcp-h-shared-vpcext-use4` | `10.80.2.128/26` |
| `extranet-default` | us-east4 | `fdagcp-h-shared-vpcint-extranet-use4` | `10.80.1.0/26` |
| `intranet-default` | us-east4 | `fdagcp-h-shared-vpcint-intranet-use4` | `10.80.0.128/26` |
| `non-prod-default` | us-east4 | `fdagcp-h-shared-vpcint-non-prod-use4` | `10.80.0.0/26` |
| `palo-managment-default` | us-east4 | `fdagcp-h-shared-vpc-palo-managment-use4` | `10.80.2.0/26` |
| `shared-tools-default` | us-east4 | `fdagcp-h-shared-tools-vpc-use4` | `10.87.0.0/28` |
| `shared-tools-hub-default` | us-east4 | `fdagcp-h-shared-vpcint-shared-tools-use4` | `10.80.1.128/26` |
| `shared-tools-sys-image-sub` | us-east4 | `fdagcp-h-shared-tools-vpc-use4` | `10.87.0.16/28` |

`palo-managment-default` is the live subnet name (spelling as in GCP).

Shared Tools spoke VPC `fdagcp-h-shared-tools-vpc-use4` holds both `shared-tools-default` (`10.87.0.0/28`) and `shared-tools-sys-image-sub` (`10.87.0.16/28`). Further spoke subnets must come from `10.87.0.0/16` and must not overlap those two.

## Shared Tools projects (five more)

Client needs **five additional factory projects** under the Shared Tools folder, each attached to the Shared Tools spoke (`host_project: shared-tools-spoke` → VPC `fdagcp-h-shared-tools-vpc-use4` in `fdagcp-shared-shared-net`). One project per admin group. Do not put these under a Center / env folder.

Group domain is `@fda.hhs.gov` (same as `RB-FDAIT-GCP-firewall-admins`). Confirm the live Cloud Identity emails before apply.

GCP project IDs and subnet names must be lowercase. Client account names are shown as given; YAML `name` is the suffix after prefix `fdagcp-`.

`fdagcp-st-fdait-active-directory` is 33 characters (GCP project ID max is 30), so the project ID is `fdagcp-st-fdait-ad`. The subnet can keep the full client name.

Live spoke already uses `10.87.0.0/28` and `10.87.0.16/28` (`10.87.0.0/27`). Citrix is a `/27`, so it starts at the next `/27`-aligned block `10.87.0.32/27`. The four `/28`s follow it. Networking must sign off before apply.

| Account name | Project ID | YAML name | Admin group | Spoke subnet | CIDR |
| --- | --- | --- | --- | --- | --- |
| `FDAGCP-ST-FDAIT-CITRIX` | `fdagcp-st-fdait-citrix` | `st-fdait-citrix.yaml` | `RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov` | `fdagcp-st-fdait-citrix` | `10.87.0.32/27` |
| `FDAGCP-ST-FDAIT-SOLARWINDS` | `fdagcp-st-fdait-solarwinds` | `st-fdait-solarwinds.yaml` | `RB-FDAIT-GCP-SOLARWINDS-ADMINS@fda.hhs.gov` | `fdagcp-st-fdait-solarwinds` | `10.87.0.64/28` |
| `FDAGCP-ST-FDAIT-BIGFIX` | `fdagcp-st-fdait-bigfix` | `st-fdait-bigfix.yaml` | `RB-FDAIT-GCP-BIGFIX-ADMINS@fda.hhs.gov` | `fdagcp-st-fdait-bigfix` | `10.87.0.80/28` |
| `FDAGCP-ST-FDAIT-NESSUS` | `fdagcp-st-fdait-nessus` | `st-fdait-nessus.yaml` | `RB-FDAIT-GCP-NESSUS-ADMINS@fda.hhs.gov` | `fdagcp-st-fdait-nessus` | `10.87.0.96/28` |
| `FDAGCP-ST-FDAIT-ACTIVE-DIRECTORY` | `fdagcp-st-fdait-ad` | `st-fdait-ad.yaml` | `RB-FDAIT-GCP-ACTIVE-DIRECTORY-ADMINS@fda.hhs.gov` | `fdagcp-st-fdait-active-directory` | `10.87.0.112/28` |

### IAM on each project (that group only)

Grant these on **that group's project**, not org-wide and not on the other four. YAML `iam:` is project-scoped. Do not grant `roles/owner` or hub `roles/compute.networkAdmin`.

| Requirement | Role on the project | Why |
| --- | --- | --- |
| Create / update / delete VMs | `roles/compute.instanceAdmin.v1` | Disks, instance templates, MIGs, start/stop/update |
| Attach a service account to a VM | `roles/iam.serviceAccountUser` | Required with instanceAdmin or create VM fails on the SA |
| Provision load balancers | `roles/compute.loadBalancerAdmin` | Forwarding rules, backend services, health checks, URL maps, proxies, SSL policies |
| TLS certs on those LBs | `roles/certificatemanager.editor` | Certificate Manager certs used by HTTPS LBs |
| NIC on the Shared Tools spoke | `roles/compute.networkUser` | Use the attached Shared VPC in **this** project |
| View logs | `roles/logging.viewer` | Logs Explorer, log-based metrics read |
| View metrics / dashboards | `roles/monitoring.viewer` | Metrics Explorer, default dashboards, alerting read |
| Open the project in console | `roles/browser` | `resourcemanager.projects.get` if instanceAdmin is not enough in the picker |

Also grant `roles/compute.networkUser` on **that project's spoke subnet** on the hub host (`network_subnet_users` / `host_subnets.network_users`). Without the subnet grant, VM and internal LB create fail even with the project roles above.

Do **not** put `roles/compute.networkAdmin`, `roles/compute.securityAdmin`, or NCC hub/spoke admin on these five projects. Those are hub/NVA only.

Factory YAML `iam:` shape (replace the group per project):

```yaml
iam:
  roles/browser:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/compute.instanceAdmin.v1:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/iam.serviceAccountUser:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/compute.loadBalancerAdmin:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/certificatemanager.editor:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/compute.networkUser:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/logging.viewer:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
  roles/monitoring.viewer:
    - group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov
```

If an owner must **create** log sinks, metrics, or alerting policies (not only view), add `roles/logging.configWriter` and `roles/monitoring.editor` on that project only.

Internal LBs that reserve an IP in the spoke subnet still need the subnet `networkUser` on the hub. External HTTP(S) LBs stay in the service project with `loadBalancerAdmin`.

Each YAML must set:

- `parent: shared-tools`
- `host_project: shared-tools-spoke`
- `iam` as the table above for **that one group**
- `network_subnet_users` on `us-east4/<spoke-subnet>` for that group
- `host_subnets` to create the spoke subnet (`vpc: shared-tools-spoke`, `region: us-east4`, `private_ip_google_access: true`)

### VPC-SC membership and ingress (3-security)

IAM on the project is not enough. `compute`, `iam`, `logging`, `monitoring`, `storage`, and `cloudresourcemanager` are restricted. Console or `gcloud` from the internet is denied until ingress matches the **group** (VPC-SC evaluates identity, not the YAML `iam:` block).

**Membership.** After factory apply, add all five project IDs to `vpc_sc.perimeters.platform.project_ids` (Shared Tools = platform, not `dev` / `prod-int`). Re-apply 3-security. The hub `fdagcp-shared-shared-net` must already be on platform (NIC and internal LB APIs hit the host).

**Ingress.** One policy for all five groups. `vpc_sc.ingress_policies` keys attach to **platform only**. To keep the same users from being denied when an API is evaluated on **any** perimeter (hub, spoke, later env attach), also list the policy name on every perimeter.

`access_levels = ["*"]` is required (laptop / Cloud Console). Empty `access_levels` is what blocked factory Terraform and Splunk.

Look up numbers after the projects exist:

```bash
for p in fdagcp-st-fdait-citrix fdagcp-st-fdait-solarwinds fdagcp-st-fdait-bigfix \
         fdagcp-st-fdait-nessus fdagcp-st-fdait-ad fdagcp-shared-shared-net; do
  echo "$p $(gcloud projects describe "$p" --format='value(projectNumber)')"
done
```

Live 3-security `terraform.tfvars`:

```hcl
  ingress_policies = {
    st-fdait-tool-admins = {
      from = {
        access_levels = ["*"]
        identities = [
          "group:RB-FDAIT-GCP-CITRIX-ADMINS@fda.hhs.gov",
          "group:RB-FDAIT-GCP-SOLARWINDS-ADMINS@fda.hhs.gov",
          "group:RB-FDAIT-GCP-BIGFIX-ADMINS@fda.hhs.gov",
          "group:RB-FDAIT-GCP-NESSUS-ADMINS@fda.hhs.gov",
          "group:RB-FDAIT-GCP-ACTIVE-DIRECTORY-ADMINS@fda.hhs.gov",
        ]
      }
      to = {
        operations = [
          { service_name = "compute.googleapis.com",              method_selectors = ["*"] },
          { service_name = "iam.googleapis.com",                  method_selectors = ["*"] },
          { service_name = "logging.googleapis.com",              method_selectors = ["*"] },
          { service_name = "monitoring.googleapis.com",           method_selectors = ["*"] },
          { service_name = "cloudresourcemanager.googleapis.com", method_selectors = ["*"] },
          { service_name = "storage.googleapis.com",              method_selectors = ["*"] },
          { service_name = "iaptunnel.googleapis.com",            method_selectors = ["*"] },
          { service_name = "oslogin.googleapis.com",              method_selectors = ["*"] },
        ]
        resources = [
          "projects/<CITRIX_PROJECT_NUMBER>",
          "projects/<SOLARWINDS_PROJECT_NUMBER>",
          "projects/<BIGFIX_PROJECT_NUMBER>",
          "projects/<NESSUS_PROJECT_NUMBER>",
          "projects/<AD_PROJECT_NUMBER>",
          "projects/<HUB_PROJECT_NUMBER>",
        ]
      }
    }
  }

  perimeters = {
    platform   = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* ... + five ST projects ... */] }
    dev        = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
    test       = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
    "pprod-int" = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
    "pprod-ext" = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
    "prod-int"  = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
    "prod-ext"  = { ingress_policies = ["st-fdait-tool-admins"], project_ids = [/* existing */] }
  }
```

`perimeters.*.ingress_policies` is **additive**. `fast-automation` and `fast-org-log-sinks` stay attached. Extra keys in `vpc_sc.ingress_policies` already attach to platform; listing `st-fdait-tool-admins` on platform twice is harmless (`distinct`).

`to.resources` is the five tool projects **plus the hub**. Omit the hub and VM / internal LB calls on `fdagcp-h-shared-tools-vpc-use4` still VPC-SC deny. `certificatemanager.googleapis.com` is not a restricted service in this stage; it does not need an ingress line.

Do **not** set `resources = ["*"]`. That would let these groups call Compute/IAM/Logging on every project in every perimeter (prod hosts included). Nessus scans of env VMs are a separate Security decision: add those env project numbers to `resources`, do not open `*`.

Apply 3-security after factory creates the projects (need the project numbers). Groups must exist in Cloud Identity or the perimeter apply fails on identity expansion.
