# HTTP ping

The hub supporting [ALPN](../alpn.md) and/or [protocol detection](../protocol-detection.md)
should also serve an HTTP endpoint for pingers:

```
GET /api/v0/hubinfo.json
accept: application/json
user-agent: PingerX/1.0
```

`user-agent` header must be set and include the name and the version of a pinger software.

The response must be a valid JSON document containing hub information and live statistics.

```
content-type: application/json
server: HubX/1.0
```

`server` header must be set and include the name and the version of a hub software.

```json
{
    "name": "Hub name",
    "desc": "Hub description",
    "addr": [
        "adcs://some.hub.net:411",
        "dchub://fallback.hub.net:411"
    ],
    "icon": "/relative/img/path/favicon.png",
    "website": "https://www.hub.net",
    "email": "webmaster@hub.net",
    "users": {
        "cur": 1000,
        "max": 2000,
        "top": 1200
    },
    "share": {
        "cur": 1234567890,
        "min": 1000,
        "top": 12345678900
    },
    "uptime": 12345678,
    "encoding": "utf-8"
}
```

`name` is a short name of the hub that will be displayed in the hublist.

`desc` is a full hub description, that may include the topic, hub rules, etc.

`addr` is a list of alternative hub addresses, including fallback servers. List should be
sorted starting from the canonical address and followed by secondary addresses and
fallback servers. Addresses must include the protocol.

`icon` is a relative URL path for a hub's favicon. Absolute URLs are discouraged for
security reasons. The icon should be served by hub's own HTTP server.

`email` is a contact emails for hub's administration.

`website` is a website for this hub. Should match the [redirect](./website.md#redirect) address, if it is set.

`users.cur` is a current number of users connected to the hub.

`users.max` is a maximal number of users allowed on the hub. 

`users.top` is a maximal number of users seen on the hub for its lifetime.

`share.cur` is a current total size of files shared by users on the hub. In MB, rounded up.

`share.min` is a minimal user share required to enter the hub. In MB, rounded up.

`share.top` is a maximal total share of users seen on the hub for its lifetime. In MB, rounded up.

`uptime` is a hub's uptime in seconds (unix epoch).

`encoding` is an optional field for [NMDC](../nmdc/nmdc.md) hubs to specify encoding used
for search and chat messages. If not specified, `utf-8` must be assumed. Value of this field
must follow [W3C encoding](https://www.w3.org/TR/encoding/) recommendation.

### User list

Hubs may also serve a user list. The pinger may pass a `users=1` query parameter
to request the user list:
```
GET /api/v0/hubinfo.json?users=1
accept: application/json
user-agent: PingerX/1.0
```

The response is the same as for the fist case, but should include an additional
`users.list` field, containing an array of objects:

```json
{
    ...
    "users": {
        "cur": 2,
        "list": [
            {
                "name": "User 1",
                "share": 123456789
            },
            {
                "name": "User 2",
                "share": 123456789
            }
        ]
    }
}
```

`name` is a display name of a user.

`share` is a current total size of files shared by the user. In MB, rounded up.
