# Roblox Cross-Server Routing Protocol version 2

This is the specification for the Roblox Cross-Server Routing Protocol (CSRP) version 2. CSRP is a protocol that allows Roblox servers to communicate with each other using the MessagingService. It is designed to be as simple as possible to implement in both Roblox and external applications using the MessagingService API.

## Introduction

### Motivation

The [previous version of CSRP](../v1/README.md) (referred to as CSRPv1 from now on) requires significant overhead for each message fragment, as the same header is sent with each fragment that is part of the same original message. This introduces a significant amount of overhead for messages with multiple recipients or otherwise large headers. The overhead in turn reduces the maximum length of the message fragment body, requiring more fragments to be sent for the same message. This results in the increased chance of hitting the MessagingService's rate limit, which can cause messages to be dropped.

CSRPv2 seeks to address these issues by sending a minimal header with each message fragment and allowing most of the message header to be fragmented with the body. This reduces the amount of duplicate data sent with each message fragment.

This document proposes a uniform and simple way to route messages to specific servers with fragmentation improvements over CSRPv1.

The proposed syntax is designed to be as simple as possible to implement in both Roblox and external applications using the MessagingService API, while still being flexible enough to support a wide range of use cases.

### Scope

This document is intended to be a specification for the Roblox Cross-Server Routing Protocol version 2 (CSRPv2). It is not intended to be a specification for the MessagingService API.

The CSRP is intended to provide a simple cross-server communication standard for servers that are part of the same game.

The CSRP is not intended to provide a standard for cross-game communication or communication between a Roblox game and web APIs.

The CSRP is not intended to provide encrypted messaging between servers. If such a functionality is necessary, the implementation is left up to the user. However, this should not be necessary as server-side code should be able to be trusted with cross-server messaging.

### About this document

This document is written in Markdown and is intended to be read on GitHub. The document provides a specification of the behavior required of a CSRPv2 implementation, both in its interactions with higher level systems and in its interactions with other CSRPv2 implementations.

## Functional Specification

### Message Composition

In CSRPv2, the message format has changed significantly. Message header segments are still separated using semicolons (`;`), but fragmented messages are now composed of a minimal fragment header and message content. The full message header, which has also changed, is encapsulated into the fragment content, allowing it to be fragmented with the message body.

#### Raw CSRP Message

```
 Header                                                      | Body
interaction_id;source_server_id;destination_server_id;options;message body with any characters
```

#### Fragmented CSRP Message

```
 Minimal Fragment Header                              | Fragment Content
interaction_id;fragment_sequence_number;fragment_count;fragment of raw CSRP message including its header, minus the interaction_id
```

### Header Format

CSRPv2 messages use two different header formats: a minimal fragment header and a full message header. The minimal fragment header is prepended to the content of each fragment of a message. The full message header is prepended to the body of the full message.

#### Minimal Fragment Header

The CSRPv2 minimal fragment header contains three (3) parts. The value of each part should not contain any semicolon (`;`) characters to avoid ambiguity. Sections can be left blank by not entering any characters between semicolons. The header sections are:

1. `Interaction ID` - The ID of the message. This is a string that uniquely identifies the message. This field is required and should be unique for each interaction. This can be used to pair requests and responses by using the same ID for both messages. The Interaction ID should be the same for all fragments of the same message.
2. `Fragment Sequence Number` - The sequence number of the message fragment. This is a number that uniquely identifies the message fragment. This field is required if the message is fragmented. If left blank, the value is assumed to be equal to the `Fragment Count` value.
3. `Fragment Count` - The total number of fragments in the message. This is a number that indicates the total number of fragments in the message. If left blank, the value is assumed to be `1`.

Any characters after the minimal fragment header are considered to be part of the fragment content.

#### Full Message Header

A full CSRPv2 message header is composed of four (4) parts. The value of each part should not contain any semicolon (`;`) characters to avoid ambiguity. Sections can be left blank by not entering any characters between semicolons. The header sections are:

1. `Interaction ID` - The ID of the message. This is a string that uniquely identifies the message. This field is required and should be unique for each interaction. This can be used to pair requests and responses by using the same ID for both messages. The Interaction ID should be the same for all fragments of the same message.
2. `Source Server ID` - The ID of the server that sent the message. This is a string that uniquely identifies the server that sent the message. This field is required and should be origin server's job ID.
3. `Destination Server ID` - The ID(s) of the server(s) that the message is intended for. This is a string that uniquely identifies the server(s) that the message is intended for. Can be empty, a single server job ID or a comma-separated list of job IDs. If left blank, the message is intended for all servers. If a job ID is prefixed with a minus sign (`-`), that job ID is excluded from the list of recipients.
4. `Options` - A comma-separated list of key-value pairs that provide additional information about the message. The key and value of each option should not contain any semicolon (`;`), comma (`,`), or equals (`=`) characters to avoid ambiguity. The key and value of each option should be separated by an equals (`=`) character. This field can be left blank.

Any characters after the full message header are considered to be the message body.

### Message Routing

When a server receives all fragments of a CSRP message and reassembles it into a message with a full header, it should first check the Source Server ID field to determine if the message is intended for it. The Destination Server ID string should first be split into a list of Job IDs using the comma (`,`) character as a delimiter. If any of the items in the resulting list are an empty string, then the message is intended for all servers. If the Job ID of the server that received the message is present in the list, then the message is intended for that server, unless the Job ID is prefixed with a minus sign (`-`).

If the Job ID of the server is prefixed with a minus sign (`-`), the message should be ignored regardless of any other matching Job IDs or blank items in the list.

If the recipient filter list contains an item that is just a minus sign (`-`), then the message should be ignored by all servers.

If the Destination Server ID field is empty, then the message is intended for all servers.

### Message Fragmentation

The Roblox MessagingService has a maximum message size of 1 kB, limiting the amount of data that can be sent in a single message. The CSRPv2 allows messages to be fragmented into multiple messages to allow for larger messages to be sent.

CSRPv2 requires all messages to be fragmented. If a message is short enough, it can be encapsulated into a single fragment. Each fragment's header must have a unique Fragment Sequence Number and the Fragment Count must be equal to the total number of fragments. The Fragment Sequence Number of the first fragment must be `1`, and the Fragment Sequence Number of the last fragment must be equal to the Fragment Count. The CSRPv2 allows the fragment size to be any value as long as the Interaction ID, Fragment Sequence Number and Fragment Count fields fit in each fragment along with 1 or more characters of the remaining header and the message body.

In order to fragment a message, the message should be concatenated into a single string with the following format:

```
source_server_id;destination_server_ids;options;message body with any characters
```

Note that the Interaction ID is not included in the message body. It will be included in the minimal fragment header of each fragment.

After formatting, the message should be split into fragments of the desired size. The fragments should be prefixed with the minimal fragment header containing the Interaction ID, Fragment Sequence Number and Fragment Count separated by semicolons (`;`). The minimal fragment header should be followed by a semicolon and the content of the fragment.

Once a fragmented message has been received, the fragments should be stored in a temporary buffer until all fragments have been received. Once all fragments have been received, the fragments should be reassembled in the correct order by concatenating the fragment content and prepending the Interaction ID to the final message. The reassembled message should then be processed as a normal CSRPv2 message.

In order to be reassembled correctly, each fragment must contain the Interaction ID, Fragment Sequence Number and Fragment Count fields. The other header fields can be retrieved from the reassembled message. If a fragment contains no Interaction ID, it can be processed as a single fragment message with a Fragment Count of `1`.

## Examples

This section contains example CSRP messages for different use cases. In these examples, placeholder values are used for the Source Server ID, Destination Server ID and Interaction ID. These placeholder values are:

* `Source Server ID`: `9999-ffff`
* `Destination Server ID`: `1111-2222` and `3333-4444`
* `Interaction ID`: `aaaa-bbbb`

Unless otherwise specified, the example messages are not fragmented. Note that while Interaction IDs in this section are reused for different examples, each message should have a unique Interaction ID in a real implementation.

### Example 1: Broadcasting a Message to All Servers

To broadcast a message to all servers, the Destination Server ID field should be left blank.

```
aaaa-bbbb;;;9999-ffff;;;This is the message body
```

### Example 2: Sending a Message to a Specific Server

To broadcast a message to a specific server, the Destination Server ID field should contain the Job ID of the server that the message is intended for.

```
aaaa-bbbb;;;9999-ffff;1111-2222;;This is sent to server 1111-2222
```

### Example 3: Sending a Message to Multiple Servers

To broadcast a message to multiple servers, the Destination Server ID field should contain a comma-separated list of the Job IDs of the servers that the message is intended for.

```
aaaa-bbbb;;;9999-ffff;1111-2222,3333-4444;;This is sent to servers 1111-2222 and 3333-4444
```

### Example 4: Sending a Message to All Servers Except One

To broadcast a message to all servers except one, the Destination Server ID field should contain a comma-separated list of the Job IDs of the servers that the message is intended for, with a minus sign (`-`) prefixing the Job ID of the server that the message should not be sent to. In this case, the message is sent to all servers except one, so one of the items in the comma-separated list is blank and one is the Job ID of the server to exclude prefixed with a minus sign (`-`).

```
aaaa-bbbb;;;9999-ffff;,-1111-2222;;This is sent to all servers except 1111-2222
```

### Example 5: Sending a Fragmented Message

To send a fragmented message, the message payload should be split into multiple fragments. The Fragment Sequence Number header field should contain the index of the fragment. The Fragment Count header field should contain the total number of fragments in the message.

```
aaaa-bbbb;2;3; second fragment without other headers
aaaa-bbbb;1;3;9999-ffff;1111-2222;;First fragment,
aaaa-bbbb;3;3; and finally the third fragment.
```

These fragments will be reassembled to the following message:

```
aaaa-bbbb;;;9999-ffff;1111-2222;;First fragment, second fragment without other headers and finally the third fragment.
```

Note that the fragments can be sent in any order. The recipient should reassemble the fragments in the correct order before processing the message.