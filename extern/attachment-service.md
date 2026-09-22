# Helsemelding — Attachment Service

Dialog messages may contain file attachments such as PDFs or images. Because attachments can be
too large to transport reliably over Kafka, they are stored separately in Google Cloud Storage
(GCS) by the Attachment Service. A reference to the number of attachments (`numberOfAttachments`)
is included in the Kafka message, and the actual files are retrieved via the Attachment Service
REST API.

> ⚠️ Attachments are automatically deleted from GCS after **90 days**.

The Attachment Service exposes an endpoint `GET /attachments/{messageId}` to retrieve attachments for a given message.

---

## Authentication

The endpoint requires a valid **Azure AD Bearer token** in the `Authorization` header.

```
Authorization: Bearer <token>
```

Token is obtained via the OAuth 2.0 **client credentials** flow against the Azure AD token
endpoint. The required scope is:

```
api://dev-gcp.helsemelding.attachment-service/.default
```

Your application must also be configured in Nais. Add the following to your `nais.yaml`:

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

---

## API Reference

**Base URL:** `https://attachment-service.intern.dev.nav.no`

### `GET /attachments/{messageId}`

Retrieves all attachments stored for a given message.

#### Path parameters

| Parameter | Type | Description |
|---|---|---|
| `messageId` | string (UUID) | Unique identifier of the message. |

#### Response body

`Content-Type: application/json` — a JSON array of attachment objects.

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
| `contentType` | string | MIME type of the attachment |
| `contentBase64` | string | Binary content of the file encoded as Base64 |

#### Responses

| Status | Description |
|---|---|
| `200 OK` | Attachments retrieved successfully |
| `400 Bad Request` | `messageId` is not a valid UUID |
| `404 Not Found` | No attachments found for the given `messageId` |
| `500 Internal Server Error` | An unexpected error occurred while reading from GCS |

---

## Attachment Client

The `attachment-client` library provides a Kotlin HTTP client for the Attachment Service,
handling authentication and serialization automatically.

### Adding the dependency

Check the latest release of Attachment Client on [GitHub](https://github.com/navikt/helsemelding-attachment-service/releases)

```kotlin
dependencies {
    implementation("no.nav.helsemelding:attachment-client:<version>")
}
```

### Usage

```kotlin
val client = HttpAttachmentClient()

// Retrieve attachments
client.getAttachments(messageId)
    .onSuccess { attachments -> /* process attachments */ }
    .onFailure { error -> /* handle error */ }

// Close the client when done
client.close()
```

`getAttachments()` method returns a `Result<List<Attachment>>`. On failure, the result contains an `AttachmentError` with an
HTTP code and a human-readable message.
