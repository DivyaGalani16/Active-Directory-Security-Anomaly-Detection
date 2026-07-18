# 1. Introduction

Active Directory (AD) is a centralized directory service used by enterprises to manage 
users, devices, and security policies across networks [1]. It is responsible for handling 
authentication, authorization, and resource allocation, playing a critical role in 
enterprise identity management systems. AD enables administrators to control access to 
shared resources, apply security policies, and enforce user permissions across large 
infrastructures. However, its extensive reach and complex configuration make it a frequent 
target for attackers who exploit vulnerabilities and misconfigurations to gain unauthorized 
access and escalate privileges.

Attackers leverage the hierarchical structure and trust relationships in Active Directory to 
move laterally within the network, compromise privileged accounts, and ultimately gain 
control over domain controllers. The presence of legacy protocols, weak password policies, 
and excessive privileges on service accounts further expose AD environments to a range of 
sophisticated attacks.

## 1.1 Common Attacks in Active Directory

Cyber attackers use well-established methods to exploit weaknesses within Active Directory 
environments [2]. These attacks typically begin with reconnaissance and enumeration to 
gather information about domain structures, service accounts, and administrative paths. 
Once attackers identify vulnerable areas, they proceed to credential theft and privilege 
escalation. Common attack techniques include:

**Kerberoasting:**
- Service tickets (TGS) for service accounts are requested by attackers.
- These tickets, once captured, are cracked offline to recover the service account password.
- Often targets accounts with weak passwords, providing attackers with privileged access [2].

**Pass-the-Hash (PtH):**
- Allows authentication using captured NTLM hashes without knowing the plaintext password.
- Enables lateral movement between systems by impersonating legitimate users [2].

**Silver Ticket Attacks:**
- Involves forging Kerberos service tickets to gain unauthorized access to services.
- Attackers use compromised service account keys to craft these tickets and bypass KDC 
  validation [20].

They then push unauthorized changes directly into Active Directory, modifying group 
memberships and security policies without detection [2]. Exploitation of these techniques 
is often facilitated by known vulnerabilities. **CVE-2020-1472 (Zerologon)** allows 
attackers to compromise domain controllers by exploiting weaknesses in the Netlogon 
protocol, leading to immediate domain takeover. The combined abuse of **CVE-2021-42278** 
and **CVE-2021-42287** enables privilege escalation to domain admin by manipulating service 
account attributes and ticket requests. Recent vulnerabilities such as **CVE-2025-21293** 
and **CVE-2025-21193** demonstrate that privilege escalation and impersonation threats 
continue to evolve, requiring constant vigilance.

## 1.2 Indicators and Detection Techniques

The ability to detect attacks in Active Directory depends on continuous monitoring of 
suspicious patterns and behaviors. Certain indicators serve as early warning signs of 
potential compromise:

- Excessive Kerberos ticket requests from a single host, especially for service accounts.
- Unauthorized or unusual modifications to Group Policy Objects (GPOs).
- The creation of new privileged accounts without proper change control approval.
- Multiple failed authentication attempts and lateral login attempts across systems in 
  rapid succession.
- Unauthorized changes in trust relationships between domains or unexpected domain 
  replication traffic.

Organizations use event log analysis and SIEM integration to monitor domain controller 
activities and detect these signatures. Key security logs, such as Kerberos authentication 
events, directory service changes, and privileged account logins, are critical data sources 
for real-time threat detection.

## 1.3 Sources of Vulnerabilities in Active Directory

Vulnerabilities in AD often stem from misconfigurations and weak security practices that 
remain unnoticed over time. The most common sources include:

- Weak passwords for service accounts, especially those with high privileges and 
  long-standing credentials.
- Excessive permissions granted to user accounts and groups without periodic review.
- Insecure delegation settings, allowing attackers to impersonate users or services.
- Unmonitored trust relationships between domains or forests that can be exploited for 
  lateral movement.
- Legacy protocols like NTLM, which remain enabled and vulnerable to relay and 
  hash-based attacks.
- Lack of effective auditing, leading to undetected privilege escalations or GPO changes.

Such misconfigurations and overlooked security controls create an environment where 
attackers can move stealthily and compromise critical assets. In large enterprise 
environments, managing thousands of users, groups, and permissions adds to the complexity, 
making consistent security hygiene challenging.

## 1.4 Challenges in Active Directory Security

Securing Active Directory against advanced threats is challenging for several reasons:

**Complex configurations:**
- The intricate hierarchy of domains, trusts, and organizational units makes it difficult 
  to detect subtle misconfigurations.
- Permissions are often layered and can result in hidden escalation paths.

**Evolving attack techniques:**
- Attackers continually refine methods to bypass security measures.
- Techniques like Kerberoasting and DCShadow evolve, becoming harder to detect.

**Monitoring limitations:**
- Not all changes in AD environments are logged or correlated.
- Large volumes of log data make it difficult to spot anomalies without dedicated 
  detection systems.

**Hybrid environments:**
- The integration of on-prem AD with Azure Active Directory introduces additional 
  complexities.
- Hybrid setups expand the attack surface, increasing the likelihood of overlooked 
  configurations and vulnerabilities.

**Insider threats:**
- Authorized users with excessive privileges or misused administrative accounts can 
  cause damage or open doors for external attackers.

Addressing these challenges requires continuous assessment, privilege reviews, regular 
audits, and an understanding of evolving attack methodologies.
