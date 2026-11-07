# Reaching a server through a Seshat edge

A server can be reached directly (the default) or **through a Seshat edge**: a small agent installed on the
server's own network that dials out to Seshat and nothing else, so that network accepts no inbound connection.
This page is for the people who set it up and for anyone writing code near it. The adapter does not change:
Kea, Windows Server and every other adapter run their own TLS, SSH or WinRM session, end to end, inside a byte
stream the edge opens. Seshat and the edge see ciphertext, and the DHCP server's credentials never leave
Lachesis.

## Set-up

1. In Seshat, create an integration client that holds the `edge.relay.tcp` capability with the edge in its scope,
   and issue it a token. On the edge, turn the relay on (mode `read` or `read+write`) and allow the server's
   address and port (`address:port`, `cidr:port` or `name@cidr:port`). A bare name is refused by Seshat: every
   address it resolves to must fall inside the pinned network.
2. In Lachesis, add or edit the server (Servers, Add a server or Edit): under **How Lachesis reaches this server**
   choose **Via a Seshat edge**, then fill in:
   - **Seshat address**: the console's base address, `https://seshat.example.com`. https only.
   - **Edge id**: the edge as Seshat names it.
   - **Seshat integration token**: write-only. Save the server first, then probe it: a connection cannot be
     tested before the token is stored.
   - **Plain text acknowledgement**: only for an `http://` endpoint or WinRM on 5985.

The same settings are `transport` in the server API (`mode` `direct` or `seshat_edge`, `seshat_url`, `edge_id`,
`plaintext_ack`) and the token is `PUT /api/servers/{id}/transport/token` (`DELETE` removes it). The token is
never returned by any API, never logged and never in an audit record; a server read says only whether one is
set. Going back to a direct connection removes the stored token.

Lachesis verifies the Seshat console's certificate against the system roots (`SSL_CERT_FILE` and `SSL_CERT_DIR`
apply, so a private authority is added the usual way for a Go program). There is no switch to skip it.

## What a relayed server may reach

The relayed dialer reaches the endpoint's host and port and the extra targets the adapter declares (for example a
Kerberos KDC), exactly those, and refuses everything else before any network I/O. It does **not** resolve the
target on the Lachesis host: the edge resolves names, on its own network, and enforces its own allow-list.
Lachesis still refuses to register an endpoint that is a loopback, link-local or metadata literal. The address of
the Seshat console is judged by the egress guard like any endpoint, and the connection to it is made through the
guard (ALPN `http/1.1`, no proxy, one `POST` per connection, the token as a Bearer header).

The relay adds no confidentiality. An endpoint whose own protocol is clear text (`http://`, or WinRM on port
5985) would send its credentials unencrypted through Seshat and the edge, so Lachesis refuses to save or dial it
unless the administrator ticks the acknowledgement.

## Intent: read and write

Every relayed session declares an intent to Seshat, which records it and refuses `write` on an edge whose relay
mode is `read`. Seshat cannot see inside the stream, so the guarantee is Lachesis's own:

| Session | Intent |
|---|---|
| Probe, connection reads (scopes, leases, reservations, search), drift reads, plan building | `read` |
| **Plan apply** (the sessions that write) | `write` |
| The verify that follows an apply (it opens its own sessions) | `read` |

The default is `read`, so a code path that forgot to declare an intent is refused rather than over-declared.
Only the plan engine's apply step asks for `write` (`servers.WithWriteIntent`); nothing else in Lachesis does.
Connections are closed when their session closes, so an idle keep-alive connection never holds one of Seshat's
relay slots (a client holds 2 by default).

## Errors and what to change

Each failure is a typed error with a message that says what to change. It is shown in the server's probe result
and in the plan's apply result. The text is fixed wording plus the edge id and the target; the token and Seshat's
own message or response body are never repeated.

| Seshat answers | Lachesis says (summary) | Kind |
|---|---|---|
| `edge_offline` (503) | the edge is not connected to Seshat; check it is running | unreachable |
| `relay_disabled` (403) | relaying is off on the edge or its allow-list is empty; set a relay mode and allow the server | invalid input |
| `target_not_allowed` (403) | the edge does not allow this address and port; add it to the allow-list | invalid input |
| `write_not_allowed` (403) | the edge accepts read sessions only; set `read+write`, then plan the change again (nothing was written) | invalid input |
| `dial_failed` (502, `error.fields.class`) | the edge could not connect: `refused` (not listening), `timeout`, `unreachable` (no route) | unreachable |
| `relay_timeout` (504) | the edge did not open the connection in time | unreachable |
| `limit_exceeded` (429) | one retry after `Retry-After` (at most 10 s), then busy; nothing was written | busy |
| 401 | Seshat did not accept the token | auth failed |
| `capability_missing`, `out_of_scope` (403) | the client lacks `edge.relay.tcp`, or the edge is not in its scope | auth failed |
| 404 | Seshat does not know the edge id | invalid input |
| anything else | "Seshat refused the relay request" with the status or a code of the right shape | unreachable |
| Seshat unreachable, or its certificate is not trusted | check the address, that Lachesis may connect, and the certificate | unreachable |
| no token stored | set the token in the server's transport settings | invalid input |

A `limit_exceeded` at the first step of an apply writes nothing, so the plan goes back to `ready` and the same
plan can be applied again (409 `server_busy`).

## Audit

Every relayed session writes one `relay.session` audit record when it closes, and one per refused attempt:
the server, the edge, the target (`host:port`), the intent, the outcome (`ok`, `error`, `refused`), the code of a
refusal, the bytes in each direction and the duration. It holds no token and no payload. Setting or removing the
token is recorded as `secret.set` / `secret.delete` with the field name only, and a change to the transport is
listed in `server.update`. Seshat audits the same sessions independently on its side.

## Limits

- The stream has no half-close (Seshat's relay has none): closing it ends the session. Read and write deadlines
  set by the adapter pass through to the stream.
- A connection test of a server that is not saved yet is refused for a relayed server (there is no stored token
  to present); save it, set the token, then probe.
- Seshat's integration token roll-over (two live tokens at a time) is done on the Seshat side: set the new token
  here when the old one is about to lapse.
