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

<img width="885" height="915" alt="02-vm-adapter-vmnet1" src="https://github.com/user-attachments/assets/785ae15a-a083-41d0-b533-f1407bb42125" />

## 4. Configure a Static IP for the Server

1. Press `Win + R`, type `ncpa.cpl`, and press Enter.
2. Open the Ethernet adapter properties, then **Internet Protocol Version 4 (TCP/IPv4)**.
3. Use these settings:
   - **IP address:** `192.168.175.10`
   - **Subnet mask:** `255.255.255.0`
   - **Default gateway:** leave empty (the network is isolated)
   - **Preferred DNS server:** `192.168.175.10` (the server itself, because it will host the domain's DNS)
4. Rename the computer to `DC01` and restart.

<img width="1018" height="726" alt="03-dc-static-ip" src="https://github.com/user-attachments/assets/281e37b6-f988-4896-807b-0528bfb8ce37" />

---

# Part 2: Creating a Domain Controller

## 1. Install Active Directory Domain Services (AD DS)

1. Open **Server Manager**, click **Manage**, and select **Add Roles and Features**.
2. Leave the defaults until **Server Roles**, then select **Active Directory Domain Services**.

<img width="1010" height="805" alt="04-select-server-roles" src="https://github.com/user-attachments/assets/6912c929-0e49-42d7-bc4d-f46bbcb0f7f9" />

3. Click **Add Features**, continue to **Confirmation**, and click **Install**.

<img width="1010" height="732" alt="05-confirm-install" src="https://github.com/user-attachments/assets/91a35f3f-c5e1-4209-a4dc-b07237995ba0" />

## 2. Promote the Server to a Domain Controller

1. After installation, click the **flag icon** in Server Manager and select **Promote this server to a domain controller**.
2. In **Deployment Configuration**, choose **Add a new forest** and enter the root domain name `preetham.local`.
3. Keep **DNS server** selected and enter a DSRM (Directory Services Restore Mode) password.
4. On **DNS Options**, a warning says a DNS delegation cannot be created because the authoritative parent zone cannot be found. This is expected in an isolated `.local` lab and can be ignored.

<img width="1007" height="730" alt="06-dns-options-warning" src="https://github.com/user-attachments/assets/309a0487-c1e8-49a5-987c-fcc959a894c2" />

5. Complete the prerequisite checks and click **Install**.

## 3. Restart the Server

The server restarts automatically. The logon screen now shows `PREETHAM\Administrator`, which confirms the server is part of the new domain.

<img width="1240" height="787" alt="07-domain-login-screen" src="https://github.com/user-attachments/assets/a68c46ac-2be9-46e6-9350-01bef104e3ab" />

## 4. Verify the Domain and DNS

1. Open **Tools > Active Directory Users and Computers**. The domain `preetham.local` is listed.

<img width="1022" height="727" alt="08-aduc-domain" src="https://github.com/user-attachments/assets/472d08b3-d34f-436d-81db-ee0fa9bdc1e1" />

2. Open **Tools > DNS** and expand **Forward Lookup Zones > preetham.local**. The `dc01` host record points to `192.168.175.10`, and the `_msdcs`, `_sites`, `_tcp` and `_udp` folders are present.

<img width="747" height="490" alt="09-dns-zone-records" src="https://github.com/user-attachments/assets/fa5d8aa2-7185-40ae-afe7-2a9e6cd5c747" />

---

# Part 3: Managing Organizational Units, Users and Groups

## 1. Creating Organizational Units (OUs)

1. In **Active Directory Users and Computers**, right-click the domain and select **New > Organizational Unit**.
2. Enter the name (for example `IT`) and click **OK**.

<img width="1022" height="727" alt="10-create-ou-it" src="https://github.com/user-attachments/assets/ebff2bdf-a550-44ec-9741-3f7e5daf7683" />

3. Repeat for `HR` and `Workstations`.

> **Note:** When "Protect container from accidental deletion" is checked, the OU cannot be deleted directly.

<img width="1020" height="727" alt="11-ous-created" src="https://github.com/user-attachments/assets/3e49e824-510b-41b8-a96f-90a5eacd5dd5" />

**Why OUs?** The default `Users` and `Computers` folders are containers, and Group Policy can only be linked to OUs (and to the domain or sites). OUs also let you organize objects by department.

## 2. Creating Users

Right-click the **IT** OU, select **New > User**, and enter the first name, last name and user logon name.

<img width="432" height="372" alt="12-create-user-alex" src="https://github.com/user-attachments/assets/0219db4e-21c8-4393-b1ed-eeae2b3b7158" />

Click **Next**, set a password, and review the options:

- **User must change password at next logon:** forces a new password at first sign-in (selected for this lab)
- **User cannot change password:** the user is blocked from changing it
- **Password never expires:** commonly used for service accounts
- **Account is disabled:** used to create an account for future use

<img width="432" height="372" alt="13-user-password-options" src="https://github.com/user-attachments/assets/5cdf6f2d-3940-4a52-9840-08e10b61ff20" />

Click **Next**, then **Finish**.

<img width="433" height="372" alt="14-user-alex-confirm" src="https://github.com/user-attachments/assets/7ad6c215-10d9-4081-a5c9-a0c8d37a53af" />

Repeat the same steps to create **Priya Patel** in the **HR** OU.

<img width="431" height="371" alt="15-create-user-priya" src="https://github.com/user-attachments/assets/4d7812fd-a7d1-4f0d-bab9-56680b5cb143" />

## 3. Creating a Security Group

Right-click the **IT** OU and select **New > Group**. Name it `IT-Support`, with scope **Global** and type **Security**.

<img width="431" height="370" alt="16-create-group-it-support" src="https://github.com/user-attachments/assets/1d7b3a35-b8cd-441a-8064-5d1deb8db267" />

The IT OU now contains the user and the group.

<img width="1023" height="723" alt="17-it-ou-contents" src="https://github.com/user-attachments/assets/968d1602-2c54-44d0-8947-b332228cede5" />

---

# Part 4: Group Policy

## 1. Create and Link a GPO

1. Open **Tools > Group Policy Management** and expand the domain.
2. Right-click the **IT** OU and select **Create a GPO in this domain, and Link it here**.
3. Name it `IT-Workstation-Policy`.

<img width="1023" height="726" alt="18-create-and-link-gpo" src="https://github.com/user-attachments/assets/e3ee5191-1617-48e3-89b1-a110533a616d" />

## 2. Configure the Policy

1. Right-click the GPO and select **Edit**.
2. Go to **User Configuration > Policies > Administrative Templates > Control Panel > Personalization**.
3. Open **Prevent changing desktop background**, select **Enabled**, and click **OK**.

<img width="680" height="631" alt="19-gpo-setting-enabled" src="https://github.com/user-attachments/assets/5a379e79-1115-4284-a8bc-ac89cb698b8b" />

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

<img width="395" height="452" alt="20-client-static-ip" src="https://github.com/user-attachments/assets/edd08115-1c74-4009-8c39-499b655b8e07" />

Test the connection with `ping 192.168.175.10`. All four replies should come back (0% loss).

<img width="670" height="503" alt="21-client-ping-dc" src="https://github.com/user-attachments/assets/a7c01757-ba17-42df-8b59-79d22f163da1" />

## 2. Join the Domain

1. Press `Win + R`, type `sysdm.cpl`, and press Enter.
2. On the **Computer Name** tab, click **Change**, select **Domain**, and enter `preetham.local`.
3. Enter domain credentials that are allowed to join computers, then restart when prompted.

<img width="315" height="385" alt="22-join-domain" src="https://github.com/user-attachments/assets/4bccdd8f-d06e-4435-8aa1-43ffec14bf08" />

## 3. Sign in with a Domain Account

After the restart, choose **Other user** and sign in as `PREETHAM\alex.sharma`. Because the account is set to change its password at first logon, Windows asks for a new password.

Open Command Prompt and run `whoami` and `echo %LOGONSERVER%`. The output shows the domain account and that `DC01` authenticated the logon.

<img width="687" height="170" alt="23-whoami-logonserver" src="https://github.com/user-attachments/assets/a8353200-c369-429e-891b-3bfe55ba8b1e" />

---

# Part 6: Verifying Domain Services and Group Policy

## 1. Domain Controller Discovery and SYSVOL

Run:

- `nltest /dsgetdc:preetham.local` to confirm the client finds `DC01.preetham.local`
- `dir \\DC01\SYSVOL` to confirm the SYSVOL share (where Group Policy files live) is reachable

<img width="672" height="491" alt="24-nltest-sysvol" src="https://github.com/user-attachments/assets/43b0a722-fd57-4716-b5e9-2fcb56eb5652" />

## 2. Confirm the GPO Is Applied

Run:

- `gpupdate /force`
- `gpresult /r`

Under **Applied Group Policy Objects** for `CN=Alex Sharma,OU=IT,DC=preetham,DC=local`, `IT-Workstation-Policy` is listed and was applied from `DC01.preetham.local`.

<img width="797" height="800" alt="25-gpresult-applied" src="https://github.com/user-attachments/assets/c6e64ba9-199c-4400-b4ae-aca25f763ce4" />

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
