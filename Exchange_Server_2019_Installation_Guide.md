# Exchange Server 2019 Installation Guide (GUI and Command Line)

**Last reviewed:** 18 November 2026  
**Target:** A new Exchange Server 2019 Mailbox server on a supported Windows Server, in an Active Directory forest  
**Example values:** Replace **EX01**, **dc01.contoso.com**, **Contoso**, and **mail.contoso.com** with your environment's values.

> [!IMPORTANT]
> **Exchange Server 2019 reached end of support on 14 October 2025.** Microsoft no longer provides normal product support. New post-end-of-support security updates are available only to customers enrolled in Microsoft's Exchange Server Extended Security Update (ESU) program. For a new production deployment, plan for Exchange Server Subscription Edition (SE) or Exchange Online instead. Exchange 2019 CU14 or CU15 has a supported in-place upgrade path to Exchange SE. This guide is for environments that specifically need Exchange 2019, such as a lab or a deployment covered by an approved support and update plan. See [Microsoft's end-of-support roadmap](https://learn.microsoft.com/en-us/troubleshoot/exchange/administration/exchange-2019-2016-end-of-support) and [Exchange SE upgrade guidance](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deploy-new-installations/upgrade-to-exchange-server-se).

## 1. Scope and version

This guide installs the **Mailbox server role**, which includes the Exchange management tools and the client access services used by Outlook on the web and the Exchange Admin Center. Exchange 2019 does not have a separate Client Access server role.

Use the **latest Exchange 2019 CU15 installation media**. CU15 is a complete installation package; earlier CUs do not need to be installed first. Before exposing the server to clients or the internet, install the newest security update that your organization is entitled to use. Check Microsoft's [Exchange update downloads](https://learn.microsoft.com/en-us/exchange/new-features/updates) and [build number list](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates) on the day you install. After end of support, December 2025 and later Exchange 2019 security updates require ESU enrollment.

This is a first-server/single-server installation procedure. A migration, hybrid deployment, Database Availability Group (DAG), Edge Transport deployment, or installation into a complex existing Exchange organization needs a deployment-specific plan. Use the [Microsoft Exchange Server Deployment Assistant](https://setup.cloud.microsoft/) for those environments.

## 2. Supported platform and prerequisites

Confirm these items before running Setup. Microsoft's requirements can change; use the links below to confirm the exact CU and Windows combination you plan to install.

### Server and Active Directory

- Use a supported 64-bit Windows Server edition. Exchange 2019 supports Windows Server 2019, 2022, and 2025; check the [supportability matrix](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/supportability-matrix) for CU-specific details.
- For the **GUI wizard**, install Windows Server with **Desktop Experience**. Server Core is also supported and is recommended by Microsoft for production, but it is managed without the full graphical desktop.
- Install Exchange on a **domain-joined member server**, not on a domain controller. Exchange's server name and fully qualified domain name (FQDN) must be final before installation. Renaming the server after Exchange is installed is unsupported.
- The Active Directory site that contains the Exchange server must have a writable domain controller that is also a Global Catalog. Verify domain controller operating systems and forest/domain functional levels against the supportability matrix.
- Patch Windows, confirm DNS and time synchronization, and ensure the server can reach domain controllers and Global Catalogs. The Remote Registry service must not be disabled.
- For a Mailbox server, Microsoft lists **128 GB as the minimum recommended memory**. Size storage for Exchange binaries, mailbox databases, transaction logs, queues, growth, and backups. The documented free-space minimums include 30 GB on the Exchange installation drive, 200 MB on the system drive, and 500 MB on the queue database drive; these minimums are not a production storage plan.

### Software prerequisites

- **.NET Framework:** On Windows Server 2019, use .NET Framework 4.8 for CU15. On Windows Server 2022/2025, use a .NET version supported by the CU and OS combination in the [supportability matrix](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/supportability-matrix). Do not install a .NET version solely because it is newer.
- **Visual C++ redistributables:** Install the Microsoft Visual C++ Redistributable packages for Visual Studio 2012 and 2013 required by the Mailbox role. Use Microsoft's prerequisite page for the current links.
- **Windows components:** Setup can install the required Windows roles and features. The GUI wizard has a checkbox for this; command-line Setup uses **/InstallWindowsComponents**.
- **UCMA 4.0:** Install the Unified Communications Managed API 4.0 package only when integrating with Skype for Business presence/instant messaging as required by that environment. The package is on the Exchange media under **UCMARedist**.
- Do not install a standalone Windows Management Framework/PowerShell version on the Exchange server. Use the PowerShell version that ships with Windows Server.

See Microsoft's [Exchange Server prerequisites](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/prerequisites?view=exchserver-2019) and [system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements?view=exchserver-se).

### Accounts and permissions

Use a separate, approved administrative account and elevated setup session.

- To extend the schema, the account must be in **Schema Admins** and **Enterprise Admins**.
- To prepare the Exchange organization, the account must be in **Enterprise Admins**; if that step also extends the schema, it needs **Schema Admins** too.
- Installing the first Exchange server requires **Enterprise Admins**. If the Exchange organization/domain was already prepared, the installing account needs the **Organization Management** role group. A delegated setup account can install only when an Exchange administrator has provisioned the server for it.
- Avoid leaving installation accounts in elevated groups longer than your organization's policy requires.

## 3. Prepare the environment

1. Give the server its final name, configure DNS, and join it to the domain.
2. Install Windows updates and reboot until no restart is pending.
3. Confirm the server's AD site has a writable Global Catalog and healthy replication.
4. Download and verify the Exchange 2019 CU15 media and the applicable security update from Microsoft.
5. Mount the ISO, note its drive letter (the examples below use **E:**), and stage prerequisite installers locally if the server has no internet access.
6. Decide the Exchange organization name before setup. The name is internal, limited to 64 characters, and cannot be changed after the organization is created.
7. Plan the mail, client access, certificate, firewall, and storage design before creating production mailboxes.

## 4. Install with the graphical Setup wizard

Use this path on Windows Server with Desktop Experience.

1. Sign in with an account that has the required permissions. In File Explorer, right-click the CU15 ISO and choose **Mount**.
2. Open the mounted drive and right-click **Setup.exe** → **Run as administrator**.
3. On **Check for Updates**, choose **Connect to the Internet and check for updates** when the server can reach Microsoft Update. This checks updates for the selected CU; it does not substitute a newer CU. If the server is offline, choose **Don't check for updates right now** and apply the approved update package later.
4. Continue through the welcome and license pages. Review and accept the license terms.
5. On **Recommended Settings**, choose whether to enable error reporting and usage feedback according to organizational policy.
6. On **Server Role Selection**, select **Mailbox role**. Select **Automatically install Windows Server roles and features that are required to install Exchange**. Setup adds the required Windows components; it does not install all software prerequisites, so install .NET and the Visual C++ packages separately.
7. On **Installation Space and Location**, keep the default path or choose a prepared local drive with sufficient space. Do not use a network, removable, or root-of-drive location.
8. If this is the first Exchange server and Active Directory is not prepared, Setup may show **Exchange Organization**. Enter the planned organization name. The setup account needs the required AD rights, and the server must meet the AD site/schema-master requirements. In a multi-site forest, prepare AD separately as described in section 6.
9. On **Malware Protection Settings**, leave malware scanning enabled unless your security design explicitly requires otherwise.
10. On **Readiness Checks**, resolve every error. Read warnings and confirm they are acceptable for this topology. Select **Retry** after correcting issues.
11. When the readiness check passes, select **Install** and allow Setup to complete. Do not rename the server or interrupt the setup process.
12. Select **Finish**, restart Windows, then install the newest applicable Exchange security update before opening client or internet access.

## 5. Install with the command line

Use an elevated **Command Prompt** on the Exchange server. These examples use **E:** for the mounted ISO and **dc01.contoso.com** for the domain controller. Replace them with your actual values. The switches shown are for an unattended install; the diagnostic-data choice must be accepted as part of Setup.

### 5.1 Prepare Active Directory

For a new Exchange organization, run the following steps in order. You can run them from a domain-joined server with the Exchange media and required permissions; it does not have to be the Exchange server.

**Extend the schema.** The specified domain controller must hold the Schema Master FSMO role:

~~~cmd
E:\Setup.exe /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF /PrepareSchema /DomainController:dc01.contoso.com
~~~

Wait for schema changes to replicate across the forest before continuing. Do not run the next command while replication is still in progress.

**Prepare the Exchange organization:**

~~~cmd
E:\Setup.exe /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF /PrepareAD /OrganizationName:"Contoso" /DomainController:dc01.contoso.com
~~~

For an existing Exchange organization, do not supply a new organization name. Follow the instructions for the CU and your existing topology.

**Prepare additional domains only when needed.** For a single-domain forest, **/PrepareAD** prepares the domain, so this step is not needed. In a multi-domain forest, prepare every domain that will contain an Exchange server or mail-enabled users. Either prepare all domains:

~~~cmd
E:\Setup.exe /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF /PrepareAllDomains /DomainController:dc01.contoso.com
~~~

Or prepare one specific domain at a time:

~~~cmd
E:\Setup.exe /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF /PrepareDomain:child.contoso.com /DomainController:dc01.contoso.com
~~~

Wait for AD replication after each preparation stage. Use a domain controller appropriate to the operation; for **/PrepareSchema** and schema-extending **/PrepareAD**, specify the Schema Master. The full permissions and site requirements are in [Prepare Active Directory and domains for Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/prepare-ad-and-domains?view=exchserver-2019).

> For a hybrid organization, Microsoft requires additional tenant configuration for AD preparation. Follow the official hybrid-specific procedure rather than using the simple commands above unchanged.

### 5.2 Install the Mailbox role

On the target Exchange server, open elevated Command Prompt, mount the CU15 media, then run:

~~~cmd
E:\Setup.exe /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF /Mode:Install /Roles:Mailbox /InstallWindowsComponents /DomainController:dc01.contoso.com
~~~

**/InstallWindowsComponents** tells Setup to add the Windows features required for the Mailbox role. Setup also performs readiness checks. If it returns an error, correct the reported prerequisite or AD issue and rerun Setup. After a successful install, reboot and install the latest applicable Exchange security update.

If your organization chooses to send diagnostic data, replace **DiagnosticDataOFF** with **DiagnosticDataON**. One of these switches is required for unattended commands on the CU versions covered by this guide.

## 6. Optional: graphical Active Directory preparation

The GUI path for AD preparation is the Exchange Setup wizard during the first-server installation. In a straightforward environment, Setup can extend the schema and prepare the Exchange organization and domain if the setup account has the required rights. The wizard cannot prepare AD when the Exchange server is in a different AD site from the Schema Master. If an AD team controls schema changes, the forest has multiple sites, or you need to stage permissions and replication separately, use section 5.1 from a domain-joined computer in the Schema Master's site. Microsoft's prerequisite guidance says a separate computer used for GUI preparation needs the Exchange Management Tools role and its prerequisites; see the official preparation page for the exact procedure.

## 7. Reference architecture: identity, DNS, network, and mail

This section shows how the Exchange server relates to Active Directory, internal users, the public email domain, external clients, and internet mail. The names and IP addresses below are examples only.

### 7.1 Keep these namespaces distinct

| Name or concept | Example | What it is for |
|---|---|---|
| Active Directory DNS domain | ad.contoso.com | Internal AD domain and DNS service discovery. It does not need to be the public email domain. Keep domain controllers and AD locator records private. |
| User sign-in name (UPN) | alex@contoso.com | Identity used to sign in. Microsoft recommends matching the UPN suffix to the user's primary email address where practical. |
| SMTP/accepted domain | contoso.com | Public email address domain. Add each domain that Exchange should receive mail for as an accepted domain. |
| Client access namespace | mail.contoso.com | HTTPS endpoint used for Outlook, OWA, EAC, Autodiscover targets, and Exchange web services. |
| Autodiscover name | autodiscover.contoso.com | Outlook's discovery endpoint. Publish it for every accepted SMTP domain used by clients, using CNAME or SRV as appropriate. |
| Optional OWA name | owa.contoso.com | Optional friendly browser URL. If used, include it in DNS, Exchange virtual directory URLs, and the TLS certificate. |
| Inbound mail host | mx.contoso.com or a provider's gateway host | Public MX target. This can be a filtering service or Edge Transport host rather than the Mailbox server. |

A public email domain is a mail-routing namespace; it does not authenticate users. User passwords are checked against Active Directory Domain Services (AD DS), or an explicitly configured identity provider such as AD FS or Microsoft Entra ID in a hybrid design. Do not publish domain controller, LDAP, Kerberos, or Global Catalog records in public DNS.

### 7.2 Logical architecture

The diagram shows a straightforward one-server deployment with an optional perimeter mail gateway and web reverse proxy. A gateway filters or relays SMTP; a web reverse proxy protects HTTPS client access. These are separate functions and may be supplied by different systems.

~~~mermaid
flowchart LR
    subgraph PUBLIC["Public internet"]
        EXTCLIENT["Remote Outlook / browser / mobile"]
        EXTMAIL["External mail servers"]
        PUBDNS["Public DNS<br/>MX, A/CNAME, TXT"]
    end

    subgraph DMZ["Perimeter network (optional but recommended)"]
        WEBPROXY["HTTPS reverse proxy / WAF<br/>TCP 443"]
        SMTPEDGE["SMTP security gateway or<br/>Exchange Edge Transport<br/>TCP 25"]
    end

    subgraph LAN["Internal network"]
        INTCLIENT["Domain-joined Outlook clients"]
        INTDNS["Internal DNS"]
        AD["AD DS / writable Global Catalog<br/>Internal only"]
        MBX["Exchange 2019 Mailbox server<br/>Client Access + Transport + Mailbox"]
        DB["Mailbox databases and logs"]
    end

    EXTCLIENT -->|"DNS lookup"| PUBDNS
    PUBDNS -->|"returns public HTTPS VIP"| EXTCLIENT
    EXTCLIENT -->|"HTTPS 443"| WEBPROXY
    WEBPROXY -->|"HTTPS 443"| MBX
    EXTCLIENT -.->|"Direct HTTPS if no proxy"| MBX

    EXTMAIL -->|"MX lookup"| PUBDNS
    PUBDNS -->|"returns MX hostname/IP"| EXTMAIL
    EXTMAIL -->|"SMTP 25"| SMTPEDGE
    SMTPEDGE -->|"SMTP 25"| MBX
    EXTMAIL -.->|"Direct SMTP 25 if no gateway"| MBX
    MBX -->|"Send connector: SMTP 25"| SMTPEDGE
    SMTPEDGE -->|"SMTP 25"| EXTMAIL
    MBX -.->|"Direct SMTP 25 if no gateway"| EXTMAIL

    INTCLIENT -->|"DNS and Autodiscover SCP lookup"| INTDNS
    INTDNS -->|"mail/autodiscover resolve to private VIP"| MBX
    INTCLIENT -->|"HTTPS 443 / MAPI over HTTP"| MBX
    INTCLIENT -->|"Kerberos/LDAP as required"| AD
    MBX <-->|"AD, DNS, authentication and directory lookups"| AD
    MBX -->|"Transport delivery"| DB
    DB -->|"Mailbox access"| MBX
~~~

For a very small lab, the optional gateway and reverse proxy can be omitted, but the Mailbox server then has a larger internet-facing attack surface and is a single point of failure. Do not expose AD DS to the internet. If using an Edge Transport server, it handles SMTP mail flow; it does not replace the HTTPS reverse proxy for Outlook/OWA.

### 7.3 DNS record plan

Create records in the correct zone. The internal and public answers may differ while clients use the same URLs; this is split DNS. Microsoft recommends using the same client URL internally and externally for a new namespace plan where possible.

Before publishing DNS, make sure the organization controls the registered public domain and its authoritative DNS zone, has a static public IP or provider-managed VIP, has a working firewall/NAT path to the intended gateway or Exchange endpoint, and has selected certificate names. Confirm that the ISP/hosting provider permits the intended outbound SMTP route. DNS records only publish names and routes; they do not create a firewall rule, Exchange connector, accepted domain, or certificate.

#### Internal DNS

| Record | Example target | Purpose |
|---|---|---|
| A record for EX01.ad.contoso.com | Exchange server private IP | Resolves the server's internal computer FQDN for domain services and administration.
| A record for mail.contoso.com | Private load balancer/VIP, or the Mailbox server's private IP for a one-server lab | Gives internal clients the private HTTPS endpoint. |
| CNAME or A record for autodiscover.contoso.com | CNAME to mail.contoso.com, or the same private VIP | Resolves internal Autodiscover for clients that use DNS. |
| Optional owa.contoso.com | CNAME to mail.contoso.com or same private VIP | Supports a separate OWA browser URL if configured on the virtual directory and certificate. |
| AD locator records | Normally registered and maintained by domain controllers in the internal AD DNS zone | Lets domain members find LDAP, Kerberos, and Global Catalog services. Do not create these manually or copy them into public DNS. |

Domain-joined Outlook first discovers Exchange using the Autodiscover Service Connection Point (SCP) in Active Directory, then uses HTTPS. Configure the SCP URL to a name that resolves internally and appears on the certificate. If you host a split copy of a public DNS zone internally, make sure it contains all internal records clients need so the private zone does not unintentionally shadow public services. DNS Autodiscover records are still needed for clients and scenarios that use DNS-based discovery. For multiple accepted SMTP domains, publish the appropriate Autodiscover alias or SRV record for each client email domain.

In Exchange Management Shell, inspect the current SCP and set the internal Autodiscover URL to the chosen HTTPS name. Do this only after the name resolves internally and is included on the certificate:

~~~powershell
Get-ClientAccessService | Format-List Name,AutoDiscoverServiceInternalUri
Set-ClientAccessService -Identity EX01 -AutoDiscoverServiceInternalUri "https://autodiscover.contoso.com/Autodiscover/Autodiscover.xml"
~~~

For a multi-server or multi-site organization, configure the correct URL and site scope for each Client Access service; do not blindly copy the single-server example.

#### Public DNS

| Record | Example | Purpose and notes |
|---|---|---|
| MX for contoso.com | Priority 10 → mx.contoso.com | Directs inbound email to the public SMTP gateway/Edge host. If there is no gateway, publish mx.contoso.com to the SMTP endpoint/NAT that forwards TCP 25 to Exchange. Use mail.contoso.com as the MX target only if that endpoint also accepts SMTP. The MX target must be a hostname with an A/AAAA address record, not a CNAME. |
| A for mx.contoso.com | Public IP of SMTP gateway/Edge or mail endpoint | Publish this only when the MX target is a hostname in your DNS zone. If using a provider gateway, use the provider-specified MX target. Never publish a private RFC1918 address. |
| A for mail.contoso.com | Public HTTPS VIP / reverse-proxy address | Publishes the external client-access endpoint. If HTTPS and SMTP use different systems, use different hostnames and records. |
| CNAME for autodiscover.contoso.com | mail.contoso.com | Points Outlook discovery to the client-access namespace. An SRV record on _autodiscover._tcp is an alternative when required by the DNS design. |
| Optional CNAME for owa.contoso.com | mail.contoso.com | Use only if this separate OWA name is configured in Exchange and present on the certificate. |
| SPF TXT for contoso.com | A policy listing every authorized outbound sender | Publish one SPF record per domain and include all actual senders, including any relay, bulk sender, or cloud service. Do not copy a generic example without inventorying senders. |
| DKIM selector TXT | Supplied by the system that signs outbound messages | Publish the selector/key record from the chosen signing gateway or service. Configure the signer as well as DNS. |
| DMARC TXT at _dmarc.contoso.com | A policy selected by the mail/domain owner | Start with reporting/monitoring if the sender inventory is incomplete; tighten enforcement only after legitimate sources pass alignment. |
| Optional MTA-STS TXT at _mta-sts.contoso.com | Version and policy ID; policy file hosted at https://mta-sts.contoso.com/.well-known/mta-sts.txt | Publish only when the HTTPS policy host, certificate, and MX targets are configured and maintained. |
| Optional SMTP TLS reporting TXT at _smtp._tls.contoso.com | Reporting URI managed by the domain owner | Publish only when the report mailbox or HTTPS endpoint is monitored. This provides reports; it does not configure Exchange TLS by itself. |
| PTR for outbound public IP | Set by the ISP or hosting provider | Configure reverse DNS for the public SMTP egress IP. Align it with the intended mail hostname and SMTP identity where possible. |

MTA-STS and SMTP TLS reporting are optional transport-security enhancements, not Exchange installation prerequisites. They require a maintained HTTPS policy endpoint and reporting destination. A DNS CNAME changes name resolution; it does not change the URL in the browser or the name checked on a TLS certificate. The certificate must cover every hostname clients actually request, commonly mail.contoso.com and autodiscover.contoso.com, plus owa.contoso.com if used. Exchange Setup does not create public DNS, MX, SPF, DKIM, or DMARC records for you.

Exchange initially configures the forest-root Active Directory FQDN as an accepted domain; do not assume the public email domain is already accepted. In EAC, go to **Mail flow > Accepted domains** and add each public SMTP domain. Confirm the exact domain value before creating it; the domain value cannot be edited later.

Choose the domain type based on where recipients live:
- **Authoritative:** all recipients for the domain exist in this Exchange organization.
- **Internal Relay:** the domain is shared with another internal mail system or Exchange organization; configure the routing connector for recipients not hosted here.
- **External Relay:** use only for a specifically designed external relay scenario.

Assign the domain to recipients with an email address policy or mailbox settings. Then publish its MX and sender-authentication records, and an Autodiscover record for autodiscover.contoso.net if users sign in with that email domain.

In Exchange Management Shell, inspect domains and, only if the domain is not already configured, create an authoritative domain for a domain whose recipients are all hosted here:

~~~powershell
Get-AcceptedDomain | Format-Table -Auto Name,DomainName,DomainType,Default
New-AcceptedDomain -Name "Contoso Public Email" -DomainName contoso.com -DomainType Authoritative
~~~

Do not use the sample command for a shared or hybrid mail-routing design; choose the accepted-domain type and connectors for the actual recipient locations. For a new Exchange organization, the first accepted domain may be the AD forest root, which is not necessarily the public email domain.

#### DNS validation examples

Run checks from an internal network and from outside the organization, using the appropriate DNS resolver. Replace the sample internal DNS server address and names with your own:

~~~powershell
Resolve-DnsName mail.contoso.com -Server 10.10.0.10
Resolve-DnsName autodiscover.contoso.com -Server 10.10.0.10
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.ad.contoso.com -Server 10.10.0.10
~~~

~~~cmd
nslookup -type=MX contoso.com 1.1.1.1
nslookup mail.contoso.com 1.1.1.1
nslookup autodiscover.contoso.com 1.1.1.1
nslookup -type=TXT contoso.com 1.1.1.1
nslookup -type=TXT _dmarc.contoso.com 1.1.1.1
nslookup -type=TXT _mta-sts.contoso.com 1.1.1.1
nslookup -type=TXT _smtp._tls.contoso.com 1.1.1.1
~~~

Use the actual public resolver available to you. Confirm that the MX answer points to the intended SMTP receiver, the HTTPS names point to the intended public VIP, and internal names resolve to private addresses. Check each additional accepted email domain separately.

### 7.4 Network and firewall requirements

Use this as a planning summary, not a complete Exchange firewall rule set. Ports depend on enabled protocols and topology. Exchange-to-AD and Exchange-to-Exchange traffic must not be arbitrarily filtered.

| Source → destination | Port/protocol | Requirement |
|---|---|---|
| Internal or external client → Exchange client-access VIP | TCP 443 HTTPS | Required for Outlook MAPI over HTTP, Autodiscover, OWA/EAC, EWS, ActiveSync, and OAB. If a reverse proxy is used, allow its supported HTTPS path to Exchange. |
| External sending MTA → SMTP gateway/Edge or Mailbox server | TCP 25 SMTP | Required for inbound server-to-server mail. Point the MX record at the actual receiving service. |
| Exchange Send Connector → SMTP gateway or recipient mail servers | TCP 25 SMTP | Required for outbound server-to-server delivery. A firewall/provider may block direct outbound TCP 25; use an approved smart host if so. |
| Approved device/client → Exchange | TCP 587 SMTP submission | Optional. Use only when authenticated client submission is intentionally deployed; require TLS and restrict who can submit. Do not treat port 25 as a general user submission service. |
| Exchange → internal DNS resolvers | TCP/UDP 53 | Required for AD and mail-routing name resolution. Exchange transport also needs DNS to find external next hops when using DNS routing. |
| Exchange ↔ domain controllers / Global Catalogs | Unrestricted internal traffic; common AD ports include TCP/UDP 53, 88, 389, and 464; TCP 135, 445, 3268, and dynamic RPC 49152–65535. TCP 636/3269 only if LDAPS is deliberately used. | Required for directory, authentication, policy, and setup operations. Microsoft says internal Exchange-to-AD server traffic must not be restricted or altered. The listed ports are examples, not a complete strict allowlist. |
| Exchange ↔ other Exchange servers | Any required ports and protocols, including dynamic RPC | Microsoft requires unrestricted internal Exchange-to-Exchange communication; do not filter or alter this traffic. This matters for multiple servers, DAGs, and backend proxying. |
| Mailbox servers ↔ subscribed Edge Transport servers | TCP 25 SMTP; TCP 50636 secure LDAP for EdgeSync | Required when Edge Transport is used. Keep EdgeSync limited to the trusted internal path. |
| Client → Exchange on TCP 80 | TCP 80 HTTP | Avoid where possible. Some redirect/fallback scenarios use it; prefer HTTPS 443 for credentials and mailbox traffic. |
| Client → POP/IMAP | TCP 110/995 POP; 143/993 IMAP | Not enabled by default. Open only if these protocols are explicitly required and configured securely. |

Do not port-forward LDAP, Kerberos, SMB, RPC, or Global Catalog ports from the internet to domain controllers. See Microsoft's [Exchange network port reference](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/network-ports) and the [Windows Server network port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements) for topology-specific firewall planning.

### 7.5 Mail flow

Exchange creates the Receive connectors required for inbound mail during installation. The default Front End Receive connector accepts anonymous SMTP from external mail servers on TCP 25; it is for server-to-server delivery and must not become an open relay. Exchange does **not** create an internet Send connector by default. Create one for direct DNS/MX delivery or configure a smart host/security gateway.

~~~mermaid
sequenceDiagram
    participant Inbound as External sending mail server
    participant DNS as Public DNS
    participant Gateway as SMTP gateway / Edge (optional)
    participant FE as Exchange Front End Transport
    participant Transport as Exchange Transport
    participant DB as Recipient mailbox database
    participant User as Internal user
    participant Outbound as External recipient mail server

    Inbound->>DNS: Query recipient domain MX
    DNS-->>Inbound: MX target and public address
    alt Gateway or Edge is deployed
        Inbound->>Gateway: SMTP 25
        Gateway->>FE: SMTP 25
    else Direct inbound to Exchange
        Inbound->>FE: SMTP 25
    end
    FE->>Transport: Pass message through Exchange transport pipeline
    Transport->>DB: Deliver to accepted recipient
    DB-->>User: Mailbox available

    User->>FE: Submit over HTTPS / MAPI over HTTP
    FE->>Transport: Submit to Exchange transport


    alt Smart host or gateway routing
        Transport->>Gateway: SMTP 25 to configured next hop
        Gateway->>DNS: Resolve recipient domain if gateway routes by DNS
        DNS-->>Gateway: Recipient mail host
        Gateway->>Outbound: SMTP 25
    else Direct DNS routing
        Transport->>DNS: Resolve recipient domain MX
        DNS-->>Transport: Recipient mail host
        Transport->>Outbound: SMTP 25
    end
~~~

Internal messages are routed inside the Exchange organization; they do not need public MX lookup.

~~~mermaid
sequenceDiagram
    participant A as Internal sender
    participant FE as Exchange Client Access
    participant T as Exchange Transport
    participant AD as AD DS
    participant DB as Recipient mailbox database
    participant B as Internal recipient

    A->>FE: Submit over HTTPS / MAPI over HTTP
    FE->>AD: Resolve sender and mailbox directory data
    AD-->>FE: Directory result
    FE->>T: Submit message
    T->>AD: Resolve recipient and routing
    AD-->>T: Recipient mailbox location
    T->>DB: Deliver inside Exchange organization
    DB-->>B: Message available in mailbox
~~~

To receive internet mail, the recipient domain must be accepted by Exchange and its public MX must point to the chosen inbound service. To send mail, create an internet Send connector or subscribe an Edge Transport server. Decide whether direct outbound delivery or a smart host is appropriate for the ISP, filtering, logging, and IP-reputation requirements.

### 7.6 Authentication and identity

Authentication is separate from public DNS and SMTP routing. The public domain directs clients and mail to services; it does not store Exchange user passwords.

| Connection | Typical authentication model | Important note |
|---|---|---|
| Domain-joined Outlook inside the network | Outlook uses Autodiscover, including the AD SCP; Exchange client access uses the configured Windows authentication method, commonly Kerberos/Negotiate or NTLM, with AD DS validating the account | Client and server need working internal DNS, time, domain connectivity, and a valid HTTPS certificate. |
| OWA/EAC over HTTPS | By default, OWA and EAC use Forms-Based Authentication (FBA); Microsoft notes that FBA relies on Basic authentication | Protect the connection with TLS. OWA and EAC should use consistent authentication settings. Restrict EAC administration to trusted admin networks/VPN or an access-controlled proxy; avoid broad public EAC exposure. |
| Mobile/remote Outlook or other HTTPS clients | HTTPS to the client-access namespace, then the authentication method enabled for the relevant virtual directory/protocol | Basic authentication is not the same as Modern Authentication. Do not disable an authentication method until a working replacement is configured and all affected clients are tested. |
| Internet mail server delivering to TCP 25 | Anonymous server-to-server SMTP delivery to the Front End Receive connector, subject to recipient/domain and mail-flow controls | This is not an end-user login and must not grant anonymous relay to arbitrary external recipients. Use a gateway, anti-spam controls, TLS policy, and logging as appropriate. |
| Internal application/device submitting mail | Authenticated SMTP submission on TCP 587 with TLS, or a tightly scoped relay design | Prefer dedicated service identity and source-IP restrictions. Do not grant broad relay rights or expose a relay connector to the internet. |
| Modern Authentication | Exchange 2019 CU13+ can use OAuth Modern Authentication with AD FS in a pure on-premises design; Exchange hybrid uses Hybrid Modern Authentication (HMA) with Microsoft Entra ID | This is an additional identity architecture, not enabled just by installing Exchange. Plan its prerequisites, supported clients, certificates, and MFA policy. HMA is Microsoft's recommended modern-auth path for hybrid organizations. |

~~~mermaid
sequenceDiagram
    participant Client as Internal Outlook
    participant AD as AD DS
    participant DNS as Internal DNS
    participant Exchange as Exchange Client Access
    participant Mailbox as Mailbox database

    Client->>AD: Read Autodiscover SCP / obtain domain identity
    Client->>DNS: Resolve mail and Autodiscover names
    DNS-->>Client: Private Exchange VIP
    Client->>Exchange: HTTPS 443 / MAPI over HTTP
    Exchange->>AD: Validate identity and query directory
    AD-->>Exchange: User/group/mailbox directory data
    Exchange->>Mailbox: Proxy request to mailbox services
    Mailbox-->>Client: Mailbox data over HTTPS
~~~

For an external Outlook/OWA client, public DNS resolves the client namespace to the public HTTPS VIP. The client connects over TLS through the approved reverse proxy or directly to the internet-facing Exchange endpoint. With default OWA FBA, the user submits credentials over TLS and Exchange validates the account against AD DS. With Modern Authentication, the client is redirected to the configured identity provider and Exchange validates a token. In either design, expose only the intended HTTPS service and do not expose AD DS to the client network.

~~~mermaid
sequenceDiagram
    participant User as External user
    participant DNS as Public DNS
    participant Proxy as Reverse proxy / WAF
    participant EX as Exchange Client Access
    participant IDP as AD FS or Entra ID
    participant AD as AD DS
    participant DB as Mailbox database

    User->>DNS: Resolve mail.contoso.com
    DNS-->>User: Public HTTPS VIP
    User->>Proxy: HTTPS 443
    Proxy->>EX: HTTPS 443

    alt Default OWA Forms authentication
        User->>Proxy: Submit credentials in HTTPS request
        Proxy->>EX: Forward request over HTTPS
        EX->>AD: Validate account
        AD-->>EX: Authentication result
    else Modern Authentication is configured
        EX-->>User: Redirect to identity provider
        User->>IDP: Authenticate (MFA if policy requires)
        IDP-->>User: Return OAuth token
        User->>Proxy: HTTPS request with token
        Proxy->>EX: Forward request over HTTPS
        EX->>EX: Validate token signature, issuer, and audience
        
        EX->>AD: Resolve mailbox and directory data
        AD-->>EX: Directory result
    end

    EX->>DB: Access the user's mailbox
    DB-->>User: Mailbox content over HTTPS
~~~

For a pure on-premises Modern Authentication design, Microsoft documents AD FS on Windows Server 2019 or later and Exchange 2019 CU13 or later; do not install AD FS on the Exchange server. For hybrid deployments, use the Hybrid Configuration Wizard and HMA guidance for Entra ID rather than copying the standalone AD FS steps.

### 7.7 Availability and production design

- The installation procedure above creates one Mailbox server. One server has no Exchange server failover and is a single point of failure.
- For production availability, design at least two Mailbox servers and a Database Availability Group (DAG), redundant AD DS/DNS/Global Catalog services, and an HTTPS load balancer or reverse-proxy tier. Use a supported design and validate quorum, witness placement, storage, backup, and recovery requirements.
- Use a dedicated perimeter SMTP gateway or Edge Transport tier when the organization needs isolation, filtering, or controlled SMTP routing. Keep it separate from domain controllers and do not confuse it with the HTTPS client-access proxy.
- A hybrid design adds Microsoft 365/Exchange Online, Entra ID synchronization, Hybrid Configuration Wizard connectors, and possibly HMA. The MX target depends on whether on-premises Exchange or Exchange Online is the front door for inbound mail. Use the hybrid deployment guide for the intended routing pattern.

## 8. Post-install configuration and verification

Installation alone does not complete mail service. Finish the following tasks before users rely on the server.

### Verify Setup and services

Open **Exchange Management Shell** as an Exchange administrator and run:

~~~powershell
Get-ExchangeServer | Format-List Name,Edition,AdminDisplayVersion
Test-ServiceHealth
Get-MailboxDatabase -Status | Format-Table Name,Server,Mounted
Get-EcpVirtualDirectory | Format-List Identity,InternalUrl,ExternalUrl
Get-OwaVirtualDirectory | Format-List Identity,InternalUrl,ExternalUrl
~~~

Confirm that the new server appears with the expected version, required services are healthy, and the initial mailbox database is mounted. Review **C:\ExchangeSetupLogs\ExchangeSetup.log** if setup reports a failure or the version is unexpected. Microsoft's [Verify Exchange Server installations](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/verify-installation?view=exchserver-2019) page describes the setup-log checks.

### Configure client access and certificates

- Choose the production namespace, for example **mail.contoso.com**, and configure internal/external URLs for the client protocols you use. Configure the EAC/ECP URL for the intended admin access path and restrict administration to trusted networks, VPN, or an access-controlled proxy; do not assume a public URL should be generally accessible.
- Obtain a certificate whose subject names cover the namespaces clients use. Import and assign it through Exchange Admin Center or Exchange Management Shell. **Do not bind the Exchange certificate manually in IIS Manager**; Exchange manages its IIS bindings.
- Configure internal/external DNS, load balancer or reverse proxy, and firewall/NAT rules to match the design. TCP 443 is normally required for HTTPS client access; TCP 25 is normally used for SMTP mail flow. Open TCP 587 only if authenticated client submission is intentionally configured. Other ports depend on the topology and enabled protocols.
- Configure accepted domains, send/receive connectors, outbound routing or smart host, MX records, and spam/security controls as required. A valid certificate and open ports alone do not configure mail flow.

### Test access and mail flow

- From an authorized workstation, open **https://mail.contoso.com/ecp** and **https://mail.contoso.com/owa** in a current supported browser.
- Create or use a test mailbox and verify internal mail flow. Then test inbound and outbound internet mail if the server is intended to handle internet mail.
- Confirm client name resolution and certificate trust from representative client networks.

## 9. Common corrections to the supplied notes

- The supplied notes specify **CU11**. This guide uses the final CU line, **CU15**, and requires checking current updates before deployment.
- The 128 GB memory figure is a Microsoft minimum recommendation for the Mailbox role, not a universal deployment sizing answer. Size for workload and mailbox/database growth.
- The source listed only one Visual C++ package. The Mailbox prerequisites include both the 2012 and 2013 redistributables; follow Microsoft's current prerequisite page.
- **/PrepareAllDomains** is not required in every forest. **/PrepareAD** prepares the current domain; use the all-domain or targeted-domain command only when additional domains need preparation.
- Schema preparation must target the Schema Master when a domain controller is specified, and AD replication must complete before later steps.
- Do not manually adjust Exchange IIS application pools or bind Exchange's certificate directly in IIS as generic post-install steps. Configure Exchange URLs and certificates with Exchange tools.
- The browser note about an Internet Explorer icon is obsolete; use a currently supported browser.

## 10. Microsoft references

- [Exchange Server updates and CU downloads](https://learn.microsoft.com/en-us/exchange/new-features/updates)
- [Exchange Server build numbers and security updates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
- [Exchange Server 2019 end-of-support roadmap](https://learn.microsoft.com/en-us/troubleshoot/exchange/administration/exchange-2019-2016-end-of-support)
- [Exchange Server prerequisites](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/prerequisites?view=exchserver-2019)
- [Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements?view=exchserver-se)
- [Exchange Server supportability matrix](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/supportability-matrix)
- [Prepare Active Directory and domains](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/prepare-ad-and-domains?view=exchserver-2019)
- [Install Mailbox servers with the Setup wizard](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deploy-new-installations/install-mailbox-role)
- [Unattended Exchange Setup](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deploy-new-installations/unattended-installs?view=exchserver-2019)
- [Verify an Exchange installation](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/verify-installation?view=exchserver-2019)
- [Upgrade to Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deploy-new-installations/upgrade-to-exchange-server-se)
- [Configure mail flow and client access](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/configure-mail-flow-and-client-access)
- [Autodiscover service and DNS](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains?view=exchserver-se)
- [Set-ClientAccessService PowerShell reference](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-clientaccessservice?view=exchange-ps)
- [Exchange network port reference](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/network-ports)
- [Receive connectors](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/receive-connectors?view=exchserver-se)
- [Send connectors](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/send-connectors?view=exchserver-2019)
- [Internet Send connector setup](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/internet-mail-send-connectors)
- [Exchange on-premises Modern Authentication](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enable-modern-auth-in-exchange-server-on-premises)
- [Disable Basic authentication on Exchange virtual directories](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/disable-basic-authentication-on-exchange-server-virtual-directories)
- [Windows Server network port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements)
- [MTA-STS specification (RFC 8461)](https://www.rfc-editor.org/rfc/rfc8461.html)
- [SMTP TLS Reporting specification (RFC 8460)](https://www.rfc-editor.org/rfc/rfc8460.html)














