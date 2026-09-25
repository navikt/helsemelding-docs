# Helsemelding — Kafka Topics

This document describes the Kafka topics used by the Helsemelding platform. It is intended for
internal teams that want to integrate their systems with the platform to send or receive dialog messages.

---

## Overview

The Helsemelding platform bridges internal NAV systems and external healthcare providers. Communication happens
over Kafka topics on the **Aiven** (`nav-dev` / `nav-prod`) platform.

```
  EPJ                                                                                    Fagsystem
   │                                                                                         │
   │────▶ edi-adapter ───▶ [dialog.in.xml] ──▶ inbound-processing ───▶ [dialog.in] ──▶ consume
   │                                                                                         │
   │◀─── edi-adapter ◀── [dialog.out.xml] ◀── outbound-processing ◀── [dialog.out] ◀─── produce
   │                                                  │                                      │
   │                                        [dialog.out.status] ────────────────────────▶ consume (delivery status)
   │                                        [dialog.out.error]  ────────────────────────▶ consume (validation errors)
```

**Topics relevant to internal departments:**

| Topic | Action | Format | Retention | Cleanup policy | Purpose |
|---|---|---|---|---|---|
| `helsemelding.dialog.in` | **Consume** | JSON | 7 days | delete | Receive inbound messages from external healthcare providers |
| `helsemelding.dialog.out` | **Produce** | JSON | 7 days | delete | Send outbound messages to external healthcare providers |
| `helsemelding.dialog.out.status` | **Consume** | JSON | 7 days | compact,delete | Track delivery status of outbound messages |
| `helsemelding.dialog.out.error` | **Consume** | JSON | 7 days | delete | Handle messages rejected during outbound validation |

---

## Common Record Format

All topics use **String serialization** for both key and value.

### Record Key

The Kafka record key **must** be a valid UUID.  
Example: `3fa85f64-5717-4562-b3fc-2c963f66afa6`

Records with a missing or non-UUID key are rejected and routed to the error topic.

### Kafka Headers

The following header is **required** on records produced to the outbound JSON topic:

| Header | Type | Description |
|---|---|---|
| `sourceSystem` | String | Identifier of the producing application (e.g. your Nais application name), ytelse or område |

This value will be specified in status updates (in `helsemelding.dialog.out.status`) and error messages (in `helsemelding.dialog.out.error`), 
so that the producing system can identify status updates and errors related to its own messages.
Records missing this header are rejected and routed to the error topic.

---

## Topic Reference

### `helsemelding.dialog.in`

**Direction:** Helsemelding → Fagsystem  
**Format:** JSON  
**Schema version:** v1  

Contains inbound dialog messages that have been received from external healthcare providers,
validated, and converted from XML to JSON by Helsemelding platform.

**JSON schema (accessible via NAIS-device):**
- [Latest](https://helsemelding-json-schema.intern.dev.nav.no/api/v1/schemas/incoming-dialog-message/latest)
- [v1](https://helsemelding-json-schema.intern.dev.nav.no/api/v1/schemas/incoming-dialog-message/v1)

#### Message structure

```json
{
  "version": 1,
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "type": "PATIENT_INQUIRY",
  "receivedAt": "2024-06-01T10:00:00Z",
  "patientIdent": "12345678901",
  "sender": {
    "providerId": "123456",
    "signingProviderId": "123456"
  },
  "conversationReference": {
    "parentMessageId": "2bb9fdc1-f851-4604-ab58-e812a9d3d03e",
    "conversationId": "91ef57f9-3798-47de-8b5d-6d8748949705"
  },
  "message": "The patient requests a follow-up appointment.",
  "numberOfAttachments": 0
}
```

#### Field descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | ✅ | Schema version |
| `id` | string (UUID) | ✅ | Unique identifier of the dialog message |
| `type` | string (enum) | ✅ | Type of dialog message ([see below](#inbound-message-types)) |
| `receivedAt` | string (ISO 8601, UTC) | ✅ | When the message was received by Helsemelding platform |
| `patientIdent` | string | ✅ | National identity number (11 digits) of the patient |
| `sender.providerId` | string | ✅ | Provider registry ID of the sending healthcare provider |
| `sender.signingProviderId` | string | ✅ | Provider registry ID of the provider who signed the message |
| `conversationReference` | object \| null | ❌ | Link to an existing conversation, or `null` for new conversations |
| `conversationReference.parentMessageId` | string (UUID) | ✅ | ID of the previous message in the conversation |
| `conversationReference.conversationId` | string (UUID) | ✅ | ID of the conversation (typically same as the first message) |
| `message` | string \| null | ❌ | Free-text message body |
| `numberOfAttachments` | integer | ✅ | Number of attachments included in the original message |

#### Inbound message types

| Value | Application | Description | Possible response to |
|---|---|---|---|
| `ACCEPTS_MEETING_INVITATION` | Ja, jeg kommer | Healthcare provider accepts a meeting invitation | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `REQUESTS_NEW_MEETING_TIME` | Jeg ønsker nytt møtetidspunkt | Healthcare provider requests a new meeting time | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `DECLINES_MEETING_WITH_REASON` | Jeg kan ikke komme / begrunnelse for manglende oppmøte | Healthcare provider declines a meeting with a stated reason | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `PATIENT_REQUEST_RESPONSE` | Svar på forespørsel | Response to a patient request | `PATIENT_REQUEST`, `PATIENT_REQUEST_REMINDER` |
| `SICK_LEAVE_FOLLOW_UP_INQUIRY` | Henvendelse om sykefraværsoppfølging | Inquiry regarding sick leave follow-up | N/A |
| `PATIENT_INQUIRY` | Henvendelse om pasient | General inquiry from a healthcare provider about a patient | N/A |

---

### `helsemelding.dialog.out`

**Direction:** Fagsystem → Helsemelding  
**Format:** JSON  
**Schema version:** v1  

Produce to this topic to send a dialog message to an external healthcare provider. Helsemelding platform
validates the message, converts it to XML, and forwards it via the EDI-adapter.

**JSON schema (accessible via NAIS-device):**
- [Latest](https://helsemelding-json-schema.intern.dev.nav.no/api/v1/schemas/outgoing-dialog-message/latest)
- [v1](https://helsemelding-json-schema.intern.dev.nav.no/api/v1/schemas/outgoing-dialog-message/v1)

#### Message structure

```json
{
  "version": 1,
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "patientIdent": "12345678901",
  "providerId": "08e86b4e-9ffb-403f-b81c-aa81f9408b21",
  "conversationReference": {
    "parentMessageId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "conversationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  },
  "type": "MEETING_INVITATION_2",
  "message": "We would like to invite the patient to a meeting on June 10.",
  "attachment": "JVBERi0xLjQKJcOkw7zDtsO..."
}
```

#### Field descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | ✅ | Schema version |
| `id` | string (UUID) | ✅ | Unique identifier of the dialog message — **must match the Kafka record key** |
| `patientIdent` | string | ✅ | National identity number (11 digits) of the patient |
| `providerId` | string | ✅ | Provider registry ID of the receiving healthcare provider or provider office |
| `conversationReference` | object \| null | ❌ | Link to an existing conversation, or `null` for new conversations. This value is ignored for `FOLLOW_UP_PLAN` message type. |
| `conversationReference.parentMessageId` | string (UUID) | ✅ | ID of the previous message in the conversation. If not specified, `id` value will be used. |
| `conversationReference.conversationId` | string (UUID) | ✅ | ID of the conversation. If not specified, `id` value will be used. |
| `type` | string (enum) | ✅ | Type of dialog message ([see below](#outbound-message-types)) |
| `message` | string \| null | ❌ | Free-text message body |
| `attachment` | string \| null | ❌ | Attachment encoded as a Base64 string, or `null` |

`conversationReference` is ignored for `FOLLOW_UP_PLAN` message type.

#### Outbound message types

| Value | Application | Description | Response requirement |
|---|---|---|---|
| `MEETING_INVITATION_2` | Innkalling dialogmøte 2 | Invitation to a dialog meeting (dialogmøte). Contains the proposed meeting time and location. | Must be shown in the EPJ system immediately upon receipt. If the doctor does not respond, the proposed time and place is considered accepted. |
| `MEETING_RESCHEDULE_2` | Endring dialogmøte 2 | Reschedule of a dialog meeting. Contains the previous and the new proposed meeting time and location. | Same as `MEETING_INVITATION_2`. |
| `MEETING_INVITATION_3` | Innkalling dialogmøte 3 | Invitation to a dialog meeting, third meeting in the sequence. | Same as `MEETING_INVITATION_2`. |
| `MEETING_RESCHEDULE_3` | Endring dialogmøte 3 | Reschedule of a dialog meeting, third meeting in the sequence. | Same as `MEETING_INVITATION_2`. |
| `PATIENT_REQUEST` | Forespørsel om pasient | Request concerning a patient. Contains various questions and information about a specific patient. | Must be shown in the EPJ system immediately upon receipt. NAV automatically sends reminder letters if the response deadline is not met. |
| `PATIENT_REQUEST_REMINDER` | Påminnelse forespørsel om pasient | Reminder to answer a previously sent patient request. | Same as `PATIENT_REQUEST`. |
| `FOLLOW_UP_PLAN` | Oppfølgingsplan | Follow-up plan for a patient. | — |
| `RETURN_TO_WORK_NOTIFICATION` | Friskmelding til arbeidsformidling | Contains a copy of the decision on return-to-work notification (friskmelding til arbeidsformidling) sent to the sick-listed person. | Informational — should be filed in the patient's record in the EPJ system. |
| `MEDICAL_CERTIFICATE_RETURN` | Retur av legeerklæring | Contains a request to submit a new medical certificate. | Must be answered by submitting a new medical certificate. The doctor should be made aware of this as soon as it is received. |
| `MEETING_CANCELLATION` | Avlysning dialogmøte | Contains information that a previously scheduled dialog meeting has been cancelled. | The doctor should be made aware of this as soon as it is received. |
| `MEETING_EXEMPTION` | Unntak dialogmøte | Contains information that an exemption has been granted from the requirement to hold a dialog meeting for a long-term sick-listed person. | Informational — should be filed in the patient's record in the EPJ system. |
| `NAV_FEEDBACK` | Tilbakemelding fra NAV | Feedback from a NAV caseworker on a previous inquiry from the doctor to NAV. | The doctor should be made aware of this as soon as it is received. |
| `NAV_MESSAGE` | Melding fra NAV | Other messages from NAV, which may be automatically generated. | The doctor should be made aware of this as soon as it is received. |
| `NAV_INFORMATION` | Informasjon fra NAV | Other information from NAV. | Informational — should be filed in the patient's record in the EPJ system. |

> ℹ️ Message types under `MEETING_INVITATION_*` / `MEETING_RESCHEDULE_*` and `PATIENT_REQUEST*`
> require a response from the doctor and can be answered more than once for the same request
> (e.g. an initial "I will attend" followed later by "I cannot attend" with a reason). All other
> message types are informational and do not have a defined response message.

---

### `helsemelding.dialog.out.status`

**Direction:** Helsemelding → Fagsystem  
**Format:** JSON  

Contains delivery status updates for outbound messages as they are processed by Helsemelding platform and
forwarded to external systems. A new event is published each time the status of a message changes.

This topic uses **compact,delete** cleanup policy. Kafka retains the latest status event per message ID
and removes older events for the same key. This means consumers can always read the current
delivery status for any message, even if they missed earlier status transitions.

#### Message structure

```json
{
  "messageId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "timestamp": "2024-06-01T10:05:00Z",
  "status": "REJECTED_APPREC",
  "apprec": {
    "receiverHerId": 123456,
    "status": "REJECTED",
    "errorList": [
      {
        "code": "INVALID_RECIPIENT",
        "description": "The recipient could not process the message.",
        "oid": "2.16.578.1.12.4.1.1.8117"
      }
    ]
  },
  "error": null
}
```

For processing errors, `error` contains an error code and details, while `apprec` is `null`:

```json
{
  "messageId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "timestamp": "2024-06-01T10:05:00Z",
  "status": "INVALID",
  "apprec": null,
  "error": {
    "code": "INVALID_STATE",
    "details": "Unable to evaluate next state for the outbound message."
  }
}
```

#### Field descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `messageId` | string (UUID) | ✅ | ID of the outbound message (matches the Kafka record key on `out`) |
| `timestamp` | string (ISO 8601, UTC) | ✅ | When this status event was produced |
| `status` | string (enum) | ✅ | Current status of the message (see below) |
| `apprec` | object \| null | ❌ | Acknowledgement information returned by the receiving healthcare system. Present when an AppRec has been received. |
| `apprec.receiverHerId` | integer \| null | ❌ | HER-id of the receiver as reported in the AppRec |
| `apprec.status` | string \| null | ❌ | AppRec status string as reported by the external system |
| `apprec.errorList` | array | ❌ | List of errors reported in the AppRec (empty if none) |
| `apprec.errorList[].code` | string \| null | ❌ | Error code from the AppRec |
| `apprec.errorList[].description` | string \| null | ❌ | Error description from the AppRec |
| `apprec.errorList[].oid` | string \| null | ❌ | OID reference for the error code |
| `error` | object \| null | ❌ | Internal processing error. Present when the message could not be processed. |
| `error.code` | string | ✅ | Machine-readable error code |
| `error.details` | string | ✅ | Human-readable description of the error |

#### Message statuses

| Status | Description |
|---|---|
| `NEW` | Message has been received by the platform and registered |
| `PENDING_TRANSPORT` | Message is being forwarded to the external system via the EDI-adapter |
| `PENDING_APPREC` | Message has been delivered; waiting for acknowledgement (AppRec) from the receiver |
| `COMPLETED` | Message was successfully delivered and acknowledged |
| `REJECTED_APPREC` | The receiving system rejected the message (negative AppRec) |
| `REJECTED_TRANSPORT` | The message could not be delivered due to a transport-level failure |
| `INVALID` | The message was rejected by the platform due to a processing error |

---

### `helsemelding.dialog.out.error`

**Direction:** Helsemelding → Fagsystem  
**Format:** JSON  

Contains messages that were **rejected** during outbound validation (produced to
`helsemelding.dialog.out` but failed validation). Consuming this topic allows your system
to detect and handle delivery failures.

#### Message structure

```json
{
  "processedAt": "2024-06-01T10:05:00Z",
  "sourceSystem": "my-application",
  "errors": [
    {
      "category": "VALIDATION",
      "code": "INVALID_KAFKA_VALUE",
      "message": "Kafka record value is not valid JSON"
    }
  ],
  "originalMessage": {
    "createdAt": "2024-06-01T10:04:59Z",
    "key": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "payload": "<the original message value>"
  }
}
```

#### Field descriptions

| Field | Type | Description |
|---|---|---|
| `processedAt` | string (ISO 8601, UTC) | When the error was detected |
| `sourceSystem` | string | Value of the `sourceSystem` header from the rejected record |
| `errors` | array | One or more validation errors |
| `errors[].category` | string | Error category |
| `errors[].code` | string (enum) | Machine-readable error code (see below) |
| `errors[].message` | string | Human-readable error description |
| `originalMessage.createdAt` | string (ISO 8601, UTC) | Timestamp of the original record |
| `originalMessage.key` | string | Kafka record key of the rejected message |
| `originalMessage.payload` | string | Original record value |

#### Error codes

| Code | Cause |
|---|---|
| `INVALID_KAFKA_KEY` | The record key is missing or is not a valid UUID |
| `INVALID_KAFKA_VALUE` | The record value is null, empty, or not valid JSON. Check [JSON schema](https://helsemelding-json-schema.intern.dev.nav.no/api/v1/schemas/outgoing-dialog-message/latest) (accessible via NAIS-device) to get the expected structure. |
| `MISSING_SOURCE_SYSTEM_HEADER` | The required [`sourceSystem` Kafka header](#kafka-headers) is absent or empty |

---

## Access Configuration

Access to Kafka topics is controlled via Kubernetes network policies defined in your Nais
application manifest. Add the relevant topic names to your application's `kafka` configuration
to request access.

Example (`nais.yaml`):

```yaml
spec:
  kafka:
    pool: nav-dev   # or nav-prod
```

Contact the Helsemelding team to be granted read or write access to the required topics.
