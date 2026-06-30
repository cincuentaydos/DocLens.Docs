# Tenant Onboarding

This page describes how a new tenant (company) gains access to DocLens: how credentials are provisioned, how the `tenantId` is assigned, and how the authentication token is obtained for API calls.

---

## Identity Model

DocLens uses **Amazon Cognito** as the identity provider. Each API request carries a Cognito-issued JWT (ID token) in the `Authorization` header. The token contains a custom attribute — `custom:tenantId` — that binds the authenticated user to their tenant.

`TenantMiddleware` in the Lambda reads this claim on every request. All downstream operations (S3 keys, DynamoDB keys, Bedrock filters, log fields) are scoped to this `tenantId`. There is no way to act on behalf of a different tenant from within a single request.

---

## Tenant ID Convention

A `tenantId` is a **lowercase alphanumeric slug** assigned at onboarding time and never changed.

```
tenant-acme
tenant-globex
tenant-initech
```

The slug is used as a path component in S3 keys and as part of DynamoDB partition keys, so it must:

- Contain only lowercase letters, digits, and hyphens
- Start with a letter
- Be unique across all tenants
- Be stable — changing it would orphan all existing documents

---

## Cognito User Pool Configuration

DocLens uses a **single Cognito User Pool** shared across all tenants. Tenant isolation is enforced via the `custom:tenantId` attribute, not via separate pools.

| Setting | Value |
|---|---|
| User Pool | One pool per DocLens environment (dev / staging / production) |
| App Client | One app client per environment, used by the frontend |
| Auth flows | `USER_PASSWORD_AUTH` (dev); `USER_SRP_AUTH` (staging / production) |
| Custom attribute | `custom:tenantId` — read-only after set; not modifiable by the user |
| Token | ID token (carries custom attributes); access token does not |
| Token TTL | 1 hour (ID token); 30 days (refresh token — configurable) |

!!! warning "ID token, not access token"
    The Lambda validates the **ID token** (`Authorization: Bearer <id-token>`). The access token does not carry custom attributes and will fail the `tenantId` claim check.

---

## Onboarding Flow

Tenant onboarding is an **admin-managed process** — tenants do not self-register. A new tenant is provisioned by an administrator using the AWS CLI or Terraform.

### Step 1 — Assign a tenant ID

Choose a unique slug following the convention above. This is the canonical identifier — record it in the tenant registry.

### Step 2 — Create the Cognito user

```bash
aws cognito-idp admin-create-user \
  --user-pool-id <pool-id> \
  --username <email> \
  --user-attributes \
      Name=email,Value=<email> \
      Name=email_verified,Value=true \
      Name=custom:tenantId,Value=<tenant-slug> \
  --temporary-password <temp-password> \
  --message-action SUPPRESS
```

The `custom:tenantId` attribute is set here and cannot be changed by the user — Cognito enforces this via the attribute's `Mutable: false` pool configuration.

### Step 3 — Force password change

On first login, Cognito requires the user to set a permanent password. This happens via the frontend login flow or directly via the CLI:

```bash
aws cognito-idp admin-set-user-password \
  --user-pool-id <pool-id> \
  --username <email> \
  --password <permanent-password> \
  --permanent
```

### Step 4 — Deliver credentials

Provide the tenant with:

- Their login email
- Their initial (or permanent) password
- The frontend URL for their environment

---

## Authentication Flow (Client)

Once provisioned, a tenant authenticates via the Cognito-hosted UI or directly using the Cognito API:

```
POST https://cognito-idp.eu-west-1.amazonaws.com/
X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth

{
  "AuthFlow": "USER_PASSWORD_AUTH",
  "ClientId": "<app-client-id>",
  "AuthParameters": {
    "USERNAME": "<email>",
    "PASSWORD": "<password>"
  }
}
```

**Response:**

```json
{
  "AuthenticationResult": {
    "IdToken": "eyJ...",
    "AccessToken": "eyJ...",
    "RefreshToken": "eyJ...",
    "ExpiresIn": 3600
  }
}
```

The `IdToken` is the bearer token used in all DocLens API calls:

```
Authorization: Bearer <IdToken>
```

---

## Token Renewal

Tokens expire after 1 hour. The frontend refreshes them automatically using the `RefreshToken`:

```
POST https://cognito-idp.eu-west-1.amazonaws.com/
X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth

{
  "AuthFlow": "REFRESH_TOKEN_AUTH",
  "ClientId": "<app-client-id>",
  "AuthParameters": {
    "REFRESH_TOKEN": "<refresh-token>"
  }
}
```

---

## Multi-User Tenants

A single tenant may have multiple users (e.g., different employees of the same company). All users of the same tenant share the same `custom:tenantId` — they see the same documents and extraction results.

User management within a tenant (invite, deactivate, role assignment) is not in scope for the current phase. All users of a tenant have the same permissions.

---

## Tenant Offboarding

To revoke a tenant's access:

1. **Disable all Cognito users** for that tenant — prevents new token issuance. Existing tokens remain valid until expiry (max 1 hour).
2. **Optionally delete their data** from S3 and DynamoDB using the `tenantId` prefix/partition key.
3. **Optionally remove their Knowledge Base data** by triggering a deletion ingestion job scoped to their `tenantId` metadata filter.

!!! danger "Data deletion is irreversible"
    Deleting S3 objects and DynamoDB items cannot be undone. Always confirm the tenant ID before executing bulk deletes.

---

## Open Questions

- Should tenant onboarding be automated via a management API endpoint rather than admin CLI commands?
- Should multi-user tenants support role-based access control (e.g., admin vs. read-only users)?
- What is the data retention policy after offboarding — immediate deletion or a grace period?
