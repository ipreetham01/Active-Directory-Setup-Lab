# Setting Up Active Directory and Building a Domain Controller in VMware Workstation

This guide provides a step-by-step process to build a Windows Server-based Active Directory (AD) lab on virtual machines in **VMware Workstation**. It covers creating a Domain Controller, organizing users with OUs and groups, applying Group Policy, and joining a Windows 10 client to the domain.

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

---

# Part 1: Setting Up the Lab Environment

## 1. Download Required Files

- Windows Server 2022 evaluation ISO: https://www.microsoft.com/en-us/evalcenter/
- Windows 10 installation media (or an existing Windows 10 VM)
- VMware Workstation: https://www.vmware.com/

## 2. Check the Host-only Network

1. Open VMware Workstation and go to **Edit > Virtual Network Editor**.
2. Select **VMnet1**. It should be type **Host-only** with subnet `192.168.175.0` and mask `255.255.255.0`.

<img width="693" height="697" alt="01-vmnet1-host-only" src="https://github.com/user-attachments/assets/9682c619-5827-4cc9-8c05-e116a047641d" />


## 3. Connect the Server VM to VMnet1

1. Create the Windows Server VM and install Windows Server from the ISO (choose **Custom: Install Windows only** and set an Administrator password).
2. Go to **VM > Settings > Network Adapter**.
3. Change the connection from **NAT** to **Custom: VMnet1 (Host-only)**.

![VM network adapter set to custom VMnet1](screenshots/02-vm-adapter-vmnet1.png)

## 4. Configure a Static IP for the Server

1. Press `Win + R`, type `ncpa.cpl`, and press Enter.
2. Open the Ethernet adapter properties, then **Internet Protocol Version 4 (TCP/IPv4)**.
3. Use these settings:
   - **IP address:** `192.168.175.10`
   - **Subnet mask:** `255.255.255.0`
   - **Default gateway:** leave empty (the network is isolated)
   - **Preferred DNS server:** `192.168.175.10` (the server itself, because it will host the domain's DNS)
4. Rename the computer to `DC01` and restart.

![Static IP configuration on the Domain Controller](screenshots/03-dc-static-ip.png)

---

# Part 2: Creating a Domain Controller

## 1. Install Active Directory Domain Services (AD DS)

1. Open **Server Manager**, click **Manage**, and select **Add Roles and Features**.
2. Leave the defaults until **Server Roles**, then select **Active Directory Domain Services**.

![Select server roles](screenshots/04-select-server-roles.png)

3. Click **Add Features**, continue to **Confirmation**, and click **Install**.

![Confirm installation selections](screenshots/05-confirm-install.png)

## 2. Promote the Server to a Domain Controller

1. After installation, click the **flag icon** in Server Manager and select **Promote this server to a domain controller**.
2. In **Deployment Configuration**, choose **Add a new forest** and enter the root domain name `preetham.local`.
3. Keep **DNS server** selected and enter a DSRM (Directory Services Restore Mode) password.
4. On **DNS Options**, a warning says a DNS delegation cannot be created because the authoritative parent zone cannot be found. This is expected in an isolated `.local` lab and can be ignored.

![DNS Options warning](screenshots/06-dns-options-warning.png)

5. Complete the prerequisite checks and click **Install**.

## 3. Restart the Server

The server restarts automatically. The logon screen now shows `PREETHAM\Administrator`, which confirms the server is part of the new domain.

![Domain logon screen](screenshots/07-domain-login-screen.png)

## 4. Verify the Domain and DNS

1. Open **Tools > Active Directory Users and Computers**. The domain `preetham.local` is listed.

![Active Directory Users and Computers](screenshots/08-aduc-domain.png)

2. Open **Tools > DNS** and expand **Forward Lookup Zones > preetham.local**. The `dc01` host record points to `192.168.175.10`, and the `_msdcs`, `_sites`, `_tcp` and `_udp` folders are present.

![DNS forward lookup zone records](screenshots/09-dns-zone-records.png)

---

# Part 3: Managing Organizational Units, Users and Groups

## 1. Creating Organizational Units (OUs)

1. In **Active Directory Users and Computers**, right-click the domain and select **New > Organizational Unit**.
2. Enter the name (for example `IT`) and click **OK**.

![Creating the IT OU](screenshots/10-create-ou-it.png)

3. Repeat for `HR` and `Workstations`.

> **Note:** When "Protect container from accidental deletion" is checked, the OU cannot be deleted directly.

![OUs created under the domain](screenshots/11-ous-created.png)

**Why OUs?** The default `Users` and `Computers` folders are containers, and Group Policy can only be linked to OUs (and to the domain or sites). OUs also let you organize objects by department.

## 2. Creating Users

Right-click the **IT** OU, select **New > User**, and enter the first name, last name and user logon name.

![Creating the user Alex Sharma](screenshots/12-create-user-alex.png)

Click **Next**, set a password, and review the options:

- **User must change password at next logon:** forces a new password at first sign-in (selected for this lab)
- **User cannot change password:** the user is blocked from changing it
- **Password never expires:** commonly used for service accounts
- **Account is disabled:** used to create an account for future use

![Password options](screenshots/13-user-password-options.png)

Click **Next**, then **Finish**.

![User confirmation](screenshots/14-user-alex-confirm.png)

Repeat the same steps to create **Priya Patel** in the **HR** OU.

![Creating the user Priya Patel](screenshots/15-create-user-priya.png)

## 3. Creating a Security Group

Right-click the **IT** OU and select **New > Group**. Name it `IT-Support`, with scope **Global** and type **Security**.

![Creating the IT-Support group](screenshots/16-create-group-it-support.png)

The IT OU now contains the user and the group.

![IT OU contents](screenshots/17-it-ou-contents.png)

---

# Part 4: Group Policy

## 1. Create and Link a GPO

1. Open **Tools > Group Policy Management** and expand the domain.
2. Right-click the **IT** OU and select **Create a GPO in this domain, and Link it here**.
3. Name it `IT-Workstation-Policy`.

![Create and link a GPO to the IT OU](screenshots/18-create-and-link-gpo.png)

## 2. Configure the Policy

1. Right-click the GPO and select **Edit**.
2. Go to **User Configuration > Policies > Administrative Templates > Control Panel > Personalization**.
3. Open **Prevent changing desktop background**, select **Enabled**, and click **OK**.

![Prevent changing desktop background set to Enabled](screenshots/19-gpo-setting-enabled.png)

> **Note:** This is a *user* policy, so it applies to user accounts inside the IT OU. Microsoft's help text also says the "Desktop Wallpaper" policy must be enabled to fully stop users from changing their wallpaper. This lab verifies that the GPO is delivered and applied, shown in Part 6.

---

# Part 5: Joining a Windows 10 Client to the Domain

## 1. Configure the Client Network

Connect the Windows 10 VM to **VMnet1** and set a static IP. **The client must use the Domain Controller as its DNS server**, otherwise it cannot find the domain.

| Setting | Value |
|---|---|
| IP address | `192.168.175.20` |
| Subnet mask | `255.255.255.0` |
| Preferred DNS server | `192.168.175.10` |

![Client static IP and DNS](screenshots/20-client-static-ip.png)

Test the connection with `ping 192.168.175.10`. All four replies should come back (0% loss).

![Ping from the client to the Domain Controller](screenshots/21-client-ping-dc.png)

## 2. Join the Domain

1. Press `Win + R`, type `sysdm.cpl`, and press Enter.
2. On the **Computer Name** tab, click **Change**, select **Domain**, and enter `preetham.local`.
3. Enter domain credentials that are allowed to join computers, then restart when prompted.

![Joining the domain preetham.local](screenshots/22-join-domain.png)

## 3. Sign in with a Domain Account

After the restart, choose **Other user** and sign in as `PREETHAM\alex.sharma`. Because the account is set to change its password at first logon, Windows asks for a new password.

Open Command Prompt and run `whoami` and `echo %LOGONSERVER%`. The output shows the domain account and that `DC01` authenticated the logon.

![whoami and LOGONSERVER](screenshots/23-whoami-logonserver.png)

---

# Part 6: Verifying Domain Services and Group Policy

## 1. Domain Controller Discovery and SYSVOL

Run:

- `nltest /dsgetdc:preetham.local` to confirm the client finds `DC01.preetham.local`
- `dir \\DC01\SYSVOL` to confirm the SYSVOL share (where Group Policy files live) is reachable

![nltest and SYSVOL](screenshots/24-nltest-sysvol.png)

## 2. Confirm the GPO Is Applied

Run:

- `gpupdate /force`
- `gpresult /r`

Under **Applied Group Policy Objects** for `CN=Alex Sharma,OU=IT,DC=preetham,DC=local`, `IT-Workstation-Policy` is listed and was applied from `DC01.preetham.local`.

![gpresult showing IT-Workstation-Policy applied](screenshots/25-gpresult-applied.png)

---

# Troubleshooting

**`nslookup` shows "DNS request timed out" and "Server: UnKnown".**
`nslookup` first does a reverse lookup of the DNS server's own IP. No reverse lookup zone exists in this lab, so it prints the timeout. The forward lookup still returns the correct answer, so this is harmless.

**DNS returned an extra address (`192.168.87.135`).**
That was the address from the NAT network the VM used before it was moved to VMnet1. I confirmed the DNS Server service was running, ran `ipconfig /flushdns`, and queried the DC directly with `nslookup preetham.local 192.168.175.10`, which returned only `192.168.175.10`.

**`gpupdate /force` failed with "lack of network connectivity to a domain controller".**
I checked that the client could ping the DC, used the DC as its DNS server, found `DC01` with `nltest /dsgetdc`, and could open `\\DC01\SYSVOL`. After that, the policy update succeeded and the GPO appeared in `gpresult`.

**General checklist**

- The client's only DNS server should be the Domain Controller.
- The GPO must be linked to the OU that contains the user (for user policies).
- Run `gpupdate /force`, then `gpresult /r` or `gpresult /h report.html`.
- Make sure the DNS Server service is running on the Domain Controller.

---

# Security Considerations

- Work was done in isolated virtual machines on a host-only network.
- Only test accounts were used, and passwords are masked in the screenshots.
- New accounts require a password change at first logon.
- Windows Server was an evaluation copy used for learning.

# Skills Demonstrated

- VMware Workstation virtual networking
- Windows Server and Active Directory Domain Services
- DNS fundamentals (zones, A and SRV records, `nslookup`)
- OU, user and security group management
- Group Policy creation, linking and verification (`gpupdate`, `gpresult`)
- Domain join and troubleshooting (`ping`, `ipconfig`, `nltest`)

# Conclusion

This lab builds a working Active Directory environment end to end: an isolated VMware network, a Domain Controller with DNS, OUs, users, a security group, a linked GPO, and a domain-joined Windows 10 client with verified logon and policy application.
<img width="693" height="697" alt="01-vmnet1-host-only" src="https://github.com/user-attachments/assets/19fdf0ac-55aa-4603-814e-8d10ff3549d9" />
