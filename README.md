# helsemelding-docs

## Documentation

### External integration guides

- [Sending dialog messages](extern/sending-dialog-message-guide.md)  
  Guide for publishing outbound dialog messages, tracking delivery status, and handling validation and delivery errors.
- [Receiving dialog messages](extern/receiving-dialog-message-guide.md)  
  Guide for consuming inbound dialog messages and retrieving their attachments from the Attachment Service.
- [Kafka topics](extern/kafka-topics.md)  
  Reference for the Kafka topics used to send and receive dialog messages, including record formats, message schemas, status events, and validation errors.
- [Attachment Service](extern/attachment-service.md)  
  API reference for retrieving message attachments, including authentication, response formats, and the Kotlin client.

### Migration

- [iSYFO migration changes](migration/isyfo_endringer.md)  
  Overview of the changes required in iSYFO services to support the migration to EDI 2.0 and the Helsemelding platform.

### Current limitations

- There is currently no way of receiving a response with the correct values for `ConversationRef` (`RefToParent` 
and `RefToConversation`). This means that it is possible to send a request, but not receive a response to said request. 
That being said, we are periodically sending ourselves messages from a different HER-id to simulate receiving messages 
from an EPJ.
- `providerId` of outbound messages will be mapped to our own HER-id (`8142520`) instead of the value returned by 
Provider registry. This is done to be able to simulate the entire message flow without the need for external input.

#### Missing features

These will be fixed, but have not been deemed crucial enough to wait for these to be fully implemented before testing.

- No validation of incoming dialog message (XML and values). Planned validation can be found in this 
[issue on GitHub](https://github.com/navikt/helsemelding-inbound-message-service/issues/21).
- No signature information - The value used for `signingProviderIdent` is the provider from the XML
- `receivedAt` from the `helsemelding.dialog.in` will be slightly inaccurate, but nothing of significance.  

The error topic `helsemelding.dialog.out.error` is still being worked on, but there are only expected to be minor 
changes, which is as following:

```kotlin
data class ErrorMessage(
  val processedAt: Instant,
  val sourceSystem: String,
  val errors: List<ProcessingError>,
  val originalMessage: OriginalMessage
)

data class OriginalMessage(
  val createdAt: Instant,
  val payload: String
)
```

To:
```kotlin
data class ErrorMessage(
    val processedAt: Instant,
    val sourceSystem: String,
    val messageId: Uuid?,
    val errors: List<ProcessingError>,
    val originalMessage: OriginalMessage?,
)

data class OriginalMessage(
    val publishedAt: Instant,
    val payload: String
)
```

Notable changes:
- `messageId` will be added to `ErrorMessage`
- `OriginalMessage.createdAt` will be renamed to `publishedAt` 

#### Difference in behavior

This is functionality that exist in the current implementation that most likely won't be present in the new.

- When processing outbound messages the service will not generate any file(s)/attachment(s) (typically PDF) based on the 
message provided.

### Contact

If you work in [@navikt](https://github.com/navikt) you can reach us at the Slack 
channel [#team-helsemelding](https://nav-it.slack.com/archives/C0A7WPMMUC9)
