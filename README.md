# Intune Help Desk Case Study

| | |
|---|---|
| **Incident** | User locked out of Outlook after device flagged non-compliant |
| **Severity** | High — Blocked from email with a client presentation deadline |
| **Root Cause** | Defender real-time protection disabled → Intune compliance failed → Conditional Access blocked Exchange Online |
| **Status** | Resolved — Same-day fix |
| **Environment** | Microsoft 365 E5 Developer tenant, Microsoft Intune, Entra ID, March 2026 |

---

## Overview

Sarah Chen, a new marketing hire, got locked out of Outlook after her device's Defender real-time protection was disabled. That one change broke the Intune compliance chain, triggered Conditional Access, and cut off her email. This case study walks through the setup, the failure, and how I diagnosed and fixed it.

---

## Environment

| Component | Detail |
|---|---|
| **Tenant** | TestLabCo279.onmicrosoft.com (M365 Developer Program) |
| **Identity** | Microsoft Entra ID (Azure AD) |
| **Endpoint Management** | Microsoft Intune |
| **Access Control** | Entra ID Conditional Access |
| **Device** | Windows 11 VM (VirtualBox), AAD-joined |
| **User** | Sarah Chen, Marketing Coordinator |
| **Apps** | Microsoft 365 Apps (Word, Excel, Outlook, Teams, PowerPoint) |

---

## Part 1: Tenant Setup & Device Enrollment

Before the ticket, I needed a working environment. M365 E5 Developer tenant, a test user (Sarah Chen, Marketing), and three policies: compliance, Conditional Access, and app deployment.

### The Policies

The compliance policy — "Corporate Device Baseline" — requires BitLocker, Defender Antimalware, real-time protection, and a password with minimum 8 characters. These aren't arbitrary settings. BitLocker is a baseline requirement for any corporate device. Defender and real-time protection are the baseline you'd expect from any security team. The password policy prevents weak credentials. Assigned to All Users.

![Compliance policy — Device Health settings (BitLocker required)](screenshots/03-compliance-policy-device-health.png)

![Compliance policy — Defender settings (real-time protection required)](screenshots/04-compliance-policy-defender-settings.png)

![Compliance policy — Password requirements](screenshots/05-compliance-policy-password-requirements.png)

![Compliance policy — assigned to All Users](screenshots/06-compliance-policy-assignment.png)

The Conditional Access policy — "Require Compliant Device for Email" — is the enforcement layer. Without it, a non-compliant device just gets a warning in Intune and the user keeps working. With CA enabled, a non-compliant device gets blocked from Exchange Online entirely. Policy set to On, not Report-only.

![Conditional Access — "Require Compliant Device for Email"](screenshots/07-conditional-access-policy.png)

App deployment was straightforward: Microsoft 365 Apps (Word, Excel, Outlook, Teams, PowerPoint), Required assignment to All Users, Current Channel for updates. Apps push automatically once a device enrolls.

![M365 Apps configuration](screenshots/08-m365-apps-configuration.png)

![M365 Apps assignment — Required, All Users](screenshots/09-m365-apps-assignment.png)

### Enrollment

I created Sarah Chen in Entra ID, assigned her an M365 E5 license, and moved to the VM.

![Sarah Chen user profile](screenshots/01-sarah-chen-user-profile.png)

![Sarah Chen license assignment — M365 E5](screenshots/02-sarah-chen-license-assignment.png)

The VM was running as a local account — no work or school accounts connected. Standard fresh machine state.

![VM accounts page — local account, no work connections](screenshots/10-vm-before-aad-join.png)

I went to Settings > Accounts > Access work or school, clicked Connect, and selected "Join this device to Azure Active Directory." Signed in as `schen@TestLabCo279.onmicrosoft.com`. After the join and a restart, the VM was cloud-joined to the tenant.

![VM accounts — Sarah Chen's work account connected](screenshots/11-vm-after-aad-join-sarah.png)

Forced an Intune sync from the VM and checked the portal. DESKTOP-FHRMLFC showed up under All Devices — managed by Intune, compliance status: **Compliant**. Everything passed: BitLocker, Defender, real-time protection, password requirements.

![Device sync — successful](screenshots/12-device-sync-successful.png)

![Intune All Devices — DESKTOP-FHRMLFC, Compliant](screenshots/13-intune-device-compliant.png)

M365 Apps deployment was showing in the portal — assigned and tracking.

![Apps > All Apps — Microsoft 365 Apps Standard deployed](screenshots/14-m365-apps-deployed.png)

Opened Outlook on the VM, signed in as Sarah. Inbox loaded. Baseline confirmed: compliant device, working email, apps deploying.

![Outlook working — Sarah Chen signed in, compliant device](screenshots/15-outlook-working-compliant.png)

---

## Part 2: The Ticket

> **Ticket #1047**
> **From:** Sarah Chen (Marketing)
> **Priority:** High
> **Subject:** Can't access Outlook — getting error about device security
>
> "I was working fine this morning and now I can't open my email. It says my device doesn't meet security requirements. I have a client presentation at 2 PM and need my email ASAP."

---

## Part 3: Diagnosis

First thing I needed to establish: was this Sarah's device, or was something broken at the tenant level? I checked the Intune portal — no other compliance failures, no service health alerts. This was isolated to Sarah's machine.

Sarah's device had its Defender real-time protection disabled. In a real environment this could happen a few ways — a user turning it off because an app flagged a false positive, a bad script, malware disabling it, or someone following bad advice online. The why matters less than the chain of events it triggers.

![Windows Security — real-time protection OFF](screenshots/16-realtime-protection-off.png)

Once real-time protection goes down, the rest happens automatically. Intune re-evaluates compliance on the next sync, the device fails the "Corporate Device Baseline" check, the status flips to Noncompliant, and Conditional Access blocks Exchange Online. Sarah sees an error page. From her perspective, email just stopped working with no warning.

After forcing a sync from the VM, I checked the portal. DESKTOP-FHRMLFC was now showing **Noncompliant**.

![Intune All Devices — DESKTOP-FHRMLFC, Noncompliant](screenshots/17-intune-device-noncompliant.png)

On the VM, Outlook now showed the Conditional Access block page: "Device must comply with your organization's compliance requirements."

![Access blocked — device non-compliant](screenshots/18-access-blocked-compliance.png)

I drilled into the failure: Devices > DESKTOP-FHRMLFC > Device compliance. The Corporate Device Baseline policy was showing **Not compliant** for `schen@TestLabCo279.onmicrosoft.com`. This confirmed exactly which policy failed and for which user — no guessing.

![Device compliance detail — Corporate Device Baseline, Not compliant](screenshots/19-compliance-detail-not-compliant.png)

---

## Part 4: Remediation & Resolution

Root cause confirmed: Defender real-time protection disabled, which broke the compliance chain all the way up to email access. The fix was straightforward.

On the VM, I opened Windows Security > Virus & threat protection > Manage settings and toggled real-time protection back **On**. Then I forced another Intune sync.

![Windows Security — real-time protection ON](screenshots/20-realtime-protection-on.png)

Checked the portal — DESKTOP-FHRMLFC was back to **Compliant**.

![Intune All Devices — DESKTOP-FHRMLFC, Compliant again](screenshots/21-intune-device-compliant-again.png)

Opened Outlook on the VM. Email access restored. Sarah's inbox loaded normally.

![Outlook working again — access restored](screenshots/22-outlook-working-again.png)

Device compliance detail confirmed it — Corporate Device Baseline back to **Compliant**.

![Device compliance detail — Corporate Device Baseline, Compliant](screenshots/23-compliance-detail-compliant-again.png)

**Why this didn't bounce back:** I didn't just re-enable Defender and walk away. I waited for the compliance status to flip back in the portal, then verified Outlook access was actually restored before closing the ticket. That extra step is the difference between resolved and reopened.

> "Hi Sarah, your email access has been restored. The issue was a security setting on your device — real-time antivirus protection had been turned off, which triggered our security policy and blocked email access. I've re-enabled it and verified everything is working. Please avoid changing Windows Security settings — if an application is being flagged, submit a ticket and we'll find a safe solution. Let me know if you need anything else before your 2 PM presentation."

**Prevention:** I documented a recommendation to configure Intune remediation scripts that automatically re-enable Defender if it gets disabled, add a grace period in Conditional Access so users get a warning before access is cut, and set up compliance notification emails so users know something's wrong before they're locked out. From a Tier 1 perspective, the fix was my job. The policy improvements are escalation items — document them, pass them up, let the team decide.

---

## What This Taught Me

**The compliance chain is invisible until it breaks.** Intune evaluates compliance, Conditional Access enforces it, and the user just sees "you can't get in." Understanding how those three layers connect — and knowing where to look at each layer — is the difference between a 10-minute fix and an hour of guessing.

**Sync timing matters.** Intune compliance doesn't evaluate in real-time. There's a sync interval, and if you're troubleshooting a compliance issue and don't force a sync, you might be looking at stale data. I hit this during both the break and the fix — the portal didn't update until I manually triggered a sync from the device.

**The portal tells you exactly what failed.** Devices > Device compliance > drill into the policy. It shows which specific setting failed for which user. No need to guess or compare settings manually. Knowing that path exists saves time on every compliance ticket.

**Don't just fix it, prevent it.** Restoring access took two clicks. Making sure the same ticket doesn't come back next month is the harder problem. Auto-remediation, grace periods, user education — that's the stuff that actually reduces ticket volume.

---

## Tools & Technologies Used

| Tool | Purpose |
|---|---|
| **Microsoft Intune** | Device enrollment, compliance policies, app deployment |
| **Microsoft Entra ID** | Identity management, Conditional Access enforcement |
| **Exchange Online** | Email access (Outlook Web) — the resource being protected |
| **Windows Defender** | The security setting that triggered the compliance failure |
| **VirtualBox** | Lab environment — Windows 11 VM for the enrolled device |
| **M365 Developer Program** | Free E5 tenant for the lab (no credit card required) |

---

_Case Study — Lab Simulation — March 2026_
