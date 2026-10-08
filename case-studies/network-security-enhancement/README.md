# Network Optimisation & Security Enhancement

> Multi-year programme to assess and improve the performance and security of a managed-services network estate: Cisco switching, Fortinet firewalls and intrusion detection, Veeam and Acronis backup, regular security audits, and PowerShell automation for the routine work.

**Category:** Infrastructure (professional) · **Status:** Completed · **Period:** May 2022 – Jan 2025 · **Role:** Junior Systems Engineer → Systems Engineer, Next Day Heroes

> Client names, addresses and configurations are omitted. This describes how the work was structured and what it delivered.

## Objective

Evaluate and enhance the performance and security of the company's network infrastructure, reduce operational cost, and raise overall reliability, without disrupting businesses that could not afford downtime.

## How the work was structured

### 1. Assessment

- Comprehensive assessments of the existing network systems to surface inefficiencies and security vulnerabilities: configuration review, inventory, and identification of single points of failure.
- Findings prioritised by risk and effort into a remediation plan.

### 2. Security controls

- **Firewalls:** standardised Fortinet deployments and rule-base cleanup, with policy between network segments rather than flat trust.
- **Intrusion detection:** IDS/IPS profiles enabled on the perimeter with logging and alerting.
- **Regular security audits:** recurring reviews of firewall rules, administrative accounts and patch status, so security was a cadence rather than a one-off project.

### 3. Backup and recovery

- Deployment and standardisation of **Veeam** and **Acronis** across the estate, with documented restore procedures and tested recoveries.

### 4. Automation with PowerShell

- Routine system-management tasks scripted to remove manual effort and inconsistency: Active Directory user and group administration, reporting, and health checks.
- A sanitised subset of this tooling is published in [PowerShell-Scripts](https://github.com/Dstanfield-Creator/Powershell-Scripts).

### 5. Integration and people

- Worked with cross-functional teams to integrate new technology, including **Cisco**, **Veeam**, **Acronis** and **Fortinet**, with minimal disruption.
- Provided training and support to staff on the new systems and security practices, and wrote the procedures they followed.

## Technologies

Cisco Catalyst switching · Fortinet FortiGate · Veeam Backup & Replication · Acronis Cyber Protect · Windows Server / Active Directory · virtualisation (VMware vSphere) · Microsoft 365 · PowerShell

## Outcome

Significant improvement in network performance and security, reduced operational cost, and higher overall system reliability, with a standard build, backup and audit cadence that outlived the project.

## Skills demonstrated

Network assessment · firewall and IDS policy · segmentation · backup and recovery · PowerShell automation · security awareness training · managed-services change management

## Related

- [Cisco Enterprise Network Design](../../design/cisco-enterprise-network-design/) — a lab design that practises the same segmentation pattern
- [server-administration](https://github.com/Dstanfield-Creator/server-administration) · [guides](https://github.com/Dstanfield-Creator/server-administration/tree/master/guides)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
