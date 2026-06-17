[Versão em português](README.md)

# Active Directory and GPO with Windows Server 2025

Lab project for configuring an Active Directory domain on Windows Server 2025 and testing Group Policies on Windows 11 workstations.

The environment covers AD DS installation, DNS configuration, users and computers organized into OUs, security groups and GPOs for administration and security testing.

---

### Environment

```text
VMware NAT
│
├── DC01 - Windows Server 2025
│   ├── Active Directory Domain Services
│   ├── DNS
│   └── Group Policy Management
│
└── CL01 - Windows 11
    └── Domain-joined workstation
```

| Machine | Role | Example IP |
|---|---|---|
| DC01 | Domain controller and DNS | 192.168.100.10 |
| CL01 | Client workstation | 192.168.100.20 |
| VMware gateway | Internet access | 192.168.100.2 |

> The addresses are examples. Adjust the subnet and gateway to match the VMware network.

---

### Requirements

- VMware Workstation Pro
- Windows Server 2025
- Windows 11 Pro or Enterprise
- VMware NAT or Host-only network
- Administrator access on both machines
- A server snapshot before installing AD DS

---

### Domain

| Item | Value |
|---|---|
| DNS domain | corp.riat.test |
| NetBIOS name | RIAT |
| Domain controller | DC01 |
| Client | CL01 |

The `.test` suffix is used because this is a lab environment.

---

## 1. Server preparation

Before installing Active Directory:

- Rename the server to `DC01`
- Configure a static IPv4 address
- Set the server itself as the preferred DNS server
- Install Windows updates
- Check date, time and time zone
- Create a VM snapshot

```powershell
Rename-Computer -NewName "DC01" -Restart
```

Check the network interface name:

```powershell
Get-NetAdapter
```

Example configuration:

```powershell
New-NetIPAddress `
    -InterfaceAlias "Ethernet" `
    -IPAddress "192.168.100.10" `
    -PrefixLength 24 `
    -DefaultGateway "192.168.100.2"

Set-DnsClientServerAddress `
    -InterfaceAlias "Ethernet" `
    -ServerAddresses "192.168.100.10"
```

---

## 2. AD DS and DNS installation

In **Server Manager**:

1. Open **Add Roles and Features**
2. Select **Role-based or feature-based installation**
3. Select **Active Directory Domain Services**
4. Add the management tools
5. Complete the installation
6. Select **Promote this server to a domain controller**

Create a new forest:

```text
corp.riat.test
```

Use `RIAT` as the NetBIOS name and install DNS with AD DS.

PowerShell installation:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
    -DomainName "corp.riat.test" `
    -DomainNetbiosName "RIAT" `
    -InstallDNS
```

---

## 3. Domain validation

After the restart, sign in with:

```text
RIAT\Administrator
```

Run:

```powershell
Get-ADDomain
Get-ADForest
Get-Service NTDS, DNS, Netlogon
net share
dcdiag
nslookup corp.riat.test
```

The `SYSVOL` and `NETLOGON` shares must be available.

---

## 4. OU structure

```text
corp.riat.test
│
└── OU=LAB
    ├── OU=Usuarios
    │   ├── OU=TI
    │   ├── OU=Financeiro
    │   └── OU=RH
    ├── OU=Computadores
    │   ├── OU=Workstations
    │   └── OU=Servers
    ├── OU=Grupos
    ├── OU=Administracao
    └── OU=Desabilitados
```

Create the structure manually or run:

```powershell
.\scripts\New-LabStructure.ps1
```

The script also creates:

```text
GG-TI-Usuarios
GG-Financeiro-Usuarios
GG-RH-Usuarios
GG-Suporte-LocalAdmins
```

---

## 5. Test users

The `data/users.csv` file contains three sample users.

Run:

```powershell
.\scripts\New-LabUsers.ps1
```

The script requests a temporary password, creates the accounts in the correct OUs and adds them to the department groups. The password is not stored in the repository.

---

## 6. Join Windows 11 to the domain

On `CL01`, configure the preferred DNS server:

```text
192.168.100.10
```

Test name resolution:

```powershell
nslookup corp.riat.test
ping DC01
```

Join the domain:

```powershell
Add-Computer `
    -DomainName "corp.riat.test" `
    -Credential "RIAT\Administrator" `
    -Restart
```

Move the `CL01` computer object to:

```text
OU=Workstations,OU=Computadores,OU=LAB
```

---

## 7. Group Policies

Open:

```text
gpmc.msc
```

Create one GPO for each objective and link it to the correct OU.

Naming convention:

```text
GPO-C-[PURPOSE] -> computer configuration
GPO-U-[PURPOSE] -> user configuration
```

Policies included in the lab:

| GPO | Purpose | Link |
|---|---|---|
| GPO-C-Firewall | Enable Windows Defender Firewall profiles | Workstations |
| GPO-C-Auditoria-Logon | Audit successful and failed logons | Workstations |
| GPO-U-Bloqueio-Tela | Enable password-protected screen lock | Usuarios |
| GPO-U-Bloquear-Painel-Controle | Block Control Panel and Settings | Test user OU |
| GPO-C-Mensagem-Logon | Display a logon notice | Workstations |
| GPO-C-Bloquear-Armazenamento-Removivel | Block removable storage | Test computer OU |

Detailed policy paths are available in the Portuguese README.

---

## 8. GPO validation

On the client:

```powershell
gpupdate /force
gpresult /r

New-Item -ItemType Directory -Path C:\Temp -Force
gpresult /h C:\Temp\gpo-report.html
```

Other tools:

```text
rsop.msc
gpmc.msc
Event Viewer
```

Check the target OU, GPO link, security filtering, inheritance and whether the setting is under User or Computer Configuration.

---

## 9. Backup and reports

Run:

```powershell
.\scripts\Export-GPOs.ps1
```

The script creates GPO backups and HTML reports under:

```text
backups\
reports\
```

These generated files are excluded from Git.

---

## Repository structure

```text
windows-server-2025-ad-gpo-lab
├── README.md
├── README.en.md
├── LICENSE
├── .gitignore
├── data
│   └── users.csv
├── scripts
│   ├── New-LabStructure.ps1
│   ├── New-LabUsers.ps1
│   └── Export-GPOs.ps1
├── evidence
│   └── .gitkeep
├── backups
│   └── .gitkeep
└── reports
    └── .gitkeep
```

---

## Next tests

- Security Filtering
- GPO inheritance and precedence
- Block Inheritance and Enforced
- WMI Filters
- Loopback Processing
- OU delegation
- Windows LAPS
- ADMX Central Store
- Password and account lockout policies

---

## Result

The lab provides a practical environment for learning Windows domain administration, from AD DS installation to Group Policy deployment and validation.

---

## References

- [Install Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [Group Policy Management Console](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-management-console)
- [Group Policy processing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)
- [gpresult](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult)
- [Windows LAPS](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)
