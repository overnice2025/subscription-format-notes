# Proxy Subscription Format Notes

Why the same subscription URL works in one client and fails in another — the three formats, the metadata header, and the conversion traps that silently break nodes.

This is the technical layer under most "import failed" and "invalid yaml" errors. If your client rejects a subscription that works in a browser, the answer is almost always in this document.

---

## The three formats

A subscription URL returns **one** of these. They are not interchangeable, and most providers serve a different one per path or query parameter.

### 1. Base64 node list (v2ray-style)

The body is a base64-encoded block of one-per-line share links:

```
dm1lc3M6Ly9leUoySWpvaU1UQXd...   (base64 of the block below)
```

Decoded:

```
vmess://eyJ2IjoiMiIsInBzIjoi...
vless://uuid@host:443?type=ws&security=tls#name
trojan://password@host:443?sni=example.com#name
ss://base64method:password@host:port#name
```

**Consumed by:** v2rayN, v2rayNG, and most Xray-family clients.

Two traps specific to this format:

- `vmess://` is **not** a standard URI. The part after `://` is base64-encoded JSON (v2rayN's own convention). A strict URI parser rejects it.
- The whole block is base64 with possible line breaks. Clients that strip newlines first succeed; clients that don't fail with a decode error.

### 2. Clash / Mihomo YAML

```yaml
proxies:
  - name: "node-1"
    type: vmess
    server: example.com
    port: 443
    uuid: xxxx
    alterId: 0
    cipher: auto
    tls: true
proxy-groups:
  - name: "PROXY"
    type: select
    proxies: ["node-1"]
rules:
  - DOMAIN-SUFFIX,example.com,PROXY
  - MATCH,PROXY
```

**Consumed by:** Clash Verge Rev, FlClash, Stash (compatible), and anything Mihomo-based.

A Clash config is an **entire configuration**, not a node list. This is the key asymmetry: a base64 node list cannot become a Clash config without a converter adding `proxy-groups` and `rules`.

### 3. sing-box JSON

```json
{
  "outbounds": [
    { "type": "vmess", "tag": "node-1", "server": "example.com", "server_port": 443, "uuid": "xxxx" }
  ],
  "route": { "rules": [], "final": "node-1" }
}
```

**Consumed by:** sing-box, Hiddify, Karing.

sing-box has moved fields between versions — most notably around `tunnel-per-resolver` and rule-set handling. A config written for a newer core fails on an older one with an `unknown field` error, and vice versa.

---

## The metadata header

Most clients read subscription quota from a response header, not the body:

```
subscription-userinfo: upload=123456789; download=987654321; total=107374182400; expire=1767225600
```

| Field | Meaning | Unit |
|---|---|---|
| `upload` | bytes sent | bytes |
| `download` | bytes received | bytes |
| `total` | quota | bytes |
| `expire` | expiry | **Unix timestamp**, seconds |

Practical notes:

- `expire=0` usually means no expiry, but some panels use `0` to mean "unknown" — they're not distinguishable from the header alone.
- Clients display this in GB (÷ 1024³). A provider advertising "100GB" that shows 93.1GB in the client is using decimal GB in marketing and binary GB in the header. Both are common.
- If the header is missing, clients show "no quota info" and fall back to whatever you configure manually. Absence of the header does **not** mean the subscription is broken.

---

## Why conversion breaks nodes

People run subscriptions through a converter (subconverter or similar) to switch formats. This is where silent data loss happens.

**What a converter typically does:**

| Action | Consequence |
|---|---|
| Renames nodes | Your rules and proxy-group references point at names that no longer exist |
| Drops unsupported fields | `alterId`, `flow`, `packet-encoding`, plugin options disappear silently |
| Reorders outbounds | The "first node" or default selection changes |
| Rewrites the transport | `ws` path/host may be normalized; non-standard values get dropped |
| Re-encodes base64 | Padding and line-break differences break clients with strict decoders |

**What it cannot do:** infer information the source format didn't carry. If the base64 node list has no `rules`, the converter invents a default ruleset — which is why converted configs often route traffic differently from the original.

---

## Diagnostic sequence

When a subscription works in one client and fails in another:

1. **Fetch the URL with `curl -i` and look at the first bytes.** Is it `proxies:`, `{`, a base64 string, or `<`? If `<`, you have an HTML page (login wall, 404, or a challenge) with a 200 status — that is the fault, and nothing downstream will fix it.
2. **Check `Content-Encoding`.** A gzip body handed to a client expecting plain text produces a parse error that looks like a format problem.
3. **Check `Content-Type`.** `text/html` where you expected `text/yaml` or `application/json` confirms step 1.
4. **Compare byte size against expectation.** A subscription that returns 200 bytes is an error page, not a config.
5. **Decode the base64 manually.** If it doesn't decode, the client isn't the problem.
6. **Only then** consider the client's parser.

---

## Redaction before you post

Subscription URLs contain a token that grants access to your account's traffic. Before pasting into an issue:

- Strip the token from the URL path/query
- Remove `uuid`, `password`, and node server addresses
- Replace subscription hostnames
- Remove any `subscription-userinfo` values that map to your account

A leaked subscription token lets someone else consume your quota, and in many panels it's the only credential — there is no second factor.

---

---

If this explained an import failure you'd been debugging, star the repo.

## Related

- [Error code reference](https://github.com/overnice2025/clash-error-codes) — the exact errors these format mismatches produce
- [Proxy client comparison matrix](https://github.com/overnice2025/proxy-client-matrix) — which client consumes which format
- [Subscription basics (ZH)](https://jiedianshouce.com/articles/subscription-basics/) — full walkthrough from purchase to import to expiry
- [Subscription conversion safety (ZH)](https://jiedianshouce.com/articles/subscription-conversion-safety/)

## Contributing

If a converter behavior listed here has changed, open an issue with the converter version and a minimal input/output pair.

## License

MIT — see [LICENSE](LICENSE).
