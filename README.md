# Roblox Cross-Server Routing Protocol

This is the specification for the Roblox Cross-Server Routing Protocol (CSRP). CSRP is a protocol that allows Roblox servers to communicate with each other using the MessagingService. It is designed to be as simple as possible to implement in both Roblox and external applications using the MessagingService API.

## Introduction

### Motivation

The default behavior of the Messaging Service does not allow routing messages towards any specific servers, and instead all messages are broadcast to all servers at the same time. This forces the user to implement their own recipient filtering if such a thing is necessary for their use case.

This document proposes a uniform and simple way to route messages to specific servers.

The proposed syntax is designed to be as simple as possible to implement in both Roblox and external applications using the MessagingService API, while still being flexible enough to support a wide range of use cases.

### Scope

This document is intended to be a specification for the Roblox Cross-Server Routing Protocol (CSRP). It is not intended to be a specification for the MessagingService API.

The CSRP is intended to provide a simple cross-server communication standard for servers that are part of the same game.

The CSRP is not intended to provide a standard for cross-game communication or communication between a Roblox game and web APIs.

### About this document

This document is written in Markdown and is intended to be read on GitHub. The document provides a specification of the behavior required of a CSRP implementation, both in its interactions with higher level systems and in its interactions with other CSRP implementations.

## Functional Specification

### Message Composition

Each CSRP message must be a single string containing both the message header and the message body. The message header is divided into six (6) parts, each separated by a single semicolon (`;`) character. The message body is the remainder of the message string, and must be separated from the message header by a single semicolon (`;`) character. Any semicolons (`;`) in the message body should be ignored when parsing the message header.

This message format was chosen in favour of fixed-length header fields because it allows for greater flexibility in terms of recipient filtering and header options.

#### CSRP Message

```
 Header                                                                                              | Body
source_server_id;destination_server_id;interaction_id;fragment_sequence_number;fragment_count;options;message body with any characters
```

### Header Format

A CSRP message header is composed of six (6) parts. The value of each part should not contain any semicolon (`;`) characters to avoid ambiguity. Sections can be left blank by not entering any characters between semicolons. The header sections are:

1. `Source Server ID` - The ID of the server that sent the message. This is a string that uniquely identifies the server that sent the message. This field is required and should be origin server's job ID.
2. `Destination Server ID` - The ID(s) of the server(s) that the message is intended for. This is a string that uniquely identifies the server(s) that the message is intended for. Can be empty, a single server job ID or a comma-separated list of job IDs. If left blank, the message is intended for all servers. If a job ID is prefixed with a minus sign (`-`), that job ID is excluded from the list of recipients.
3. `Interaction ID` - The ID of the message. This is a string that uniquely identifies the message. This field is required and should be unique for each interaction.
4. `Fragment Sequence Number` - The sequence number of the message fragment. This is a number that uniquely identifies the message fragment. This field is required if the message is fragmented. If left blank, the value is assumed to be equal to the `Fragment Count` value.
5. `Fragment Count` - The total number of fragments in the message. This is a number that indicates the total number of fragments in the message. If left blank, the value is assumed to be `1`.
6. `Options` - A comma-separated list of key-value pairs that provide additional information about the message. The key and value of each option should contain any semicolon (`;`), comma (`,`), or equals (`=`) characters to avoid ambiguity. The key and value of each option should be separated by an equals (`=`) character. This field can be left blank.

### Message Routing

When a server receives a CSRP message, it should first check the Destination Server ID field to determine if the message is intended for it. The Destination Server ID string should first be split into a list of Job IDs using the comma (`,`) character as a delimiter. If any of the items in the resulting list are an empty string, then the message is intended for all servers. If the Job ID of the server that received the message is present in the list, then the message is intended for that server, unless the Job ID is prefixed with a minus sign (`-`).

If the Job ID of the server is prefixed with a minus sign (`-`), the message should be ignored regardless of any other matching Job IDs or blank items in the list.

If the recipient filter list contains an item that is just a minus sign (`-`), then the message should be ignored by all servers.

If the Destination Server ID field is empty, then the message is intended for all servers.

### Message Fragmentation

The Roblox MessagingService has a maximum message size of 1 kB. If a message (including both the header and the body) is larger than 1 kB, it must be split into multiple fragments. Each fragment's header must have a unique Fragment Sequence Number and the Fragment Count must be equal to the total number of fragments. The Fragment Sequence Number of the first fragment must be `1`, and the Fragment Sequence Number of the last fragment must be equal to the Fragment Count.

Once a message has been split into fragments, the fragments can be sent in any order. The recipient should reassemble the fragments in the correct order before processing the message.

Once a fragmented message has been received, the fragments should be stored in a temporary buffer until all fragments have been received. Once all fragments have been received, the fragments should be reassembled in the correct order and the message should be processed as a single message.

To reassemble a fragmented message, the recipient should sort the fragments by Fragment Sequence Number and concatenate the fragment bodies in the correct order. Header values should be taken from the first fragment. Fragmentation header fields should be left blank in the reassembled message.

For simplicity, the CSRP allows handling non-fragmented messages as fragmented messages with a Fragment Count of `1`. This allows the same code to be used to handle both fragmented and non-fragmented messages.

## Examples

This section contains example CSRP messages for different use cases. In these examples, placeholder values are used for the Source Server ID, Destination Server ID and Interaction ID. These placeholder values are:

* `Source Server ID`: `9999-ffff`
* `Destination Server ID`: `1111-2222` and `3333-4444`
* `Interaction ID`: `aaaa-bbbb`

Unless otherwise specified, the example messages are not fragmented. Note that while Interaction IDs in this section are reused for different examples, each message should have a unique Interaction ID in a real implementation.

### Example 1: Broadcasting a Message to All Servers

To broadcast a message to all servers, the Destination Server ID field should be left blank.

```
9999-ffff;;aaaa-bbbb;;;;This is the message body
```

### Example 2: Sending a Message to a Specific Server

To broadcast a message to a specific server, the Destination Server ID field should contain the Job ID of the server that the message is intended for.

```
9999-ffff;1111-2222;aaaa-bbbb;;;;This is sent to server 1111-2222
```

### Example 3: Sending a Message to Multiple Servers

To broadcast a message to multiple servers, the Destination Server ID field should contain a comma-separated list of the Job IDs of the servers that the message is intended for.

```
9999-ffff;1111-2222,3333-4444;aaaa-bbbb;;;;This is sent to server 1111-2222 and 3333-4444
```

### Example 4: Sending a Message to All Servers Except One

To broadcast a message to all servers except one, the Destination Server ID field should contain a comma-separated list of the Job IDs of the servers that the message is intended for, with a minus sign (`-`) prefixing the Job ID of the server that the message should not be sent to. In this case, the message is sent to all servers except one, so one of the items in the comma-separated list is blank and one is the Job ID of the server to exclude prefixed with a minus sign (`-`).

```
9999-ffff;,-1111-2222;aaaa-bbbb;;;;This is sent to all servers except 1111-2222
```

### Example 5: Sending a Fragmented Message

To send a fragmented message, the message payload should be split into multiple fragments. The Fragment Sequence Number header field should contain the index of the fragment. The Fragment Count header field should contain the total number of fragments in the message.

```
9999-ffff;1111-2222;aaaa-bbbb;1;3;; this is the second fragment and
9999-ffff;1111-2222;aaaa-bbbb;1;3;;This is the first fragment,
9999-ffff;1111-2222;aaaa-bbbb;3;3;; this is the third fragment.
```

These fragments will be reassembled to the following message:

```
9999-ffff;1111-2222;aaaa-bbbb;;;;This is the first fragment, this is the second fragment and this is the third fragment.
```

Note that the fragments can be sent in any order. The recipient should reassemble the fragments in the correct order before processing the message.
