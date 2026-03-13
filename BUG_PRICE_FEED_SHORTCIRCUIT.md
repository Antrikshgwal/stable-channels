# Bug: A single blocked price feed stalls the user app indefinitely

## TL;DR

If any one of the five default BTC/USD price feeds is unreachable from the
user's jurisdiction or network (geo-blocked, firewalled, rate-limited, or
DNS-failed), `fetch_prices` returns `Err` on the *first* failure via the `?`
operator instead of continuing to the next feed. With no cached price,
`update_balances()` never clears `is_syncing`, and the app is stuck on the
**"Syncing… Fetching latest prices"** screen forever — no timeout, no skip
button, no error surfaced to the user.

The function looks like it was designed to tolerate per-feed failures (it
declares a `'feeds:` label and uses `continue 'feeds` for JSON-shape errors),
but the network-level failure path short-circuits before that logic is
reachable.

**This bug currently breaks the app 100% of the time for users in India**,
because Bitstamp is geo-blocked there (regulatory block — Bitstamp doesn't
serve Indian IPs) and Bitstamp is the first entry in the default feed list.
Any other jurisdiction that blocks the first feed would see the same failure
mode; this isn't a transient or edge-case network issue.

---

## Symptoms

- Fresh start of `stable-channels user` on a restrictive network (campus
  wifi, captive portals, some corporate/VPN egress, or regions that block
  specific financial APIs) hangs on the syncing screen.
- `~/.local/share/StableChannels/user/audit_log.txt` shows repeating
  `STABILITY_SKIP` entries with `reason: "no valid price available"` and
  **zero** `PRICE_FETCH` events for the session.
- Switching to an unrestricted network (e.g. mobile hotspot) unblocks the app
  within ~30 seconds — because then *all* feeds are reachable.

### Confirmed repro (2026-04-19)

On an Indian residential/mobile connection,
`curl https://www.bitstamp.net/api/v2/ticker/btcusd/` fails with
`(35) Recv failure: Connection reset by peer` — Bitstamp is geo-blocked for
Indian IPs as part of their regulatory compliance, so the TLS handshake is
terminated before a response is sent. From the **same** machine, the other
four feeds (CoinGecko, Kraken, Coinbase, Blockchain.com) all returned HTTP 200.
The app still stalled because Bitstamp is the first entry in the default feed
list and its failure short-circuits the whole fetch before the other feeds
are tried.

---

## Root cause

### The short-circuit

[src/price_feeds.rs:97-110](src/price_feeds.rs#L97-L110):

```rust
'feeds: for price_feed in price_feeds {
    ...
    let response = retry(
        Fixed::from_millis(PRICE_FETCH_RETRY_DELAY_MS).take(PRICE_FETCH_MAX_RETRIES),
        || match agent.get(&url).call() {
            Ok(resp) if (200..300).contains(&resp.status()) => Ok(resp),
            Ok(resp) => Err(format!("Received status code: {}", resp.status())),
            Err(e) => Err(e.to_string()),
        },
    )
    .map_err(|e| -> Box<dyn Error> { Box::new(std::io::Error::other(e.to_string())) })?;
    //                                                                               ^^^
    //                                          returns Err from fetch_prices on the
    //                                          FIRST feed that exhausts its retries
```

When `retry` exhausts its `PRICE_FETCH_MAX_RETRIES` attempts against a blocked
host, the `?` propagates the error out of `fetch_prices` immediately. The
loop never advances to the next feed. This is inconsistent with the rest of
the function, which uses `continue 'feeds` for JSON path / shape errors and
only returns `Err` from the bottom if `prices.is_empty()`
([src/price_feeds.rs:153-155](src/price_feeds.rs#L153-L155)).

### How this cascades into a stuck UI

1. **Background thread** at [src/user.rs:613-619](src/user.rs#L613-L619) calls
   `get_latest_price`, gets `Err`, falls back to `get_cached_price()` which
   itself calls `get_latest_price` on a cold cache and also returns the cached
   value `0.0`.
2. **`update_balances`** at [src/user.rs:1489-1523](src/user.rs#L1489-L1523)
   reads `get_cached_price_no_fetch()`. The clear-syncing branch at
   [src/user.rs:1511](src/user.rs#L1511) is gated on `current_price > 0.0`.
3. **`update` loop** at [src/user.rs:7144-7147](src/user.rs#L7144-L7147)
   renders `show_syncing_screen` as long as `is_syncing` is true.
4. No timeout, no skip button, no user-visible error. The spinner runs
   forever.

### Why the first feed dominates

Default feed order is defined at
[src/constants.rs:147-175](src/constants.rs#L147-L175):

1. Bitstamp ← geo-blocked in India
2. CoinGecko
3. Kraken
4. Coinbase
5. Blockchain.com

Because the short-circuit fires on the first failure, whichever feed is listed
first and inaccessible wins. Rotating the list (e.g. moving Bitstamp last)
would make the app work in India today, but it would just shift the problem
to whichever jurisdiction or network blocks the new first feed — not a fix.

---

## Fix

Turn the retry error into a logged `continue 'feeds` so one unreachable host
doesn't kill the whole fetch. The existing `prices.is_empty()` check at
[src/price_feeds.rs:153](src/price_feeds.rs#L153) already handles the
"nothing worked" case.

```rust
// src/price_feeds.rs, replace lines 97-110

let response = match retry(
    Fixed::from_millis(PRICE_FETCH_RETRY_DELAY_MS).take(PRICE_FETCH_MAX_RETRIES),
    || match agent.get(&url).call() {
        Ok(resp) if (200..300).contains(&resp.status()) => Ok(resp),
        Ok(resp) => Err(format!("Received status code: {}", resp.status())),
        Err(e) => Err(e.to_string()),
    },
) {
    Ok(resp) => resp,
    Err(e) => {
        eprintln!("Feed {} unreachable: {}", price_feed.name, e);
        continue 'feeds;
    }
};
```

Also worth tightening while we're here:

- The `into_json()?` one line below
  ([src/price_feeds.rs:112](src/price_feeds.rs#L112)) has the same problem —
  a feed returning unparseable bytes (e.g. a Cloudflare HTML challenge page
  served with `Content-Type: application/json`) will blow up the whole fetch.
  Replace with a `match` that `continue 'feeds` on error.
- `ureq::Agent::new()` at [src/user.rs:615](src/user.rs#L615) uses library
  defaults for timeouts. A hung TCP connection can block the background loop
  for minutes per feed per retry. Use the same
  `AgentBuilder::timeout_connect(10s).timeout(15s)` pattern that
  `get_cached_price` already uses at
  [src/price_feeds.rs:59-62](src/price_feeds.rs#L59-L62).

---

## UX follow-up (separate PR)

Even with the fetch fix, a user with *all* feeds blocked still hangs. The
sync screen should not be an unconditional gate:

- Timeout (e.g. 30s) → show a "Couldn't reach price feeds — check network"
  message with a Retry button and a "Continue anyway" escape hatch.
- Let the user into the main screen with a prominent "Price unavailable"
  banner instead of blocking it entirely. Most of the UI (balances in sats,
  on-chain deposit address, channel status) doesn't actually require a USD
  price.

File: [src/user.rs:3236-3262](src/user.rs#L3236-L3262) (`show_syncing_screen`)
and the gate at [src/user.rs:7144](src/user.rs#L7144).

---

## How to verify the fix

1. On a machine where at least one but not all feeds are blocked
   (reproducible by dropping one host via `/etc/hosts` or an egress rule),
   confirm the app reaches the main screen within ~30s and the audit log
   shows a successful `PRICE_FETCH` entry.
2. Block *all* feeds and confirm `fetch_prices` returns the existing
   `"No valid prices fetched."` error (unchanged behavior).
3. Unit test in `src/price_feeds.rs` (new): feed list containing one URL
   pointing at `127.0.0.1:1` (guaranteed-refused) plus one pointing at a
   mock HTTP server returning valid JSON; assert that `fetch_prices`
   returns `Ok` with one entry.
