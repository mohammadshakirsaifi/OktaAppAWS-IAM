# Create Okta App for AWS IAM Identity Center

## Step 1. Log in to Okta Admin Console
<img width="596" height="76" alt="image" src="https://github.com/user-attachments/assets/e5ec8ad5-f25f-41a2-8f68-1cb3a95c9fd5" />

## Step 2. Add AWS IAM Identity Center app
<img width="785" height="155" alt="image" src="https://github.com/user-attachments/assets/ab914578-2929-43ea-b76b-9ad65e5d9e0d" />

## Step 3. Configure SAML Settings
<img width="960" height="251" alt="image" src="https://github.com/user-attachments/assets/a092635b-c44a-4d8c-bde2-874f093f5d54" />

## Step 4. Assign Users or Groups
<img width="678" height="125" alt="image" src="https://github.com/user-attachments/assets/4b71bde0-935c-490b-a5e7-aafafaa7ae09" />


## Step 5. Download the Okta IdP Metadata
<img width="881" height="181" alt="image" src="https://github.com/user-attachments/assets/5c20a624-f144-4e61-8104-4f50cdb52ec5" />


### Configure AWS IAM Identity Center with Okta as IdP
## Step 1. Log in to AWS IAM Identity Center
<img width="709" height="123" alt="image" src="https://github.com/user-attachments/assets/33199c7a-df53-478d-94e5-898e421e82d0" />


## Step 2. Go to Identity Source Settings
<img width="530" height="128" alt="image" src="https://github.com/user-attachments/assets/e661d65c-1bc2-45f2-bef4-5c28e4990dba" />


## Step 3. Upload Okta Metadata
<img width="761" height="153" alt="image" src="https://github.com/user-attachments/assets/6214d7bd-2766-42cc-a7b8-90ff2c966ed1" />

## Step 4. Download AWS Metadata for Okta
<img width="956" height="189" alt="image" src="https://github.com/user-attachments/assets/e8fbd64d-2a20-4c66-8f78-f1c05615ea77" />


## Step 5. Enable SCIM Provisioning (Recommended)
<img width="702" height="216" alt="image" src="https://github.com/user-attachments/assets/8a3fdded-d0f3-4d58-ad77-83624e768de8" />

 Now Okta can automatically create/update/remove AWS users when their lifecycle changes in Okta.

## Step 6. Create Permission Sets in AWS
<img width="584" height="192" alt="image" src="https://github.com/user-attachments/assets/d4858d1c-1089-4cbe-8dce-28fa256e462d" />


## Step 7. Map Okta Groups → AWS Permission Sets
<img width="834" height="125" alt="image" src="https://github.com/user-attachments/assets/abd89f2a-8a7c-4980-abf2-9327ef6c2077" />

## Step 8. Test SSO
<img width="810" height="205" alt="image" src="https://github.com/user-attachments/assets/8b833705-e08a-4a80-9bfc-2848770bfa28" />

## Okta + AWS IAM Identity Center — Terraform starter repo

This repository scaffold contains Terraform code to create:
- Okta SAML app (AWS IAM Identity Center) and groups
- Group assignments in Okta
- AWS IAM Identity Center (SSO) Permission Set and Account Assignment example

# okta-aws-terraform-starter

## What this repo contains
- `providers.tf` — Okta and AWS providers
- `variables.tf` — variables used by the examples
- `okta/` — Okta resources: app, groups, group assignment
- `aws/` — AWS SSO (Identity Center) permission set + account assignment example
- `outputs.tf` — helpful outputs

## How to use
1. Populate `terraform.tfvars` or export environment variables for Okta API token and AWS credentials.
2. Run `terraform init` then `terraform plan` and `terraform apply` (use a non-production Okta org first).
3. Import existing Okta app into Terraform if you prefer to adopt an existing app.


--- filename: providers.tf ---
```hcl
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 3.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "okta" {
  org_name = var.okta_org_name
  base_url = var.okta_base_url
  api_token = var.okta_api_token
}

provider "aws" {
  region = var.aws_region
  # Authentication can use environment, shared creds or SSO as needed
}
```

--- filename: variables.tf ---
```hcl
variable "okta_org_name" {
  type = string
}

variable "okta_base_url" {
  type = string
}

variable "okta_api_token" {
  type = string
  sensitive = true
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "aws_account_id" {
  type = string
}

variable "permission_set_name" {
  type    = string
  default = "AWS-Admins"
}
```

--- filename: okta/main.tf ---
```hcl
# Create an Okta group that will map to an AWS permission set
resource "okta_group" "aws_admins" {
  name = "aws-admins"
  description = "Group for AWS admins mapped to IAM Identity Center permission set"
}

# Create a SAML app in Okta for AWS Identity Center
# Note: creating SAML apps via Terraform may require additional fields and activation steps.
resource "okta_app_saml" "aws_identity_center" {
  label = "AWS IAM Identity Center"
  # If you have a preconfigured app shortname you could use preconfigured_app
  # preconfigured_app = "amazon_aws"

  # Example SAML settings (these will need editing to match AWS SP metadata)
  # The exact assertion and ACS URLs should be set after you enable Identity Center and obtain SP metadata from AWS.
  sso_url = "https://example.awsapps.com/start"
  audience = "urn:amazon:webservices"
  recipient = "https://example.awsapps.com/saml/assertion"

  # Optional: user_name_template and subject_name_id_template can be set as needed
}

# Assign Okta app to the group so users in group see the app
resource "okta_app_group_assignment" "assign_aws_app" {
  app_id   = okta_app_saml.aws_identity_center.id
  group_id = okta_group.aws_admins.id
}
```

--- filename: aws/main.tf ---
```hcl
# Example: create an AWS IAM Identity Center permission set and assign it to an account
# Note: You must have IAM Identity Center (AWS SSO) enabled in the target region before using these resources.

resource "aws_ssoadmin_permission_set" "admin_ps" {
  name        = var.permission_set_name
  description = "Administrator permission set"
  instance_arn = data.aws_ssoadmin_instances.default.arns[0]

  # Attach an inline policy or managed policy ARNs
  managed_policies = ["arn:aws:iam::aws:policy/AdministratorAccess"]
}

# Example account assignment for the permission set
resource "aws_ssoadmin_account_assignment" "assign_admin" {
  instance_arn     = data.aws_ssoadmin_instances.default.arns[0]
  permission_set_arn = aws_ssoadmin_permission_set.admin_ps.arn
  principal_type   = "GROUP" # could be USER or GROUP
  principal_id     = var.aws_group_principal_id # set to the identity store group's id (from SCIM or identity store)
  target_id        = var.aws_account_id
  target_type      = "AWS_ACCOUNT"
}

# Data source to find the SSO (Identity Center) instance
data "aws_ssoadmin_instances" "default" {}
```

--- filename: outputs.tf ---
```hcl
output "okta_group_id" {
  value = okta_group.aws_admins.id
}

output "okta_app_id" {
  value = okta_app_saml.aws_identity_center.id
}

output "aws_permission_set_arn" {
  value = aws_ssoadmin_permission_set.admin_ps.arn
}
```

# Notes & Caveats
- The Okta SAML app resource may require manual adjustment for SAML certificates and activation. It's common to create the app in the Okta console, export its settings, then import into Terraform state and manage thereafter.
- AWS IAM Identity Center (SSO) must be **enabled** in the AWS console (one-time) before you can reliably manage SSO resources via Terraform.
- SCIM provisioning: AWS Identity Center exposes a SCIM endpoint/token in the console. Use that in the Okta app provisioning settings (manually or via Terraform if you create the app configuration fully).

# Next steps
- Test this in a dev Okta org.
- Replace placeholder SSO URLs with the real AWS SP metadata values (upload SP metadata to Okta app settings).
- Optionally add a module to import existing Okta app metadata and activate the SAML certificate if required.
