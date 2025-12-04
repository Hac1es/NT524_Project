# Setup Keycloak as SAML IdP for AWS IAM Identity Center

This document describes how to configure AWS IAM Identity Center (AWS SSO) to use Keycloak as an external SAML identity provider. It includes: prerequisites, step-by-step setup for both AWS and Keycloak, testing, and quick troubleshooting tips.

Prerequisites
- A running Keycloak instance with admin access.
- AWS account with permissions to configure IAM Identity Center (previously AWS SSO).
- A user in Keycloak to test SSO with (example in this guide: `tungzeka`).

Important notes
- Enabling IAM Identity Center for an AWS Organization may change billing (Organization features, pay-as-you-go). Consider using a secondary AWS account for non-production testing.

## Part 1: AWS IAM Identity Center — enable and export metadata

1. Sign in to the AWS Management Console and open **IAM Identity Center**.
2. If you haven't enabled Identity Center, enable it. When prompted about the Organization/instance, review billing implications and confirm.
3. In **Settings** → **Identity source**, select **Change identity source** → **External identity provider**.
4. Choose the option to upload IdP metadata. Download the provided AWS SAML metadata XML file (save as `aws-metadata.xml`).

Keep the AWS browser tab open — you'll return to it after configuring Keycloak.

## Part 2: Keycloak — import AWS metadata and create a SAML client

1. In Keycloak, select the target **Realm**.
2. Go to **Clients** → **Import client** and upload the file you downloaded from AWS (`aws-metadata.xml`).
  - Optionally set a friendly client name (e.g. `aws-iam-idp`).
3. After import, open **Realm Settings** → **SAML 2.0 Identity Provider Metadata** and download Keycloak's SAML metadata (save as `descriptor.xml`).

## Part 3: AWS — finish IdP configuration

1. Return to the AWS IAM Identity Center browser tab where you began the change identity source flow.
2. Upload Keycloak's SAML metadata file (`descriptor.xml`) as the IdP metadata.
3. Keep default settings unless you have specific requirements, then finish the configuration.

## Part 4: Keycloak — create SAML client scope and mappers (NameID)

1. In Keycloak, go to **Client Scopes** → **Create client scope**.
  - Name: `aws`
  - Type: `Default`
  - Protocol: `saml`
2. Save and open the new scope, then go to the **Mappers** tab → **Create mapper**.
  - Mapper Type: `User Attribute` (or `User Property` depending on Keycloak version)
  - Name: `aws-nameid`
  - SAML Attribute Name: set to NameID format appropriate for AWS (e.g. `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`)
  - User Attribute: set to `email` or `username` depending on how you mapped identities in AWS (email is common).
3. Go to **Clients**, select the client created from `aws-metadata.xml`, then **Client Scopes** → **Add client scope** and add the `aws` scope as a Default scope.

Notes on NameID and attributes
- AWS frequently expects NameID to be an email address (or another unique identifier). Choose the `User Attribute` mapper to populate NameID with the Keycloak `email` (or `username`) claim.

## Part 5: AWS — create or sync test user and assign access

1. In IAM Identity Center → **Users**, create a test user that matches the attribute Keycloak will send (for example, email `nchinhhtung@gmail.com` or username `tungzeka`).
2. In IAM Identity Center → **AWS Accounts**, select the target AWS account and **Assign users**.
  - Assign the test user and choose or create an IAM role that the user will assume (choose a suitable permission set).

## Part 6: Test the SSO flow

1. From the IAM Identity Center dashboard, click the **AWS access portal URL**.
2. You should be redirected to Keycloak's login page. Log in with the Keycloak user (example: `tungzeka`).
3. After successful authentication, Keycloak will post a SAML response to AWS; you should be redirected back to the AWS access portal and see assigned accounts/roles.