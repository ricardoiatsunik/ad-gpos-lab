[English version](README.en.md)

# Active Directory e GPO com Windows Server 2025

Projeto de laboratório para configurar um domínio Active Directory no Windows Server 2025 e testar Group Policies em estações Windows 11.

O ambiente foi criado para praticar a instalação do AD DS, configuração de DNS, organização de usuários e computadores em OUs, criação de grupos e aplicação de GPOs voltadas à administração e segurança.

---

### Ambiente

```text
VMware NAT
│
├── DC01 - Windows Server 2025
│   ├── Active Directory Domain Services
│   ├── DNS
│   └── Group Policy Management
│
└── CL01 - Windows 11
    └── Estação ingressada no domínio
```

| Máquina | Função | IP de exemplo |
|---|---|---|
| DC01 | Controlador de domínio e DNS | 192.168.100.10 |
| CL01 | Estação cliente | 192.168.100.20 |
| Gateway VMware | Saída para a internet | 192.168.100.2 |

> O endereçamento é apenas um exemplo. A sub-rede e o gateway devem ser ajustados conforme a rede configurada no VMware.

---

### Requisitos

- VMware Workstation Pro
- Windows Server 2025 instalado
- Windows 11 Pro ou Enterprise
- Rede NAT ou Host-only configurada no VMware
- Acesso de administrador nas duas máquinas
- Snapshot do servidor antes da instalação do AD DS

---

### Domínio

| Item | Valor |
|---|---|
| Domínio DNS | corp.riat.test |
| Nome NetBIOS | RIAT |
| Controlador de domínio | DC01 |
| Cliente | CL01 |

A extensão `.test` foi usada porque o ambiente é apenas de laboratório.

---

## 1. Preparação do servidor

Antes de instalar o Active Directory:

- Renomeie o servidor para `DC01`
- Configure um endereço IPv4 fixo
- Use o próprio servidor como DNS preferencial
- Instale as atualizações do Windows
- Confira data, horário e fuso
- Crie um snapshot da VM

### Renomear o servidor

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Configurar o endereço IP

Confira primeiro o nome da interface:

```powershell
Get-NetAdapter
```

Exemplo de configuração:

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

Teste a configuração:

```powershell
ipconfig /all
ping 192.168.100.2
```

---

## 2. Instalação do AD DS e DNS

No **Server Manager**:

1. Abra **Add Roles and Features**
2. Selecione **Role-based or feature-based installation**
3. Marque **Active Directory Domain Services**
4. Adicione as ferramentas de gerenciamento
5. Conclua a instalação
6. Clique em **Promote this server to a domain controller**

Crie uma nova floresta com o domínio:

```text
corp.riat.test
```

Defina o nome NetBIOS como:

```text
RIAT
```

O DNS deve ser instalado junto com o AD DS. A senha do DSRM não deve ser salva no repositório.

### Instalação pelo PowerShell

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Depois:

```powershell
Install-ADDSForest `
    -DomainName "corp.riat.test" `
    -DomainNetbiosName "RIAT" `
    -InstallDNS
```

O servidor será reiniciado ao final da promoção.

---

## 3. Validação do domínio

Após reiniciar, entre com a conta:

```text
RIAT\Administrator
```

Confira o domínio e a floresta:

```powershell
Get-ADDomain
Get-ADForest
```

Confira os serviços:

```powershell
Get-Service NTDS, DNS, Netlogon
```

Verifique os compartilhamentos do domínio:

```powershell
net share
```

Os compartilhamentos `SYSVOL` e `NETLOGON` devem aparecer.

Execute também:

```powershell
dcdiag
nslookup corp.riat.test
```

---

## 4. Estrutura de OUs

A estrutura usada no laboratório separa usuários, computadores, grupos e contas administrativas.

```text
corp.riat.test
│
└── OU=LAB
    ├── OU=Usuarios
    │   ├── OU=TI
    │   ├── OU=Financeiro
    │   └── OU=RH
    │
    ├── OU=Computadores
    │   ├── OU=Workstations
    │   └── OU=Servers
    │
    ├── OU=Grupos
    ├── OU=Administracao
    └── OU=Desabilitados
```

A estrutura pode ser criada manualmente no **Active Directory Users and Computers** ou com o script:

```powershell
.\scripts\New-LabStructure.ps1
```

O script também cria estes grupos:

```text
GG-TI-Usuarios
GG-Financeiro-Usuarios
GG-RH-Usuarios
GG-Suporte-LocalAdmins
```

---

## 5. Usuários de teste

O arquivo `data/users.csv` possui três contas de exemplo:

| Nome | Usuário | Departamento |
|---|---|---|
| Ana Lima | ana.lima | TI |
| Bruno Souza | bruno.souza | Financeiro |
| Carla Mendes | carla.mendes | RH |

Para criar as contas:

```powershell
.\scripts\New-LabUsers.ps1
```

O script solicita uma senha temporária, cria os usuários na OU correta e adiciona cada conta ao grupo do departamento.

A senha não é armazenada no arquivo CSV ou no código.

---

## 6. Ingresso do Windows 11 no domínio

Na estação `CL01`, configure o DNS para apontar para o controlador de domínio:

```text
DNS preferencial: 192.168.100.10
```

Teste a resolução:

```powershell
nslookup corp.riat.test
ping DC01
```

Entre no domínio pelo menu:

```text
Configurações
└── Sistema
    └── Sobre
        └── Domínio ou grupo de trabalho
```

Também é possível usar o PowerShell:

```powershell
Add-Computer `
    -DomainName "corp.riat.test" `
    -Credential "RIAT\Administrator" `
    -Restart
```

Depois do reinício, mova o objeto `CL01` para:

```text
OU=Workstations,OU=Computadores,OU=LAB
```

Faça logon com um usuário do domínio:

```text
RIAT\ana.lima
```

---

## 7. Criação das GPOs

Abra o console:

```text
gpmc.msc
```

Crie uma GPO separada para cada objetivo e vincule-a à OU correta. Evite colocar todas as configurações na `Default Domain Policy`.

Convenção usada:

```text
GPO-C-[OBJETIVO]  -> configuração de computador
GPO-U-[OBJETIVO]  -> configuração de usuário
```

---

### GPO-C-Firewall

Mantém o Windows Defender Firewall ativo nos três perfis.

Caminho:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Windows Defender Firewall with Advanced Security
```

Configuração:

```text
Domain Profile  -> Firewall state: On
Private Profile -> Firewall state: On
Public Profile  -> Firewall state: On
```

Vincular em:

```text
OU=Workstations
```

---

### GPO-C-Auditoria-Logon

Registra tentativas de logon com sucesso e falha.

Caminho:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Advanced Audit Policy Configuration
                └── Logon/Logoff
                    └── Audit Logon
```

Configuração:

```text
Success
Failure
```

Vincular em:

```text
OU=Workstations
```

Os eventos podem ser consultados no **Event Viewer**, em:

```text
Windows Logs
└── Security
```

---

### GPO-U-Bloqueio-Tela

Ativa o protetor de tela e exige senha para desbloquear.

Caminho:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
            └── Personalization
```

Configurações:

```text
Enable screen saver                  -> Enabled
Password protect the screen saver    -> Enabled
Screen saver timeout                 -> Enabled / 600 seconds
```

Vincular em:

```text
OU=Usuarios
```

---

### GPO-U-Bloquear-Painel-Controle

Bloqueia o Painel de Controle e o aplicativo Configurações para os usuários de teste.

Caminho:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
            └── Prohibit access to Control Panel and PC settings
```

Configuração:

```text
Enabled
```

Vincular inicialmente em uma OU de teste, como:

```text
OU=TI
```

---

### GPO-C-Mensagem-Logon

Exibe um aviso antes do logon.

Caminho:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── Security Options
```

Configurações:

```text
Interactive logon: Message title for users attempting to log on
Interactive logon: Message text for users attempting to log on
```

Exemplo:

```text
Título: Ambiente de Laboratório
Texto: Acesso permitido somente para testes autorizados.
```

Vincular em:

```text
OU=Workstations
```

---

### GPO-C-Bloquear-Armazenamento-Removivel

Bloqueia o uso de dispositivos de armazenamento removível.

Caminho:

```text
Computer Configuration
└── Policies
    └── Administrative Templates
        └── System
            └── Removable Storage Access
                └── All Removable Storage classes: Deny all access
```

Configuração:

```text
Enabled
```

Essa política deve ser testada em uma OU separada antes de ser aplicada a todas as estações.

---

## 8. Aplicação e validação das GPOs

Na estação cliente:

```powershell
gpupdate /force
```

Confira as políticas aplicadas:

```powershell
gpresult /r
```

Gere um relatório HTML:

```powershell
New-Item -ItemType Directory -Path C:\Temp -Force
gpresult /h C:\Temp\gpo-report.html
```

Outras ferramentas:

```text
rsop.msc
gpmc.msc
Event Viewer
```

Para uma GPO de computador, confira se o objeto da máquina está na OU vinculada.

Para uma GPO de usuário, confira se a conta está na OU vinculada.

Também valide:

- Resolução DNS do domínio
- Comunicação com o controlador de domínio
- Link da GPO
- Security Filtering
- Herança das OUs
- Configuração em User ou Computer Configuration

---

## 9. Backup e relatórios

O script abaixo exporta um backup e um relatório HTML de cada GPO:

```powershell
.\scripts\Export-GPOs.ps1
```

Os arquivos são salvos em:

```text
backups\
reports\
```

Essas pastas não são enviadas ao GitHub porque podem conter dados específicos do domínio.

---

## Estrutura do repositório

```text
windows-server-2025-ad-gpo-lab
│
├── README.md
├── README.en.md
├── LICENSE
├── .gitignore
│
├── data
│   └── users.csv
│
├── scripts
│   ├── New-LabStructure.ps1
│   ├── New-LabUsers.ps1
│   └── Export-GPOs.ps1
│
├── evidence
│   └── .gitkeep
│
├── backups
│   └── .gitkeep
│
└── reports
    └── .gitkeep
```

A pasta `evidence` pode ser usada para capturas de tela da instalação, estrutura do AD e testes das GPOs.

---

## Próximos testes

- Security Filtering
- Herança e precedência de GPOs
- Block Inheritance e Enforced
- WMI Filters
- Loopback Processing
- Delegação de controle em OUs
- Windows LAPS
- Central Store para arquivos ADMX
- Política de senha e bloqueio de conta

---

## Resultado

O laboratório permite praticar a administração básica de um domínio Windows, desde a instalação do AD DS até a aplicação de políticas em usuários e computadores.

Com as OUs, os objetos ficam separados por função e podem receber políticas diferentes. As GPOs permitem controlar configurações das estações, aplicar regras de segurança e validar o resultado pelo `gpresult`, RSoP e GPMC.

---

## Referências

- [Install Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [Group Policy Management Console](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-management-console)
- [Group Policy processing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)
- [gpresult](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult)
- [Windows LAPS](https://learn.microsoft.com/en-us/windows-server/identity/laps/laps-overview)
