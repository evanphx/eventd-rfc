# eventd - A syslog replacement

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

#### What's good about heka?

* Structured logging at the bottom
  * Include units per attribute (seconds, milliseconds, uri, email, etc)
* Reliable delivery via builtin TCP transport
* Highly configurable, easy to route logs to interesting places

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



