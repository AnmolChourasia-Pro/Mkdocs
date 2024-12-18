# Azure Api Permissions for SharePoint

To access SharePoint Online, configure the following permissions in Azure.

---

## Required API Permissions

1. **Microsoft Graph API**  
     - Type: Delegated (user context)
         - **Permissions**: `User.Read` (for basic user info)

2. **SharePoint API**  
     
     - Type: Application.
         - **Permissions**: `Sites.Selected` , `User.Read.All`
     - Type: Delegate.
         - **Permissions**:  `Sites.Search.All` , `AllSites.Read`(user context)

3. **Grant FullControl permission with PowerShell**
     - Open PowerShell as an administrator. 
     - Run the following command 
         - `Connect-PnPOnline "<site-url>" -Interactive`
         - `Grant-PnPAzureADAppSitePermission -AppId "<app-id>" -DisplayName "<app-display-name>" -Permissions FullControl -Site <site-url>`

      **UAT Site Example** 

         Connect-PnPOnline "https://cromaretail.sharepoint.com/sites/LegaDoxUat" -Interactive 

         Grant-PnPAzureADAppSitePermission -AppId "Application (Client) ID" -DisplayName "Application Display Name" -Permissions FullControl -Site https://cromaretail.sharepoint.com/sites/LegaDoxUat 

---
# Azure Api Permissions for AD user 

To access Azure AD users, configure the following permissions in Azure.

---

## Required API Permissions
1. **Microsoft Graph API**  
     - **Permissions**: `User.ReadBasic.All`
     - Type: Application

---

# Azure Api Permissions for Mail 

To Send mail via Graph Api as any user .

---
## Required API Permissions
1. **Microsoft Graph API**  
     - **Permissions**: `Mail.Send`
     - Type: Application


---

## Steps to Add Permissions

1. Go to **API Permissions** in your app registration.
2. Click **Add a permission**.
3. Choose **Microsoft APIs** > **SharePoint** or **Microsoft Graph**.
4. Add the permissions mentioned above.
5. **Grant Admin Consent** if required.

---


## Verify Permissions
To verify, use tools like **Postman** or **Graph Explorer** to make API calls with the configured permissions.
