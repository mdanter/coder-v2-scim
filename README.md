# Okta SCIM Setup with Coder - Complete Guide

## What is SCIM?

SCIM (System for Cross-domain Identity Management) lets Okta automatically create, update, and remove users in Coder - so you don't have to manage them manually.

## Prerequisites

- **Enterprise Coder license** (SCIM is enterprise-only)
- **OIDC/SSO already working** between Okta and Coder
- Admin access to both Okta and Coder

## Setup Steps

### 1. Enable SCIM in Coder

1. Set the SCIM API key environment variable:
   ```bash
   CODER_SCIM_API_KEY="your-secret-token-here"
   ```

2. Restart your Coder server

3. Verify SCIM endpoint is accessible:
   ```bash
   curl -H "Authorization: Bearer your-secret-token-here" \
        https://your-coder-domain.com/scim/v2/ServiceProviderConfig
   ```

### 2. Configure Okta SCIM Provisioning

1. **Go to your existing Coder app in Okta Admin Console**
   - Applications → Your Coder App → Provisioning tab

2. **Enable SCIM provisioning**
   - Click "Configure API Integration"
   - Check "Enable API integration"

3. **Configure SCIM settings:**
   - **SCIM Base URL**: `https://your-coder-domain.com/scim/v2`
   - **Authorization**: Bearer Token
   - **Bearer Token**: Your `CODER_SCIM_API_KEY` value

4. **Test the connection** and save

5. **Enable provisioning features:**
   - ✅ Create Users
   - ✅ Update User Attributes  
   - ✅ Deactivate Users
   - ✅ Push Groups (optional)

### 3. Test the Setup

1. **Assign a test user** to the Coder app in Okta
2. **Check Coder UI** - user should appear automatically
3. **Unassign the user** from Okta
4. **Verify in Coder** - user should be suspended

## What SCIM Does

- **Creates Coder users** when assigned in Okta
- **Updates user information** when changed in Okta  
- **Suspends Coder users** when unassigned in Okta
- **Can sync group memberships** (if configured)

## Important Notes

- **No DELETE requests**: Okta only sends PATCH requests with `active: false`
- **Immediate processing**: Coder processes SCIM requests synchronously
- **User status mapping**:
  - `active: true` → User becomes `dormant` (until first login)
  - `active: false` → User becomes `suspended`

## Debugging SCIM Issues

### Enable SCIM Logging in Coder

```bash
# Enable SCIM-specific logging
CODER_LOG_FILTER=".*scim.*"

# Or enable all debug logs
CODER_LOG_FILTER=".*"
```

### Check Coder Logs

Look for SCIM operations tagged with:
```json
{
  "automatic_actor": "coder",
  "automatic_subsystem": "scim"
}
```

### Verify SCIM Endpoint

```bash
# Test endpoint accessibility
curl -H "Authorization: Bearer YOUR_SCIM_API_KEY" \
     https://your-coder-domain.com/scim/v2/ServiceProviderConfig
```

### Check Okta System Logs

1. Go to **Reports > System Log** in Okta Admin Console
2. Filter by the specific user ID or Coder app name
3. Look for SCIM-related events and error codes

### Manual SCIM Testing

```bash
# Test user deactivation manually
curl -X PATCH \
  -H "Authorization: Bearer YOUR_KEY" \
  -H "Content-Type: application/scim+json" \
  -d '{"active": false}' \
  https://your-coder.com/scim/v2/Users/USER_UUID
```

## Common Issues & Solutions

### Issue: Users not provisioning
**Check:**
- "Create Users" enabled in Okta provisioning settings?
- SCIM Base URL correct? (no trailing slash)
- Bearer token matches `CODER_SCIM_API_KEY`?
- User actually assigned to Coder app?

### Issue: Users not being removed
**Check:**
- "Deactivate Users" enabled in Okta?
- Check Okta System Logs for failed SCIM requests
- Verify user was properly created initially

### Issue: SCIM requests timing out
**Check:**
- Network connectivity between Okta and Coder
- Coder server performance/load
- Database connectivity issues

### Issue: Authentication failures
**Check:**
- Bearer token exactly matches (no extra spaces)
- SCIM endpoint URL is correct
- Coder server is running and accessible

## SCIM API Endpoints

Coder implements these SCIM 2.0 endpoints:

- `GET /scim/v2/ServiceProviderConfig` - Service provider capabilities
- `GET /scim/v2/Users` - List users (always returns empty)
- `GET /scim/v2/Users/{id}` - Get user (always returns 404)
- `POST /scim/v2/Users` - Create user
- `PATCH /scim/v2/Users/{id}` - Update user (mainly for activation status)
- `PUT /scim/v2/Users/{id}` - Replace user (mainly for activation status)

## Security Considerations

- **Keep SCIM API key secure** - treat like a database password
- **Use HTTPS only** - never configure HTTP endpoints
- **Monitor SCIM audit logs** - track all provisioning activities
- **Regular key rotation** - consider rotating SCIM API keys periodically

## Resources

- [Coder OIDC Authentication Docs](https://coder.com/docs/admin/users/oidc-auth)
- [Coder Monitoring & Logs](https://coder.com/docs/admin/monitoring/logs)
- [Okta SCIM Documentation](https://developer.okta.com/docs/reference/scim/)

---
