# Sending Outbound Dialog Messages

This guide describes how to send a dialog message to an external healthcare provider through the
Helsemelding platform, track its delivery status, and handle errors.

## Overview

```
  Consumer                         Kafka                          Helsemelding
      │                               │                                 │
      │─── produce message ────▶ [dialog.out] ──────── consume ────────▶│
      │                               │                                 │
      │◀──── consume ────── [dialog.out.status] ◀─── publish status ────│
      │                               │                                 │
      │◀──── consume ────── [dialog.out.error] ◀─── publish on error ───│
      │                               │                                 │
```

The consumer is responsible for:
1. Publishing the outbound message to `helsemelding.dialog.out`
2. Consuming status updates from `helsemelding.dialog.out.status` to track delivery
3. Consuming error events from `helsemelding.dialog.out.error` to detect validation and transport (e.g., network issues) failures

---

## Step 1: Publish a message to `helsemelding.dialog.out`

To send a dialog message, produce a record to `helsemelding.dialog.out` with the following
requirements:

### Record key

The record key must be a **unique UUID** that identifies the message. This ID is used to correlate
status updates and error events back to the original message.

Example:
```
3fa85f64-5717-4562-b3fc-2c963f66afa6
```

### Kafka header

The following header is required on every record:

| Header | Value |
|---|---|
| `sourceSystem` | Name of your application (e.g. your Nais application name) |

Example (Kotlin, using the Kafka Producer API):

```kotlin
val record = ProducerRecord<String, String>(
    "helsemelding.dialog.out",
    messageId,
    payload
).apply {
    headers().add(RecordHeader("sourceSystem", "my-fagsystem".toByteArray()))
}
```

### Record value

The record value must be a valid JSON object conforming to the `OutgoingDialogMessage` schema.

```json
{
  "version": 1,
  "id": "a1b2c3d4-5717-4562-b3fc-2c963f66afa6",
  "patientIdent": "12345678901",
  "providerId": "08e86b4e-9ffb-403f-b81c-aa81f9408b21",
  "conversationReference": {
    "parentMessageId": "a1b2c3d4-5717-4562-b3fc-2c963f66afa6",
    "conversationId": "a1b2c3d4-5717-4562-b3fc-2c963f66afa6"
  },
  "type": "PATIENT_REQUEST",
  "message": "Request for updated medical certificate.",
  "attachment": "JVBERi0xLjQKJcOkw7zDtsO..."
}
```

#### Field descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | ✅ | Schema version. |
| `id` | string (UUID) | ✅ | Unique identifier of the message. |
| `patientIdent` | string | ✅ | National identity number (11 digits) of the patient. |
| `providerId` | string | ✅ | Provider registry ID of the receiving healthcare provider or provider office. |
| `conversationReference` | object \| null | ❌ | Link to an existing conversation, or `null` for new conversations. |
| `conversationReference.parentMessageId` | string (UUID) | ✅ | ID of the previous message in the conversation. |
| `conversationReference.conversationId` | string (UUID) | ✅ | ID of the conversation (typically the same as the first message in the thread). |
| `type` | string (enum) | ✅ | Type of dialog message ([see below](#message-types)). |
| `message` | string \| null | ❌ | Free-text message body. |
| `attachment` | string \| null | ❌ | Base64-encoded PDF document representing the message. |

#### Message types

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

## Step 2: Track delivery status on `helsemelding.dialog.out.status`

After a message is published, the Helsemelding platform publishes status events to
`helsemelding.dialog.out.status`. Each event is keyed by the `messageId`, which corresponds to
the Kafka record key used when publishing to `helsemelding.dialog.out`.

### Status transitions

A message progresses through the following statuses:

```
NEW ──▶ PENDING_TRANSPORT ──▶ PENDING_APPREC ──▶ COMPLETED
                │                    │
                ▼                    ▼
       REJECTED_TRANSPORT    REJECTED_APPREC
```

| Status | Description |
|---|---|
| `NEW` | Message received by the platform |
| `PENDING_TRANSPORT` | Message is being forwarded to the EPJ-system via NHN |
| `PENDING_APPREC` | Message delivered; waiting for acknowledgement (AppRec) from the receiver |
| `COMPLETED` | Message successfully delivered and acknowledged |
| `REJECTED_TRANSPORT` | Message could not be delivered due to a transport-level failure. Check `error` field for more details. |
| `REJECTED_APPREC` | The receiving system rejected the message (negative AppRec). Check `apprec.errorList` field for more details. |
| `INVALID` | Message rejected by the platform due to a processing error. Check `error` field for more details. |


### Correlating status events

Use the `messageId` field in the status event to correlate it with the original message:

```json
{
  "messageId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "timestamp": "2024-06-01T10:05:00Z",
  "status": "COMPLETED",
  "apprec": {
    "receiverHerId": 123456,
    "status": "OK",
    "errorList": []
  },
  "error": null
}
```

### Reading the latest status

`helsemelding.dialog.out.status` uses **compact** cleanup policy. This means the topic always
retains the latest status event per `messageId`, allowing consumers to look up the current
delivery status of any message at any time — even after a restart.

---

## Step 3: Handle errors from `helsemelding.dialog.out.error`

Error events are published to `helsemelding.dialog.out.error` when a message fails **validation**
before it is processed by the Helsemelding platform. This is distinct from delivery failures,
which are reported as `REJECTED_TRANSPORT` or `REJECTED_APPREC` status events on `helsemelding.dialog.out.status`.

### When does an error end up on `helsemelding.dialog.out.error`?

| Error code | Cause |
|---|---|
| `INVALID_KAFKA_KEY` | The record key is missing or is not a valid UUID |
| `INVALID_KAFKA_VALUE` | The record value is null, empty, or not valid JSON |
| `MISSING_SOURCE_SYSTEM_HEADER` | The required `sourceSystem` Kafka header is absent or empty |

### Correlating error events

Use the `originalMessage.key` field to correlate the error with the original message:

```json
{
  "processedAt": "2024-06-01T10:05:00Z",
  "sourceSystem": "my-fagsystem",
  "errors": [
    {
      "category": "VALIDATION",
      "code": "MISSING_SOURCE_SYSTEM_HEADER",
      "message": "Kafka record header 'sourceSystem' is missing or empty"
    }
  ],
  "originalMessage": {
    "createdAt": "2024-06-01T10:04:59Z",
    "key": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "payload": "{ ... }"
  }
}
```

### Error vs. delivery failure — summary

| Scenario | Where to look |
|---|---|
| Invalid key, value, or missing header | `helsemelding.dialog.out.error` |
| Message could not be delivered to external system | `helsemelding.dialog.out.status` (`REJECTED_TRANSPORT`) |
| External system rejected the message | `helsemelding.dialog.out.status` (`REJECTED_APPREC`) |

---

## Technical considerations

### Idempotency

The platform does not deduplicate messages. If the same record key is published to `helsemelding.dialog.out`
multiple times, each message will be processed independently and generate its own status events
on `helsemelding.dialog.out.status`. It is the responsibility of the consumer to ensure each message is published
only once.

### No guaranteed timeout for AppRec

After a message reaches `PENDING_APPREC`, the platform waits for an acknowledgement (AppRec)
from the receiving external system. There is no guaranteed timeout — if the external system
never responds, the message will remain in `PENDING_APPREC` indefinitely. The consumer should
implement its own timeout logic if a timely response is required.

### Consuming `helsemelding.dialog.out.status` after a restart

Because `helsemelding.dialog.out.status` uses compact cleanup policy, the latest status event
for each message is always available on the topic. A consumer that restarts or falls behind can
replay the topic from the beginning and reconstruct the current delivery status for all messages
without missing any final states.

### Order of operations

Status events and error events are produced asynchronously. A consumer should not assume that a
status event will arrive within a specific time window after publishing to `helsemelding.dialog.out`.
