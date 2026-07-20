# IIS Remote Management

## 1. Installation

To manage IIS from other computers, you need to first install the Web Management Service. *(Assuming that you've already installed IIS itself.)*

```powershell
Install-WindowsFeature Web-Mgmt-Service
```

## 2. Configuration

Now that the management service is installed, you need to enable remote management. To do so, open **regedit** and edit the following variable:

```powershell
[HKEY_LOCAL_MACHINE\Software\Microsoft\WebManagement\Server]

"EnableRemoteManagement":=dword:00000001
```

Now that remote management is enabled, start the service and set it to automatically start upon reboot.

```powershell
Start-Service WMSVC
Set-Service -Name WMSVC -StartupType Automatic
```

Now you can connect to IIS Management from a remote computer with the IIS console installed.
