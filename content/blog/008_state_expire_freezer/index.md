+++
title = "LED - Loadable Expired Data."
description = "LED is a practical state expire protocol."
date = 2025-01-15T12:00:00+00:00
updated = 2025-11-15T12:00:00+00:00
draft = true
template = "blog/page.html"

[taxonomies]
authors = ["draganrakita"]
+++

# SEF - State Expire Freezer

State bloat is every Blockchain long-term scaling problem. Every new account, storage slot, or contract sticks around forever, so disk requirements relentlessly climb without any bounds. Some blockchains just ignore it—bigger disks and better hardware buy a few more years—but that is postponing the problem, not solving it.

Introducing tiers storage should be first and most important step to resolve this.

State expire is not one idea, it is a stack of them. Each idea below solves one specific piece of the puzzle: where frozen state lives, who fetches it back, what counts as keeping state alive, how new accounts avoid paying for old ones. None of them is enough on its own. I will first walk through all the major ideas one by one, and at the end combine them into the **LED (Loadable Expired Data)** protocol.

## First idea: state leaves the client's disk and clients fetch it back

The starting point is simple: old state has to live somewhere other than every client's local disk. If it stays on disk forever, we haven't solved anything - we've just renamed the problem. So frozen state moves off-disk, and getting it back can't be instant. Loading it takes N blocks (a few seconds), because the data has to be fetched from wherever it was parked and verified before it can be touched again.

The non-obvious part is *who* does that fetching. **Clients are responsible for fetching frozen state and verifying it**. A node that wants to stay in sync must be able to retrieve the requested data and prove it against the state root. *How* the data gets fetched is deliberately not defined by the protocol—it can be a p2p network, a torrent, or a plain AWS server. It doesn't matter where the bytes come from, because the client verifies them against the hash anyway. Trust lives in the verification, not in the source.

This is a hard reversal from the older line of thinking, where the *user* was supposed to carry the data: hold your own witnesses, ship a proof with every transaction, keep your accounts alive yourself. That does not work at all. It pushes proof-management UX onto every wallet and every user, and quietly assumes everyone runs archival infrastructure, or has access to it. Making users custodians of their own state is bad ux so the responsibility goes back to the clients, where it belongs.

## Second idea: state is split into epochs, old epochs must be revived

Freezing isn't a single line between "hot" and "cold"—it's a sequence of **epochs**. An epoch is a window of time during which some slice of state lives in the active trie. State that gets touched inside an epoch belongs to that epoch; once an epoch closes, the state it held is no longer part of the current trie. It still exists—it's just parked in an older epoch, out of the working set.

The current trie only ever holds the current epoch. That's what keeps the live state bounded: as time advances and new epochs open, old epochs roll off the active trie instead of accumulating forever. The total state still grows, but the part every node must keep hot is capped to a rolling window rather than the whole history of the chain.

The consequence is **revival**. If a transaction needs to touch state that lives in an older epoch, that state has to be brought back—revived—into the current trie before it can be used. Reviving means fetching the data from wherever its epoch was archived, proving it against the commitment for that epoch, and re-inserting it into the current epoch's trie. After revival the state belongs to the current epoch again, with its clock reset, and it stays hot until it once more rolls off.

This is what makes epochs more than just timestamps: each epoch is its own committed unit of state. A node doesn't need to carry every past epoch on disk, only the current one plus the ability to prove and pull from the rest. Old data isn't deleted and it isn't lost—it's epoch-shifted, and revival is the explicit, priced act of pulling it forward into the present.

## Third idea: only writes renew state, reads leave the trie untouched

If touching state advances its epoch, the obvious question is *what counts as touching it*. The naive answer—any access, read or write—is wrong, and it's wrong in a way that quietly breaks the whole scheme. A read is the cheapest, most common operation there is. If every `SLOAD` or balance check re-stamped state into the current epoch, then the act of *reading* the working set would constantly drag everything back into the hot trie, the live state would never shrink, and we'd be back to unbounded growth with extra bookkeeping on top.

So the rule is sharp: **state's epoch is renewed only on write, never on read.** An account or storage slot carries a "last written" marker, and only a mutation—a balance change, a storage update, a nonce bump—resets that clock and pulls the item into the current epoch. Reads observe state without moving it. You can read a value that lives in the current epoch as many times as you like and it stays exactly where it is; reading does not keep cold data warm.

This is the same principle [EIP-8188](https://eips.ethereum.org/EIPS/eip-8188) ("State Tiering by Write Age") formalizes: it tracks a `last_written_period` per account and slot, prices writes by how long the state has sat unmutated, and explicitly leaves reads unpriced by renewal age. Tiering by *write* age rather than access age is what gives the freezer a clean, predictable boundary—the hot set is precisely the state that has been *mutated* recently, not the much larger set that has merely been *looked at*. It also matches how clients actually want to optimize: the data worth keeping on fast storage is the data that keeps getting rewritten, and that's exactly what the write clock identifies.

Note this is about *renewal*, not *revival*. Reviving frozen state—pulling an old epoch's data forward via a load transaction—still places that data into the current epoch, because revival is itself a write into the live trie. What the read/write distinction rules out is the silent, automatic renewal that would happen if ordinary reads counted. Cold data you only read stays cold; only when you write does its clock reset.

## Fourth idea: accounts carry an epoch number, so new accounts skip revival

Epochs raise an awkward question: what happens when a transaction creates an account—or touches a storage slot—at an address that *might* already hold frozen state in an older epoch? The cautious answer is to revive first: fetch the old epoch, prove there's nothing there (or pull forward whatever is), and only then write. But that forces a load round-trip on the most common operation there is—writing to fresh state—just to rule out a collision with cold data nobody asked for.

The fix is to give every account an **epoch number**, set to the epoch in which the account was created. When a transaction creates an account in the current epoch, it stamps the current epoch as the account's epoch number and writes directly into the live trie—**no fetch, no revival of any older epoch required.** The new account is unambiguously a current-epoch entity; any state parked at the same address under an earlier epoch belongs to a past life of that address and does not interfere.

This is what makes epochs composable without constant revival. The epoch number lets a client decide, locally and instantly, whether the state it's looking at is current or belongs to a past life of that address. Creating in the present never has to reach into the past—revival is reserved for the cases that genuinely need old data, not paid as a tax on every new account.

One clarification before moving on, because the two numbers are easy to conflate: the account's **epoch number** is attached to the account *address* and records when the account was *created*; the state's **epoch** records when it was last *written to*. Same-looking integers, different jobs. The account's epoch number answers "is this a fresh account or a past life at this address?"; the state's epoch answers "how stale is this state, and which tier does it belong to?". Don't let the symmetry fool you—an account created long ago (old epoch number) can sit in the current epoch if it keeps getting written, and a freshly created account can roll into an older epoch if it goes quiet.

`CREATE` and `CREATE2` follow the same rule: a contract deployed by either opcode gets stamped with the latest epoch as its epoch number. That means everything the deployment touches—code, nonce, every storage slot the constructor writes—lands in the current trie, so all of the new contract's state is immediately accessible. The deployment never has to look into older epochs; whatever may sit at that address from a past epoch stays parked there and the fresh contract starts clean in the present.

## Fifth idea: the freezer is custom storage outside the protocol, with slow reads and small, steady writes

Everything so far has described *when* state moves out of the hot trie. The fifth idea is about *where* it goes—and the answer is deliberately unspecified by the protocol. The **freezer is custom storage that lives outside consensus**. The protocol commits to what frozen state *is* (its values, provable against an epoch's commitment), but never dictates how an operator stores it. HDDs, a remote archive, IPFS-like networks, a managed RPC—any of them is fine, as long as the operator can present correct, provable data when a load asks for it.

What makes this practical is the shape of the I/O. Reads from the freezer are **slow on purpose**: a load is delayed a few blocks, because the data has to be fetched and verified before it can re-enter the hot trie. That latency is the whole reason the freezer can sit on cheap, slow storage—nothing on the hot path waits on it. Writes into the freezer, by contrast, are **constant over time and small in number**. Every block the hot trie prunes a handful of storage slots and accounts—the ones whose epoch just rolled off—and that same handful is what gets appended to the freezer. The freezer's write rate is therefore bounded and steady, not bursty: a constant trickle of items per block, regardless of how big the total frozen set has grown. This is what keeps the freezer cheap to operate and friendly to append-only storage backends (the same property the [appendable state commitment](../006_appendable_state_commitment/) post leans on).


## Sixth idea: expiring itself happens in epochs

Expiry is not one cliff, it's a pipeline. Epoch N is the active one—the hot trie where all writes land. Epoch N-1 is being deposited: every block a chunk of it is read and stored into long-term storage. Epoch N-2 is being expired: some slots per block get deleted from the client's disk, which is safe because they were already deposited an epoch earlier. Every epoch walks the same path: active, deposited, expired.

## Seventh idea: touching a previous-epoch account halts execution

Accessing storage of an account created in a previous epoch can't just proceed—the data may no longer be on disk—so execution halts on the `SSTORE`/`CALL`. To make that decidable, **address expansion** is needed: the address carries the account's epoch number, so a client knows from the address alone what epoch an account is from, without reaching into old state.


# Combining all ideas you create the LED protocol.

Let's work out how all of this would look, epoch by epoch.

**Epoch 0** is the starting point: the entire current state of the chain. Nothing is frozen, nothing is deposited, the trie looks exactly like it does today. Every existing account gets epoch number 0.

**Epoch 1 opens.** Epoch 0 becomes the N-1 epoch, and the deposit pipeline starts: every block a chunk of epoch-0 state is read and stored into long-term storage. Nothing is deleted yet—all of epoch 0 is still on disk. Meanwhile, normal activity continues:

1. Reading an epoch-0 slot works exactly as before. Reads renew nothing, so the slot stays in epoch 0 and keeps marching toward the freezer.
2. Writing to an epoch-0 account also works—the data is still on disk—and the write renews it: that account's state moves into epoch 1 and drops out of the set being deposited. An account from epoch 0 can still be changed.
3. A `CREATE`/`CREATE2` deploys a contract stamped with epoch number 1. All of its storage lives in the current trie, fully accessible, no revival anywhere.
4. A transfer to a fresh address uses the expanded address carrying epoch number 1, so every client knows instantly that this account has no past life to check against.

**Epoch 2 opens.** Now epoch 0 starts actually expiring: some slots per block are deleted from disk, which is safe because they were deposited during epoch 1. Epoch 1 takes over the deposit slot and starts trickling into long-term storage. Touching expired epoch-0 state is no longer a plain access—execution halts on the `SSTORE`/`CALL`, because the data may already be gone from disk.

This is where **revival** comes in. Reviving needs either a new transaction type—a revive tx that requests the old data, waits the N-block fetch delay, and lands it in the hot trie—or a write of the state that sat two epochs back: the data is fetched from the freezer, proven against epoch 0's commitment, and written into the latest epoch with a fresh clock. Either way, revival is an explicit, priced act—old state never sneaks back in on its own.

**Epoch 3 opens.** Epoch 1's data is now expired as well, having been deposited during epoch 2. But note that expired does not mean lost: the first epoch's data is not destroyed—it sits in the freezer, provable against its epoch commitment. It's just that the only way to access it anymore is a data revive tx. The hot trie stays bounded to the rolling window, and everything older is one revive away.