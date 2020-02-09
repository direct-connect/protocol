# Application-Level Protocol Negotiation

[ALPN](https://tools.ietf.org/html/rfc7301) is a TLS extension used to negotiate the
application protocol that will be used after a TLS connection is established.
It helps DC clients to select between supported protocol like [ADC](./adc), [NMDC](./nmdc)
and [HTTP](./http) before sending any protocol messages. It also allows to [evolve](./post-alpn.md)
existing protocols without breaking old clients and hubs.

DC software should list supported protocols via ALPN when making secure client-client (C-C),
client-hub (C-H), pinger-hub (P-H) and other types of connections.

ALPN can be used in conjunction with [protocol detection](./protocol-detection.md).

- [Overview](#overview)
- [Supported protocols](#supported-protocols)
- [Client/Pinger to Hub](#clientpinger-to-hub)
- [Hub accepting connections](#hub-accepting-connections)

## Overview

When HTTP2 was introduced, developers were faced with a challenge to update existing
seb servers and browsers to a new protocol without breaking existing clients.

Instead of using special HTTP methods, a new mechanism was designed: Application-Layer
Protocol Negotiation. ALPN was built as an extension into the TLS handshake, thus a client
and the server can negotiate a protocol before sending any HTTP data. 

This allows to make any kind of changes that otherwise will be break users.

Also, since it is an extension, both parties will know if they are connected to a legacy
software that doesn't support ALPN, and can fallback to an old protocol.

The negotiation begins when the client sends `ClientHello` TLS message with a list of
supported protocols for ALPN.

The server picks a best protocols considering its own list and the list sent by the client
and sends the selected protocol back with a `ServerHello`. If the TLS handshake is successful,
application can start using the selected protocol over established TLS connection without
further feature negotiation.

For the full specification, see https://tools.ietf.org/html/rfc7301

## Supported protocols

When accepting or opening a new TLS connection, DC software must list one or more
supported protocols:

- `adc` for [ADC](./adc/ADC.txt)
- `nmdc` for [NMDC](./nmdc/nmdc.md)
- `h2` for [HTTP2](./http)
- `http/1.0`, `http/1.1` for [HTTP1](./http)

Protocols are specified in the order of descending preference, meaning that the first
protocol is preferred over the next one. For example:

```
h2,adc,nmdc
```

If both peers support `adc` and `nmdc` but not `h2`, `adc` will be selected.

## Client/Pinger to Hub

When making a secure C-H connection (`adcs://` and `nmdcs://` URI schemes), or P-H connection
(`https://`, `adcs://` and `nmdcs://` URI schemes) the client (or pinger) should not assume
that the scheme matches the protocol, and instead set the list of supported protocols via ALPN:

```
adc,nmdc
```

Note that the pinger must prioritize [HTTP pinger API](./http/ping.md) when connecting to
a TLS-enabled hub:

```
h2,http/1.1,http/1.0,adc,nmdc
```

There might be 3 possible outcomes:

1) The hub does not support ALPN. In this case, the client should use the protocol specified
   by the URI scheme: [ADC](./adc/ADC.txt) for `adcs://`, [NMDC](./nmdc/nmdc.md) for `nmdcs://`
   and [HTTP](./http/ping.md) for `https://` for pingers.
2) Both peers support ALPN, and a protocol was negotiated. Client should ignore the URI
   scheme and use the negotiated [protocol](#supported-protocols).
3) ALPN is supported, but an intersection was not found. The client should fallback to the
   protocol specified by URI scheme (see 1).
   
For all non-TLS URI schemes the client should continue using the protocol specified by URI.

### Implementation

Implementation of the client side can be divided into two stages that can be implemented atomically.

#### Stage 1: Single protocol

The first stage requires only few lines of change, as long as client already supports TLS.

The client sets a single protocol in ALPN - the one expected from the URI scheme:
`nmdc` on the NMDC-over-TLS code path (`nmdcs://`) and `adc` for ADC-over-TLS (`adcs://`).

It won’t be a real protocol negotiation, because the client only advertises a single protocol,
but it will be the simplest way to test the feature.

**Benefits for the client:**

- Allows to [evolve](./post-alpn.md) a specific protocol (`nmdc`, `nmdc/1`, …).
- Faster connections to hybrid hubs (that support both ADC and NMDC).

**What won't be supported:**

- Protocol negotiation when connecting to a hybrid hub (client will use the protocol from URI).
- Protocol auto-detection when connected to a secure hub that supports a single protocol.

#### Stage 2: Protocol negotiation

Specifying multiple protocols will enable full support for protocol negotiation.

This may require more changes depending on the client architecture. The client should be
able to open the TLS connection first, ignoring the URI scheme and pick the protocol
implementation only after the successful TLS handshake.

**Benefits for the client:**

- Allows to [evolve](./post-alpn.md) or switch DC protocol (`adc/1`, `nmdc/1`, …).
- Allows to experiment with new protocols.
- Faster connections to hybrid hubs (that support both ADC and NMDC).
- Protocol negotiation and auto-detection when making a secure connection to the hub.

### Examples

#### C/C++ and OpenSSL

ALPN is supported starting from OpenSSL v1.0.2.

The list of supported protocols is similar to other SSL parameters and should be set on a
[`SSL_CTX`](https://www.openssl.org/docs/man1.1.1/man3/SSL_CTX_new.html) instance using
[`SSL_CTX_set_alpn_protos`](https://www.openssl.org/docs/man1.1.1/man3/SSL_set_alpn_protos.html)
or on the [`SSL`](https://www.openssl.org/docs/man1.1.1/man3/SSL_new.html) instance using
[`SSL_set_alpn_protos`](https://www.openssl.org/docs/man1.1.1/man3/SSL_set_alpn_protos.html).

The list itself is a byte array in the ALPN wire encoding. For example, the following list
of protocols:
```
adc,nmdc
```

Corresponds to the following byte array in C/C++: 
```c
unsigned char alpn_protos[] = {
    3, 'a', 'd', 'c',
    4, 'n', 'm', 'd', 'c'
};
```

The first byte of each line specifies the length of the protocol name. The list is **not**
null-byte-delimited, thus the length must be passed to those functions:
```c
SSL_CTX_set_alpn_protos(ctx, alpn_protos, sizeof(alpn_protos));
```

After the call to [`SSL_connect`](https://www.openssl.org/docs/man1.1.1/man3/SSL_connect.html)
the selected protocol can be obtained using [`SSL_get0_alpn_selected`](https://www.openssl.org/docs/man1.1.1/man3/SSL_set_alpn_protos.html):

```c
const unsigned char* proto = 0;
unsigned int len = 0;

SSL_get0_alpn_selected(ssl, &proto, &len);

if (len != 0)
    printf("ALPN negotiated %.*s\n", len, proto);
```

## Hub accepting connections

The hub may accept TLS and unencrypted connections on a single port by using [protocol detection](./protocol-detection.md).

Still, when serving TLS connections, the hub should set one or more supported protocols via ALPN.

For hubs supporting a single protocol the list may be the following:
```
adc
```

Note that even when the hub only supports a single protocol, it can still benefit from
ALPN by supporting multiple protocol revisions (see [post-ALPN features](./post-alpn.md)).

It is strongly recommended to also support [HTTP pinger API](./http/ping.md) to reduce the
load on the hub. In this case the hub should also list HTTP1 protocols:
```
http/1.1,http/1.0
```

and/or HTTP2 protocol:
```
h2
```

The resulting list will be the following (HTTP is preferred):
```
h2,http/1.1,http/1.0,adc
```

Hybrid hubs that support both ADC and NMDC should specify both protocols as well:
```
h2,http/1.1,http/1.0,adc,nmdc
```

There might be 3 possible outcomes:

1) The client does not support ALPN. In this case, the hub should use the [protocol detection](./protocol-detection.md)
   or fallback to the default protocol instead.
2) Both peers support ALPN, and a protocol was negotiated. Hub should use the negotiated
   [protocol](#supported-protocols).
3) ALPN is supported, but an intersection was not found. The hub should fallback to case 1.

### Implementation

Implementation of the hub side can be divided into two stages for a single-protocol hub.
For hybrid hubs, the first stage doesn't make sense, thus they should implement stage 2 directly.

#### Stage 1: Single protocol

The first stage requires only few lines of change, as long as the hub already supports TLS.

The hub sets a single protocol in ALPN - the one it already serves over TLS:
`nmdc` for the NMDC-over-TLS (`nmdcs://`) and `adc` for ADC-over-TLS (`adcs://`).

It won’t be a real protocol negotiation, because the hub only advertises a single protocol,
but it will be the simplest way to test the feature.

**Benefits for the hub:**

- Allows to [evolve](./post-alpn.md) a specific protocol (`nmdc`, `nmdc/1`, …).
- Maybe a reason to switch to secure TLS connections by default.

**What won't be supported:**

- Protocol negotiation for clients and pingers.
- Embedded HTTPS server.

#### Stage 2: Protocol negotiation

Specifying multiple protocols will enable full support for protocol negotiation.

For single-protocol hubs, this will allow to use [HTTP ping](./http/ping.md), and
[evolve](./post-alpn.md) the protocol they already support.

For hybrid hubs, this will additionally allow to select the client protocol more efficiently.

**Benefits for the hub:**

- Allows to [evolve](./post-alpn.md) or switch DC protocol (`adc/1`, `nmdc/1`, …).
- Allows to experiment with new protocols.
- Faster connections for hybrid hubs.
- Protocol negotiation for clients and pingers.
- Allows to embed HTTPS server.

### Examples

#### C/C++ and OpenSSL

ALPN is supported starting from OpenSSL v1.0.2.

The the callback to select the protocol is similar to other SSL parameters and should be set on a
[`SSL_CTX`](https://www.openssl.org/docs/man1.1.1/man3/SSL_CTX_new.html) instance using
[`SSL_CTX_set_alpn_select_cb`](https://www.openssl.org/docs/man1.1.1/man3/SSL_set_alpn_protos.html).

The callback can be set the following way:
```c
SSL_CTX_set_alpn_select_cb(ctx, alpn_select_protocol, NULL);
```
The third parameter is an optional pointer that will be passed as a parameter to the
`alpn_select_protocol` and can be sued to track any server-related state.

The protocol list itself is a byte array in the ALPN wire encoding. For example, the
following list of protocols:
```
adc,nmdc
```

Corresponds to the following byte array in C/C++: 
```c
unsigned char alpn_protos[] = {
    3, 'a', 'd', 'c',
    4, 'n', 'm', 'd', 'c'
};
```

The first byte of each line specifies the length of the protocol name. The list is **not**
null-byte-delimited, thus the length must be passed to the selection function (`SSL_select_next_proto`).

The callback code may be the following:
```c
int alpn_select_protocol(SSL *ssl, const unsigned char **out, unsigned char *outlen,
                                 const unsigned char *in, unsigned int inlen, void *arg)
{
    int res = SSL_select_next_proto((unsigned char **)out, outlen,
                    alpn_protos, sizeof(alpn_protos), in, inlen);
    if (res == OPENSSL_NPN_NO_OVERLAP)
    {
        // set default protocol
        *out = alpn_protos;
        *outlen = 1+alpn_protos[0];
    }
    return SSL_TLSEXT_ERR_OK;
}
```

The `arg` parameter corresponds to the third argument in the `SSL_CTX_set_alpn_select_cb`
and can be used to pass the server structure that will track any relevant state.

## Client to Client

Clients should also use ALPN when making client-client connections. It allows to use ADC
protocol for C-C when connected to NMDC hub, or benefit from other [post-ALPN features](./post-alpn.md).

### NMDC hub

When connected to [NMDC](./nmdc/nmdc.md) hub, the clients use [`$ConnectToMe`](./nmdc/nmdc.md#connecttome)
or [`$RevConnectToMe`](./nmdc/nmdc.md#revconnecttome) to connect to each other.

Clients must prefer TLS connections when sending those commands:
```
$ConnectToMe user 192.168.1.2:3000S|
```

### ADC hub

When connected to [ADC](./adc/ADC.txt) hub, the clients use [`CTM`](./adc/ADC.txt#ctm)
or [`RCM`](./adc/ADC.txt#rcm) to connect to each other.

Clients must prefer TLS connections by specifying `ADCS/0.10` as a protocol:
```
DCTM AAAA BBBB ADCS/0.10 3000 token|
```

### Implementation

#### Stage 1: Single protocol

Similar to the Stage 1 for C-H and H-C connections, the client should specify a single
protocol for ALPN - the one that is used for communication with the hub.

**Benefits for the client:**

- Allows to [evolve](./post-alpn.md) a specific protocol (`nmdc`, `nmdc/1`, …).

**What won't be supported:**

- Protocol negotiation when connecting to a hybrid hub (client will use a single protocol for C-C).

#### Stage 2: Protocol negotiation

Similar to the Stage 2 for C-H and H-C connections, the client should specify multiple
protocol that it supports:
```
adc,nmdc
```

This will effectively allow to make [ADC](./adc/ADC.txt) C-C connections on [NMDC](./nmdc/nmdc.md)
hubs and vise-versa.

**Benefits for the client:**

- Allows to [evolve](./post-alpn.md) or switch DC protocol (`adc/1`, `nmdc/1`, …).
- Allows to experiment with new protocols.
- Protocol negotiation when connected to a hybrid hub.

However, there is really no point in making [NMDC](./nmdc/nmdc.md) C-C connections, because
[ADC](./adc/ADC.txt) is more compact, easy to parse and provides all the C-C functionality
that [NMDC](./nmdc/nmdc.md) supports. Thus, clients should always specify `adc` as a supported
protocol in Stage 2, even when connected to [NMDC](./nmdc/nmdc.md) hub.

Note, that clients still need to fallback to [NMDC](./nmdc/nmdc.md) protocol for C-C
connection to be able to connect to old clients.
For details, see outcome 1 and 3 in the C-H stage 2 implementation.

Also note that making [ADC](./adc/ADC.txt) connections on a [NMDC](./nmdc/nmdc.md) hub
requires the client to use a generated [CID](./adc/ADC.txt#client-id) to prove their identity
to the peer. See NMDC section in [Client identification](./hybrid.md#client-identification)
for more details.

### Examples

See examples in C-H and H-C sections of this document.
