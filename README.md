# Complete Guide: Okta OIDC + SCIM Setup with Coder

## Table of Contents

- [What is SCIM?](#what-is-scim)
- [Prerequisites](#prerequisites)
- [Understanding the Okta Limitation](#understanding-the-okta-limitation)
- [Setup Overview](#setup-overview)
- [Step-by-Step Configuration](#step-by-step-configuration)
  - [Part 1: Coder SCIM Configuration](#part-1-coder-scim-configuration)
  - [Part 2: Okta OIDC App Setup](#part-2-okta-oidc-app-setup)
  - [Part 3: Okta SCIM App Setup](#part-3-okta-scim-app-setup)
  - [Part 4: User Assignment](#part-4-user-assignment)
- [Testing Your Setup](#testing-your-setup)
- [Debugging SCIM Issues](#debugging-scim-issues)
- [Common Issues & Solutions](#common-issues--solutions)
- [How User Removal Works](#how-user-removal-works)
- [Resources](#resources)

---

## What is SCIM?

SCIM (System for Cross-domain Identity Management) lets Okta automatically create, update, and remove users in Coder - so you don't have to manage them manually.

**What SCIM does:**
- **Creates Coder users** when assigned in Okta
- **Updates user information** when changed in Okta  
- **Suspends Coder users** when unassigned in Okta
- **Can sync group memberships** (if configured)

---

## Prerequisites

- **Enterprise Coder license** (SCIM is enterprise-only)
- **Admin access** to both Okta and Coder
- Coder deployment accessible from Okta (public URL or proper network access)

**Note:** You do NOT need OIDC working first, but it's recommended to set up authentication before provisioning.

---

## Understanding the Okta Limitation

**CRITICAL: Okta does NOT support adding SCIM provisioning to OIDC apps through the UI.**

This is a known limitation. The Okta documentation states:
> "Adding SCIM provisioning to an OpenID Connect (OIDC) integration isn't currently supported."

**The Solution:**
You must create **TWO separate apps in Okta**:
1. **OIDC app** - Handles user authentication (login)
2. **SAML app with SCIM** - Handles user provisioning (lifecycle management)

Users will be assigned to both apps, but will only interact with the OIDC app for login.

---

## Setup Overview

Here's the architecture:

```
┌─────────────────────────────────────────────────┐
│                    Okta                         │
│                                                 │
│  ┌──────────────────┐  ┌──────────────────┐   │
│  │   OIDC App       │  │   SAML App       │   │
│  │  (Authentication)│  │   (SCIM Only)    │   │
│  │                  │  │                  │   │
│  │  Users: Group A  │  │  Users: Group A  │   │
│  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │              │
└───────────┼─────────────────────┼──────────────┘
            │                     │
            │ Auth requests       │ SCIM requests
            │                     │
            ▼                     ▼
┌───────────────────────────────────────────────┐
│              Coder Server                     │
│                                               │
│  /api/v2/users/oidc/callback  /scim/v2       │
└───────────────────────────────────────────────┘
```

---

## Step-by-Step Configuration

### Part 1: Coder SCIM Configuration

1. **Set the SCIM API key** in your Coder deployment:

   ```bash
   CODER_SCIM_API_KEY="your-secret-token-here"
   ```

   Generate a secure random token (e.g., `openssl rand -hex 32`)

2. **Restart your Coder server**

3. **Verify SCIM endpoint is accessible:**

   ```bash
   curl -H "Authorization: Bearer your-secret-token-here" \
        https://your-coder-domain.com/scim/v2/ServiceProviderConfig
   ```

   Should return JSON with SCIM capabilities (not 401/404)

---

### Part 2: Okta OIDC App Setup

#### Option A: If You Already Have an OIDC App

**Skip to Part 3** - keep your existing OIDC app as-is.

#### Option B: Create a New OIDC App

1. **Okta Admin Console** → Applications → Applications
2. Click **Create App Integration**
3. Select **OIDC - OpenID Connect** → **Web Application** → Next

4. **Configure General Settings:**
   - **App integration name**: `Coder` (or your preference)
   - **Logo**: (optional)
   - **Grant type**: 
     - ✅ Authorization Code
     - ✅ Refresh Token (recommended)
   - **Sign-in redirect URIs**: `https://your-coder-domain.com/api/v2/users/oidc/callback`
   - **Sign-out redirect URIs**: `https://your-coder-domain.com/` (optional)
   - **Controlled access**: Choose who can access
   - Click **Save**

5. **Copy Client Credentials:**
   - Client ID
   - Client Secret

6. **Configure Group Claims (Optional but Recommended):**
   - Go to **Sign On** tab
   - Under **OpenID Connect ID Token**, click **Edit**
   - **Groups claim type**: Filter
   - **Groups claim filter**: `groups` Matches regex `.*`
   - Click **Save**

7. **Configure Coder with OIDC settings:**

   ```bash
   CODER_OIDC_ISSUER_URL="https://your-okta-domain.okta.com"
   CODER_OIDC_CLIENT_ID="your-client-id"
   CODER_OIDC_CLIENT_SECRET="your-client-secret"
   CODER_OIDC_EMAIL_DOMAIN="your-company.com"
   CODER_OIDC_SCOPES="openid,profile,email,groups"
   
   # Optional: Group sync
   CODER_OIDC_GROUP_FIELD="groups"
   ```

8. **Restart Coder** and test OIDC login

---

### Part 3: Okta SCIM App Setup

Now create a separate app for SCIM provisioning:

#### Step 1: Create a SAML App (for SCIM)

1. **Okta Admin Console** → Applications → Applications
2. Click **Create App Integration**
3. Select **SAML 2.0** → **Next**

4. **General Settings:**
   - **App name**: `Coder SCIM Provisioning`
   - **App logo**: (optional, can use same as OIDC app)
   - Click **Next**

5. **SAML Settings** *(these are dummy values - we won't use SAML)*:
   - **Single sign-on URL**: `https://your-coder-domain.com/fake`
   - **Audience URI (SP Entity ID)**: `https://your-coder-domain.com`
   - Click **Next**

6. **Feedback:**
   - Select: "I'm an Okta customer adding an internal app"
   - "This is an internal app that we have created"
   - Click **Finish**

#### Step 2: Enable SCIM Provisioning

1. Go to the **Coder SCIM Provisioning** app you just created
2. Click **General** tab
3. In **App Settings** section → Click **Edit**
4. **Provisioning**: Change dropdown from "None" to **SCIM**
5. Click **Save**

   *The Provisioning tab should now appear*

#### Step 3: Configure SCIM Connection

1. Click **Provisioning** tab
2. Under **Settings** → **Integration** → Click **Edit**

3. **Configure SCIM Settings:**
   - **SCIM connector base URL**: `https://your-coder-domain.com/scim/v2`
   - **Unique identifier field for users**: `userName`
   - **Supported provisioning actions**: 
     - ✅ Push New Users
     - ✅ Push Profile Updates
   - **Authentication Mode**: HTTP Header
   - **Authorization**: (paste the exact value below)
     ```
     Bearer your-coder-scim-api-key-here
     ```
     ⚠️ **Important**: Include the word "Bearer" with a space, then your key

4. Click **Test Connector Configuration**
   - Should show "Coder SCIM Provisioning was verified successfully"
   - If it fails, check the [Debugging section](#debugging-scim-issues)

5. Click **Save**

#### Step 4: Enable Provisioning Features

1. Still in **Provisioning** tab
2. Click **Settings** → **To App** → Click **Edit**

3. **Enable these features:**
   - ✅ **Create Users**: Creates users in Coder when assigned
   - ✅ **Update User Attributes**: Syncs profile changes
   - ✅ **Deactivate Users**: Suspends users when unassigned

4. Click **Save**

---

### Part 4: User Assignment

**Critical Step:** Assign the same users/groups to BOTH apps.

#### Recommended Approach: Use Okta Groups

1. **Create an Okta Group:**
   - Directory → Groups → Add Group
   - Name: `Coder Users`
   - Add members

2. **Assign Group to OIDC App:**
   - Go to your **Coder** OIDC app
   - Assignments tab → Assign → Assign to Groups
   - Select `Coder Users` → Assign → Done

3. **Assign Group to SCIM App:**
   - Go to your **Coder SCIM Provisioning** app
   - Assignments tab → Assign → Assign to Groups
   - Select `Coder Users` → Assign → Done

#### Manual User Assignment

If not using groups:
1. Assign individual users to **both** apps
2. Ensure the same users are in both apps

---

## Testing Your Setup

### Test 1: SCIM User Provisioning

1. **Assign a test user** to the SCIM app in Okta
2. **Check Coder UI** → Users page
   - User should appear automatically within seconds
   - Status will be "Dormant" (until first login)

### Test 2: OIDC Authentication

1. **Open Coder** in incognito/private window
2. Click **Login with OIDC** (or your SSO button)
3. Should redirect to Okta
4. After authentication, should return to Coder
5. User status changes from "Dormant" to "Active"

### Test 3: User Deactivation

1. **Unassign the test user** from the SCIM app in Okta
2. **Check Coder UI** → User should be marked as "Suspended"
3. User cannot log in anymore

---

## Debugging SCIM Issues

### Enable SCIM Logging in Coder

```bash
# Enable SCIM-specific logging
CODER_LOG_FILTER=".*scim.*"

# Or enable all debug logs
CODER_LOG_FILTER=".*"
```

Look for SCIM operations tagged with:
```json
{
  "automatic_actor": "coder",
  "automatic_subsystem": "scim"
}
```

### Check Coder Logs

```bash
# If running in Kubernetes
kubectl logs deployment/coder -n coder --follow

# If running as systemd service
journalctl -u coder --follow

# Look for SCIM requests
grep "scim" /var/log/coder.log
```

### Verify SCIM Endpoint

```bash
# Test endpoint accessibility
curl -v -H "Authorization: Bearer YOUR_SCIM_API_KEY" \
     https://your-coder-domain.com/scim/v2/ServiceProviderConfig
```

**Expected response:**
- Status: 200 OK
- Content-Type: application/scim+json
- Body: JSON with SCIM capabilities

**Common failures:**
- 401 Unauthorized: Wrong API key or missing "Bearer" prefix
- 404 Not Found: Wrong URL or SCIM not enabled
- Connection refused: Network/firewall issue

### Check Okta System Logs

1. **Okta Admin Console** → Reports → System Log
2. **Filter by:**
   - Application: Your SCIM app
   - Event Type: `app.provision.*`
3. **Look for:**
   - Success events: `app.provision.user.push`
   - Error events: Red icons with error messages

### Manual SCIM Testing

```bash
# Test user creation
curl -X POST \
  -H "Authorization: Bearer YOUR_KEY" \
  -H "Content-Type: application/scim+json" \
  -d '{
    "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
    "userName": "testuser",
    "emails": [{"primary": true, "value": "[email protected]", "type": "work"}],
    "name": {"givenName": "Test", "familyName": "User"},
    "active": true
  }' \
  https://your-coder.com/scim/v2/Users

# Test user deactivation (replace USER_UUID)
curl -X PATCH \
  -H "Authorization: Bearer YOUR_KEY" \
  -H "Content-Type: application/scim+json" \
  -d '{"active": false}' \
  https://your-coder.com/scim/v2/Users/USER_UUID
```

---

## Common Issues & Solutions

### Issue: SCIM Connection Test Fails

**Symptoms:**
- "Unable to verify credentials" error in Okta
- 401 Unauthorized response

**Solutions:**
1. **Verify Bearer token format:**
   - Must be: `Bearer your-key-here` (with space)
   - Not: `your-key-here` or `Bearer: your-key-here`

2. **Check API key matches:**
   ```bash
   # On Coder server
   echo $CODER_SCIM_API_KEY
   ```

3. **Test endpoint directly:**
   ```bash
   curl -H "Authorization: Bearer $CODER_SCIM_API_KEY" \
        https://your-coder.com/scim/v2/ServiceProviderConfig
   ```

4. **Check Coder logs** for authentication errors

### Issue: Users Not Provisioning

**Symptoms:**
- User assigned in Okta but doesn't appear in Coder
- No SCIM requests in Coder logs

**Solutions:**
1. **Verify "Create Users" is enabled:**
   - SCIM app → Provisioning → To App
   - "Create Users" should be checked

2. **Check user is assigned to SCIM app:**
   - SCIM app → Assignments tab
   - User should be listed

3. **Check Okta System Logs:**
   - Look for provisioning errors
   - Filter by user and SCIM app

4. **Verify email domain:**
   - Coder may reject users with wrong email domain
   - Check `CODER_OIDC_EMAIL_DOMAIN` setting

### Issue: Users Not Being Removed/Suspended

**Symptoms:**
- User unassigned from Okta but still active in Coder
- No deactivation request in logs

**Solutions:**
1. **Verify "Deactivate Users" is enabled:**
   - SCIM app → Provisioning → To App → Edit
   - "Deactivate Users" should be checked

2. **Understand Okta behavior:**
   - Okta sends PATCH request with `active: false`
   - Okta does NOT send DELETE requests
   - User becomes "suspended" in Coder (not deleted)

3. **Check Okta System Logs:**
   - Look for `app.provision.user.deprovision` events
   - Check for errors

4. **Verify initial provisioning succeeded:**
   - If user creation failed, deactivation won't work
   - Check Okta's view of user status

### Issue: OIDC Authentication Works But User Not in Coder

**Symptoms:**
- User can authenticate via OIDC
- But gets "User not found" error

**Solutions:**
1. **User is in OIDC app but not SCIM app:**
   - Assign user to SCIM app for provisioning
   - Or enable JIT (Just-In-Time) provisioning in Coder

2. **User emails don't match:**
   - OIDC email must match SCIM email
   - Check user profile in both systems

### Issue: "Deactivate Users" Option Not Appearing

**Symptoms:**
- Can't find "Deactivate Users" checkbox in Okta

**Solutions:**
1. **Make sure you enabled SCIM first:**
   - General tab → Edit → Provisioning: SCIM

2. **Check Provisioning tab exists:**
   - If no Provisioning tab, SCIM wasn't enabled properly

3. **Contact Okta support:**
   - Feature may need to be enabled for your org

### Issue: Authentication Failures

**Symptoms:**
- 401 errors in Coder logs
- "Invalid authorization" responses

**Solutions:**
1. **Check Bearer token has no extra spaces:**
   ```bash
   # Wrong
   "Bearer  your-key"  # Two spaces
   "Bearer your-key "  # Trailing space
   
   # Correct
   "Bearer your-key"
   ```

2. **Verify API key is set in Coder:**
   ```bash
   # Should output your key
   printenv | grep CODER_SCIM_API_KEY
   ```

3. **Restart Coder after changing env vars**

---

## How User Removal Works

### Okta Behavior

- **Okta does NOT send DELETE requests** to SCIM endpoints
- Instead, it sends: `PATCH /Users/{id}` with `{"active": false}`
- This happens immediately when user is unassigned

### Coder Response

1. **Receives SCIM PATCH request** with `active: false`
2. **Updates user status** synchronously:
   - `active: true` → User becomes "Dormant" (if not logged in) or stays "Active"
   - `active: false` → User becomes "Suspended"
3. **Suspended users:**
   - Cannot log in
   - Cannot access workspaces
   - Don't count toward license seats
   - Can be reactivated by setting `active: true`

### Timing

- **No built-in delay** in Coder
- Processing is **synchronous** (happens immediately)
- Typical response time: < 1 second
- If there's a delay, it's from:
  - Okta processing time
  - Network latency
  - Coder server load

### Verification

```bash
# Watch Coder logs during deprovisioning
tail -f /var/log/coder.log | grep scim

# You should see:
# - PATCH request received
# - User status updated
# - Response sent
```

---

## Important Notes

### User Status Mapping

| Okta State | SCIM Request | Coder Status |
|------------|--------------|---------------|
| Assigned (never logged in) | `active: true` | Dormant |
| Assigned (logged in) | `active: true` | Active |
| Unassigned | `active: false` | Suspended |

### Authentication vs Provisioning

- **Authentication (OIDC)**: Who you are, handled by OIDC app
- **Provisioning (SCIM)**: Do you exist, handled by SCIM app
- Both are required for a complete user lifecycle

### Security Considerations

1. **Keep SCIM API key secure**
   - Treat like a database password
   - Rotate periodically
   - Store in secrets management (e.g., Kubernetes secrets)

2. **Use HTTPS only**
   - Never configure HTTP SCIM endpoints
   - Validate SSL certificates

3. **Monitor SCIM audit logs**
   - Track all provisioning activities
   - Set up alerts for failures

4. **Limit network access**
   - Whitelist Okta IPs if possible
   - Use firewall rules

---

## SCIM API Endpoints

Coder implements these SCIM 2.0 endpoints:

| Method | Endpoint | Purpose | Coder Behavior |
|--------|----------|---------|----------------|
| GET | `/scim/v2/ServiceProviderConfig` | Service capabilities | Returns supported features |
| GET | `/scim/v2/Users` | List users | Always returns empty (forces Okta to create) |
| GET | `/scim/v2/Users/{id}` | Get user by ID | Always returns 404 (forces Okta to create) |
| POST | `/scim/v2/Users` | Create user | Creates or returns existing user |
| PATCH | `/scim/v2/Users/{id}` | Update user | Updates active status |
| PUT | `/scim/v2/Users/{id}` | Replace user | Updates active status |

### Why GET Always Returns Empty/404

This is intentional design:
- Forces Okta to always attempt creation
- Simplifies implementation (no need to list/serialize users)
- POST endpoint handles "create or return existing"

---

## Complete Configuration Reference

### Coder Environment Variables

```bash
# ============================================
# OIDC Configuration (Authentication)
# ============================================
CODER_OIDC_ISSUER_URL="https://your-okta-domain.okta.com"
CODER_OIDC_CLIENT_ID="0oa1abc2def3GHI456"
CODER_OIDC_CLIENT_SECRET="your-oidc-client-secret"
CODER_OIDC_EMAIL_DOMAIN="your-company.com,contractor-company.com"
CODER_OIDC_SCOPES="openid,profile,email,groups"

# Optional: Group sync from OIDC claims
CODER_OIDC_GROUP_FIELD="groups"
CODER_OIDC_GROUP_AUTO_CREATE="true"

# Optional: Ignore email verification
CODER_OIDC_IGNORE_EMAIL_VERIFIED="false"

# Optional: Custom username field
CODER_OIDC_USERNAME_FIELD="preferred_username"

# ============================================
# SCIM Configuration (Provisioning)
# ============================================
CODER_SCIM_API_KEY="your-secret-scim-token-here"

# ============================================
# Logging (for debugging)
# ============================================
# CODER_LOG_FILTER=".*scim.*"  # SCIM only
# CODER_LOG_FILTER=".*"        # All debug logs
```

### Okta Apps Configuration Summary

**OIDC App:**
- Name: `Coder`
- Type: OIDC - OpenID Connect, Web Application
- Redirect URI: `https://your-coder.com/api/v2/users/oidc/callback`
- Grants: Authorization Code, Refresh Token
- Scopes: openid, profile, email, groups

**SCIM App:**
- Name: `Coder SCIM Provisioning`
- Type: SAML 2.0 (but only using SCIM)
- SCIM Base URL: `https://your-coder.com/scim/v2`
- Auth: Bearer token
- Features: Create Users, Update Attributes, Deactivate Users

---

## Resources

### Coder Documentation
- [OIDC Authentication](https://coder.com/docs/admin/users/oidc-auth)
- [Groups and Roles](https://coder.com/docs/admin/users/groups-roles)
- [Monitoring and Logs](https://coder.com/docs/admin/monitoring/logs)

### Okta Documentation
- [SCIM Protocol](https://developer.okta.com/docs/concepts/scim/)
- [Create OIDC Apps](https://developer.okta.com/docs/guides/create-an-app-integration/openidconnect/main/)
- [Add SCIM Provisioning](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_scim.htm)

### Coder Source Code
- [SCIM Implementation](https://github.com/coder/coder/blob/main/enterprise/coderd/scim.go)
- [SCIM Types](https://github.com/coder/coder/blob/main/enterprise/coderd/scim/scimtypes.go)

---

## Summary

✅ **Two apps required**: OIDC for auth, SAML with SCIM for provisioning

✅ **Assign same users** to both apps (use groups to simplify)

✅ **Test thoroughly**: Provisioning → Authentication → Deprovisioning

✅ **Monitor logs**: Both Okta System Logs and Coder server logs

✅ **Security first**: Treat SCIM API key as a secret, use HTTPS only

This setup provides complete user lifecycle management:
- Users created automatically when assigned
- Authentication via OIDC SSO
- Users suspended automatically when unassigned
- All changes logged for audit

---

*Generated from Slack thread discussion - Last updated: October 2025*
