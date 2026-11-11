# signet-nip46-client

A NIP-46 remote-signer client in Rust. An app pairs with a signer (such as a
My Signet persona) over Nostr relays, keeps a session that holds no connect
secret, and then asks the signer for its public key or for a signature. The
app never holds the signer's key.

This crate replaces `signet-nip46-client` 0.1, which wrapped rust-nostr's
`NostrConnect` and ran its own relay pool. This version sends and receives
over the `Relay` trait of [`fray-relay-client`](https://github.com/forgesworn/fray-relay-client),
so the whole flow can be tested against that crate's in-memory relay and a fake
signer. See "Coming from 0.1" below.

## What it offers

- `parse_bunker_uri` reads a `bunker://` URI into a `BunkerUri`.
  `RemoteSigner::from_bunker` makes a signer from it, and
  `connect_with_secret` pairs.
- `PendingPair::start` begins the client-initiated flow and returns a
  `nostrconnect://` URI to show to the signer. `await_signer` waits for the
  signer's answer and returns a `RemoteSigner`.
- `RemoteSigner` makes requests: `connect`, `connect_with_secret`,
  `get_public_key` and `sign_event`, or any other method through `request`.
  `request_with_id` does the same under an id the caller supplies.
- `RemoteSigner::persist(&persona)` gives a `PersistedSession`, and
  `RemoteSigner::restore` rebuilds a signer from one without sending anything.
- `build_nostrconnect_uri` builds the URI alone. `generate_connect_secret`
  makes a fresh secret or request id.
- `Error` is `Uri`, `Key`, `Connect`, `Signer` or `Timeout`.

`await_signer` and the request methods (`request`, `request_with_id`, `connect`,
`connect_with_secret`, `get_public_key`, `sign_event`) are `async` and use
`tokio::time`, so they must run on a tokio runtime with the time driver enabled.
`parse_bunker_uri`, `PendingPair::start`, `from_bunker`, `restore` and `persist`
are plain functions. `fray-relay-client` has one: `Task::spawn` runs a
future on it. The `Relay` you pass in is the caller's. This crate opens no
connection of its own.

## The wire

Every message is a Nostr event of kind 24133 whose content is encrypted with
NIP-44 (version 2) between the app's client key and the signer's key, and which
carries a `p` tag naming the receiver.

- Request, plaintext: `{"id": "<id>", "method": "<name>", "params": ["<string>", ...]}`.
  `params` is a list of strings. From `request`, the `id` is 16 random bytes in hex.
- Reply, plaintext: `{"id": "<id>", "result": "<string>", "error": "<string>"}`.
  Each field is optional.
- A request opens its reply subscription before it publishes, filtered to
  kind 24133, tagged to the client key and authored by the signer. A reply is
  taken only if it is from the signer, carries exactly one `p` tag and that
  tag names the client key, has a valid event signature, decrypts and parses,
  and carries the request's id. Anything else is ignored. A reply whose result
  is `auth_url` is ignored too, and the request goes on waiting. A non-empty
  `error` fails the request with `Error::Signer`; no `result` is
  `Error::Signer("empty reply")`. The call gives up with `Error::Timeout` at
  the signer's timeout.
- Content over 256 KiB is not decrypted.
- `sign_event` sends the template `{kind, pubkey, created_at, tags, content}`
  with `pubkey` set to the persona, and checks what comes back: a valid
  signature, signed by the persona, with the same kind, content and tags, and a
  `created_at` within 600 seconds of the one asked for. A signer that
  substitutes another event, key or time is refused with `Error::Signer`.

## Pairing

### `bunker://` (the signer gives the URI)

1. `parse_bunker_uri(uri)`. The URI must be `bunker://<64 hex signer key>` with
   at least one `relay=` that is a `wss://` URL, and may carry `secret=`. Relays
   that are not `wss://` are skipped. At most 8 relays are kept, each of at most
   256 bytes and deduplicated, and a URI over 2048 bytes or a secret over 256
   bytes is refused. Anything else is `Error::Uri`.
2. `RemoteSigner::from_bunker(&bunker, relay, timeout)` makes a signer with a
   fresh client key. Nothing is sent yet.
3. `connect_with_secret(bunker.secret.as_deref())` sends the URI's secret in one
   `connect` request and keeps it nowhere. Any reply that has a `result` and no
   `error` counts as success here, because the reply must already come from the
   signer's key (see "The wire").
4. `get_public_key()` returns the persona.

A `BunkerUri` prints its secret as `<redacted>`.

### `nostrconnect://` (the app gives the URI)

1. `PendingPair::start(relay, app_name, perms)` makes a client key and a secret
   (16 bytes from the operating system's random source, in hex), subscribes
   for replies to the client key, and returns the `nostrconnect://` URI with the
   client key, every relay, the `secret`, the `perms` if any, the app name as
   `name` and as `metadata={"name":...}`. Requested permissions remain subject to the
   signer's approval. It fails with `Error::Uri` if `relay` has no URLs.
2. Show the URI to the signer. `await_signer(timeout, on_listening)` calls
   `on_listening` once every relay has answered the reply subscription, after
   10 seconds (`LISTEN_GRACE`), or when the reply arrives, whichever is first;
   show the URI at that point, since the reply is
   ephemeral and one sent before anyone listens is lost.
3. The signer sends a `connect` reply. `await_signer` returns a `RemoteSigner`
   whose signer key is the sender of the accepted reply. It consumes the
   `PendingPair`, so one pairing accepts one answer.
4. `get_public_key()` returns the persona.

**The secret-echo rule.** A reply ends the wait with success only if its
`result` is exactly the secret from the URI, compared in constant time. A bare
`"ack"` is ignored. The relay sees our subscription, and so the client key, and
anyone who knows that key can send an encrypted `{"result":"ack"}` from a key of
their own. Only the signer that was shown the URI knows the secret, so only a
reply that echoes it identifies the signer. A reply that decrypts but has the
wrong secret, no secret, or is not a `connect` reply is ignored, and the wait
goes on. A reply with a non-empty `error` ends the wait with `Error::Signer`. It does
not have to carry the secret, so anyone who knows the client key (any listed
relay, or whoever saw the URI) can cancel a pairing that way, though they
cannot complete one. Unless the refusal also echoes the secret, its text is
replaced by `UNPROVEN_REFUSAL`, so an outsider cannot put words in the
signer's mouth.

## The persisted session

`PersistedSession` is plain data for the app to store (it derives `serde`
`Serialize` and `Deserialize`):

| Field | Holds |
|---|---|
| `app_secret_key` | hex secret of this app's own NIP-46 client key, not the persona's |
| `remote_signer` | hex public key of the signer that answers requests |
| `relays` | the relay URLs the signer listens on |
| `persona` | hex public key of the persona, as given to `persist` |

It holds no persona key and no connect secret. `app_secret_key` still lets
whoever has it talk to the signer as this app, so keep it as you would any
credential. `Debug` prints it as `<redacted>`. `bunker_uri()` gives the
equivalent secret-less `bunker://` URI. `RemoteSigner::restore(&session, relay,
timeout)` rebuilds a signer without sending anything; a bad `app_secret_key` is
`Error::Key` and a bad `remote_signer` is `Error::Uri`. `connect()` sends a
`connect` with an empty secret, for a signer that already knows the client.

`persist` stores the `persona` it is given and does not check it.

## Coming from 0.1

0.1 wrapped rust-nostr's `NostrConnect`. This version has no `NostrSigner`
implementation and no async wrapper. An app that wants one writes a thin adapter
over `get_public_key`, `sign_event` and `request`.

| 0.1 | Now |
|---|---|
| `BunkerSession::pair` | `parse_bunker_uri`, `RemoteSigner::from_bunker`, then `connect_with_secret` |
| `pair_nostrconnect` | `PendingPair::start`, then `await_signer` |
| `restore` | `RemoteSigner::restore(&session, relay, timeout)` |
| `persist` | `RemoteSigner::persist(&persona)` |
| `user_public_key` | `RemoteSigner::get_public_key().await`, a round trip each time |
| `shutdown` | none: the crate owns no connection. Drop the `RemoteSigner`; the `Relay` is the caller's to keep or drop |
| `impl NostrSigner` | none; use `sign_event`, `request` and `get_public_key` |

Besides the new transport, you now pass in the `Relay` and the timeouts.

## One copy of `fray-relay-client`

`PendingPair::start`, `RemoteSigner::restore` and `RemoteSigner::from_bunker`
take an `Arc<dyn Relay>` from `fray-relay-client`. Your build must hold one copy
of that crate, at the revision this crate pins, or the `Relay` you build and the
`Relay` this crate expects are different types.

```toml
fray-relay-client = { git = "https://github.com/forgesworn/fray-relay-client", rev = "7b2243046a01814ad4ad8797ce563425dbcd9011" }
```

## Known open

The `nostrconnect://` flow's `get_public_key` step has not been proven against
every signer. This crate's own tests do not run a pairing or request round trip
against a fake signer.

## Tests

```
cargo test
```

The unit tests build signers over `fray-relay-client`'s in-memory relay and need
no network. They cover the URIs, the reply check, the persisted session and the
signature check. They do not run a full pairing or request round trip.
`tests/no_game_names.rs` fails if a game or app name appears in library code.

## Licence

MIT. See `LICENSE`. `NOTICE` says where the code came from.
