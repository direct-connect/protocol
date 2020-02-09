# Automatic protocol detection for DC hubs

Hubs may use a described approach to auto-detect the protocol used by DC clients.
This allows to upgrade existing hubs from [NMDC](./nmdc/nmdc.md) to [ADC](./adc/ADC.txt),
or from NMDC/ADC to TLS without breaking existing clients.

First, the hub should set a read deadline on the connection to ~2x of a maximal expected
RTT (round trip time). Assuming a maximal RTT of 300ms, a deadline of 650ms may be used.
This number is a sensible default for a generic use case.

The hub should then read at least 4 bytes from the connection.

If the deadline is reached, the hub should assume NMDC protocol. It must increase the
read deadline for this connection and send [`$Lock`](./nmdc/nmdc.md#lock) as usual.
It may also send a [`$ForceMove`](./nmdc/nmdc.md#forcemove) and drop the connection
if NMDC protocol is not supported.

If received 4 bytes match the `HSUP` sequence (`0x48 0x53 0x55 0x50`), the hub must
assume [ADC](./adc/ADC.txt) protocol. It must increase the read deadline on this connection,
keep `HSUP` on the read buffer and proceed with ADC handshake.
It may also send an error:
```
ISTA 240 ADC\snot\ssupported<0x0a>
```
and drop the connection if ADC is not supported.

If first 4 bytes match any HTTP methods prefixes like `HEAD`, `GET `, `POST`, `OPTI`, etc.,
the hub should serve [HTTP](./http) on this connection.

If fist 4 bytes match the `NICK` sequence, the hub may serve IRC protocol for this connection,
or drop it otherwise.

If the first two received bytes match `0x16 0x03` sequence (`HANDSHAKE` record type
and major version `3`, meaning TLS 1.x), the hub must assume TLS for this connection
It must increase the read deadline, keep 4 bytes on the read buffer and proceed with
TLS handshake, setting an appropriate list of supported protocols for [ALPN](./alpn.md).

Note that if client does not support [ALPN](./alpn.md), the hub may start the protocol
detection again, this time reading 4 bytes from a TLS connection. Implementations must
disable TLS detection for this case to reject TLS-over-TLS connections.