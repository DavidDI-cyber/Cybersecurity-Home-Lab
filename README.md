Active Directory Identity Threat Detection Lab

A self built enterprise security lab that simulates a small Windows Active Directory network and uses a central SIEM (Wazuh) to detect identity-based attacks such as Kerberoasting. Built from scratch in VirtualBox as hands on blue team practice.

Status: 🟡 In progress — the core infrastructure is complete and verified (domain, domain-joined client, working authentication). Detection pipeline and attack simulation are the next phase. See Roadmap for more info

Overview

This project builds a realistic (if miniature) corporate network and then tries to break into it and catch the break in. It consists of a Windows Server 2022 domain controller, a domain-joined Windows 10 workstation, and a Wazuh SIEM collecting and correlating logs from both.

The point isn't just to stand the infrastructure up  it's to generate the kind of attack a real environment faces (starting with Kerberoasting) and prove it can be detected in the logs. To be honest, that detection step is what the project is really about.

I'm a second-year Cyber Security student building this toward a placement year, so the lab is also a running record of what I've learned and the problems I've had to debug along the way.

Why I built it
To understand how enterprise identity (Active Directory) actually works and therefore how it's attacked rather than just reading about it.
To practise the core blue-team workflow: collect logs → establish a baseline of "normal" →  then detect the anomaly.
To have something concrete I can explain end to end in an interview, including the parts that went wrong.
Architecture

All thee machines sit on an isolated VirtualBox host-only network (192.168.1.0/24) so lab traffic — including anything malicious — never touches my real network.

                 Host-only network  192.168.1.0/24  (isolated)
     ┌─────────────────────────┬──────────────────────────┐
     │                         │                          │
┌────┴─────────┐      ┌────────┴────────┐        ┌────────┴────────┐
│   DC01       │      │   CLIENT01      │        │   WAZUH          │
│ Win Server   │      │ Windows 10 Pro  │        │ Wazuh 4.14.7     │
│   2022       │      │ domain member   │        │ SIEM manager     │
│ AD DS + DNS  │      │ CLIENT01.lab... │        │ + dashboard      │
│ 192.168.1.10 │      │ 192.168.1.20    │        │ 192.168.1.x      │
└──────────────┘      └─────────────────┘        └─────────────────┘
  domain: lab.local      logs ───────────────────────▶ (Wazuh agent)

VirtualBox showing the lab VMs

Environment
Machine	OS	Role	IP
DC01	Windows Server 2022	Domain Controller, AD DS, DNS	192.168.1.10
CLIENT01	Windows 10 Pro	Domain-joined workstation	192.168.1.20
WAZUH	Wazuh OVA 4.14.7 (Linux)	SIEM manager + dashboard	192.168.1.x
Domain: lab.local
Virtualisation: Oracle VirtualBox, host-only networking
Storage: VMs run from an external SSD to keep the host disk free

DC01 Server Manager showing AD DS and DNS roles

Build summary
1. Domain Controller (DC01)
Installed Windows Server 2022 and promoted it to a Domain Controller, creating the lab.local domain.
Created an Organizational Unit (LabUsers) and a test user (testuser1) via Active Directory Users and Computers.

ADUC showing the LabUsers OU and testuser1

2. Isolated network
Moved all VMs onto a host-only adapter so the environment is self-contained and safe to attack.
DC01 holds a static IP (192.168.1.10) and serves DNS for the domain.
3. Client workstation (CLIENT01)
Installed Windows 10 Pro (Pro is required for domain join).
Set a static IP (192.168.1.20) with DNS pointed at the DC (192.168.1.10) — this is what lets the client resolve and join lab.local.
Joined CLIENT01 to the lab.local domain and verified it from the server side.

CLIENT01 System Properties showing membership of lab.local

ADUC Computers container showing CLIENT01 joined

4. Verifying the domain works end-to-end
Confirmed connectivity and name resolution from CLIENT01 (ping to the DC and nslookup lab.local resolving to 192.168.1.10).
Logged into CLIENT01 as a domain user (LAB\testuser1) and confirmed with whoami — proving authentication flows all the way through.

CLIENT01 network tests — ping and nslookup

Domain user logged into CLIENT01 (whoami shows lab\testuser1)

5. SIEM (Wazuh) — next phase
Wazuh manager + dashboard deployed.
Next: install Wazuh agents and Sysmon on DC01 and CLIENT01 so authentication and process activity are shipped to the SIEM for analysis.
Detection goals — next phase
Baseline — log in/out as testuser1 to capture what normal authentication looks like (Windows Event ID 4624).
Attack — simulate Kerberoasting from CLIENT01: request service tickets that can be cracked offline to recover credentials (a common real-world AD attack).
Detect — identify the attack in Wazuh via the service-ticket requests it generates (Event ID 4769) and corresponding Wazuh alerts.
Troubleshooting log

Real problems hit while building this, and how I diagnosed and fixed them. (I added this bit because the debugging was where I learned the most.)

VMs showed as "Inaccessible" after an unsafe SSD disconnect. The small .vbox config files had been zeroed, but the large .vdi disk images were intact. Recovered by registering new VM shells pointed at the existing .vdi files (and understanding that .vbox-prev is VirtualBox's automatic config backup). I also matched the new VM's MAC address to the original so the DC kept its static IP binding. Lesson: always "safely remove" the drive before unplugging.
Windows 11 client install hung on a permanent black screen. Traced it past the VM settings (ICH9, TPM, Secure Boot all correct) to the root cause: my laptop's Hyper-V backend was breaking VirtualBox's EFI boot — BIOS-mode VMs booted fine, EFI-mode ones didn't. Resolved by switching the client to a BIOS-mode Windows 10 build instead of fighting the host's virtualization stack.
Install failed repeatedly with 0x800701B1 / 0x80070015 (device not ready). The external SSD was dropping its USB connection mid-write because Windows' USB selective suspend was powering it down under sustained load. Fixed by disabling USB selective suspend and the hard-disk idle timeout, and moving to a more stable USB port; the install then completed cleanly. (Also learned the hard way not to eject the install ISO mid-install — setup still reads from it during "Getting files ready".)
Domain join rejected valid credentials ("username or password is incorrect"). Ruled out clock skew and DNS (the DC was reachable and lab.local resolved). The real cause was a keyboard-layout mismatch: the admin password had been set containing a character that typed differently across US/UK layouts (@ vs "), so the password I was "correctly" typing on the client didn't match what was stored. Fixed by resetting the account to a password with no layout-sensitive characters. Lesson: keep keyboard layouts consistent across VMs, or avoid ambiguous characters in service passwords.
Domain user login failed with "your account has been disabled". The testuser1 account was disabled in AD. Enabled it via ADUC (and cleared the "user must change password at next logon" flag) — a good reminder of how account state in AD directly gates authentication.
What I learned
(to fill in as the detection phase completes — e.g. how Kerberoasting works at the ticket level, what Event 4769 actually looks like, how Wazuh rules map to Windows events)
Why DNS must point at the DC for domain membership to work at all.
How much of "it won't authenticate" comes down to unglamorous basics — clock, DNS, account state, and even keyboard layout — rather than the thing you first suspect.
Roadmap
 Build DC01 and the lab.local domain
 Isolate the lab on a host-only network
 Build and domain-join CLIENT01
 Verify domain authentication (domain user login on the client)
 Deploy Wazuh agents + Sysmon on DC01 and CLIENT01
 Capture a normal authentication baseline
 Simulate a Kerberoasting attack
 Detect it in Wazuh and document the alert
 Add Group Policy hardening and re-test
