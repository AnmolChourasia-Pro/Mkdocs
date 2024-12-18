# Microsoft Azure App 

To interact with SharePoint Online securely, register your app in Azure.

## Steps to Register an App

1. **Application Registration on Entra Portal**  
     - Go to [Microsoft Entra Portal](https://entra.microsoft.com/) and sign in with your Microsoft azure account. 
   
     - Under **Identity**, select **Applications** and then **App Registrations**. Click on New Registration. 
     - Provide Appropriate Display Name, select blue highlight option and click on Register button.
     - **Supported Account Types**: Choose based on your scenario (e.g., single-tenant or multi-tenant).
     - Add **Redirection URL** in below box and click on save button eg–  `https://uat-clms.croma.com/` or `https://localhost`
     - check **Access Token** and **ID token** checkbox and click on save button. 


2. **Configuring an X.509 Certificate for the application**
    - Open Windows PowerShell (Version 7.4.2) with admin privileges.
    - Create one folder on Local machine and use this folder for certificate creation.
    - Run below command in PowerShell window. (Update certificate name and local folder path) 

        
            $cert = New-PnPAzureCertificate -CommonName "my-certificate-common-name" -OutPfx .\my-certificate.pfx -OutCert .\my-certificate.cer -ValidYears 2 -CertificatePassword ("Read-Host -AsSecureString -Prompt Enter Certificate Password")
        
    
    - When asked, enter a password which will be required while creating App Context.   
    - The above script creates a new X.509 certificate and it stores its .PFX and .CER files in the specified file paths. Then, it outputs the thumbprint of the generated certificate. 
    - Go back to the Azure AD web page showing the application information and select on the **Certificates & secrets** menu on the left side of the application page. Select the **Certificates** tab in the page and select on **Upload certificate** and upload the .CER file from there. 



3. **Collect Application Details**  
   After registration, note the following details:
    - **Application (Client) ID**
    - **Directory (Tenant) ID**
    - **Certificate path and password**
---

## Summary
Once registered, your app is ready to interact with SharePoint. Move to the next section to configure permissions.
