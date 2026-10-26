# AI-Powered Customer Support Ticketing System

**Platform:** Salesforce Service Cloud, Salesforce Flow Builder, Agentforce, Lightning App Builder, and Salesforce Reports  
**Project context:** SkillWallet Salesforce Administrator with AI Agentforce capstone  
**Industry scenario:** Telecommunications customer support

## Project overview

This project configures a Salesforce-based support workflow for telecom incidents such as broadband outages, eSIM activation problems, roaming-billing disputes, and regional network outages. It combines Case fields, SLA rules, record-triggered Flows, email alerts, support profiles, reports, and an Agentforce assistant configuration.

**Important implementation note:** The before-save Flow deterministically populates the `Resolution_Time__c`, `SLA_Due_Date__c`, and `AI_Suggested_Reply__c` Case fields according to its configured logic. In the live Agentforce conversation-preview test, the assistant responded and offered escalation, but did **not** provide the requested broadband troubleshooting guidance. Agentforce preview execution is verified; successful end-to-end AI troubleshooting is **not** verified and remains a limitation.

## Features and components

- Custom objects: `Support_Category__c`, `SLA_Policy__c`, and `AI_Feedback__c`.
- Custom Case fields for issue type, support category, SLA deadline, resolution-time target, and suggested reply.
- High-priority validation rule: `SLA_Due_Date_Required_For_High_Priority`.
- Before-save Flow: `Case_Before_Save_SLA_and_Triage_Flow`.
- After-save notification Flow: `Case_SLA_and_Triage_Automation_Flow`.
- Resolution email Flow: `Case_Resolution_Email_Flow`.
- Case summary Flow and Agentforce metadata components.
- Lightning application: `AI_Powered_CST_System`.
- Reports: `Open_Cases_by_Priority` and `SLA_Breach_and_Resolution_Time_Analysis`.
- Support Agent and Support Manager profiles.
- Three email templates and associated email alerts.

## SLA behavior

The configured target windows are:

| Priority | Target resolution time |
|---|---:|
| High | 4 hours |
| Medium | 12 hours |
| Low | 24 hours |

The active validation rule rejects a High-priority Case if its SLA deadline is blank. The before-save Flow can populate the deadline when the configured conditions are met. Validate the interaction between the validation rule and Flow in any target org before relying on it operationally.

## Automation design

### `Case_Before_Save_SLA_and_Triage_Flow`
A record-triggered before-save Flow on Case. It assigns the configured resolution target, calculates the SLA deadline under its configured conditions, and populates the suggested-reply field. Before-save field updates avoid a separate self-update of the Case record.

### `Case_SLA_and_Triage_Automation_Flow`
A record-triggered after-save Flow. The included metadata contains notification paths for a transition to High priority and a transition to the `Escalated` status. The Flow metadata does not contain a self-record-update element.

### `Case_Resolution_Email_Flow`
A record-triggered after-save Flow intended to send the configured resolution notification when a Case transitions to `Closed`.

## Reports

- **Open Cases by Priority:** groups open Cases by priority.
- **SLA Breach & Resolution Time Analysis:** includes the `SLA Breach Status` row-level formula (`CDF1`) and SLA/resolution-time analysis.

The formula in the packaged report metadata uses these exact output labels: `No SLA Target`, `SLA Breached`, `Within SLA`, `SLA Breached (Overdue)`, and `Within SLA (Active)`. Report formulas are report-level calculations, not persistent Case fields.

## Live development-org verification reported

The following results were reported from a live Salesforce Developer Edition test. Case numbers and auto-numbered feedback IDs are specific to that development org and will differ in other orgs.

| Record | Scenario | Reported result |
|---|---|---|
| Case `00001026` | Broadband Fiber Connectivity Failure | High priority; target 4 hours; validation and before-save automation tested; report classified it as overdue |
| Case `00001027` | Duplicate International Roaming Billing Dispute | Medium priority; target 12 hours; closed; report classified it as within SLA |
| Case `00001028` | 5G eSIM Activation Failure | Medium priority; target 12 hours; report classified it as active/within SLA |
| Case `00001029` | Regional Cell Tower Outage | High priority; escalated; past SLA deadline retained for breach testing; breach notification path triggered |
| `AF-00001` | Feedback linked to Case `00001026` | Saved with `Helpful` status and `Suggested Reply` type |
| `AF-00002` | Feedback linked to Case `00001027` | Saved with `Helpful` status and `Case Classification` type |

These are reported test outcomes, not portable seeded data. The package contains metadata; it does not automatically recreate these live records in another org.

## Agentforce test result and limitation

The `AI_Customer_Support_Assistant` Version 1 was active in the development org. A conversation-preview test asked how to classify, prioritize, and troubleshoot a broadband outage with a red LOS light. The assistant responded that it could not retrieve the specific steps and offered escalation to a live agent.

Therefore:
- **Verified:** the conversation preview ran and returned a response.
- **Not verified:** successful end-to-end invocation of the custom Case summary action or grounded troubleshooting guidance.
- **Recommended follow-up:** inspect the agent's topic/action configuration and knowledge grounding in the target org, then repeat the preview test with representative cases.

## Repository structure

```text
force-app/main/default/
  applications/
  bots/
  email/
  flows/
  genAiFunctions/
  genAiPlanners/
  genAiPlugins/
  objects/
  profiles/
  reports/
  tabs/
  workflows/
manifest/package.xml
sfdx-project.json
README.md
AI_Powered_CST_System_Documentation.pdf
```

## Deploying to a separate Salesforce org

This repository is Salesforce DX source metadata. Deployment has **not** been validated with Salesforce CLI in the environment used for this package review.

1. Install Salesforce CLI and authenticate to a disposable development/test org.
2. Confirm that required Salesforce features and Agentforce/Einstein capabilities are enabled in the target org.
3. Review profile permissions and org-specific settings. The SLA breach warning alert is configured for the Case owner to avoid a hardcoded development-org user recipient; its email template addresses the Support Team.
4. Validate the deployment before deploying:
   ```bash
   sf project deploy validate --manifest manifest/package.xml --target-org <your-org-alias>
   ```
5. If validation reports missing dependencies or feature limitations, resolve them before deployment.
6. After deployment, configure any org-specific Agentforce setup and run the functional test cases described above.

Do not use this package as proof of successful deployment until the Salesforce CLI validation and post-deployment tests pass in the intended target org.

## Package validation status

At the time of preparation, the package was reported to have passed ZIP integrity, XML parsing, manifest cross-reference, and credential scanning checks. Those checks are distinct from Salesforce deployment validation. This refined package adds explicit `CustomField` entries to the manifest, aligns the README formula labels with the report metadata, and removes a hardcoded development-org email recipient from the SLA breach warning alert.

## Security and publishing notes

- Do not commit Salesforce passwords, session IDs, access tokens, private keys, or other credentials.
- Do not publish org IDs, internal org URLs, or user-specific email addresses unless there is a clear reason and explicit approval.
- The live Case numbers and AI Feedback auto-numbers are development-org records and are included only as verification references.
