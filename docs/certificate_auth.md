# Certificate-Based Authentication

To improve security, use a certificate instead of a client secret for authentication in Azure AD.

---

## Prerequisites

1. **Registered App in Azure AD** (see [Azure App Registration](azure_app_registration.md)).
2. **Certificate**: Exported `.pfx` file with a secure password.
3. **Dot Net framework**: `version 4.5` 

---

## Code Implementation

### **Set Up Configuration**
Add these settings to your `app.config` or `web.config` file:

```xml
<appSettings>
    <add key="SpHostUrl" value="https://yoursharepointsite.sharepoint.com" />
    <add key="clientid" value="your-client-id" />
    <add key="certificate" value="path-to-your-certificate.pfx" />
    <add key="certificatePassword" value="your-certificate-password" />
    <add key="AzureTenantID" value="your-tenant-id" />
</appSettings>
```

