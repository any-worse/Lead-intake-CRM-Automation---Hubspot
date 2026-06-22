# Lead-intake-CRM-Automation---Hubspot

## Overview

This project is a business workflow automation built with **n8n** and **HubSpot CRM**.

The workflow receives incoming lead requests through a webhook, validates and normalizes the data, searches HubSpot for an existing contact by email, creates a new contact if needed, checks whether the same deal request already exists, creates a new deal, associates the deal with the correct contact, sends Telegram notifications, and writes internal technical events into an n8n event log.

The main goal of this project is to demonstrate how n8n can be used to automate a real business lead intake process and connect external requests with a CRM system.

---

## Business Problem

Many businesses receive leads from website forms, landing pages, advertising campaigns, partner forms, or external systems.

Without automation, these leads are often processed manually. This can cause:

* delayed response times;
* duplicate CRM records;
* lost or ignored leads;
* manual data entry mistakes;
* poor visibility into lead processing history;
* no clear tracking of failed or duplicate requests.

This workflow solves these problems by automatically processing incoming lead data and sending it to HubSpot CRM in a structured and controlled way.

---

## Business Value

This automation helps a business:

* reduce manual CRM work;
* prevent duplicate deals;
* improve lead response speed;
* keep contacts and deals properly connected in HubSpot;
* notify responsible people about new leads in real time;
* track invalid requests, duplicate deals, and automation errors;
* maintain an internal technical event log for audit and debugging.

---

## Tools Used

* **n8n** — workflow automation platform
* **HubSpot CRM** — contacts and deals management
* **HubSpot CRM Search API** — search contacts and deals by properties
* **Telegram Bot** — real-time notifications
* **n8n Data Tables** — internal technical event logging
* **Webhook** — receiving incoming lead requests
* **HTTP Request node** — HubSpot API search requests
* **HubSpot node** — creating contacts and deals
* **IF / Merge / Edit Fields nodes** — workflow logic and data transformation

---

## Workflow Features

The workflow includes:

* webhook-based lead intake;
* incoming data normalization;
* required field validation;
* contact search in HubSpot by email;
* automatic contact creation if the contact does not exist;
* custom deal duplicate detection through `n8n_deal_key`;
* deal creation in HubSpot;
* contact-to-deal association;
* Telegram notifications for key events;
* internal event logging in n8n Data Tables;
* structured HTTP responses;
* error handling for HubSpot and HTTP request failures.

---

## Workflow Logic

The workflow follows this process:

1. Receive lead data through a Webhook.
2. Normalize incoming fields.
3. Validate required fields: `name`, `email`, and `message`.
4. If required fields are missing, stop the process and return `400 Bad Request`.
5. Search HubSpot Contacts by email.
6. If the contact exists, use the existing contact ID.
7. If the contact does not exist, create a new HubSpot contact.
8. Generate a unique `n8n_deal_key`.
9. Search HubSpot Deals by `n8n_deal_key`.
10. If the deal already exists, return `409 Duplicate`.
11. If the deal does not exist, create a new HubSpot deal.
12. Associate the deal with the correct HubSpot contact.
13. Send a Telegram notification.
14. Save the event into the internal n8n event log.
15. Return a structured HTTP response.

---

## Workflow Structure

```text
Webhook
↓
Normalize Lead
↓
IF - Required Fields?
├── false
│   ↓
│   Log - Invalid Request
│   ↓
│   Telegram - Invalid Request
│   ↓
│   Respond - Invalid Request 400
│
└── true
    ↓
    HTTP - Search Contact by Email
    ↓
    Normalize Contact Search Result
    ↓
    IF - Contact Exists?
    ├── true
    │   ↓
    │   Normalize Existing Contact
    │
    └── false
        ↓
        HubSpot - Create Contact
        ↓
        Normalize New Contact
    ↓
    Merge - Contact Result
    ↓
    HTTP - Search Deal by Deal Key
    ↓
    Normalize Deal Search Result
    ↓
    IF - Deal Exists?
    ├── true
    │   ↓
    │   Log - Duplicate Deal
    │   ↓
    │   Telegram - Duplicate Deal
    │   ↓
    │   Respond - Duplicate 409
    │
    └── false
        ↓
        HubSpot - Create Deal
        ↓
        Log - Deal Created
        ↓
        Telegram - Deal Created
        ↓
        Respond - Created 201
```

---

## Input Example

Example JSON request sent to the webhook:

```json
{
  "name": "Dmitriy Demo",
  "email": "dmitriy.demo@example.com",
  "company": "Automation Demo Company",
  "message": "I want to automate lead processing with n8n and HubSpot",
  "budget": 1500,
  "source": "website_form"
}
```

---

## Normalized Lead Fields

The workflow transforms incoming data into a normalized structure:

```json
{
  "lead_name": "Dmitriy Demo",
  "lead_firstname": "Dmitriy",
  "lead_lastname": "Demo",
  "lead_email": "dmitriy.demo@example.com",
  "lead_company": "Automation Demo Company",
  "lead_message": "I want to automate lead processing with n8n and HubSpot",
  "lead_budget": 1500,
  "lead_quality": "high",
  "lead_source": "website_form",
  "lead_key": "dmitriy.demo@example.com",
  "deal_key": "dmitriy.demo@example.com|website_form|i want to automate lead processing with n8n and hubspot|1500",
  "received_at_utc": "16.06.2026 22:30:53 UTC"
}
```

---

## Duplicate Detection Logic

The workflow does not treat the same email as a full duplicate.

Instead, it separates two concepts:

```text
Contact = a person or lead
Deal = a specific request or sales opportunity
```

A single contact can have multiple deals.

Duplicate deals are detected using the custom HubSpot deal property:

```text
n8n_deal_key
```

The deal key is generated from:

```text
email + source + message + budget
```

This means:

* same email + same request = duplicate deal;
* same email + different request = new deal for existing contact;
* new email + new request = new contact and new deal.

---

## HubSpot Data Model

HubSpot stores the main business data:

* Contacts;
* Deals;
* Contact-to-Deal associations;
* Deal amount;
* Deal description;
* Lead source;
* Lead quality;
* Lead message;
* n8n deal key;
* n8n execution ID.

n8n Data Tables are used only for internal technical logging.

---

## Internal Event Log

The workflow writes internal events into an n8n Data Table called:

```text
lead_events
```

Example event types:

```text
invalid_request
new_contact_and_deal_created
new_deal_for_existing_contact
duplicate_deal
hubspot_error
```

The event log stores:

* event type;
* lead email;
* deal key;
* HubSpot contact ID;
* HubSpot deal ID;
* contact status;
* HTTP status code;
* workflow name;
* execution ID;
* timestamp;
* error stage;
* error message.

---

## Telegram Notifications

The workflow sends Telegram notifications for the following cases:

### New Contact and Deal Created

```text
New contact and deal created in HubSpot

Name: Dmitriy Demo
Email: dmitriy.demo@example.com
Company: Automation Demo Company
Budget: 1500
Quality: high
Source: website_form

Contact ID: 123456
Deal ID: 987654

Time UTC: 16.06.2026 22:30:53 UTC
```

### New Deal for Existing Contact

```text
New deal created for existing contact

Name: Dmitriy Demo
Email: dmitriy.demo@example.com
Company: Automation Demo Company
Budget: 2500
Quality: high
Source: website_form

Contact ID: 123456
Deal ID: 987655

Time UTC: 16.06.2026 22:45:10 UTC
```

### Duplicate Deal

```text
Duplicate deal ignored

Email: dmitriy.demo@example.com
Source: website_form
Message: I want to automate lead processing with n8n and HubSpot
Existing Deal ID: 987654

Reason: Same n8n_deal_key already exists in HubSpot
Time UTC: 16.06.2026 22:50:22 UTC
```

### Invalid Request

```text
Invalid lead request

Email: missing
Message: Need automation
Reason: Missing required fields
Execution ID: 123
Time UTC: 16.06.2026 22:55:01 UTC
```

### Automation Error

```text
Lead automation error

Email: dmitriy.demo@example.com
Deal Key: dmitriy.demo@example.com|website_form|test request|1500
Error: HubSpot API request failed

Execution ID: 124
Time UTC: 16.06.2026 23:00:10 UTC
```

---

## HTTP Responses

The workflow returns structured HTTP responses.

### 201 Created

```json
{
  "status": "created",
  "message": "Deal created in HubSpot",
  "contact_status": "new_contact",
  "crm_contact_id": "123456",
  "crm_deal_id": "987654",
  "deal_key": "dmitriy.demo@example.com|website_form|i want to automate lead processing with n8n and hubspot|1500"
}
```

### 400 Bad Request

```json
{
  "status": "error",
  "message": "Missing required fields: name, email or message"
}
```

### 409 Duplicate

```json
{
  "status": "duplicate",
  "message": "Deal already exists in HubSpot",
  "deal_key": "dmitriy.demo@example.com|website_form|i want to automate lead processing with n8n and hubspot|1500",
  "existing_deal_id": "987654"
}
```

### 500 Automation Error

```json
{
  "status": "error",
  "message": "Lead automation failed",
  "execution_id": "124"
}
```

---

## Test Scenarios

The workflow was tested with the following scenarios.

### 1. New Email + New Request

Expected result:

* new HubSpot Contact is created;
* new HubSpot Deal is created;
* Deal is associated with Contact;
* Telegram notification is sent;
* `lead_events` receives `new_contact_and_deal_created`;
* workflow returns `201 Created`.

### 2. Existing Email + New Request

Expected result:

* existing HubSpot Contact is used;
* new HubSpot Deal is created;
* Deal is associated with existing Contact;
* Telegram notification is sent;
* `lead_events` receives `new_deal_for_existing_contact`;
* workflow returns `201 Created`.

### 3. Existing Email + Same Request

Expected result:

* Contact is not duplicated;
* Deal is not duplicated;
* Telegram duplicate notification is sent;
* `lead_events` receives `duplicate_deal`;
* workflow returns `409 Duplicate`.

### 4. Missing Required Fields

Expected result:

* HubSpot is not called;
* Telegram invalid request notification is sent;
* `lead_events` receives `invalid_request`;
* workflow returns `400 Bad Request`.

### 5. HubSpot or HTTP Request Failure

Expected result:

* workflow does not fail silently;
* error branch is triggered;
* Telegram error notification is sent;
* `lead_events` receives `hubspot_error`;
* workflow returns `500 Automation Error`.

---

## Screenshots

Recommended screenshots for portfolio:

```text
/screenshots/workflow-overview.png
/screenshots/normalize-lead-output.png
/screenshots/hubspot-contact-deal-association.png
/screenshots/lead-events-table.png
/screenshots/telegram-notification.png
```

---

## Suggested Project Structure

```text
lead-intake-hubspot/
├── README.md
├── workflow.json
├── test-data/
│   ├── new-lead.json
│   ├── existing-contact-new-deal.json
│   ├── duplicate-deal.json
│   └── invalid-request.json
└── screenshots/
    ├── workflow-overview.png
    ├── normalize-lead-output.png
    ├── hubspot-contact-deal-association.png
    ├── lead-events-table.png
    └── telegram-notification.png
```

---

## Limitations

This is a portfolio demo workflow.

In a production environment, the workflow could be extended with:

* automatic owner assignment;
* advanced lead scoring;
* company creation and company-to-deal association;
* email follow-up automation;
* CRM pipeline routing;
* Slack or Microsoft Teams notifications;
* dashboard reporting;
* retry logic for temporary API failures;
* monitoring and alerting for workflow health.

---

## Conclusion

This project demonstrates how n8n can be used as a business automation layer between external lead sources and a CRM system.

The workflow solves a real business problem: it receives incoming leads, validates them, prevents duplicate deals, creates or reuses CRM contacts, creates HubSpot deals, connects deals with contacts, sends real-time notifications, and keeps an internal technical audit log.

The project shows practical experience with n8n workflow design, HubSpot CRM integration, HTTP API search requests, data normalization, duplicate detection, Telegram notifications, structured responses, and error handling.
