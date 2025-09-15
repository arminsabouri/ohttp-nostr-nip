# NIP-X -- Nostr Relays as OHTTP Targets

Author: Armin Sabouri <me@arminsabouri.com>

This NIP defines how a Nostr relay can expose an **Oblivious HTTP (OHTTP) target** interface so that clients can send Nostr requests through an **OHTTP relay** without revealing their network identity to the target relay. Each request is encapsulated with an ephemeral HPKE key pair and routed via an OHTTP relay as per RFC 9458. The target relay processes the (decapsulated) request and returns a response; the OHTTP relay cannot read contents, and the target relay cannot see client metadata.

Critical note: this design relies on **two distinct roles** (OHTTP *relay* and OHTTP *target*). If a single operator controls both and colludes, client unlinkability collapses (a known limitation of OHTTP). See RFC 9458 for the trust split and limitations. ([IETF](https://www.ietf.org/rfc/rfc9458.html), [RFC Editor](https://www.rfc-editor.org/info/rfc9458))

## Motivation

Nostr today relies on persistent WebSocket connections that inherently expose a client’s IP address to the relay. This creates a metadata-leakage channel: even if messages are encrypted, relays can trivially correlate client activity over time and build behavioral profiles. While users can route connections through Tor or VPNs, these options add operational complexity and do not guarantee reliable performance.

Oblivious HTTP (OHTTP) provides a standardized way to separate transport metadata from application payloads. By routing requests through an OHTTP relay, clients conceal their network identity from the target relay while preserving end-to-end confidentiality and integrity of their Nostr events. This introduces per-request unlinkability without requiring heavy overlay networks.

The design keeps protocol churn minimal. Clients and relays continue to use NIP-01 semantics, but transport shifts from a connection-oriented WebSocket to stateless OHTTP messages. Deployment is incremental: relays can expose OHTTP endpoints alongside WebSocket interfaces, advertising support in NIP-11. Clients that do not adopt OHTTP remain fully interoperble.

## Specification / Protocol Flow

### NIP-11 advertisement and OHTTP key configuration

Relays MUST support fetching [RFC 9540 Key Configuration Fetching](https://www.rfc-editor.org/rfc/rfc9540.html#name-key-configuration-fetching) via GET request. RFC 9540 defines the gateway location as `/.well-known/ohttp-gateway`. RFC9540 also defines the key configuration encoding.

Relays that implement this NIP MUST advertise the following new fields in their NIP-11 document:

```json
{  
  "ohttp": {  
    "keys": "https://relay.example.com/.well-known/ohttp-gateway", 
    "suites": { // TODO do we just specify one crypto suite or allow for configuration?
      "kems": ["X25519-HKDF-SHA256"],  
      "kdfs": ["HKDF-SHA256"],  
      "aeads": ["ChaCha20Poly1305"]  
    },  
    "max_request_bytes": 65536,  
    "max_response_bytes": 1048576,
    "keyconfig": "0x....",
   }  
}
```

Relays and clients MUST support the following configurations:

* Key Encapsulation Mechanism (KEM): X25519-HKDF-SHA256
* Key Derivation Function (KDF): HKDF-SHA256
* Authenticated Encryption with Associated Data (AEAD): ChaCha20Poly1305

The following configurations we're chosen because most of the clients and relays are already using the same cryptoraphic primitives.

Relays CAN advertise their key configuration in the NIP-11 document to reduce round trips for the clients.

### Retrieve key configuration

Clients MUST retrieve the OHTTP [key configuration](https://www.ietf.org/rfc/rfc9458.html#section-3.1) from the relay using RFC 9540 at the `/.well-known/ohttp-gateway` endpoint. This request MUST be authenticated. If the relay advertises the configuration in its NIP-11 document, clients MAY use that as an alternative bootstrap source but doing so may reveal the clients network identity.

The key configuration defines the HPKE parameters required to establish secure and authenticated communication with the target relay. When fetching the configuration, clients SHOULD route the request through an OHTTP relay to avoid exposing their network identity.

Once the configuration is retrieved, the client encapsulates a NIP-1 message inside a Binary HTTP (BHTTP) request and then applies HPKE to form the OHTTP request.

* EVENT messages: Encapsulate as BHTTP `POST`. The body MUST contain the event as a JSON string.
* REQ messages: Encapsulate as BHTTP `GET`. The query string MUST carry the NIP-1 filter, encoded either as a hexified JSON string or URL-encoded JSON (final encoding is TBD).
* REQ messages MUST NOT include a subscription ID, as this leaks linkable metadata.

### Long Polling

TODO: 

#### Endpoints (logical, behind OHTTP)

Clients SHOULD interpert a 404 response as a relay that does not support OHTTP and SHOULD fallback to a an ordinary WebSocket connection if they wish to use this specific relay. 

* `POST /ohttp/` -- submit an OHTTP encapsulated event

HTTP bodies MUST be a binary payload per RFC 9458. The HTTP headers MUST include the "Content-Type" header with the value "message/ohttp-req".
Relays MUST decapsulate to obtain the BHTTP request using their key configuration.

HTTP responses are:

* `200 OK` with NDJSON for `req`.

* `202 Accepted` with server ACK or empty body.

Note responses are encapsulated back to the sender.

#### Endpoints (virtual, BHTTP request encapsulated in OHTTP)

After decapsulating the logical HTTP request, relays should decapsulate and parse out the BHTTP inner request. There are two valid methods. Responses should also be included in the BHTTP response. All responses back from the logica endpoint should be 200.

##### POST

The body is expected to be a single NIP-1 Event message. An empty 202 response is expected if the event can be processed. 400 if the NIP-1 message is malformed. Or for any other client side errors. 500 must be provided for internal related errors.

##### GET

The body is expected to be a single NIP-1 subscription message. A filter is expected to be provided in the query messages. TODO which key (?).

Relays MUST enforce request/response size limits advertised in NIP-11.

Responses must be encapsulated back to the sender.

#### Key rotation

// TODO

## Backwards Compatibility

* Relays continue to support NIP-01 over WebSockets. Clients that do not implement OHTTP interoperate unchanged. The OHTTP interface introduces new endpoints; no changes to NIP-01 message schemas.

## Privacy Considerations

* Per-request unlinkability: Clients MUST use a fresh HPKE encapsulation per request.

* Gift Wrap: Useful for application-layer metadata minimization, but orthogonal to OHTTP.

## Open Questions

* Streaming: Whether to standardize a chunked/streaming profile once IETF work stabilizes. ([ietf-wg-ohai.github.io](https://ietf-wg-ohai.github.io/draft-ohai-chunked-ohttp/draft-ietf-ohai-chunked-ohttp.html)). Subscriptions: Avoid long-lived state through OHTTP. Use poll+cursor now; consider chunked OHTTP later when the IETF draft matures

* Discovery: Whether to mandate RFC 9540 SVCB discovery in addition to NIP-11 fields. ([RFC Editor](https://www.rfc-editor.org/rfc/rfc9540.html))

* Anonymous credentials: Standardize a NIP for token issuance/redemption to control abuse while preserving unlinkability.

## Reference Implementation(s)

A client refrence can be found [here](https://github.com/arminsabouri/nostr/blob/ohttp-example/crates/nostr-sdk/examples/ohttp.rs).
A nostr relay reference can be found [here](https://github.com/arminsabouri/nostr-rs-relay/tree/ohttp-gateway).
