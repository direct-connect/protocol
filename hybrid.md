# Hybrid hub

DC software is not limited to a single protocol like [ADC](./adc/ADC.txt) or [NMDC](./nmdc/nmdc.md)
and may support multiple protocols. This is especially true for clients, but was not widely
used in hubs.

Still, having a hub that supports all clients regardless of the protocol is desirable and
this document provides recommendations how a hybrid hub may be implemented.

## Accepting connections

When accepting connections, the hub does not know what protocol the client expects. To solve
this, [protocol detection](./protocol-detection.md) should be used.

New TLS-enabled hubs may use [ALPN](./alpn.md) instead, which is easier to implement and is
more reliable. However, support for old clients still require [protocol detection](./protocol-detection.md)
alongside with [ALPN](./alpn.md).

For simplicity, the hybrid hub may assign each connection a unique [SID](./adc/ADC.txt#session-id)
even when [NMDC](./nmdc/nmdc.md) protocol is used.

## Client identification

### ADC

In [ADC](./adc/ADC.txt), all clients provide their unique [CID](./adc/ADC.txt#client-id)
and prove its identity by additionally providing [PID](./adc/ADC.txt#private-id). The hub
may continue using [CID](./adc/ADC.txt#client-id) to identify all clients.

### NMDC

For [NMDC](./nmdc/nmdc.md), there is no way to provide a [CID](./adc/ADC.txt#client-id),
thus the hub should calculate it based on other data provided by the client. The hub
may use a combination of client's IP address adn client's nick name to generate a [CID](./adc/ADC.txt#client-id):

```
cid = Tiger(IP + "|" + Name)
```

The IP address is added to allow [ADC](./adc/ADC.txt) clients to distinguish two different
clients with the same name connected to different hubs:

```
A1    C    A2
  \  / \  /
   H1   H2
```

In the following diagram, both clients A1 and A2 has the same name but a different IP.
By hashing the IP address as well, the client C can distinguish between those two clients.

This approach also has side-effects, for example, the [CID](./adc/ADC.txt#client-id) of a
[NMDC](./nmdc/nmdc.md) client may change after a reconnection if the client has a dynamic
IP allocation or uses a proxy. This is an acceptable false-negative result. It is important
for the identification to not lead to false-positive results, because it may expose information
to a third party.

## P2P connection requests

The hybrid hub may translate P2P connection requests between [ADC](./adc/ADC.txt) and [NMDC](./nmdc/nmdc.md)
protocols, however, the C-C connection will only be established if both clients support
[ALPN](./alpn.md).

### ADC to NMDC

[ADC](./adc/ADC.txt) version of connection request is a [CTM](./adc/ADC.txt#ctm) command:
```
DCTM AAAA BBBB ADCS/0.10 port token
```

Since the protocol can only be negotiated by peers when establishing a TLS connection with
[ALPN](./alpn.md), the hub should only translate requests with a `ADCS/0.10` protocol.

Note that the connection request does not specify the IP of a peer, because it is sent
as a part of [INF](./adc/ADC.txt#inf):
```
BINF AAAA ... I4ip
```

The hub should use the `I4` field of [INF](./adc/ADC.txt#inf) and translate the request to
a secure [`$ConnectToMe`](./nmdc/nmdc.md#connecttome) request: 
```
$ConnectToMe BNick ip:portS|
```

Same approach can be used for translating the [RCM](./adc/ADC.txt#rcm) to [`$RevConnectToMe`](./nmdc/nmdc.md#revconnecttome).

### NMDC to ADC

[NMDC](./nmdc/nmdc.md) version of connection request is [`$ConnectToMe`](./nmdc/nmdc.md#connecttome):
```
$ConnectToMe BNick ip:portS|
```

Since the protocol can only be negotiated by peers when establishing a TLS connection with
[ALPN](./alpn.md), the hub should only translate requests with a `S` port suffix (secure).

Note that the [CTM](./adc/ADC.txt#ctm) command does not have an IP address, thus the hub
should fake an [INF](./adc/ADC.txt#inf) when translating this request:
```
BINF AAAA I4ip
DCTM AAAA BBBB ADCS/0.10 port token
```

[NMDC](./nmdc/nmdc.md) version of the request does not contain the token, so the random token
can be sent instead. The clients will rely on [identification](#client-identification) to
distinguish between connection from multiple peers.