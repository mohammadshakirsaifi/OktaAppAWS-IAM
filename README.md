# Create Okta App for AWS IAM Identity Center
Create Okta App for AWS IAM Identity Center
## Step 1. Log in to Okta Admin Console

Go to your Okta org: https://<your-org>.okta.com/admin

Make sure you’re logged in as a Super Admin or App Admin.

## Step 2. Add AWS IAM Identity Center app

In the left-hand menu, go to Applications → Applications.

Click Browse App Catalog.

Search for AWS IAM Identity Center (sometimes shown as AWS Single Sign-On).

Select it, then click Add Integration.

## Step 3. Configure SAML Settings

On the General Settings page:

Give it a descriptive label like AWS IAM Identity Center (SSO).

Optionally assign to default user groups now, or leave for later.

On the Sign-On Options page:

Leave the default SAML 2.0 configuration.

Do not change the default unless AWS requires a custom ACS URL (you’ll configure that later in AWS IAM Identity Center).

## Step 4. Assign Users or Groups

Go to the Assignments tab of the app.

Assign one or two test users or groups who will authenticate into AWS.

(Later, you’ll map Okta groups → AWS IAM roles.)

## Step 5. Download the Okta IdP Metadata

Go to the app’s Sign On tab.

Scroll down to SAML Signing Certificates.

Click Actions → View IdP metadata.

This will open an XML file in a new browser tab.

Save it locally as okta-idp-metadata.xml — you’ll upload this into AWS IAM Identity Center.

✅ At this point you now have:

An Okta SAML app for AWS IAM Identity Center.

The IdP metadata XML file downloaded and ready for AWS.

# Configure AWS IAM Identity Center with Okta as IdP
## Step 1. Log in to AWS IAM Identity Center

Sign in to the AWS Management Console with an admin account.

In the search bar, type IAM Identity Center and open it.
(If this is your first time, AWS may ask you to enable IAM Identity Center.)

## Step 2. Go to Identity Source Settings

In the left menu, click Settings.

Under Identity Source, click Change identity source.

Select External Identity Provider (SAML 2.0).

## Step 3. Upload Okta Metadata

In the External IdP settings section:

Upload the okta-idp-metadata.xml file you downloaded earlier from Okta.

This tells AWS to trust Okta as the SAML IdP.

Save changes.

## Step 4. Download AWS Metadata for Okta

After saving, AWS will generate its own Service Provider (SP) metadata file or show ACS URLs / Entity IDs.

Download this metadata and import it back into Okta (in the AWS IAM Identity Center app → Sign On tab → Edit → upload SP metadata).
This completes the trust on both sides.

## Step 5. Enable SCIM Provisioning (Recommended)

Still in Settings → Identity Source, enable Automatic provisioning (SCIM).

AWS provides a SCIM endpoint and Bearer Token.

In Okta → your AWS IAM Identity Center app → Provisioning tab:

Enter the SCIM base URL.

Paste the Bearer token.

Enable "Create Users", "Update Users", and "Deactivate Users".

✅ Now Okta can automatically create/update/remove AWS users when their lifecycle changes in Okta.

## Step 6. Create Permission Sets in AWS

In IAM Identity Center → Permission Sets, create roles like:

AWS-Admins (AdministratorAccess)

AWS-DevOps (PowerUserAccess)

AWS-ReadOnly (ReadOnlyAccess)

Assign these permission sets to AWS accounts.

## Step 7. Map Okta Groups → AWS Permission Sets

In Okta, create groups that match your permission sets (e.g., aws-admins, aws-devops).

Assign these groups to the AWS IAM Identity Center app in Okta.

Okta will provision the groups/users into AWS via SCIM.

## Step 8. Test SSO

Go to your AWS IAM Identity Center user portal URL (something like https://<your-org>.awsapps.com/start).

Log in with Okta credentials.

You should see the AWS accounts and roles assigned.

Test both console login and CLI login (aws sso login --profile myprofile).

✅ At this point, you have:

Okta → AWS IAM Identity Center (SAML SSO).

Okta → AWS IAM Identity Center (SCIM provisioning).

Okta Groups mapped to AWS permission sets.

# # Okta + AWS IAM Identity Center — Terraform starter repo

This repository scaffold contains Terraform code to create:
- Okta SAML app (AWS IAM Identity Center) and groups
- Group assignments in Okta
- AWS IAM Identity Center (SSO) Permission Set and Account Assignment example


--- filename: README.md ---
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

--- filename: notes.md ---
# Notes & Caveats
- The Okta SAML app resource may require manual adjustment for SAML certificates and activation. It's common to create the app in the Okta console, export its settings, then import into Terraform state and manage thereafter.
- AWS IAM Identity Center (SSO) must be **enabled** in the AWS console (one-time) before you can reliably manage SSO resources via Terraform.
- SCIM provisioning: AWS Identity Center exposes a SCIM endpoint/token in the console. Use that in the Okta app provisioning settings (manually or via Terraform if you create the app configuration fully).

# Next steps
- Test this in a dev Okta org.
- Replace placeholder SSO URLs with the real AWS SP metadata values (upload SP metadata to Okta app settings).
- Optionally add a module to import existing Okta app metadata and activate the SAML certificate if required.
