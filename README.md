# Active Directory Lab — GPO Misconfiguration → Local Privilege Escalation

A self-built Active Directory lab where a Windows 11 workstation is joined to a Windows Server 2022
domain, and a standard domain user is escalated to full local **Administrator** by exploiting a
realistic Group Policy misconfiguration — no passwords or credentials of a privileged account used.

📄 Full write-up PDF: [`Cyber_Project_Part_A_ActiveDirectory.pdf`](./Cyber_Project_Part_A_ActiveDirectory.pdf)

---

## Lab Architecture

| Role | Hostname | OS | IP |
|---|---|---|---|
| Domain Controller | `DC01` | Windows Server 2022 | `192.168.180.129` |
| Domain-joined workstation | `Win11-Client` | Windows 11 Pro | `192.168.180.140` |

Domain: **adamklab.local**

![Lab topology](./images/01-lab-topology.png)

---

## 1. Domain Controller — Operational

Windows Server 2022 was promoted to a Domain Controller, standing up the new Active Directory forest
`adamklab.local` with AD DS and DNS running on a static IP.

![DC01 operational](./images/02-dc-operational.png)
![Domain visible in AD Users and Computers](./images/03-domain-visible.png)

---

## 2. Workstation Joined to the Domain

`Win11-Client` (Windows 11 Pro) was joined to `adamklab.local`. Domain authentication succeeds
directly on the workstation, confirming the join.

![Domain login succeeds](./images/04-domain-login.png)

A standard, non-privileged domain user — **Student1** — was created as the baseline account for the
escalation test. Before the exploit, it holds no administrative rights:

```
C:\> whoami /groups
...
BUILTIN\Users          Alias   S-1-5-32-545   Enabled group
(no BUILTIN\Administrators entry)
```

![Baseline — no admin rights](./images/05-baseline-no-admin.png)

---

## 3. The Misconfiguration

**Type:** Group Policy `Restricted Groups` misconfiguration on the *Default Domain Policy*.

Instead of scoping local-admin rights carefully (or using a dedicated, auditable group), the policy
was configured to add a **standard domain user directly into the local `Administrators` group** of
every computer in the domain:

```
Computer Configuration
 └─ Policies
     └─ Windows Settings
         └─ Security Settings
             └─ Restricted Groups
                 └─ Administrators → Members: Student1
```

This mirrors a common real-world incident: a single domain-wide policy change silently grants
excessive local privileges across an entire organization's fleet.

![Exploit flow](./images/06-exploit-flow.png)

![Restricted Groups policy configured](./images/07-restricted-groups-policy.png)

---

## 4. Exploitation & Verification

The policy was pushed to the workstation with `gpupdate /force`. After a restart (required to
refresh the local security token), `Student1`'s group membership shows the new privilege:

```
C:\> whoami /groups
...
BUILTIN\Administrators Alias   S-1-5-32-544   Mandatory group, Enabled by default, Enabled group, Group owner
```

![whoami after escalation](./images/08-whoami-after-escalation.png)

**Decisive proof:** launching an elevated Command Prompt as `Student1` succeeds with **zero UAC
credential prompt** — full local Administrator access obtained purely through the policy
misconfiguration.

![Elevated cmd with no prompt](./images/09-elevated-cmd-no-prompt.png)

---

## Summary

- ✅ Built a working Active Directory domain (`adamklab.local`) on Windows Server 2022.
- ✅ Joined a Windows 11 workstation to the domain.
- ✅ Created a standard user with verified zero admin rights (baseline).
- ✅ Identified and exploited a `Restricted Groups` GPO misconfiguration.
- ✅ Escalated the standard user to full local Administrator — confirmed via `whoami /groups` and
  a UAC-prompt-free elevated shell.

## Remediation Notes

- Never add individual standard users to `Restricted Groups` → `Administrators` at the domain level.
- Use **Local Administrator Password Solution (LAPS)** or a scoped, audited PAM/JIT admin group
  instead of static domain-wide membership.
- Regularly audit `Default Domain Policy` and all GPOs linked at the domain root for scope creep.
- Alert on any `BUILTIN\Administrators` membership change via Windows Event ID 4732 / 4728.

---

*All IP addresses are private (RFC 1918) and used entirely within an isolated local lab — no
production systems or public infrastructure were involved.*
