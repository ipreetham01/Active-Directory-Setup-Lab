# Active Directory Lab: Domain Controller, DNS, Users, Groups and Group Policy

## Overview

This project documents a Windows Active Directory lab built in **VMware Workstation**. A Windows Server 2022 virtual machine is promoted to a Domain Controller for a new forest (`preetham.local`). Organizational Units (OUs), users and a security group are created, a Group Policy Object (GPO) is linked to the IT OU, and a Windows 10 client is joined to the domain to verify authentication and policy delivery.

## Objectives

- Build an isolated host-only network in VMware Workstation
- Install AD DS and promote a server to a Domain Controller
- Create OUs, users and a security group
- Create and link a Group Policy Object
- Join a Windows 10 client to the domain
- Verify domain logon, SYSVOL access and Group Policy application

## Lab Environment

| Component | Configuration |
|---|---|
| Virtualization platform | VMware Workstation |
| Domain Controller OS | Windows Server 2022 Standard (Evaluation) |
| Client OS | Windows 10 Education 22H2 |
| Domain | `preetham.local` (NetBIOS: `PREETHAM`) |
| Domain Controller | `DC01` at `192.168.175.10` |
| Client | `192.168.175.20` |
| Network | VMnet1 (Host-only), `192.168.175.0/24` |
| DNS server (both machines) | `192.168.175.10` |

---

## 1. Virtual Network Setup

VMnet1 is a host-only network on `192.168.175.0/24`. Both VMs use it so the lab is isolated.

![VMnet1 host-only network in VMware Virtual Network Editor](<img width="693" height="697" alt="01-vmnet1-hostonly" src="https://github.com/user-attachments/assets/902253dc-a668-47c4-bb01-6b58242710d3" />
)

The Domain Controller uses a static IP and points to itself for DNS, because it hosts the domain's DNS zone. No gateway is set since the network is isolated.

![Static IP configuration on the Domain Controller](screenshots/02-dc-static-ip.png)

## 2. Installing AD DS and Promoting the Server

1. In Server Manager, select **Manage > Add Roles and Features**.
2. Select **Active Directory Domain Services** and install it.

![AD DS installation confirmation](screenshots/03-adds-install.png)

3. Click **Promote this server to a domain controller**, choose **Add a new forest**, and enter `preetham.local`.
4. Set a DSRM password and complete the wizard. A DNS delegation warning appears; this is expected in an isolated `.local` lab.
5. After the restart, the logon screen shows `PREETHAM\Administrator`.

![Domain logon screen after promotion](screenshots/04-domain-login.png)

## 3. Verifying DNS

The `preetham.local` zone contains the `dc01` host record (`192.168.175.10`) and the `_msdcs`, `_sites`, `_tcp` and `_udp` folders that AD depends on.

![DNS forward lookup zone for preetham.local](screenshots/05-dns-zone-records.png)

## 4. Organizational Units, Users and Groups

I created the **IT**, **HR** and **Workstations** OUs.

![OUs created under preetham.local](screenshots/06-ous-created.png)

Two test users were created: **Alex Sharma** (`alex.sharma`) in IT and **Priya Patel** (`priya.patel`) in HR, both set to change their password at next logon. A global security group, **IT-Support**, was created in the IT OU.

![IT OU containing the user and security group](screenshots/07-it-ou-user-and-group.png)

## 5. Group Policy

In Group Policy Management, I right-clicked the **IT** OU and selected **Create a GPO in this domain, and Link it here**, naming it **IT-Workstation-Policy**.

![Creating and linking the GPO to the IT OU](screenshots/08-create-and-link-gpo.png)

Under **User Configuration > Policies > Administrative Templates > Control Panel > Personalization**, I enabled **Prevent changing desktop background**.

![Prevent changing desktop background set to Enabled](screenshots/09-gpo-setting-enabled.png)

> This is a user policy, so it applies to users in the IT OU. Microsoft's help text notes that "Desktop Wallpaper" must also be enabled to fully lock the wallpaper. This lab verifies that the GPO is delivered and applied, shown in Section 7.

## 6. Windows 10 Client and Domain Join

The client uses a static IP and the Domain Controller as its DNS server. This is what lets it find the domain.

| Setting | Value |
|---|---|
| IP address | `192.168.175.20` |
| Subnet mask | `255.255.255.0` |
| Preferred DNS server | `192.168.175.10` |

![Client static IP and DNS settings](screenshots/10-client-static-ip.png)

Connectivity was confirmed with `ping 192.168.175.10`, and `nslookup preetham.local` resolved to the Domain Controller. The client was then joined using `sysdm.cpl`.

![Joining the client to preetham.local](screenshots/11-join-domain.png)

After the restart, I signed in as `PREETHAM\alex.sharma`. `whoami` confirms the domain account and `%LOGONSERVER%` shows `DC01` authenticated the logon.

![whoami and LOGONSERVER on the client](screenshots/12-whoami-logonserver.png)

## 7. Verification

`nltest /dsgetdc:preetham.local` found `DC01.preetham.local`, and the `SYSVOL` share is reachable at `\\DC01\SYSVOL`.

![nltest and SYSVOL access](screenshots/13-nltest-sysvol.png)

`gpresult /r` shows **IT-Workstation-Policy** under Applied Group Policy Objects for `CN=Alex Sharma,OU=IT,DC=preetham,DC=local`.

![gpresult showing IT-Workstation-Policy applied](screenshots/14-gpresult-applied.png)

## 8. Troubleshooting

**`nslookup` shows "DNS request timed out" and "Server: UnKnown".** `nslookup` first does a reverse lookup of the DNS server's IP. No reverse zone exists in this lab, so it prints the timeout. The forward lookup still returns the correct answer.

**DNS returned an extra address (`192.168.87.135`).** This was the NAT-network address the VM used before it was moved to VMnet1. I confirmed the DNS Server service was running, ran `ipconfig /flushdns`, and queried the DC directly with `nslookup preetham.local 192.168.175.10`, which returned only `192.168.175.10`.

**`gpupdate /force` failed with "lack of network connectivity to a domain controller".** I checked that the client could ping the DC, used the DC as its DNS server, found `DC01` with `nltest /dsgetdc`, and could reach `\\DC01\SYSVOL`. After that, the policy update succeeded and the GPO appeared in `gpresult`.

## 9. Security Considerations

- Work was done in isolated virtual machines on a host-only network.
- Only test accounts were used, and passwords are masked in screenshots.
- New accounts require a password change at first logon.
- Windows Server was an evaluation copy used for learning.

## 10. Skills Demonstrated

- VMware Workstation virtual networking
- Windows Server and Active Directory Domain Services
- DNS fundamentals (zones, A and SRV records, `nslookup`)
- OU, user and security group management
- Group Policy creation, linking and verification (`gpupdate`, `gpresult`)
- Domain join and troubleshooting (`ping`, `ipconfig`, `nltest`)

## Repository Structure

```text
├── README.md
└── screenshots/
    ├── 01-vmnet1-hostonly.png
    ├── 02-dc-static-ip.png
    ├── ...
    └── 14-gpresult-applied.png
```

## Conclusion

This lab builds a working Active Directory environment end to end: an isolated VMware network, a Domain Controller with DNS, OUs, users, a security group, a linked GPO, and a domain-joined Windows 10 client with verified logon and policy application.

