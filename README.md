# Simple Sensor Sample Streaming Push Protocol (S4PP)

## Rationale
Low-power sensor devices typically need to upload their samples to some
central location for collection, analysis and action. Some protocols have
been created which include this as a subset of their scope (see e.g. CoAP,
MQTT). These protocols however cater for a much larger feature scope, and
as such can be difficult and/or timeconsuming to implement on a small
constrained device. On the other end of the spectrum there are the web
services REST (or even SOAP) APIs, more often than not running on TLS.
Implementing support for these can be outright impossible given the
resource constraints on a microprocessor, especially with TLS in the
picture - even the initial handshake can easily eat up over 10k of RAM
due to the certificate exchange! As such, there is clearly a need for
a protocol whose only purpose is to upload samples that is both easy
to implement and has minimal state overhead.

## Design criteria and goals
The S4PP protocol sets out to achieve the following:

  - Data streaming support. At no point should the entire payload be
    needed to be constructed and kept in RAM.
  - Push RAM or CPU intensive operations to the server side.
  - Keep protocol overhead minimal.
  - Allow for both connection-oriented and connection-less transports.
  - Do not mandate Confidentiality, as in many cases the sample data is
    not secret. Confidentiality is thus left to the transport layer, to
    be used if so desired (e.g. via TLS).
  - Do mandate Integrity of data. Rubbish data is at best useless.
  - Do mandate Authentication of data. Rubbish data is at best useless.
  - Make no assumptions about the client having access to an accurate
    real-time clock source.
  - Enable pipelining of client commands and data to minimise time
    the device has to be on the network.
  - Avoid byte order issues by using the classic Internet approach of
    a text based protocol. Also makes debugging easy.

## Overview

An S4PP session logically comprises the following steps:

  1. \[Optional\] Client sends hello.
  1. Server sends hello.
  1. Server sends a challenge token.
  1. Client authenticates using the token.
  1. Client begins a data sequence.
  1. Client sends sample data.
  1. Client signs the data sequence.
  1. Server responds with whether it successfully committed the data.

These logical steps can however, thanks to pipelining, take place in a mere
three exchanges:

  1. Server sends hello and challenge token.
  1. Client sends authentication and a signed data sequence.
  1. Server accepts the data.

At any point after the hello exchange, the server may issue notification
commands to the client. These are advisory only, and may be ignored by
the client.


## Protocol format
The protocol is text-based, line oriented, using UTF-8 encoding.
The line terminator used by both the server and client MUST be
newline/linefeed (\\n, LF) **only**. Use of carriage-return+newline
(\\r\\n, CRLF) is not supported, and constitutes a malformed command.

Aside from the hello exchange and the actual data, the command format
takes the form `<command>:<param,...>\n`.

The protocol takes the old Unix style approach of being silent unless
there is an error, or an explicit response is needed. This helps keep
it both simple and efficient.

### Hello (S,C)
The initial hello message(s) are in the form:
```
S4PP/<ver> <supported-algorithms> <max-samples-per-sequence>
```
Where:
  - `ver` is the version of the S4PP protocol in use. Currently this is 1.0
  - `supported-algorithms` is a comma-separated list of cryptographic hash
    algorithms supported. This list MUST include SHA256.
  - `max-samples-per-sequence` is only included in the server hello, and
    tells the client how many samples the server is prepared to process
    in a single sequence. This value SHOULD be at least in the thousands,
    as RAM is far more abundant on the server side. If a client exceeds
    this value, the server MAY reject the upload.

The client SHOULD only ever send a hello message if using a connection-less
transport. A server SHOULD support receiving a client hello even on a
connection-oriented transport.

If a client sends a hello, the server MAY limit its `supported-algorithms`
field to algorithms it has in common with the client.

If a server and client do not share any common algorithms, both sides MUST
immediately terminate the session. This is an exceptional situation, as
SHA256 is mandated as a minimum common ground.

### Reject (S)
```
REJ:<message>
```
Where:
  - `message` is a short message describing the reason for rejection.

This command is sent by the server whenever it can't process a received
command. It can be thought of us as similar to HTTP response 400 Client
Error.

Receipt of this message typically implies the abortion of the entire
session. In case where the client has outside knowledge that this may
not be the case (e.g. implementation specifics), a client MAY attempt
to resume the session by submitting new authentication and/or data
sequences.

### Challenge token (S)
```
TOK:<challenge-token-string>
```
Where:
  - `challenge-token-string` contains a cryptographically random string
    to be used by the authentication and signature commands. It is
    suggested that it is in the form of ASCII hex-encoded bytes sourced
    from a cryptographically secure random number generator. Its length
    may vary, and is dependent on the desired cryptographic strength.
    It MUST be at least 1 character long and SHOULD be less than 128
    characters long. Clients MAY support longer token strings.

The challenge token is generated by the server and is used for authentication
and integrity confirmation. The server SHOULD use a new token for each
connection, as otherwise it is vulnerable to a replay-attack.

### Authentication (C)
```
AUTH:<algo>,<keyid>,<auth>
```
Where:
  - `algo` is the client-chosen cryptographic algorith, e.g. SHA256.
  - `keyid` is an identifier allowing the server to look up the shared key used.
  - `auth` is the Hash-based Message Authentication Code (HMAC) to
    authenticate with. It is calculated from `HMAC(key,keyid||challenge-token)`
    where || denotes concatenation.

Upon receiving this command the server MUST validate the given HMAC result
using the specified key id. If this validation fails, the server MUST send
a reject command and MAY terminate the session. On a successful validation
the server MUST stay silent.

If the client specified an algorithm not supported by the server, the
server SHOULD send a reject command, and MUST terminate the session.

### Begin sequence (C)
All sample uploads take place inside a *sequence*. Each sequence has an
explicit start, followed by data, and ends with a signature for the sequence.
A sequence can be thought of in terms of a database transaction. Until
the sequence signature has been validated the data has not been committed.

Inside the sequence, all sample timestamps are relative to the previous
timestamp. This is effectively a trivial delta-encoding to reduce the
amount of data being sent.

```
SEQ:<seqid>,<basetime>,<time-divisor>,<data-format>
```
Where:
  - `seqid` is an incrementing numeric sequence identifier. It MUST be
    larger than the previous sequence in the session, or the server
    MUST reject the command.
  - `basetime` provides an initial time stamp from which the subsequent
    samples will be relative to. It MAY be zero, in which case the first
    sample in effect contains the basetime for the sequence. The unit
    SHOULD be seconds and the epoch SHOULD be the Unix epoch, but
    exceptions MAY be made where the server and client both have outside
    knowledge of other parameters.
  - `time-divisor` is a numeric value specifying a value to divide the
    resulting timestamps (including the `basetime`) by in order to
    end up with a value in seconds from the epoch. This enables the
    avoidance of floating point calculations on the client side, where
    it could e.g. keep timestamps in milliseconds, and then set the
    `time-divisor` to 1000. This field MUST be non-zero, and if not
    explicitly used is to be set to 1.
  - `data-format` is a numeric field specifying the format of the samples
    to follow. See further down for available data formats. Values \[0,99\]
    are reserved for offical data formats. Other values MAY be used for
    experimental or site-specific formats.

### Dictionary entry (C)
Frequently, the biggest part of a sensor sample is the sensor name. Instead
of continually resubmitting this, the protocol uses a simple dictionary
approach where the client creates entries in the dictionary and simply
refers to those by id. This greatly reduces the amount of data to be
transmitted.

The dictionary is only valid for the duration of the sequence.

```
DICT:<idx>,<unit>,<unit-divisor>,<name>
```
Where:
  - `idx` is a numeric index for this dictionary entry. Entries MAY be
    redefined.
  - `unit` contains the unit the sensor's data is in, e.g. Celsius or RPM.
    This field MAY be empty if no unit is applicable or used.
  - `unit-divisor` is a numeric field specifying a value with which to divide
    the sensor readings by. E.g. a temperature sensor which records values
    in centigrade Celsius would submit the unit Celsius and a unit-divisor
    of 100. This field MUST be non-zero.
  - `name` contains the name of this sensor. Commonly this would include
    both a reference to the sensor type as well as the name of the node
    the sensor is attached to. This field MUST be non-empty.


### Data formats (C)
#### Data format 0
```
<idx>,<delta-t>,<value>
```
Where:
  - `idx` is the dictionary index referring to the sensor this sample is from.
  - `delta-t` is the delta from the previous timestamp in the sequence (with
    the initial delta-t being relative to the `basetime` in the sequence
    command). Negative deltas are valid.
  - `value` is the sample value. It SHOULD be a numeric value unless both
    the server and client has outside knowledge of other parameters (and in
    which case obviously the `unit-divisor` is not applicable). It MAY be
    NaN or +/-Inf, though these are discouraged.


### Sequence signature (C)
```
SIG:<signature>
```
Where:
  - `signature` is the HMAC of the sequence. It is calculated from
    `HMAC(key,challenge-token||sequence-data)` where the sequence data
    includes all commands and data in the sequence up to the signature
    command itself, i.e. \[SEQ,SIG\). The hash algorithm is the same
    as used in the authentication command.

The order of the components allows for streaming of data and updating
the HMAC calculation on the fly without ever having to buffer up the
entire sequence in memory. If a client does not send more than one
sequence it does not even need to store the challenge token since it
can be hashed into the HMAC right at the end of processing the token
command, and discarded thereafter.

On receipt of this command the server MUST validate the signature. If
the signature is not valid, the server MUST send a reject command and MAY
terminate the session. On a successful validation the server MUST send
a commit response command.

### Commit response (S)
```
OK:<seqid>
 or
NOK:<seqid>
```
Where:
  - `seqid` is the sequence id which was committed (or failed to commit).

Providing the data was successfully stored/handled, the OK response is used.
If for some reason (other than an invalid signature) the data could not be
stored/handled, the NOK (not-OK) response is used. The NOK can be thought of
as similar to a HTTP 500 Internal Server Error response.


### Notification (S)
```
NTFY:<code>[,args...]
```
Where:
  - `code` is a positive numeric code for a notification type. Values
    0-99 (inclusive) are reserved by this specification. Values above this
    range are available for vendor-specific extensions.
  - `args` used depends on the `code`.

At any point after the hello exchange, the server MAY issue one of more
notifications. While the intent is that a client will act on such
notifications, a client MAY ignore such directives, and MUST ignore all
(to it) unknown codes. Some notifications will only make sense to issue
once a client has successfully authenticated itself, others might be useful
to issue even prior to authentication.

A notification is purely a one-way, best-effort feature. A client does not
explicitly acknowledge receipt or processing of a notification.

The following notification codes are currently allocated:

#### Notification: Time (S)
```
NTFY:0,<utc_sec>[,<utc_ms>]
```
Where:
  - `utc_sec` is the current time expressed as UTC seconds.
  - `utc_ms` is the millisecond part of the current UTC time.

This notification is aimed at providing a simple time service "for free" to
devices which either lack time-keeping capabilities, or do not have a
sufficiently stable clock. The quality of this time service is low, and as
such the best precision offered is milliseconds, with seconds being the
only mandated precision. A server MAY omit the millisecond argument.

Applications requiring higher precision or accuracy are recommended to
employ proper time-keeping protocols, such as NTP.

There is no requirement for a client to be authenticated before this
notification is used.

#### Notification: Firmware version (S)
```
NTFY:1,<version>[,url]
```
Where:
  - `version` is the desired firmware version number of the client.
  - `url` is the location of where to source said firmware from.

This notification is aimed at providing a push notification for devices to
either verify that they're running the correct firmware, or to direct a
device to perform a firmware update. In particular for battery-operated
devices this can save the device from having to separately poll a resource
to determine whether it needs an upgrade.

The `version` argument SHOULD be a number but MAY be a string, but in such
case MUST NOT include the comma (`,`) character.

Depending on the methodology employed by the devices, the server MAY include
a URL for locating the firmware with.

Typically this notification will only be used after a client has successfully
authenticated.

#### Notification: Flags (S)
```
NTFY:2:<setflags>,<clearflags>
```
Where:
  - `setflags` is an ASCII hex-encoding of bits representing flags to set.
  - `clearflags` is an ASCII hex-encoding of bits representing flags to clear.

This notification provides a minimal approach for controlling device
behaviour. The behaviour/feature set is expressed as a bit field,
which is then hex-encoded. When encoding the flags, the server SHOULD NOT
zero-pad the values to a fixed width as this needlessly increases
the payload size. The server MUST use lower-case for any hex-digits
used. A client supporting this notification MUST accept a lower-case
hex-encoding, and MAY also accept upper-case. The total number of flags
(bits) used SHOULD NOT exceed 128. The interpretation of each flag is
implementation dependent. An empty flag set is expressed as zero. It is
thus possible (but not recommended) to use `NTFY:2,0,0` as a no-op command.

*Use case example:* A device has five LEDs, each mapped to the flags
at positions zero to four (numerical values 1, 2, 4, 8 and 16 respectively).
A notification may be sent to request the outer four LEDs be switched on,
and the middle one off: `NTFY:2,1b,4`

