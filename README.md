# attendance-checker

> **Moved.** This checker now lives in Seeds as [`presence/`](https://github.com/Birdbrain-wtf/seeds/tree/main/presence), where attendance roots are the evidence membership is admitted against, rather than a condition on a treasury payment. This repository is kept read-only so existing links and the CS33 record still resolve. New sessions are published there.

The checker for a Kusama treasury bounty that releases money only for people who were witnessed attending a live session.

A Kusama bounty needs a curator, and the curator decides when money is released. This repository is what the curator's decision is bound to. It turns a session's attendance into a Merkle root, the root is committed on Kusama Asset Hub before any award is made against it, and an award is valid only if the checker here passes against that root. Anyone can rerun it. Nobody has to trust the curator's account of who was there.

It is part of the same work as [Seeds](https://github.com/Birdbrain-wtf/seeds), which builds identity out of witnessed participation. Seeds describes who has turned up. This repository is the part that money can be conditioned on.

## What a leaf says

```
leaf = blake2_256( 0x00 ‖ P ‖ pubkey32 ‖ blake2_256(session_id) )
node = blake2_256( 0x01 ‖ sort(a, b) )
P    = blake2_256(person_id)
```

Each leaf binds a person, the one account that person claims with, and the session. Three consequences follow, and the self-test checks each one:

- **One person, one account.** A roster that gives one person two accounts, or one account to two people, is refused before a root exists. That is the nullifier, and it is the difference between paying a person once and paying one person with five passkeys five times.
- **No replay.** A proof from one session does not verify against another session's root.
- **No substitution.** A proof does not verify under anyone else's person slot.

## Why the rosters here carry no names

The leaf never sees a name. It sees `P`, the hash of an internal person identifier. So the rosters in `rosters/` give each attendee as `person` (that hash) rather than as a name, and they rebuild the anchored root exactly.

This is pseudonymity, not anonymity, and it should be read that way. The person identifiers are short, so someone who guesses one can confirm it against its hash, and the accounts are public on chain regardless. What it does do is keep names and login handles out of a public repository while leaving every claim about the root checkable.

## Who gets a leaf

Only people who were in the room **and** signed in with a passkey they hold themselves. Everyone else who was present is written into the roster's `excluded` list with the reason, so a roster can be checked against a head count rather than trusted not to have dropped anyone. The reasons are:

- joined on a guest link with no passkey, so there is no account to put in a leaf
- holds a membership only on an account an administrator key controls, so paying against it would be paying ourselves
- signed in, but the login is not yet matched to a person
- a second login by someone who already has a leaf, so they are not counted twice

## Check it yourself

You need [Bun](https://bun.sh).

```
bun install
bun run attendance-root.ts self-test          # the properties above, and a few more
bun run attendance-root.ts build rosters/CS33.json
bun run verify-chain.ts rosters/CS33.json     # read the anchoring remark off Kusama Asset Hub and compare
```

`build` prints the root and writes a proof file beside the roster, one Merkle path per leaf. To check a single leaf against a root:

```
bun run attendance-root.ts verify <root> <session> <person> <account> <proof,proof,…>
```

`<person>` is the `person` value from the roster.

`verify-chain.ts` is read-only. It fetches the block named in the roster, finds the anchoring extrinsic, decodes its `System.remark` and compares it with `birdbrain/attendance/v2:<session>:<root>` rebuilt from the roster. A match means the published roster is the one that was committed, at that block, before anything was paid against it.

## Anchored sessions

| session | date | leaves | excluded | Asset Hub block |
| --- | --- | --- | --- | --- |
| CS33 | 2026-09-14 | 4 | 5 (one a second login of a leaf) | #21,513,860 |

A session is published here only once its root is on chain. A root published first and anchored later proves nothing about ordering.

## What this does not claim

The mapping from a person to their account comes from a session's own sign-in log, co-witnessed by everyone in the room and published afterwards. It can be challenged by anyone who was there, which is a real and cheap sybil cost. It is not a cryptographic proof of personhood. The person slot is built so that a proper personhood primitive, such as Polkadot's `pallet-people`, can take it over without changing the leaf format or the award condition.

## Licence

Apache-2.0.
