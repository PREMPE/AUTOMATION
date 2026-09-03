# SOC Security Automation with n8n and Splunk

## Overview

I built this project to design and implement an end-to-end **SOC security automation and incident triage workflow** using **n8n** and **Splunk Enterprise**.

My goal was to create an automation pipeline that could receive security alerts, normalize incoming data, validate the alert, determine its severity, automatically apply the appropriate triage logic, enrich the event into a structured incident, and forward the completed incident to Splunk for centralized investigation and monitoring.

Rather than simply forwarding raw alerts from one system to another, I designed the workflow to make decisions based on the severity of each alert.

The completed workflow follows this process:

**Security Alert → Webhook → Normalization → Validation → Severity Routing → Automated Triage → Incident Enrichment → Splunk HEC → SOC Investigation**

I implemented separate processing paths for:

- **High severity** — immediate escalation and review
- **Medium severity** — analyst review
- **Low severity** — logging and monitoring
- **Invalid alerts** — rejected from the normal processing pipeline

I also implemented response checking and failure handling so that the workflow can distinguish between successful Splunk ingestion and an automation failure.

The project gave me hands-on experience building a security automation workflow across a segmented lab network rather than simply creating an isolated n8n demonstration.

---

# Project Objectives

When I started this project, I wanted the automation to perform several functions that would normally require repetitive analyst interaction.

My main objectives were to:

- Create a dedicated security automation server.
- Deploy and configure n8n.
- Build a webhook-based SOC alert ingestion point.
- Normalize inconsistent incoming severity values.
- Validate alerts before processing them.
- Route incidents according to severity.
- Apply different triage logic to High, Medium, and Low alerts.
- Generate structured incident records.
- Add SOC-relevant context to each incident.
- Forward completed incidents to Splunk through HEC.
- Store automation-generated incidents in a dedicated Splunk index.
- Verify successful ingestion from the automation workflow.
- Build failure-handling logic for unsuccessful Splunk submissions.
- Test every severity branch independently.
- Confirm the resulting incident data inside Splunk.

The result was a working automation pipeline connecting my automation infrastructure with my existing Splunk environment.

---

# Lab Architecture

I built the project inside my VMware-based cybersecurity home lab.

The main systems involved in this automation were:

| Component | Role |
|---|---|
| AUTOMATION01 | Linux server hosting Docker and n8n |
| n8n | Workflow orchestration and security automation |
| SPLUNK01 | Splunk Enterprise SIEM |
| Splunk HEC | HTTP-based ingestion interface |
| pfSense | Firewall, routing, and network segmentation |
| VMware Workstation | Virtualization platform |

The two primary servers are separated across different network segments.

| System | IP Address | Network | Purpose |
|---|---|---|---|
| AUTOMATION01 | `172.16.30.20` | Management | n8n security automation |
| SPLUNK01 | `172.16.10.30` | Servers | Splunk Enterprise |

AUTOMATION01 resides on:

`172.16.30.0/24`

while SPLUNK01 resides on:

`172.16.10.0/24`

Traffic between these networks passes through pfSense.

This was important to my implementation because the automation server had to communicate with Splunk across the segmented environment instead of both applications existing on the same host or subnet.

---

# n8n Deployment

I deployed n8n on my dedicated **AUTOMATION01** Linux virtual machine.

I used Docker to host the n8n service and configured the system with the static address:

`172.16.30.20`

The n8n web interface is hosted internally on port:

`5678`

Once n8n was operational, I created the security automation workflow and configured its webhook as the entry point for incoming SOC alerts.

The production webhook uses the path:

`/webhook/soc-alert`

I also used n8n's test webhook functionality during development before moving the workflow to the production webhook.

This gave me a controlled way to test individual stages while building the automation.

---

# Workflow Design

I designed the workflow around a sequence of security-processing stages rather than sending incoming data directly to Splunk.

The main workflow consists of:

```text
Webhook
   ↓
Normalize Alert
   ↓
Validate Alert
   ↓
Severity Router
   ├── HIGH
   ├── MEDIUM
   └── LOW

Invalid alerts are redirected before severity routing.

Each valid severity branch then performs its own triage and enrichment before forwarding the completed incident to Splunk.

Conceptually, the full workflow is:

                         ┌── HIGH → Escalate → Format → Splunk → Check Response
                         │
Webhook → Normalize → Validate → Severity Router
                         │
                         ├── MEDIUM → Review → Format → Splunk → Check Response
                         │
                         └── LOW → Log → Format → Splunk → Check Response

                  Invalid Alert
                       ↑
                 Validation Failure

Failed Splunk responses → Log Automation Failure

This design separates ingestion, validation, decision-making, enrichment, SIEM forwarding, and failure handling into individual stages.

1. SOC Alert Ingestion

I configured an n8n Webhook node as the entry point for the automation.

The webhook accepts HTTP POST requests containing security alert information.

The alert structure I used includes fields such as:

alert_name
severity
host
username
source_ip
mitre_tactic
mitre_technique
detection_source

For example, one of my High-severity test alerts represented suspicious PowerShell execution on the MANAGEMENT system.

The alert contained information such as:

Alert Name: Suspicious PowerShell Execution
Severity: High
Host: MANAGEMENT
Username: Administrator
Source IP: 172.16.30.10
MITRE Tactic: Execution
MITRE Technique: PowerShell
Detection Source: Splunk

Using a webhook gives the workflow a reusable ingestion mechanism. The automation does not depend on manually entering incidents directly into n8n.

2. Alert Normalization

After receiving an alert, I send it to the Normalize Alert stage.

I added this stage because incoming security tools do not always represent severity values in exactly the same format.

For example, the same severity could potentially arrive as:

HIGH
High
high
 high

If I routed directly against the original value, formatting differences could cause an otherwise valid alert to miss its intended workflow branch.

I therefore created a normalized_severity field.

The normalization logic:

Reads the severity from the incoming alert.
Converts the value to a string.
Removes unnecessary whitespace.
Converts the value to lowercase.
Uses unknown when no usable severity is available.

The resulting field looks like:

normalized_severity: high

This gives the rest of my automation a predictable value to evaluate.

3. Alert Validation

After normalization, I send the alert into the Validate alert stage.

I designed this stage so that malformed or unsupported alerts do not automatically enter my normal incident-processing pipeline.

The workflow expects one of three recognized severity values:

high
medium
low

When an alert contains a valid severity, it continues to the Severity Router.

When the alert fails validation, I send it to:

Invalid Alerts

For example, I tested an alert where the normalized severity became:

unknown

The validation node correctly sent the event through its False branch instead of allowing it into the normal triage workflow.

This gave the automation an important control point between external input and internal incident processing.

4. Severity-Based Routing

After validation, the alert reaches my Severity Router.

I configured three explicit routing rules using the normalized_severity field.

The routing logic is:

normalized_severity	Route
high	Output 0
medium	Output 1
low	Output 2

Each output connects to a separate incident-handling workflow.

This means the automation does not treat every security alert identically.

Instead, the workflow determines the appropriate response based on the risk represented by the alert.

5. High-Severity Incident Handling

High-severity alerts are routed to:

HIGH - Escalate Incident

I designed this path for incidents that require immediate analyst attention.

The High workflow follows:

Severity Router
      ↓
HIGH - Escalate Incident
      ↓
Format High Incident
      ↓
Send HIGH Incident to Splunk
      ↓
Check HIGH Splunk Response

During triage, I mark the incident as escalated and assign an immediate-review response.

Important fields include:

severity: high
escalated: true
response_action: immediate_review

I also add an analyst note indicating that n8n automatically triaged and escalated the incident.

One of my successful High-severity tests used:

alert_name: Suspicious PowerShell Execution
severity: high
host: MANAGEMENT
username: Administrator
source_ip: 172.16.30.10
mitre_tactic: Execution
mitre_technique: PowerShell
detection_source: Splunk
escalated: true
response_action: immediate_review

This branch demonstrates automated escalation rather than simply storing the alert.

6. Medium-Severity Incident Handling

Medium-severity alerts follow a different path:

MEDIUM - Review Incident

The workflow is:

Severity Router
      ↓
MEDIUM - Review Incident
      ↓
Format Medium Incident
      ↓
Send MEDIUM Incident to Splunk
      ↓
Check MEDIUM Splunk Response

For these incidents, I designed the automation to perform the initial triage but leave the incident marked for analyst review.

The resulting fields include values such as:

severity: medium
priority: P2
escalated: false
analyst_review_required: true
response_action: analyst_review
automation_status: triaged

This creates a distinction between an incident requiring immediate escalation and one that requires human review but does not need the same level of urgency.

7. Low-Severity Incident Handling

Low-severity alerts are sent to:

NORMAL - Log Incident

The workflow is:

Severity Router
      ↓
NORMAL - Log Incident
      ↓
Format Normal Incident
      ↓
Send LOW Incident to Splunk
      ↓
Check LOW Splunk Response

For Low-severity events, my objective was visibility without unnecessary escalation.

The resulting incident can include:

severity: low
priority: P3
escalated: false
incident_status: MONITOR
response_action: log_and_monitor

I tested this branch using a Multiple Failed Logins alert.

The workflow successfully classified the event as Low severity, processed it through the NORMAL branch, formatted the incident, and sent it to Splunk.

This allows lower-priority security activity to remain available for investigation and correlation without treating every event as an immediate incident.

8. Incident Enrichment and Formatting

I did not want Splunk to receive only the original alert.

Before sending an incident to the SIEM, I use formatting nodes to convert the incoming alert into a more useful incident record.

My workflow generates fields such as:

incident_id
created_at
alert_name
severity
priority
host
username
source_ip
mitre_tactic
mitre_technique
detection_source
escalated
analyst_review_required
incident_status
response_action
automation_status
analyst_note

Not every field is required for every severity.

For example, a Medium incident can contain analyst_review_required, while a High incident uses escalation information.

This allows the final Splunk record to describe not only what happened, but also what the automation decided to do about it.

9. Incident ID Generation

I generate a unique incident identifier for the incidents processed by the workflow.

The IDs use the format:

INC-xxxxxxxxxxxxx

For example:

INC-1788440854927

The incident ID gives me a consistent identifier that can be used to distinguish individual automated incidents inside Splunk.

Combined with the creation timestamp, this makes the records easier to track during investigation.

10. MITRE ATT&CK Context

I included MITRE ATT&CK information as part of the structured incident data.

Examples from my testing include:

mitre_tactic: Execution
mitre_technique: PowerShell

and:

mitre_tactic: Credential Access
mitre_technique: Brute Force

This allows the automation-generated incident to retain useful security context instead of reducing the event to only a generic alert name and severity.

11. Splunk HTTP Event Collector Integration

Once an incident has been triaged and formatted, I send it from n8n to Splunk Enterprise using the HTTP Event Collector (HEC).

My Splunk server is:

172.16.10.30

HEC listens on:

8088

The automation therefore communicates with the Splunk HEC service over port 8088.

I created a dedicated Splunk index for the project:

index=soc_automation

I also configured the events with:

source = n8n-security-automation
sourcetype = n8n:soc:incident

Using a dedicated index makes it easy for me to separate automation-generated incidents from the other Windows, Active Directory, firewall, Sysmon, and security telemetry already being collected in my Splunk environment.

12. Splunk Response Validation

Sending an HTTP request does not automatically mean that Splunk accepted the event.

Because of this, I added a response-checking stage after each HEC request.

I created:

Check HIGH Splunk Response
Check MEDIUM Splunk Response
Check LOW Splunk Response

A successful Splunk HEC submission returns:

text: Success
code: 0

I use that response to determine whether the incident was successfully delivered.

This gave me a way to validate the integration from inside the automation workflow rather than assuming that an HTTP node completing means the incident was indexed successfully.

13. Automation Failure Handling

I also created a centralized:

Log Automation Failure

node.

The failure paths from the Splunk response checks can be directed to this node.

The purpose of this design is to keep integration failures visible.

Without failure handling, an automation could successfully receive and triage an alert but silently fail while trying to forward it to the SIEM.

That would create a visibility gap.

By separating successful and failed responses, I can distinguish between:

Alert processing succeeded

and:

Alert processing succeeded, but SIEM delivery failed

Those are very different operational outcomes.

Troubleshooting the Splunk Integration

One of the most valuable parts of this project was troubleshooting communication between AUTOMATION01 and SPLUNK01.

The workflow logic was functioning, but n8n initially could not successfully communicate with the Splunk HEC service.

At one point, I received:

ECONNREFUSED

during a request to Splunk.

Because AUTOMATION01 and SPLUNK01 are located on separate VLAN-style lab segments, I had to investigate more than just the n8n HTTP Request node.

I worked through the communication path between:

n8n
 ↓
AUTOMATION01
 ↓
Management Network
 ↓
pfSense
 ↓
Server Network
 ↓
SPLUNK01
 ↓
Splunk HEC

This required me to examine host networking, routing, firewall behavior, the Splunk server configuration, and the HEC listener.

Splunk Network Configuration Issue

During troubleshooting, I discovered that the Splunk Linux server had conflicting network configuration.

SPLUNK01 was intended to use the static address:

172.16.10.30

but a cloud-init/DHCP configuration was also assigning another address to the same interface.

The server therefore had both its intended static configuration and an unwanted DHCP address.

I corrected the configuration by removing the conflicting cloud-init network configuration and keeping the intended static network settings.

After rebooting, SPLUNK01 had the expected static address:

172.16.10.30/24

and the expected default gateway:

172.16.10.1

I also verified that Splunk HEC was listening on port:

8088

This was an important troubleshooting lesson because the problem was not simply an n8n workflow error.

Segmented Network Troubleshooting

I also verified the network path between the Management and Server networks.

AUTOMATION01 was correctly configured with:

IP: 172.16.30.20
Gateway: 172.16.30.1

SPLUNK01 was configured with:

IP: 172.16.10.30
Gateway: 172.16.10.1

Both networks are routed by pfSense.

I verified the VMware virtual network assignments and confirmed that the automation VM was attached to the correct Management network.

I also reviewed pfSense interface assignments, routing information, firewall rules, and the communication path between the two systems.

This allowed me to troubleshoot the integration as a complete networked application instead of treating n8n and Splunk as isolated services.

Connectivity Restored

After resolving the connectivity problems, I successfully sent incidents from n8n to Splunk.

The Splunk HEC response returned:

Success
code: 0

I then confirmed that the events actually appeared inside:

index=soc_automation

This gave me confirmation at both ends:

n8n → HEC request successful

and:

Splunk → incident indexed successfully

I captured evidence of both the failed connection and the restored integration because the troubleshooting process is an important part of the project.

Splunk Incident Investigation

Once the integration was working, I used Splunk Search & Reporting to investigate the automated incidents.

My basic search was:

index=soc_automation
| sort - _time

This allowed me to inspect the full structured events.

I could see fields including:

alert_name
analyst_note
created_at
detection_source
escalated
incident_id
incident_status
mitre_tactic
mitre_technique
priority
response_action
severity
source_ip
username

I then created a more focused SOC triage view:

index=soc_automation
| table _time incident_id alert_name severity priority host username escalated analyst_review_required response_action automation_status
| sort - _time

This produced a table where I could quickly compare incidents and see how the automation handled each severity.

High-Severity Test

I tested the High workflow with:

Suspicious PowerShell Execution

The automation successfully:

Received the alert through the webhook.
Normalized the severity.
Validated the alert.
Routed it through the High branch.
Escalated the incident.
Generated a unique incident ID.
Added the creation timestamp.
Preserved host and user information.
Added MITRE ATT&CK context.
Assigned immediate_review.
Formatted the completed incident.
Sent the incident to Splunk HEC.
Received a successful HEC response.
Indexed the incident in soc_automation.

The resulting event contained:

alert_name: Suspicious PowerShell Execution
severity: high
host: MANAGEMENT
username: Administrator
source_ip: 172.16.30.10
mitre_tactic: Execution
mitre_technique: PowerShell
escalated: true
response_action: immediate_review
Medium-Severity Test

I also tested the Medium branch.

The alert was successfully routed to:

MEDIUM - Review Incident

The resulting incident contained values such as:

severity: medium
priority: P2
escalated: false
analyst_review_required: true
response_action: analyst_review
automation_status: triaged

The event was then successfully delivered to Splunk.

This confirmed that the Medium route operates independently from the High-severity escalation path.

Low-Severity Test

For the Low workflow, I tested:

Multiple Failed Logins

The automation routed the alert to:

NORMAL - Log Incident

The resulting incident contained:

alert_name: Multiple Failed Logins
severity: low
priority: P3
escalated: false
incident_status: MONITOR
response_action: log_and_monitor

The incident was successfully sent to Splunk and appeared alongside the High and Medium incidents.

This confirmed that all three routing outputs were operational.

Invalid Alert Test

I also intentionally tested an alert that did not contain a recognized severity.

The normalization process produced:

normalized_severity: unknown

The Validate alert node rejected it from the normal pipeline and sent it through the False branch to:

Invalid Alerts

The invalid event did not enter the Severity Router.

This test confirmed that my validation logic was working as intended and that unsupported alert data could not accidentally enter one of the legitimate incident-handling paths.

Final Automation Logic

The completed triage logic is:

Severity	Priority	Escalated	Analyst Review	Response
High	Immediate	Yes	Immediate	immediate_review
Medium	P2	No	Yes	analyst_review
Low	P3	No	No	log_and_monitor
Invalid	N/A	N/A	N/A	Rejected from normal pipeline

This allows the same webhook to receive different security events while n8n automatically determines how each one should be handled.

Evidence

I documented the project with screenshots covering both the successful implementation and the troubleshooting process.

n8n Workflow

n8n-security-automation-workflow.png

Shows the complete automation architecture, including alert ingestion, normalization, validation, severity routing, High/Medium/Low processing, Splunk integration, response validation, and failure handling.

Webhook Ingestion

n8n-soc-alert-webhook.png

Shows the SOC alert webhook used as the entry point for incoming security events.

Alert Normalization

n8n-alert-normalization.png

Shows the normalization stage used to generate a consistent normalized_severity value.

Alert Validation Failure

n8n-alert-validation-failure.png

Shows an unsupported unknown severity being rejected and routed toward the Invalid Alerts branch.

Severity Routing Configuration

n8n-severity-routing-configuration.png

Shows the three routing rules for High, Medium, and Low severity incidents.

High-Severity Execution

n8n-high-severity-execution.png

Shows a successful High-severity workflow execution.

Medium-Severity Execution

n8n-medium-severity-execution.png

Shows a successful Medium-severity workflow execution.

Low-Severity Execution

n8n-low-severity-execution.png

Shows a successful Low-severity workflow execution.

High Incident Enrichment

n8n-high-incident-enrichment.png

Shows the structured High-severity incident fields produced by the enrichment and formatting process.

Splunk HEC Success

n8n-splunk-hec-success.png

Shows Splunk HEC returning:

Success
code: 0

after n8n successfully submitted an incident.

Automated Incidents in Splunk

splunk-automated-incidents-overview.png

splunk-automated-incidents-details.png

Show the structured incidents successfully indexed inside Splunk.

Incident Triage Results

splunk-incident-triage-results.png

Shows High, Medium, and Low incidents together in a structured Splunk table, demonstrating the different automated response decisions.

HEC Connection Failure

splunk-hec-connection-failure.png

Documents a failed Splunk HEC connection during implementation.

HEC Connectivity Restored

splunk-hec-connectivity-restored.png

Documents the successful HEC integration after troubleshooting the connectivity problem.

Skills Demonstrated

Through this project, I gained practical experience with:

SOC security automation
Security orchestration
n8n workflow development
Splunk Enterprise
Splunk HTTP Event Collector
SIEM integration
HTTP webhooks
REST-style communication
JSON processing
Alert normalization
Input validation
Conditional workflow logic
Severity-based routing
Automated incident triage
Incident enrichment
Incident prioritization
MITRE ATT&CK mapping
Error handling
Linux server administration
Docker
VMware Workstation
pfSense
Network segmentation
Inter-subnet communication
Firewall troubleshooting
Application connectivity troubleshooting
SIEM searching
Structured security-event analysis
SOC workflow design
Security Considerations

I treated this as a controlled home-lab implementation.

I did not expose sensitive Splunk HEC credentials in my public project documentation.

In particular, authentication tokens and other secrets should not be committed to a public repository.

The automation server and Splunk server also remain separated across their respective lab network segments rather than being exposed directly as public services.

Project Outcome

I successfully built an end-to-end security automation pipeline that receives SOC alerts, normalizes their severity, validates incoming data, routes alerts according to risk, enriches them into structured incidents, forwards them to Splunk, verifies the HEC response, and makes the resulting incidents searchable by a SOC analyst.

The final workflow handles incidents differently based on their severity:

HIGH
→ Validate
→ Escalate
→ Enrich
→ Immediate Review
→ Splunk

MEDIUM
→ Validate
→ Triage
→ Enrich
→ Analyst Review
→ Splunk

LOW
→ Validate
→ Log
→ Enrich
→ Monitor
→ Splunk

INVALID
→ Reject from normal incident pipeline

Most importantly, I did not stop at proving that individual n8n nodes worked.

I verified the complete path from alert ingestion to automated decision-making to Splunk ingestion and SOC investigation.

I also encountered and resolved real integration and networking problems between the automation server and Splunk. That troubleshooting required me to work across the workflow, Linux networking, VMware networking, pfSense routing/firewall configuration, and Splunk HEC rather than treating the problem as a single application error.

The completed project demonstrates my ability to combine security operations, automation, SIEM engineering, Linux administration, and network troubleshooting into one functioning security workflow.
