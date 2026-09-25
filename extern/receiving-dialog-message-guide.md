# Receiving Inbound Dialog Messages

This guide describes how to receive a dialog message from an external healthcare provider through
the Helsemelding platform, and how to retrieve any attachments associated with the message.

## Overview

```
  Helsemelding                       Kafka                        Fagsystem
      │                                │                              │
      │──── publish message ────▶ [dialog.in] ────── consume ────────▶│
      │                                                               │
      │                        Attachment Service                     │
      │──── store attachments ────────────────── GET /attachments ───▶│
      │                                             /{messageId}      │
```

The Fagsystem is responsible for:
1. Consuming inbound messages from `helsemelding.dialog.in`
2. Checking the `numberOfAttachments` field to determine if the message has attachments
3. Fetching attachments from the Attachment Service using the `messageId`

---

## Step 1: Consume a message from `helsemelding.dialog.in`

Inbound dialog messages are published to `helsemelding.dialog.in` after they have been
received from the external healthcare provider, validated, and converted from XML to JSON by the
Helsemelding platform.

### Record value

```json
{
  "version": 1,
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "type": "PATIENT_INQUIRY",
  "receivedAt": "2024-06-01T10:00:00Z",
  "patientIdent": "12345678901",
  "sender": {
    "providerId": "08e86b4e-9ffb-403f-b81c-aa81f9408b21",
    "signingProviderId": "1b010446-2030-49ac-9df4-6df263c0ea28"
  },
  "conversationReference": {
    "parentMessageId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "conversationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  },
  "message": "The patient requests a follow-up appointment.",
  "numberOfAttachments": 2
}
```

#### Field descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | ✅ | Schema version |
| `id` | string (UUID) | ✅ | Unique identifier of the dialog message |
| `type` | string (enum) | ✅ | Type of dialog message ([see below](#message-types)) |
| `receivedAt` | string (ISO 8601, UTC) | ✅ | When the message was received by Helsemelding platform |
| `patientIdent` | string | ✅ | National identity number (11 digits) of the patient |
| `sender.providerId` | string | ✅ | Provider registry ID of the sending healthcare provider |
| `sender.signingProviderId` | string | ✅ | Provider registry ID of the provider who signed the message |
| `conversationReference` | object \| null | ❌ | Link to an existing conversation, or `null` for new conversations |
| `conversationReference.parentMessageId` | string (UUID) | ✅ | ID of the previous message in the conversation |
| `conversationReference.conversationId` | string (UUID) | ✅ | ID of the conversation (typically same as the first message) |
| `message` | string \| null | ❌ | Free-text message body |
| `numberOfAttachments` | integer | ✅ | Number of attachments included in the original message. If greater than `0`, proceed to Step 2. |

#### Message types

| Value | Application | Description | Possible response to |
|---|---|---|---|
| `ACCEPTS_MEETING_INVITATION` | Ja, jeg kommer | Healthcare provider accepts a meeting invitation | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `REQUESTS_NEW_MEETING_TIME` | Jeg ønsker nytt møtetidspunkt | Healthcare provider requests a new meeting time | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `DECLINES_MEETING_WITH_REASON` | Jeg kan ikke komme / begrunnelse for manglende oppmøte | Healthcare provider declines a meeting with a stated reason | `MEETING_INVITATION_2`, `MEETING_RESCHEDULE_2`, `MEETING_INVITATION_3`, `MEETING_RESCHEDULE_3` |
| `PATIENT_REQUEST_RESPONSE` | Svar på forespørsel | Response to a patient request | `PATIENT_REQUEST`, `PATIENT_REQUEST_REMINDER` |
| `SICK_LEAVE_FOLLOW_UP_INQUIRY` | Henvendelse om sykefraværsoppfølging | Inquiry regarding sick leave follow-up | N/A |
| `PATIENT_INQUIRY` | Henvendelse om pasient | General inquiry from a healthcare provider about a patient | N/A |

---

## Step 2: Fetch attachments from the Attachment Service

If `numberOfAttachments` is greater than `0`, fetch the attachments from the Attachment Service
using the `id` from the consumed record as the `messageId`.

### Option A: Using [`HttpAttachmentClient`](https://github.com/navikt/helsemelding-attachment-service/releases) (recommended)

The `attachment-client` library handles authentication and serialization automatically.

Add the dependency:

```kotlin
dependencies {
    implementation("no.nav.helsemelding:attachment-client:<version>")
}
```

Your application must add an outbound access policy in `nais.yaml`.
Contact Team Helsemelding, so that we can add inbound rule in `helsemelding-attachment-service` for your application.

```yaml
spec:
  azure:
    application:
      enabled: true
  accessPolicy:
    outbound:
      rules:
        - application: attachment-service
          namespace: helsemelding
          cluster: dev-gcp   # or prod-gcp
```

Fetch attachments:

```kotlin
val client = HttpAttachmentClient()

// Retrieve attachments
client.getAttachments(messageId)
    .onSuccess { attachments -> /* process attachments */ }
    .onFailure { error -> /* handle error */ }

// Close the client when done
client.close()
```

### Option B: Direct HTTP call

Requests to the Attachment Service require a valid **Azure AD Bearer token**:

```
Authorization: Bearer <token>
```

Token is obtained via the OAuth 2.0 **client credentials** flow. The required scope is:

```
api://dev-gcp.helsemelding.attachment-service/.default
```

Your application must also add an outbound access policy in `nais.yaml`.
Contact Team Helsemelding, so that we can add inbound rule in `helsemelding-attachment-service` for your application.

```yaml
spec:
  azure:
    application:
      enabled: true
  accessPolicy:
    outbound:
      rules:
        - application: attachment-service
          namespace: helsemelding
          cluster: dev-gcp   # or prod-gcp
```

```
GET https://attachment-service.intern.dev.nav.no/attachments/{messageId}
Authorization: Bearer <token>
```

A simple Kotlin example:

```kotlin
@Serializable
data class Attachment(
    val description: String,
    val contentType: String,
    val contentBase64: String
)

@Serializable
data class TokenResponse(
    val access_token: String
)

suspend fun requestAccessToken(): String {
    return HttpClient(CIO).use { client ->
        client.submitForm(
            url = "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token",
            formParameters = parameters {
                append("client_id", System.getenv("AZURE_APP_CLIENT_ID"))
                append("client_secret", System.getenv("AZURE_APP_CLIENT_SECRET"))
                append("grant_type", "client_credentials")
                append("scope", "api://dev-gcp.helsemelding.attachment-service/.default")
            }
        ).body<TokenResponse>().access_token
    }
}

suspend fun fetchAttachments(messageId: String): List<Attachment> {
    val token = requestAccessToken()

    return HttpClient(CIO).use { client ->
        client.get("https://attachment-service.intern.dev.nav.no/attachments/$messageId") {
            header(HttpHeaders.Authorization, "Bearer $token")
        }.body()
    }
}

suspend fun main() {
    val attachments = fetchAttachments("3fa85f64-5717-4562-b3fc-2c963f66afa6")
    val first = attachments.first()
    val pdfBytes = Base64.getDecoder().decode(first.contentBase64)
}
```

This matches the service implementation: the caller requests an Azure AD access token with the
`api://dev-gcp.helsemelding.attachment-service/.default` scope and then calls
`GET /attachments/{messageId}` with the bearer token in the `Authorization` header.

A successful response returns a JSON array of attachment objects:

```json
[
  {
    "description": "Medical certificate",
    "contentType": "application/pdf",
    "contentBase64": "JVBERi0xLjQK..."
  }
]
```

| Field | Type | Description |
|---|---|---|
| `description` | string | Human-readable filename or description of the attachment |
| `contentType` | string | MIME type of the attachment (e.g. `application/pdf`, `image/png`) |
| `contentBase64` | string | Binary content of the file encoded as Base64 |

#### Response codes

| Status | Description |
|---|---|
| `200 OK` | Attachments retrieved successfully |
| `400 Bad Request` | `messageId` is not a valid UUID |
| `404 Not Found` | No attachments found for the given `messageId` |
| `500 Internal Server Error` | An unexpected error occurred |

---

## Technical considerations

### Implement retry when fetching attachments

By the time a message is available on `helsemelding.dialog.in`, its attachments are already stored in the
Attachment Service. However, transient errors may occur when fetching them (e.g. network issues
or temporary unavailability). The Fagsystem should implement retry logic with a short delay when
a `500 Internal Server Error` or a `503 Unavailable` response is returned.

### Attachments are deleted after 90 days

Attachments are automatically deleted from the Attachment Service after **90 days**. The
Fagsystem should fetch and store attachments as soon as possible after receiving the message.

### A message with `numberOfAttachments: 0` has no attachments

If `numberOfAttachments` is `0`, there is no need to call the Attachment Service. Calling
`GET /attachments/{messageId}` for such a message will return `404 Not Found`.
