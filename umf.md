# Universal Messaging Format
A message format specification for use with distributed applications.

### Version
The current version of this specification is: UMF/1.4.6, which introduces the `signature` keyword.

### License

UMF is licensed under the Open Source [The MIT License (MIT)](https://github.com/cjus/umf/blob/master/LICENSE) and authored by Carlos Justiniano ([cjus34@gmail.com](mailto:cjus34@gmail.com))  This specification is hosted on Github at: [https://github.com/cjus/umf/blob/master/umf.md](https://github.com/cjus/umf/blob/master/umf.md)

# Table of Contents

  * [1. Introduction](#Introduction)
  * [2. Message Format](#Message-Format)
    * [2.1 Envelope format](#Envelope-format)
      * [2.1.1 Envelop format considerations for routing](#Envelop-format-considerations-for-routing)
    * [2.2 Reserved Fields](#Reserved-Fields)
      * [2.2.1 Mid field (Message ID)](#Mid-field-(Message-ID))
      * [2.2.2 Rmid field (Refers to Message ID)](#Rmid-field-(Refers-to-Message-ID))
      * [2.2.3 To field (routing)](#To-field-(routing))
      * [2.2.4 Forward field (routing)](#Forward-field-(routing))
      * [2.2.5 From field (routing)](#From-field-(routing))
      * [2.2.6 Type field (message type)](#Type-field-(message-type))
      * [2.2.7 Version field](#Version-field)
      * [2.2.8 Priority field](#Priority-field)
      * [2.2.9 Timestamp field](#Timestamp-field)
      * [2.2.10 Ttl field (time to live)](#Ttl-field-(time-to-live))
      * [2.2.11 Body field (application level data)](#Body-field-(application-level-data))
        * [2.2.11.1 Overriding UMF restricted key / value pairs](#Overriding-UMF-restricted-key-value-pairs)
        * [2.2.11.2 Sending binary data](#Sending-binary-data)
        * [2.2.11.3 Sending multiple application messages](#Sending-multiple-application-messages)
      * [2.2.12 Authorization field](#Authorization-field)
      * [2.2.13 For - on behalf of](#For-on-behalf-of)
      * [2.2.14 Via - sent through](#Via-sent-through)
      * [2.2.15 Headers - protocol headers](#Headers)
      * [2.2.16 Timeout - timeout recommendation](#Timeout)
      * [2.2.17 Signature field](#Signature)
  * [3. Use inside of HTTP](#Use-inside-of-HTTP)
  * [4. Peer-to-Peer Communication](#Peer-to-Peer-Communication)
  * [5. Infrastructure considerations](#Infrastructure-considerations)
    * [5.1 Message storage](#Message-storage)
    * [5.2 Message routing](#Message-routing)
      * [5.2.1 Message forwarding](#Message-forwarding)
  * [6. Short form syntax](#Short-form-syntax)

---

<a name="Introduction"></a>
## 1. Introduction

This specification describes an application-level messaging format suitable for use with WebSockets, Message Queuing and in traditional HTTP JSON payloads. The proposed message format is designed as a replacement format to avoid using inconsistent formats.

From this point forward we’ll refer to the Universal Messaging Format as UMF. We’ll also refer to UMFs as documents because they can be stored in memory, transmitted along communication channels and retained in offline storage and message queues.

<a name="Message-Format"></a>
## 2. Message Format

The UMF is a valid JSON document that is required to validate with existing JSON validators and thus be fully compliant with the JSON specification.

Example validators:

* [Curious Concept JSON formatter](http://jsonformatter.curiousconcept.com)
* [JSON Editor Online](http://www.jsoneditoronline.org)

JSON Specification: http://www.json.org/

JSON is a data interchange format based on JavaScript Object Notation. As such, UMF which is encoded in JSON, follows well established JavaScript naming conventions. For example, UMF retains the use of camel case for compound names.

<a name="Envelope-format"></a>
### 2.1 Envelope format

UMF uses an envelope format where the outer portion is a valid JSON object and the inner portion consist of directives (headers) and a message body. Variations of this approach is used in the SOAP XML format where the outer portion is called an envelope and the inner portion contains both header and body sections. In other formats headers and body may be side by side as is the case of HTTP.

In UMF the inner portion consists of UMF reserved key/value pairs with an optional body object

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "rmid": "66c61afc-037b-4229-ace4-5ec4d788903e",
  "to": "uid:123",
  "from": "uid:56",
  "type": "dm",
  "version": "UMF/1.4.3",
  "priority": "10",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "message": "How is it going?"
  }
}
```

Only UMF reserved words may be used. However, application specific (custom) key/value pairs may be used freely inside the `body` value.  This strict requirement ensures that the message format has a strict agreed upon format as defined by its version number.

<a name="Envelop-format-considerations-for-routing"></a>
## 2.1.1 Envelop format considerations for routing

A UMF message is considered to be a system routable message. The UMF message fields aid first and foremost in routing but can also secondarily be used by message processing systems (i.e. message handlers) to obtain additional routing related fields.  However, the fields are reserved for routing systems and not intended to be extensible with application specific fields.  Where application specific fields are needs they should be placed in the `body` value instead.

<a name="Reserved-Fields"></a>
## 2.2 Reserved Fields

As described earlier UMF consists of reserved key/value pairs with an optional embedded body object.  This section describes each of the reserved fields and their intended use. A reserved field consists of a name key followed by a value which is encoded in a strict format. As we’ll see later in this specification, only the body field differs from this requirement.  The take-away here is that a reserved field has a value content which follows a strict format.

<a name="Mid-field-(Message-ID)"></a>
### 2.2.1 Mid field (Message ID)

Each message must contain an mid field with a universally unique identifier (UUID) as a value.

For example:

```javascript
{
  "mid":"ef5a7369-f0b9-4143-a49d-2b9c7ee51117"
}
```

The mid field is required to uniquely identify a message across multiple applications, services and replicated servers. All programming environments support the use of UUIDs.

* AS3: https://code.google.com/p/actionscript-uuid/
* Python: import uuid
* JavaScript: http://www.javascriptoo.com/uuid-v4-js  

Because UMF is an asynchronous message format, an individual message may be generated on either a server or client application. Each is required to create a UUID for use with a newly created message.

The mid field is a required field.

<a name="Rmid-field-(Refers-to-Message-ID)"></a>
### 2.2.2 Rmid field (Refers to Message ID)

The Refers to Message ID (`rmid`) is a method of specifying that a message refers to another message.  The use of the rmid field is helpful in the case of a message that requires a reply, or where a reply finalizes or changes an application's state machine. This is also useful in threaded conversations there a message may be sent in reply to another pre-existing message.

An important use-case for the `rmid` field is when it's desirable to trace the path an originating message took as it moved through a processing pipeline.

The rmid field is NOT a required field.

<a name=" To-field-(routing)"></a>
### 2.2.3 To field (routing)

The `to` field is used to specify message routing.  A message may be routed to other users, internal application handlers or to other remote servers. Additionally, a message could be routed directly to an API server with careful use of the value specified in the `to`.

The value of a to field is a colon or forward slash separated list of sub names.

For example:

```javascript
{
  :
  "to":"server:service"
}
```

specifies that the message should be routed to a server called server and a service called service.  Another example would be a message which is intended to be routed to a specific user:

```javascript
{
  :
  "to":"UID:143"
}
```

Sending a message to a service which exposes an API might look like this:

```javascript
{
  :
  "to":"emailer:[post]/v1/send/email"
}
```

In the example above a UMF formatted message is being sent to a service called `emailer` and routed to the `v1/send/email` API endpoint. In a use case such as this, parsing the `to` field value split on `:` would yield an array with two values:

* emailer
* [post]/v1/send/email

This simplifies routing API calls, since the first array field is the service name and the second field specifies the API route. Note that you may optionally specify an HTTP VERB (such as get, post, put, delete, head, trace) inside of opening and closing braces.  In the example above and HTTP POST operation is intended. If the optional HTTP verb is missing then `post` is assumed.  Care should be taken to encode URL endpoint characters `:` and `/` to avoid mishandling when used in UMF message routing.

Where applicable the HTTP body becomes the UMF message `body`.

The use of the colon (:) or forward slash (/) separator is intended to simplify parsing using the split function available in string parsing libraries. Colon and forward slash may be used in the same key value and their use may differ across messages. Keep in mind that the value of a `to` field may not be a single entity but rather a broadcasting service which sends the message to various subscribers.

The `to` field is a required field and must be present in all messages.

<a name="Forward-field-(routing)"></a>
### 2.2.4 Forward field (routing)

The `forward` field can be used to designate where a message should be sent to.  Potential uses include:

* processing a message by the service specified in the `to` field and then having that service send its results to the service specified in the `forward` field for additional processing.
* transforming the contents (including `body`) of a message before forwarding it to another service. In this way a service might act as a proxy for one or more additional services.

Example:

```javascript
{
  :
  "to":"customer:registration:service",
  "forward":"customer:account:billing"
}
```

<a name="From field (routing)"></a>
### 2.2.5 From-field-(routing)

The `from` field is used to specify a source during message routing.  Like the `to` field, the value of a `from` field is a colon or forward slash separated list of sub names.

For example:

```javascript
{
  :
  "from":"server:service"
}
```

The above specifies that the message should be routed to a server called `server` and a service called `service`.  Another example would be a message which is intended to be routed to a specific user:

```javascript
{
  :
  "from":"UID:64"
}
```

The use of the colon (:) or forward slash (/) separator is intended to simplify parsing using the split function available in string parsing libraries. Colon and forward slash may not be used in the same key value, however their use may differ across messages.

The `from` field is a required field and must be present in all messages.

<a name="Type-field-(message-type)"></a>
### 2.2.6 Type field (message type)

The message type field describes a message as being of a particular classification.

```javascript
{
  :
  "type":"event"
}
```

The type field is a NOT a required field.  If type is missing from a message, a type of `msg` is assumed:

```javascript
{
  :
  "type":"msg"
}
```

An application developer may choose to implement a sub message type inside of their message’s body object.

```javascript
{
  :
  "body":{
    "type":"chat",
    "message":"How is it going?"
  }
}
```

However, the message will still be handled as a generic message (msg) by other servers and infrastructure components which may or may not inspect the contents of the custom body object.  For this reason it’s recommended that standard type fields be used whenever possible.

<a name="Version-field"></a>
### 2.2.7 Version field

UMF messages have a `version` field that identifies the version format for a given message.

```javascript
{
  :
  "version": "UMF/1.3"
}
```

The `version` value format is of the form of "umf/major version . minor version . revision version". The version field should begin with  `UMF/` (in caps) to identify the JSON as a UMF compatible message.

> "...the major number is increased when there are significant jumps in functionality, the minor number is incremented when only minor features or significant fixes have been added, and the revision number is incremented when minor bugs are fixed." - http://en.wikipedia.org/wiki/Software_versioning

Applications implementing UMF may choose to only consider the major and minor version segments, ignoring the revision version.

The version field is a required field, must be present in all messages and is required to include both a major and minor version number.

<a name="Priority-field"></a>
### 2.2.8 Priority field

UMF documents may include an optional `priority` field.  If not present a default value equal to default priority is assumed. If present, priority field values are in the range of 10 (highest) to 1 (lowest).  

Normal priority is valued at 5.

```javascript
{
  :
  "priority": "10"
}
```

In addition to numeric values the strings “low”,”normal”,”high” may be used to indicate message priorities:

```javascript
{
  :
  "priority": "high"
}
```

The `priority` field is NOT a required field and defaults to “normal” priority if unspecified.

<a name="Timestamp-field"></a>
### 2.2.9 Timestamp field

UMF supports a timestamp field which indicates when a message was sent. The format for a timestamp is ISO 8601, a standard date format. http://en.wikipedia.org/wiki/ISO_8601

When using an ISO 8601 formatted timestamp, UMF requires that the time be in the UTC timezone.

```javascript
{
  :
  "timestamp":"2013-09-29T10:40Z",
}
```

All programming environments include support for converting local machine time to UTC.

There are a number of reasons why message timestamps are useful:
* Communication can be ordered chronologically during display, searching and storage.
* Messages past a set time can be handled differently, including being purged from a system.

The `timestamp` field is a required UMF field.

<a name="Ttl-field-(time-to-live)"></a>
### 2.2.10 Ttl field (time to live)

The `ttl` field is used to specify how long a message may remain alive within a system. The value of this field is an amount of time specified in seconds.

```javascript
{
  :
  "ttl": "300"
}
```

The `ttl` field is optional and if not present in a UMF document the default is a `ttl` which never expires.

<a name="Body-field-(application-level-data)"></a>
### 2.2.11 Body field (application level data)

The `body` field is used to host an application-level custom object.  This is where an application may define a message content which is meaningful in the context of its application.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:56",
  "from": "game:store",
  "version": "UMF/1.3",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "type": "store:purchase",
    "itemID": "5x:winnings:multiplier",
    "expiration": "2014-02-10T10:40Z"
  }
}
```

In the example above a user receives confirmation of a purchase (power-up item) from the game store, the items can then be added to the user’s inventory.

<a name="Overriding-UMF-restricted-key-value-pairs"></a>
#### 2.2.11.1 Overriding UMF restricted key / value pairs

As mentioned earlier UMF restricted fields may not be used in occurrence to this specification, however, an application may override UMF restricted fields by including those fields in it's custom `body` object.

The following are potential use-cases:

* The application may choose to include its own sub-routing and override the `to` and `from` fields.
* It might be desirable to have an application level message version.
* The application may require its own message id (mid) format.

The application level code is free to override the meaning of UMF restricted keys by looking inside its `body` object for potential overrides.

<a name="Sending-binary-data"></a>
#### 2.2.11.2 Sending binary data

Binary or encrypted / encoded messages may be sent via the UFM by using a JSON compatible data convertor such as Base64 http://en.wikipedia.org/wiki/Base64

When using a converter the base format should be indicated in a user level field such as “contentType” whose value should be a standard Internet Media Type (formally known as a MIME type) http://en.wikipedia.org/wiki/Internet_media_type

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:134",
  "from": "uid:56",
  "version": "UMF/1.3",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "type": "private:message",
    "contentType": "text/plain",
    "base64": "SSBzZWUgeW91IHRvb2sgdGhlIHRyb3VibGUgdG8gZGVjb2RlIHRoaXMgbWVzc2FnZS4="
  }
}
```

In this way, audio and images may be transmitted via UMF.

<a name="Sending-multiple-application-messages"></a>
#### 2.2.11.3 Sending multiple application messages

An application may, for efficiency reasons, decide to bundle multiple sub-messages inside of a single UMF document. The recommended method of doing this to define a body object which contains one or more sub messages.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:134",
  "from": "chat:room:14",
  "version": "UMF/1.3",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "type": "chat:messages",
    "messages": [
      {
        "from": "moderator",
        "text": "Susan welcome to chat Nation NYC",
        "ts": "2013-09-29T10:34Z"
      },
      {
        "from": "uid:16",
        "text": "Rex, you are one lucky SOB!",
        "ts": "2013-09-29T10:30Z"
      },
      {
        "from": "uid:133",
        "text": "Rex you're going down this next round",
        "ts": "2013-09-29T10:31Z"
      }
    ]
  }
}
```

In the example above messages consists of an array of objects.

<a name="Authorization-field"></a>
### 2.2.12 Authorization field

The `authorization` field is used to pass an HTTP authorization value or authentication token.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:56",
  "from": "game:store",
  "version": "UMF/1.3",
  "authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "type": "store:purchase",
    "itemID": "5x:winnings:multiplier",
    "expiration": "2014-02-10T10:40Z"
  }
}
```
In the example above the authorization field contains a JSON Web Token.

<a name="For-on-behalf-of"></a>
### 2.2.13 For - on behalf of

The `for` field is used to indicate who a message is being send on behalf of. This helps support the use-case where a message may not be sent by an actual client but instead created by a service on behalf of a client. In this case routable information needs to be sent to help associate the entity which may ultimately require notification.

The value of the `for` field is left up to the host application, but the use of unique hashes are recommended.

<a name="Via-sent-through"></a>
### 2.2.14 Via - sent through

The presence of a `via` field indicates that the message was sent via an intermediary such as a router.  When sending replies it's often useful to send the reply to the intermediary for routing.

The format of the `via` field should be the same as `to` and `from` fields.

<a name="Headers"></a>
### 2.2.15 Headers - protocol headers

When necessary communication protocol headers may be sent inside of a UMF message.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:123",
  "from": "uid:56",
  "version": "UMF/1.4.4",
  "headers": {
    "Content-Type": "text/html"
  },
  "body": {}
}
```

The `header` field should contain an object consisting of key / value pairs.

<a name="Timeout"></a>
### 2.2.16 Timeout - timeout recommendation

Messages can carry a recommended timeout value which can be used at the descression of the application transport and routing layer. For example, an HTTP request may use this timeout to abort the wait for a response after some time.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:123",
  "from": "uid:56",
  "version": "UMF/1.4.5",
  "timeout": 5,
  "headers": {
    "Content-Type": "text/html"
  },
  "body": {}
}
```

The timeout value is a number representing a number of seconds. Sub-second timeout isn't supported.

<a name="Signature"></a>
### 2.2.17 Signature field

Messages maybe signed with an HMAC signature to help ensure that can a message was created by a known source.

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "uid:123",
  "from": "uid:56",
  "version": "UMF/1.4.6",
  "signature": "c0fa1bc00531bd78ef38c628449c5102aeabd49b5dc3a2a516ea6ea959d6658e",
  "body": {}
}
```

The creation of a signed UMF message is accomplished by first creating a UMF message and obtaining a signature using a cryptographic library and algorithm such as sha256. Once the signature is obtained it can be added to the UMF message using the signature field.  The recieving end of the UMF message would then remove the signature from the UMF message and perform the same HMAC pass using the shared secret. The message is considered valid if the resulting signatures match.

<a name="Use-inside-of-HTTP"></a>
# 3. Use inside of HTTP

UMF documents may be sent via HTTP request and responses.  Proper use requires setting HTTP content header, Content-Type: application/json

```javascript
POST http://server.com/api/v1/message HTTP/1.1
Host: server.com
Content-Type: application/json; charset=utf-8
Content-Length: {length}

{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "rmid": "66c61afc-037b-4229-ace4-5ec4d788903e",
  "to": "uid:123",
  "from": "uid:56",
  "type": "dm",
  "version": "UMF/1.3",
  "priority": "10",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "message": "How is it going?"
  }
}
```

<a name="Peer-to-Peer-Communication"></a>
# 4. Peer-to-Peer Communication

UMF documents may be used to exchange P2P messages between distributed services.  One example of this would be a service which sends its application health status to a monitoring and data aggregation service.

For example:

```javascript
{
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "stats:server",
  "from": "chat:server:23",
  "version": "UMF/1.3",
  "priority": "10",
  "timestamp": "2013-09-29T10:40Z",
  "body": {
    "totalRooms": "200",
    "averageUsersPerRoom": "13",
    "averageRoomStay": "1200"
  }
}
```

In the example above a game server is sending a message to the stats:server indicating its stats at UTC 2013-09-29T10:40Z.

<a name="Infrastructure-considerations"></a>
# 5. Infrastructure considerations

<a name="Message-storage"></a>
## 5.1 Message storage

UMF is designed with distributed servers in mind. The use of mid’s (unique message IDs) allows messages to be stored by their message id in servers such as Redis, MongoDB and in other data stores / databases.

<a name="Message-routing"></a>
## 5.2 Message routing

The use of the UMF `to` and `from` fields support message routing. The implementation of message routers is deferred to UMF implementers.  The use of colon and forward slash separators is intended to allow routes to easily parse for target handlers.

<a name="Message-forwarding"></a>
### 5.2.1 Message forwarding

It’s possible to route message between (through) servers by implementing message forwarding.

For example:

```javascript
{
  :
  "mid": "ef5a7369-f0b9-4143-a49d-2b9c7ee51117",
  "to": "router:uk-router:chat:room:12"
}
```

The message above might be sent to a router (service) which then parses the to field and realizes that the message is intended for a UK server. So it forwards the message to the uk-router which in turn sends the message to a server which hosts room 12.

<a name="Short-form-syntax"></a>
# 6. Short form syntax

UMF is being used in IoT (Internet of Things) applications where message sizes need to remain small. To allow UMF to be used in resource contained environments a short form of UMF is offered.

For each UMF keyword an abbreviated alternative may be used:

Keyword | Abbreviation
--- | ---
authorization | aut
body | bdy
for | for
forward | fwd
from | frm
headers | hdr
mid | mid
priority | pri
rmid | rmi
signature | sig
timeout | tmo
timestamp | ts
to | to
ttl | ttl
type | typ
version | ver

This is an example of a message in shorten format:

```javascript
{
  "mid": "2b9c7ee51117",
  "to": "uid:123",
  "frm": "uid:56",
  "ver": "UMF/1.4",
  "ts": "2013-09-29T10:40Z",
  "bdy": {
    "msg": "How is it going?"
  }
}
```

When using UMF in IoT applications `mid` and `rmid` fields should use short hashes where possible. Also avoid using UMF fields which your application may not require such as `priority` and `type`.

Also, it's assumed that messages are transmitted as strings without carriage returns and line feeds to reduce transmission sizes.

```
{"mid": "2b9c7ee51117","to": "uid:123","frm": "uid:56","ver": "UMF/1.4","ts": "2013-09-29T10:40Z","bdy": {"msg": "How is it going?"}}
```

The use of gzip compression also greatly reduces message sizes.
