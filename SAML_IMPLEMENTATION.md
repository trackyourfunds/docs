# SAML Authentication Implementation

This document explains how SAML authentication works in the TYF application.

## Overview

The SAML implementation provides Single Sign-On (SSO) authentication through SAML 2.0 identity providers (such as Azure AD). The implementation uses:

- **Backend**: Industry-standard SAML 2.0 protocol handling
- **Frontend**: Secure client-side authentication flow
- **Token Management**: JWT bearer tokens for API authentication with refresh token support

## Token Management Architecture

### Two-Token System

The SAML implementation uses a two-token approach for security and usability:

1. **Access Token (JWT)**
   - Short-lived (default: 15 minutes)
   - Stored securely in browser
   - Sent as `Authorization: Bearer {token}` header on web API calls from the browser to the TYF application
   - Contains user claims: email, name, organization
   - Used for authorization and permission checking

2. **Refresh Token (JWT)**
   - Long-lived (default: 7 days)
   - Stored in secure httpOnly cookie
   - Not accessible to JavaScript (XSS protection)
   - Used to obtain new access tokens silently without re-authentication
   - Validated on backend before issuing new access token

### Why This Approach?

- **Short-lived Access Tokens**: Limits exposure window if token is compromised
- **Secure Refresh Tokens**: Immune to XSS attacks, enables silent token renewal
- **Automatic Token Refresh**: Better user experience, fewer re-authentications
- **Session Management**: Tokens are cleared when browser session ends

## Configuration

### 2. Identity Provider Configuration

Configure your SAML Identity Provider with the following settings:

#### Azure AD / Entra ID

1. **Enterprise Application Setup**:
   - Create a new Enterprise Application in Azure AD
   - Select "SAML" as the authentication method
   - Set the following URLs:
     - Entity ID: `https://{your-organization}.trackyourfunds.com`
     - Reply URL (ACS): `https://{your-organization}.trackyourfunds.com/api/saml/acs`
     - Sign on URL: `https://{your-organization}.trackyourfunds.com`
     - Logout URL: `https://{your-organization}.trackyourfunds.com/api/saml/logout` (optional)

2. **Required Claims**:
   - `email` or `emailaddress`: User's email (mapped to `preferred_username` in JWT)
   - `name` or `displayname`: User's full name
   - `givenname`: First name (optional)
   - `surname`: Last name (optional)

3. **Download Metadata**:
   - Download the Federation Metadata XML
   - Note the metadata URL for configuration

## Send to TrackYourFunds

After configuring your Azure AD Enterprise Application, please send the following information to TrackYourFunds:

### Required Information

| Item | Description | Where to Find |
|------|-------------|---------------|
| **SAML Federation Metadata URL** | The URL to your IdP's SAML metadata XML file | Azure AD: Enterprise Applications ? Your App ? Single sign-on ? SAML ? **App Federation Metadata Url** |
| **SSO Sign-in Endpoint** | The login URL for your Identity Provider | Azure AD: Enterprise Applications ? Your App ? Single sign-on ? **Login URL** |
| **SSO Sign-out Endpoint URL** | The logout URL for your Identity Provider | Azure AD: Enterprise Applications ? Your App ? Single sign-on ? **Logout URL** |
| **Client ID** | Your application's unique identifier | Azure AD: Enterprise Applications ? Your App ? Overview ? **Application ID** |
| **All Claims/Assertions** | All claims/assertions sent from the application | Azure AD: Enterprise Applications ? Your App ? Single sign-on ? **Attributes & Claims** section |

### How to Send

Please provide the above information to TrackYourFunds.

Include:
- Your organization name
- All five required items listed above
- Any additional notes or questions

## Authentication Flow

### 1. Initial Login Flow

```
User navigates to https://{your-organization}.trackyourfunds.com
        ?
If already authenticated:
  ? User goes directly to homepage
        ?
If not authenticated:
  ? TYF creates a request to the IdP to authenticate
  ? Application automatically redirects to Identity Provider login panel
        ?
IdP displays login panel
        ?
User authenticates at IdP (enters credentials)
        ?
IdP handles all authentication security:
  - Password validation
  - Multi-factor authentication (2FA/MFA)
  - Conditional access policies
  - Any other security requirements
  (Note: TYF does not perform any of these security checks - 
   all authentication security is managed by your Identity Provider)
        ?
IdP validates credentials and completes authentication
        ?
IdP sends SAML response back to TYF
        ?
Backend validates SAML assertion:
  - Verifies digital signature
  - Checks issuer matches configuration
  - Validates timestamps (NotBefore, NotOnOrAfter)
  - Confirms audience (EntityId)
        ?
Extract claims from SAML assertion:
  - email (emailaddress or upn)
  - name
  - organization (from configuration)
        ?
Create JWT access token (15 min expiration)
Create refresh token (7 day expiration)
        ?
Store session data (email, name, organization)
Store refresh token securely
        ?
Redirect back to application with access token
        ?
Frontend processes callback:
  - Extracts and stores JWT access token
  - Extracts user info from JWT claims
  - Completes sign-in process
        ?
User is redirected to homepage
```

### 2. API Authentication

After successful authentication, TYF creates a bearer token. All web API requests from the browser to the TYF application use this bearer token in the Authorization header:

```typescript
// Automatic - handled by authentication manager
const response = await fetch('/api/endpoint', {
  headers: {
    'Authorization': `Bearer ${accessToken}`
  }
});
```

### 3. Token Refresh Flow

When the access token expires (after 15 minutes), the application automatically refreshes it:

- Frontend detects expired token
- Securely requests new access token using refresh token
- Backend validates refresh token
- New access token is issued and stored
- Refresh token remains valid

This provides a seamless user experience - tokens are refreshed silently in the background without requiring re-authentication.

### 4. Logout Flow

When a user logs out:

- User clicks "Logout"
- Frontend clears stored credentials
- Server session is cleared
- SAML logout request is sent to IdP
- User is redirected to IdP logout page
- IdP confirms logout
- User is returned to application login page

## Authorization & Permissions

### How User Identity is Established

1. **SAML Assertion**: IdP sends user claims in SAML response
2. **JWT Creation**: Backend transforms SAML claims into JWT token
3. **Authorization**: Token claims are validated for each API request

### User Information

The JWT access token contains user information including:
- User's email address (used as unique identifier)
- User's display name
- Organization identifier
- Token expiration timestamp

This information is used throughout the application for:
- Displaying user information
- Making authorization decisions
- Filtering data by organization/location

## Security Considerations

### Token Security

- **Access Tokens**: Short-lived (15 minutes) to limit exposure
- **Refresh Tokens**: Long-lived (7 days) for better user experience
- **Secure Storage**: Tokens are stored using industry-standard secure methods
- **HTTPS Only**: All authentication happens over encrypted connections
- **XSS Protection**: Refresh tokens are protected from JavaScript access
- **CSRF Protection**: Cookies use SameSite attributes

### SAML Assertion Validation

The backend validates all SAML assertions:

1. **Digital Signature**: Verifies IdP's signature using metadata certificate
2. **Issuer Validation**: Ensures assertion is from expected IdP
3. **Audience Restriction**: Confirms assertion is intended for this application
4. **Timestamp Validation**: Checks NotBefore and NotOnOrAfter conditions

### Token Validation

JWT tokens are validated on every API request to ensure security and authenticity.

## User Experience

The authentication system provides a seamless experience for end users:

- **Automatic Login**: Users are redirected to your corporate identity provider
- **Single Sign-On**: Use existing corporate credentials
- **Automatic Token Refresh**: Tokens are refreshed seamlessly in the background
- **Secure Logout**: Clears all authentication data and logs out of identity provider
