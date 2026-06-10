# Sensitive Data Guidelines

**FOR INTERNAL USE ONLY**

Document Owner: Kevin
Last updated: 9/29/21

**Objective 1:** Reduce exposure and surface area of sensitive data
**Objective 2:** Minimize access to sensitive data (with processes in place to manage access and timely audits)

---

## Sensitive Data Classification

"Sensitive Data" and "PII" are used interchangeably to refer to any data that falls into any of the classifications below (except Level 0).

### Definitions of PII

> "(1) any information that can be used to distinguish or trace an individual's identity, such as name, social security number, date and place of birth, mother's maiden name, or biometric records; and (2) any other information that is linked or linkable to an individual, such as medical, educational, financial, and employment information."

---

## Levels / Classifications

### Level 0 — Normal (public) data

Any data that is not categorized as sensitive.

### Level 1 — Linkable PII or sensitive/private data

Data where more than one person could share the traits. However, when linked or combined with other data, could be used to identify a specific person.

Examples: first name, DOB, gender, zip code, employer — not uniquely identifying alone, but identifying in combination. Also includes sensitive/private data once a user is identified (bankruptcies, purchase history, debts, etc.).

### Level 2 — PII (Direct)

Any information that can be used to directly identify an individual.

Examples: email, full name, home address, personal phone number(s)

### Level 3 — Highly Sensitive / Private Data

Data which, if lost, compromised, or disclosed without authorization, could result in substantial harm, embarrassment, inconvenience, or unfairness to an individual.

Examples: SSN, bank account numbers, passport number, medical information, credentials/passwords, tokens

**Current policy: never store SSN.**

### Level 4 — PCI Data

Any data related to a credit or debit card.

Examples: card number, expiration date, CVC/CVC2

**Current policy: never store PCI data.**

---

## Handling Sensitive Data

As a general principle, sensitive data should only be stored in designated and approved locations. Sensitive data should not be copied/duplicated, downloaded, or shared outside of Savi.

### Storing Sensitive Data (in Savi Infrastructure)

Do not store any sensitive data unless there is a specific business or technical requirement to do so.

| Level | Storage rule |
|---|---|
| Level 0, 1, 2 | Only in Savi encrypted databases |
| Level 3 | Only in Savi encrypted databases; also encrypted at the application level. Follow documented best practices for storing credentials/passwords. |
| Level 4 | Never store |

### Sharing/Sending Sensitive Data (with 3rd Party Services)

3rd party services include Slack, Segment, Mixpanel, Fullstory, Docusign, email services, JIRA, Asana, Zendesk, etc.

| Level | Rule |
|---|---|
| Level 0, 1 | Minimize the data sent to the minimum needed. Use best judgment and ask if unsure. |
| Level 2 | Same as Level 1, but an even higher bar must be met. Default policy: do not share. |
| Level 3, 4 | Never send/include. |

### Internal Reports (with sensitive data)

Internal reports are only seen and shared within Savi. Never shared outside Savi.

| Level | Rule |
|---|---|
| Level 0, 1 | No special considerations |
| Level 2 | Minimize sensitive data to the minimum required. Reports must be password protected and shared via Slack. |
| Level 3, 4 | Never include in any reports. |

### External Reports (with sensitive data)

External reports are any data sent or shared outside of Savi. Whenever possible, external reports should not include any sensitive data. Anonymous or aggregated metrics are strongly preferred.

Requests for external reports must be submitted through Typeform. All Level 1 & 2 data fields must get sign-off from Tobin before the request moves forward.

| Level | Rule |
|---|---|
| Level 0 | No special considerations |
| Level 1 | Requires pre-approval from manager, then Tobin |
| Level 2 | Requires pre-approval from manager, then Kevin, then Tobin |
| Level 3, 4 | Never include in any reports |

**Sending external reports with sensitive data:** sensitive data can only be sent outside of Savi via SFTP or by password-protected file + secure email. Passwords may be shared via email, but in a separate email thread.

---

## Savi Sensitive Data Storage

Highlighted fields are currently stored in the Savi DB.

### Level 1 — Linkable PII or sensitive/private data

- Geographic indicators: country, state, city, zip code
- Gender, race
- Date of birth / age, place of birth
- Business telephone number
- Religion
- Employment history/information
- Education history/information

### Level 2 — PII (Direct)

- Full name
- Home address
- Personal email address, work email address
- Personal telephone number(s)
- Owned property info (e.g. VIN)
- Device identifiers: processor/device serial number, MAC address, IP address, device IDs, cookies
- Photographic images (particularly face or other identifying characteristics), fingerprints, handwriting
- Loan information (NSLDS files)

### Level 3 — Highly Sensitive / Private Data

- SSN (not stored in DB, but exists in Docusign, Zendesk, Asana)
- Passport number, driver's license number, state ID, taxpayer ID, patient ID
- Medical information
- Financial/bank account numbers (Asset Sync + Income endpoint)
- Credentials/tokens: any credentials/passwords to Savi or external services, access/refresh tokens, private keys

### Level 4 — PCI Data

- Card number, expiration date, CVC/CVC2, card PIN, billing address

---

## Code Review Checklist

When reviewing a diff that handles user data, check:

- [ ] No Level 3 or 4 data stored, logged, or returned to the frontend
- [ ] No Level 3 or 4 data sent to any third-party service
- [ ] Level 2 data minimized — only stored/transmitted where there is a specific requirement
- [ ] Level 1 data not combined in ways that would make it uniquely identifying unless required
- [ ] Credentials, tokens, and passwords stored encrypted (Level 3) — never in plaintext, never logged
- [ ] No sensitive data included in error messages, stack traces, or debug output
- [ ] Any new external API integration does not send Level 2+ data without explicit approval