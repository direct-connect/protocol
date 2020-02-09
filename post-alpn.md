# Post-ALPN features

This document describes the features that can be introduced after the [ALPN](./alpn.md)
is implemented in at least in one client and one hub.

- [HTTP ping](#http-ping)
- [NMDC](#nmdc)
  * [nmdc/1](#nmdc1)
  * [nmdc/2](#nmdc2)
- [ADC](#adc)

## HTTP ping

Writing the pinger is currently challenging because it needs to support different
[NMDC](./nmdc/nmdc.md) protocol dialects and fight against different hub limits meant
for users and not pingers.

Instead, ALPN allows to serve a [HTTP ping API](./http/ping.md) based on JSON on the same
port as the hub itself.

This makes the pinger as simple as a single HTTP GET request.

## NMDC

[NMDC](./nmdc/nmdc.md) protocol has a pretty long history and has tons of flaws in it.
ALPN allows to not only select between different protocols, but to also implement different
protocol versions. Thus, it can be used to fix few long-standing problems in existing protocols.

ALPN allows to introduce a breaking change and ensures that the new protocol is used only
for clients that explicitly advertised the support for it. Thus, the general notion is to
introduce new protocol versions instead of adding more extensions. Note that both `nmdc`
(current protocol version) and `nmdc/x` (new versions) may coexist.

### Current issues

- Unknown encoding. Should be UTF-8 by default.
- Message delimiter interfere with string escaping.
- Too many escape sequences for strings.
- Nick names are not delimited in most commands.
- Chat messages are a special case when parsing.
- Requires a custom parser for each command.
- Unused legacy from original NMDC software.
- Too many protocol dialects.
- Handshake is too verbose: [`$Lock`](./nmdc/nmdc.md#lock) is not needed.
- Not extensible: hubs should know about the extension, or they won't broadcast it.
- [`$MyINFO`](./nmdc/nmdc.md#myinfo) is not extensible. No extension list in the user info.
- Not extensible: requires to define new commands instead of adding fields to existing ones.
- Too many de-facto required extensions.
- No timestamps in messages.
- No tokens in [`$ConnectToMe`](./nmdc/nmdc.md#connecttome).
- No support for multiple IPs (IPv4 and IPv6).
- Nick names in commands instead of short IDs.
- Problems with passive-passive connections.

### nmdc/1

To make sure the change is implemented into as many clients and hubs as possible, the scope
of the first NMDC protocol revision (`nmdc/1`) should be relatively small.

The "safe" list of changes can be divided into 3 categories:
- Deprecate unused or superseded commands.
- Declare widely-used extensions as required.
- Clarify the protocol flow, make it less verbose.

#### Deprecate commands

Those legacy commands can be safely removed since they are either unused, or a better
extension exists and are widely used.

- [`$Ping`](./nmdc/nmdc.md#ping) - never used, peers send `|` instead.
- [`$GetINFO`](./nmdc/nmdc.md#getinfo) - superseded by [`NoGetINFO`](./nmdc/nmdc.md#nogetinfo).
- [`$NickList`](./nmdc/nmdc.md#nicklist) - superseded by [`NoHello`](./nmdc/nmdc.md#nohello).
- [`$Get`](./nmdc/nmdc.md#get), [`$Send`](./nmdc/nmdc.md#send) - superseded by [`ADCGet`](./nmdc/nmdc.md#adcget).
- [`$FileLength`](./nmdc/nmdc.md#filelength), [`$GetListLen`](./nmdc/nmdc.md#getlistlen), [`$ListLen`](./nmdc/nmdc.md#listlen) - superseded by [`ADCGet`](./nmdc/nmdc.md#adcget).
- [`$MultiConnectToMe`](./nmdc/nmdc.md#multiconnecttome), [`$MultiSearch`](./nmdc/nmdc.md#multisearch) - hubs never use federation.
- [`TTHSearch`](./nmdc/nmdc.md#tthsearch) - superseded by [`TTHS`](./nmdc/nmdc.md#tths-sa-and-sp).

#### Standard extensions

This list of extensions will be required when client advertises `nmdc/1`. They are no longer
optional and will not be listed in [`$Supports`](./nmdc/nmdc.md#supports).

- UTF-8 encoding.
- Pings with `|`.
- [`NoHello`](./nmdc/nmdc.md#nohello).
- [`NoGetINFO`](./nmdc/nmdc.md#nogetinfo).
- [`SaltPass`](./nmdc/nmdc.md#saltpass).
- [`$HubTopic`](./nmdc/nmdc.md#hubtopic).
- [`UserIP2`](./nmdc/nmdc.md#userip-extension-userip2).
- [`XmlBZList`](./nmdc/nmdc.md#xmlbzlist).
- [`ADCGet`](./nmdc/nmdc.md#adcget).
- [`TTHS`](./nmdc/nmdc.md#tths-sa-and-sp).
- [`$UserCommand`](./nmdc/nmdc.md#usercommand)
- [`$MCTo`](./nmdc/nmdc.md#mcto)

#### Protocol flow

- Deprecate [`$Lock`](./nmdc/nmdc.md#lock), send [`$Supports`](./nmdc/nmdc.md#supports) directly.
- Use [`$Error`](./nmdc/nmdc.md#error) during handshake.
- Specify exact command order during the handshake.
- Specify when the user list ends.

<!-- TODO: remove client-client dice roll? -->
<!-- TODO: check if ACTM is standard -->

### nmdc/2

This version is aimed to change the way how most commands are parsed to remove ambiguities.
It should make NMDC faster, more secure and easy to develop new software for.

List of proposed changes:
- Length-delimited commands.
- Remove old character escapes, use ADC escapes instead.
- Timestamps in messages.
- Introduce `$Msg`.

#### Length-delimited commands

Currently, the NMDC parser needs to read an unknown amount of bytes to parse the command.

This forces the server to search for a delimiter (`|`) in the received data, which is O(N)
time complexity (where N is the size of the message).

It also requires the `|` character to be escaped in all protocol messages.

Instead, the proposal is to specify the message length upfront by writing the length in the
first few bytes.

```
<21>some protocol message
```

Note that the command no longer has a `|` separator at the end. Instead the length (`21`)
is used to split messages.

### Better string escapes

NMDC has too many separators and escape sequences. Instead, ADC escapes can be used
to simplify the parsing code.

### Timestamps in messages

NMDC provides no standard way to add timestamps to messages. Because of this, clients
interpret chat replays as new messages. Adding a timestamp will solve this issue.

Also, it will allow to handle chat replays properly: the client may specify the last timestamp
of a message it received, and the server can replay only messages that were not delivered.

### Introduce `$Msg`

Instead of sending the message text directly, a new `$Msg` command can be used. This will
make the protocol easier to parse and remove chat message special case.

## ADC

[ADC](./adc/ADC.txt) has a more solid design and has not that many issues, compared to [NMDC](#nmdc).

ALPN allows to not only select between different protocols, but to also implement different
protocol versions. Thus, it can be used to improve the existing protocol.

ALPN allows to introduce a breaking change and ensures that the new protocol is used only
for clients that explicitly advertised the support for it. Thus, the general notion is to
introduce new protocol versions instead of adding more extensions. Note that both `adc`
(current protocol version) and `adc/x` (new versions) may coexist.

### Current issues

- Parser needs to search for a message delimiter.
- Has de-facto required extensions.
- Too verbose when making search requests.
- Client have to repeat own SID, while the hub knows it already.
- Inconsistencies in `I` vs `H` routing (handshake).
- Inconsistencies in `I` vs `B` routing (`IQUI`).
- `E` routing is weird, should ACK instead.
- `B` routing is really an `F` with a wildcard. Forces to send both `B` and `F`.
- `H` routing can be changed to `D` if hub allocates `AAAA` SID for itself.
- `ISTA` and `DSTA` has no token for the command it relates to.
- Missing advanced OP commands, hub rules, etc.
- IPs of all users are exposed in the user list.
