# eventd - A syslog replacement

### Goals

* Establish a common format for log/event data
* Establish a standard transport for log/event data
  * The transport provide reliable delivery, meaning the sender must get acknowledgment
* Work with existing projects to integrate the new format/transport

### Non-Goals

* Replace all existing logging software with a different piece of logging software

### Current landscape

#### What's wrong with syslog

* Message loss possible even with TCP and TLS
  * Lack of ack'ing from the receiver side means logs can be lost in the transmission window
* No ability to perform custom authentication / authorization
  * TLS supports cert auth, which is only useful within a single org unit
  * There is no stream metadata
* RFC5424 structured logging barely used
  * Virtually no clients give access to it
* Data treated as text blobs mostly

#### What's good about syslog

* Simple protocol
  * Easy to implement
* Highly visible, extremely well integrated already

#### What's wrong with rsyslog

* Using structured logging is cumbersome
* Difficult to configure
  * Undesirable config file format

#### What's good about rsyslog

* High performance implementation
* Lots of plugins to route data to various places
* "A standard" amongst ops folks
* Reliable sending via RELP
* Exposes json data about logs

#### Why not systemd-journal

* Few people using it for centralization
* Only one implementation to read disk files
  * Disk format is purposfully underspecified for flexibility
* Huge dependency if needed to be build freestanding
  * Rarely if ever run on anything but a modern Linux

#### What's good about systemd-journal

* Integrated tooling to search and manipulate log data
* Structured logging provides much more than just a string
  * Automatic addition of process context to each message

#### What about heka?

* No syslog integration, requires users to switch to it entirely
* Heavy overhead per message
  * UUID per
  * HMAC per
* Highly configurable, not a great out of the box expeirence
* No text format for the native format, makes it harder to interact with
* Native TCP transport does not do reliable delivery

#### What's good about heka?

* Structured logging at the bottom
  * Include units per attribute (seconds, milliseconds, uri, email, etc)
* Reliable delivery via builtin TCP transport
* Highly configurable, easy to route logs to interesting places
* HTTP based input and outputs with header configuration

#### What's wrong with logstash?

* No mandated format
  * Leaves users guessing on how to structured their data
* Requiring java turns many users away
* Difficult (if possible at all) to dynamicly detect different message encodings
  * Meaning sending json or plain text over syslog isn't possible without treating the json as plain text
* Only supports minimal internal buffering
  * High volume servers with outputs that go away for even a moment grind to a halt
  * Highly impactful when coupled with reliable delivery output plugins

#### What's good about logstash?

* Highly, highly configurable
* Supports syslog
* An ecosystem of existing shippers to get logs into the system

#### What's wrong with GELF?

* UDP usage is recommended and is not reliable at the network and application layer
* Mandates compression per message, increasing processing power per message significantly
* Forces users into awkward underscore prefixed notation to send structured data (ie, not canonical JSON)
* Timestamps don't mandate nanosecond precision
* No reliable transmission of log data
  * Graylog mentions an HTTP input plugin, but no documentation could be found
  * Unknown if an HTTP output plugin exists

#### What's good about GELF?

* Lots of libraries, makes application integration easy
* Native structured logging mechanism
* An ecosystem of plugins support various transports

#### What's wrong with fluentd?

* Custom forwarding protocol, makes interoperating difficult
  * Enabling reliable mode of protocol slightly confusing, off by default
* Secure version of the protocol handled by separate plugin that doesn't support reliable delivery
* No ability to detect and switch formats automatically

#### What's good about fluentd?

* Custom protocol handles load balancing and fail over
  * Though lack of acking makes this process pretty lossy
* Structured logging via JSON and MessagePack
* Ability to integrate with syslog
* Large ecosystem of plugins to give users maximum flexibility

#### What's wrong with flume?

* No structured logging support
  * Events are a header (string map) and body (bytes)
* Effectively custom protocols for introducing data reliably
  * Avro and Thrift have very small mindshares wrt logging
* Very technical configuration
  * Hard for users to get started with it
* JSON format not canonical for existing JSON events
  * Sepearate header/body subobjects
  * Timestamp is a quoted number
* No ability to provide authentication/authorization with sinks

#### What's good about flume?

* A few reliable protocols for users to pick from (including HTTP)
* Well documented ability to form pipelines between flume instances
* Nice set of source types gives users flexibility
* Utilization of batching as a core concept
* Large ecosystem of plugins for maximum flexibility
* The ability to overflow data to disk if outputs are backing up

### Feature breakdown and comparison

#### Structured Logging format

* Support for JSON as a first class structured format is high
  * Flume's support is a misnomer, the users data isn't structured
* Syslog can carry JSON in the message body when necessary
* Fluentd also supports MessagePack and uses it as it's internal wire protocol too
* Logstash's JSON is the most generic, mandating only @timestamp and @version
  * https://logstash.jira.com/browse/LOGSTASH-675
  * Clear preference for storing user data at the toplevel
* Heka uses an internal protobuf representation of the data
* Using JSON only brings some typical questions:
  * Can large integers be represented as native numbers properly? Or do we resort to quoted numbers?
  * How should timestamps be represented?
    * There is a large question about leap second visibility in events as well
  * How should binary values be represented?
    * Systemd represents them as an integer of numbers
    * Golang represents them by base64 encoding the binary
    * A subobject would have to used to transmit the proper data to the receiver in either case

##### Open Questions

* Should there be a common binary format as well as text (likely JSON at this point)?
* Would a binary format allow for faster aggregator handling?

#### Transport

* syslog protocol is the only common format support (and some don't even support it)
* Every project has their own native (ie, ad-hoc) protocol 
* A common protocol would reduce bugs in all projects and increase usefulness
  * Each reliable delivery protocol uses slightly different, ad-hoc rules to enforce reliability
* Existing reliable delivery protocols utilize head blocking, meaning they're the same as batch sending
* Primary downside of sender side batching is the receiver side doesn't see the flow of messages even if reliable delivery would cause the sender to block like it was a batch
  * This can be mitigating with HTTP by having the client start a "long POST" and trickle the data across the wire as it comes in
  * This still requires the sender to buffer data in case receiver crashes or returns non-200

### Desired Features

#### Reliable Delivery

* Messages must be ack'd by the receiver side
  * Preferably done in batches/windows to keep performance up
* Never lose messages to increase performance

#### Authentication

* A stream of messages should be able to carry stream-wide authentication/authorization info
* Using TLS certificate checks is not enough
  * Difficult to adminster when dealing with large number of client certs
* Currently no log protocols or routers do this

#### Structured Logging

* Makes the log system into a true data-rich event system
* Turns logs into a database system
* **LEARN**: Heka's value unit is really interesting, gives an extra dimesion to the structured data

#### Syslog integration

* Needs the ability to accept and process syslog formatted data always
  * Can't just leave existing infrastructure high and dry
  * Want to give them an easy path into a better world
  * Opens the door for simply relays: syslog sends data to eventd on localhost, which sends it elsewhere reliabily

### Implementation

#### Transport over HTTP/1 and HTTP/2

* Piggy back on well established tooling for sending batches of data and acking them
* Persistent connections keeps it on par with spitting syslog out a TCP session
* Remote side crashing mid batch detected
  * Remote side can reject batches via HTTP codes
* HTTP based authentication well understood
  * Authentication header
    * BasicAuth
    * Bearer Tokens
* Host based routing allows for servers to aggregate logs for many different orgs and separate them out easily
* Servers can throttle clients to induce backpressure and keep the system running
* Content-Encodings such as gzip can be utilized transparently

##### The bad

* Slows down "realtime" aspects by a fixed interval to allow for a batch to fill. Can be as low as 5s.
* Clients have to have strategy for when server is failing
  * Problem can be mitigated by using on-node forwarders that deal with this issue

#### Protobuf native format

* Well oiled binary protocol readers for every language
  * Easy to write new tools/implementations in various languages
* Compact and unambigious, improving ability to understand and evolve specification

#### Canonical JSON format

* Events can be represented in JSON such that common types and formats are presented in the canonical way
* All tooling capable of reading and writing JSON rather than protobuf

##### The bad

* Not plain text, requires tools to read
* Requires some custom framing for unterminated, append only files
  * Can be as minimal as a varint size header

#### Syslog listener

* Support /dev/log protocol as well as UDP and TCP syslog protocols transparently


### Unknown features

#### HMAC

* Supported by heka and systemd-journal
* Is HMAC on individual messages really needed?
* Can/should HMAC be done on batches?
* Who verifies HMAC signatures and when?



