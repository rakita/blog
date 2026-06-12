---
name: state-expire-blog
description: >
  Use when the user is researching or drafting blog posts in this repo on
  Ethereum's state model — state expiry, state layering / tiered storage,
  state growth / bloat, state commitments, witnesses, statelessness, EIP-4444,
  Verkle/Binary tries, MPT, revival mechanics, or gas pricing of state access.
  Recognize phrases like "state expire", "state expiry", "state bloat", "state
  growth", "tiered state", "freezing state", "SEF", "stateless clients",
  "appendable state commitment", "witness", "Verkle", "MPT", or asking for a
  new post under content/blog/ on these themes. Scope: Ethereum-first;
  other-chain comparisons only if the user explicitly asks.
---

# state-expire-blog

This is a [Zola](https://www.getzola.org/) static-site blog. The author writes opinionated core-Ethereum research notes. Recent posts are an interconnected series on Ethereum state-layer design:

- `006_appendable_state_commitment` — alternative SC that is append-only, motivation behind moving away from random-insert tries.
- `008_state_expire_freezer` — SEF proposal: hot / warm / freezing tiers + load transactions.
- `009_state_expire_sucks` — survey post: why every state-expire proposal has bad tradeoffs, leading into SEF.

When the user mentions "the SEF post", "the freezer idea", "the appendable SC post", they mean one of these.

## Creating a new post

Posts live under `content/blog/NNN_short_slug/index.md`, where `NNN` is the next zero-padded number after the highest existing one (currently `009`). Co-located images go in the same directory and are referenced with relative paths (e.g. `![](./diagram.jpeg)`).

Use this frontmatter (copy from `008_state_expire_freezer/index.md`):

```toml
+++
title = "..."
description = "..."
date = YYYY-MM-DDT12:00:00+00:00
updated = YYYY-MM-DDT12:00:00+00:00
draft = false
template = "blog/page.html"

[taxonomies]
authors = ["draganrakita"]
+++
```

Keep `draft = true` while iterating; flip to `false` only when the user says to publish.

## Voice and structure

Match the existing series. Concrete cues from `008` and `009`:

- Opens with the problem in 2-3 plain sentences ("State bloat is Ethereum's long-term scaling problem…"), not with a definition dump.
- Short H2 sections, bulleted breakdowns for tiered models and numbered lists for protocol flows.
- Concrete time windows / gas numbers when discussing proposals (e.g. "Hot 0-365 days", "~3k extra gas").
- Honest closing section that names the downside — `008` ends with an "Elephant in the room" admitting "State expire sucks". Do not sand off tradeoffs.
- Cross-references to Ethereum primitives (MPT vs Verkle, EIP-4444, stateless clients, access lists, gas reforms) are expected. Other-chain comparisons (Solana rent, Sui, etc.) only when the user explicitly asks for them — default scope is Ethereum.
- Lowercase casual asides and typos are part of the voice (see `009`). Don't aggressively over-polish unless asked — ask the user before doing a copy-edit pass on existing posts.

## Ethereum state-model reference

Anchor every claim in the Yellow Paper / Execution-Spec mental model:

- **State** = mapping `address → account`, where `account = (nonce, balance, storageRoot, codeHash)`. Storage is a per-account `slot → word` mapping.
- **State commitment** today = two-layer Merkle Patricia Trie (account trie + per-account storage tries). Verkle transition collapses this to a single keyed trie; Binary trie is the post-Verkle direction being discussed.
- **Pain points to keep straight**: random-insertion cost into the MPT (~80% of block-execution time per `006`), unbounded growth of dormant accounts/slots, witness size for stateless clients, and revival/proof UX for any expiry scheme.
- **EIPs** worth cross-referencing: EIP-4444 (history expiry), EIP-4762 / Verkle gas reform, EIP-2935 (historical block hashes in state), EIP-7702 only when it touches account model, and any active state-expiry / Verkle-transition EIPs. **Verify EIP numbers and status on eips.ethereum.org before citing** — renumberings and withdrawals are common.
- **Primary venues** for state-expiry debate: ethresear.ch, EthMagicians, ACD call notes, and Vitalik's notes pages. Link them when making a non-obvious claim.
- The author works on **revm** (`/Users/draganrakita/workspace/revm`). When a post touches EVM execution mechanics, gas costs, or access-list semantics, grep revm for the relevant opcode / constant rather than guessing.

Use `WebFetch` / `WebSearch` for the above; do not invent EIP numbers or quote specs from memory.

## Cross-linking

The state-layer posts form a series — when adding a new one, propose inbound links from older posts in the series (and vice-versa) so a reader can follow the arc. Ask before editing already-published posts.

## Build & preview

```bash
zola serve          # local dev server with live reload
zola build          # produces ./public
zola check          # link + content validation, run before publishing
```

If `zola` is not installed, point the user at `brew install zola` rather than trying to work around it.

## What NOT to do

- Don't create `README.md`, planning docs, or analysis files in the repo — the blog *is* the artifact.
- Don't restructure the existing post directory naming scheme (`NNN_snake_case`).
- Don't auto-publish (`draft = false`) without explicit confirmation.
- Don't quietly rewrite the author's voice into neutral tech-blog prose. Preserve first-person opinion and the "this sucks because…" honesty.
