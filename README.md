# Active Directory Penetration Testing Lab — Notes & Methodology

A working journal of attack chains, techniques, and defensive observations from my home Active Directory penetration testing lab. This repository documents methodology and learning rather than acting as a step-by-step exploitation guide.

## Lab topology (reference)

The lab was built on a single hypervisor with the following hosts:

| Host          | Role                          | OS                       |
|---------------|-------------------------------|--------------------------|
| `DC01`        | Domain Controller             | Windows Server 2019      |
| `SRV01`       | Member server (file / SQL)    | Windows Server 2019      |
| `WS01`        | Domain workstation            | Windows 10               |
| `WS02`        | Domain workstation            | Windows 10               |
| `KALI`        | Attacker                      | Kali Linux               |

Domain: `corp.lab.local`
Subnet: `10.10.10.0/24`

The lab was intentionally configured with realistic misconfigurations: weak service account passwords, unconstrained delegation on a member server, an over-privileged help-desk group, and a kerberoastable SPN.

## Methodology — phases

Each phase below links to detailed notes and the specific techniques practised.

### 1. External / initial access

Practised against an exposed service simulation:
- Service banner enumeration with `nmap -sC -sV`
- Identifying weak authentication on exposed RDP / SMB
- Password spray against `corp.lab.local` users discovered through OSINT-equivalent enumeration
- Used `kerbrute` for username enumeration and password spray

**Key takeaway:** initial access in lab was almost always credential-based, not exploit-based. Real environments mirror this — phishing and credential reuse dominate over CVE exploitation.

### 2. Internal recon & enumeration

After landing on a workstation:
- Local enumeration: `whoami /all`, `net user /domain`, `net group "domain admins" /domain`
- BloodHound collection via `SharpHound` (CSharp ingestor) — collected with `-c All` then narrowed to `DCOnly` for stealthier replay
- Enumerated SMB shares with `NetExec smb <subnet> --shares`
- Pulled GPP `cpassword` from SYSVOL on legacy GPO (intentional misconfiguration)

**Tooling notes:**
- `NetExec` (formerly CrackMapExec) for fast subnet-wide auth and share enumeration
- `BloodHound` paired with custom Cypher queries to find shortest paths to DA
- `ldapsearch` for low-noise enumeration when avoiding endpoint detection

### 3. Credential access

Techniques exercised:
- **Kerberoasting** — `Rubeus.exe kerberoast` against the SQL service account, cracked offline with `hashcat -m 13100`
- **AS-REP roasting** — identified accounts with `Do not require Kerberos preauthentication` set, captured with `Rubeus asreproast`, cracked with `hashcat -m 18200`
- **GPP password decryption** — used `gpp-decrypt` against the recovered cpassword
- **DPAPI** — extracted credentials from local DPAPI vault on compromised workstation

### 4. Lateral movement

Once domain credentials were obtained:
- Pass-the-hash with `NetExec smb <target> -u <user> -H <ntlm>`
- WMI execution via `impacket-wmiexec`
- Service-based execution via `impacket-psexec` (noisier — used to compare detection signatures)
- Documented event log artifacts (4624, 4672, 4688) for blue team awareness

### 5. Privilege escalation

The intentional misconfigurations led to several practical paths:
- **Unconstrained delegation** — printer-bug + S4U abuse to escalate via `SRV01`
- **ACL abuse** — discovered `WriteDACL` on a high-value group via BloodHound; abused with `Set-DomainObjectOwner` then group manipulation
- **Kerberoasted service account** had local admin on multiple boxes (real-world common finding)

### 6. Domain dominance

After reaching DA-equivalent privileges:
- DCSync via `secretsdump.py -just-dc <user>:<hash>@<dc>` to extract `krbtgt`
- Generated a Golden Ticket with `Rubeus golden`
- Persisted with a Silver Ticket against the SQL service for stealthier follow-up access
- Documented detection signatures and how a defender could have identified the activity

## Defensive observations

After each attack chain, I documented:
- What event IDs fired
- What a SIEM rule would need to catch the activity
- What a hardening configuration would have prevented the path

This dual-perspective approach helped me write more useful client-style remediation guidance.

## Reporting

Each engagement in the lab was written up in a sanitised report with:
- Executive summary
- Scope and methodology
- Findings (CVSS-scored)
- Remediation guidance
- Appendices (commands, artefacts)

A redacted sample report is available in [`sample-pentest-report`](https://github.com/Zorthuun) — the format follows industry conventions used by major consultancies.

## What's intentionally not in this repo

- Active Directory configuration files / VM templates (these would expose lab-specific credentials)
- Unsanitised reports
- Any tooling that targets infrastructure I do not own

## References that shaped my methodology

- *The Hacker Recipes* — [thehacker.recipes](https://www.thehacker.recipes/)
- *PayloadsAllTheThings* — Active Directory section
- *HackTricks* — Windows / AD chapters
- *SpecterOps* blog — particularly the BloodHound and ACL abuse posts

---

*Notes are updated as I revisit the lab and learn new techniques.*
