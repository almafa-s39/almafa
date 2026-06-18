# Workfolder

> [!NOTE]
> If you have a Core server, you can done nearly everyting from another domain joined server which has GUI.
> 0. for core servers: Add the server in server manager on another server with GUI.

## Install WorkFolders

Install it from Server manager or issue the following command:

```powershell
Install-WindowsFeature FS-SyncShareService, Web-Server -IncludeManagementTools
```

> [!WARNING]
> Don't forget to restart the server (restart required)!

## Setup

Use wizard from Server Manager, it's next-next finish.

## Access setup

> [!NOTE]
> After you're done with installation and setup, you have to create a binding using a certificate for workfolders.
> For this you will need a certificate including the DNS name, you have to configure for workfolders (for HTTPS).
> Export a certificate out with the right extensions (and private key exporatble!), and import it to the server with the following  command:

```powershell
$pw = ConvertTo-SecureString "YourPW" -AsPlainText -Force
Import-PfxCertificateFile -FilePath C:\server.pfx -Password $pw -CertStoreLocation CERT:\LocalMachine\My
```

> [!NOTE]
> Save the thumbprint we will need it for  the next step.

```powershell
$guid=New-Guid
netsh http add sslcert ipport=0.0.0.0:443 certhash=<Your-Cert-Thumbprint> appid=$guid certstorename=MY
```
