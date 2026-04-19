# NIP-XX: Workspaces

## Abstract

This NIP defines a set of Nostr events for representing **workspaces** — bounded organizational groups with membership lists, extensible capability tags, channels, and end-to-end encrypted messaging via MLS. Workspaces provide a coordination layer on top of Nostr that enables team communication with cryptographic membership verification, tag-based permissions, and portable organizational structure.

Unlike NIP-28 public channels, workspaces enforce membership, support administrative capabilities through extensible tags, and scope all communication to verified members. A workspace's entire structure — its definition, membership state, channels, and permissions — is expressed as Nostr events that any compliant client or relay can index, verify, and render.

Each workspace is backed by a **norg key** (organizational key) — a separate keypair whose sole purpose is to sign membership attestations. This allows the workspace owner to delegate org-level signing without exposing their personal key, and gives members a lightweight way to prove affiliation without replaying the entire membership delta chain.

## Motivation

Nostr provides excellent infrastructure for public, unbounded communication through NIP-28 channels and private communication through NIP-04/NIP-44. However, there is no standard for **bounded organizational groups**: teams, companies, and organizations that need:

- A cryptographically verifiable membership list
- Extensible capability tags instead of hardcoded roles
- Scoped channels that only members can access
- End-to-end encrypted group messaging with forward secrecy (MLS)
- NIP-05 identity binding within an organizational domain
- Portability — the ability to move between clients and relays without losing organizational structure
- Relay-enforced privacy for membership rosters and channel content
- Lightweight affiliation verification via norg keys — members can prove org membership without replaying the full delta chain

This NIP addresses that gap by standardizing workspace events so that any client can discover, join, and participate in workspaces regardless of which relay or client software they use.

## Events

### Kind 33100 — Workspace Definition

A **parameterized replaceable** event that defines a workspace. Only the workspace owner (the key that created it) can update this event.

```json
{
  "kind": 33100,
  "content": "{\"name\":\"Acme Corp\",\"about\":\"Making widgets since 2024\",\"domain\":\"acme.com\",\"logo_url\":\"/blossom/abc...\",\"nip05_domain\":\"acme.com\",\"norg_pubkey\":\"<norg-hex-pubkey>\"}",
  "tags": [
    ["d", "<workspace-id>"],
    ["name", "Acme Corp"],
    ["domain", "acme.com"],
    ["r", "wss://relay1.example.com"],
    ["r", "wss://relay2.example.com"]
  ]
}
```

**Content JSON fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable workspace name |
| `about` | No | Short description |
| `domain` | No | Organizational domain for NIP-05 identities |
| `logo_url` | No | Blossom URL or image URL for the workspace logo |
| `nip05_domain` | No | Domain for NIP-05 verification of members |
| `norg_pubkey` | No | Hex-encoded public key of the workspace's norg key. If present, this key is used to verify kind 33110 affiliation attestations. If absent, the workspace owner's pubkey serves as the norg key. |

The `d` tag value is the **workspace identifier** — a unique string chosen at creation time. It is recommended to use a 64-character hex string derived from the workspace owner's pubkey or a random nonce to ensure global uniqueness.

The **workspace owner** is the pubkey that created the kind 33100 event. The owner has irrevocable administrative control unless they delegate it via capability tags in membership events.

The **norg key** is a separate keypair that serves as the organizational signing key for the workspace. Its public key is stored in the `norg_pubkey` field of kind 33100 content. The norg key is used exclusively to sign kind 33110 affiliation attestations and kind 33111 norg key rotation events. By separating the norg key from the owner's personal key:

- The owner can delegate org-level signing authority without exposing their personal key.
- The norg key can be rotated (via kind 33111) independently of the owner's key.
- Members can verify affiliation by checking a single norg signature rather than replaying the entire membership delta chain.

If `norg_pubkey` is absent from kind 33100, the workspace owner's pubkey serves as the norg key.

### Kind 33101 — Workspace Membership Snapshot

A **parameterized replaceable** event that declares the complete membership state of a workspace at a point in time. Only the latest snapshot for a given `d` tag is authoritative. Published by the workspace owner or a member with the `admin` capability in the previous accepted state.

```json
{
  "kind": 33101,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["p", "<owner-pubkey>", "<relay-url>", "admin,invite,create-channel"],
    ["p", "<admin-pubkey-2>", "<relay-url>", "admin,invite"],
    ["p", "<member-pubkey>", "<relay-url>", "write,read"],
    ["p", "<member-pubkey-2>", "<relay-url>", "read"],
    ["prev", "<previous-snapshot-or-delta-event-id>"]
  ]
}
```

**Capability tags:**

Capability tags are comma-separated strings in the fourth field of `p` tags. They replace static role values with an extensible system. The following **standard capability tags** are defined:

| Tag | Description |
|-----|-------------|
| `admin` | Full administrative control: manage members, capabilities, channels, settings |
| `invite` | Can send workspace invitations |
| `create-channel` | Can create new channels |
| `moderate` | Can remove messages and manage channel content |
| `write` | Can post messages in workspace channels |
| `read` | Can view workspace content and channels |

Custom capability tags MAY be used. Their meaning is application-defined and is not enforced by this NIP.

**Verification rules:**

1. The event MUST be signed by the workspace owner (creator of kind 33100), OR by a pubkey that holds the `admin` capability in the state derived from the previous accepted snapshot and delta chain.
2. A member who is removed simply does not appear in the snapshot. No separate "kick" event is needed.
3. The `prev` tag references the event ID of the previous snapshot or the most recent delta that this snapshot supersedes. If there is no previous state (first snapshot), the `prev` tag is omitted.
4. Clients SHOULD query the relay for the most recent kind 33101 event for a given `d` tag to determine the current membership snapshot.

### Kind 33102 — Workspace Membership Delta

A **regular** event that applies incremental changes to the membership state. Deltas add or remove members and capabilities without republishing the entire roster. Each delta references the event (snapshot or delta) it builds upon.

```json
{
  "kind": 33102,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["e", "<previous-event-id>", "<relay-url>", "previous"],
    ["p", "<added-pubkey>", "<relay-url>", "+invite,write,read"],
    ["p", "<admin-pubkey>", "<relay-url>", "+create-channel"],
    ["p", "<removed-pubkey>", "<relay-url>", "-"],
    ["p", "<demoted-pubkey>", "<relay-url>", "-admin"]
  ]
}
```

**Delta `p` tag format:**

The fourth field of each `p` tag uses a prefix notation:

| Format | Meaning |
|--------|---------|
| `+cap1,cap2` | Add these capabilities to the member |
| `-cap1,cap2` | Remove these capabilities from the member |
| `-` | Remove the member from the workspace entirely |

The `e` tag with marker `"previous"` references the event (snapshot or delta) that this delta builds upon. This forms a chain of state changes.

**Verification rules:**

1. The event MUST be signed by the workspace owner, OR by a pubkey that held the `admin` capability in the state derived from the referenced previous event and all accepted deltas up to that point.
2. This creates a **chain of trust**: an admin can only issue deltas if they were an admin in the immediately preceding accepted state. If a delta removes admin from a pubkey, that pubkey cannot sign any subsequent deltas.
3. Clients MUST apply deltas in chronological order starting from the latest snapshot. If deltas conflict or form forks, the delta with the earlier `created_at` wins; ties are broken by event ID lexicographic order.
4. After a sufficient number of deltas have accumulated, an admin SHOULD publish a new kind 33101 snapshot that folds in all pending changes, re-establishing a clean checkpoint.

**State reconstruction algorithm:**

```
1. Fetch the latest kind 33101 snapshot for the workspace
2. Fetch all kind 33102 deltas with created_at > snapshot.created_at
3. Sort deltas by created_at (ties broken by event ID)
4. Validate each delta's signature against cumulative state
5. Apply each valid delta in order
6. Result is the current membership state
```

### Kind 33103 — Workspace Channel

A **parameterized replaceable** event that defines a channel within a workspace. Published by a member with the `create-channel` capability.

```json
{
  "kind": 33103,
  "content": "{\"about\":\"General discussion\"}",
  "tags": [
    ["d", "<workspace-id>:<channel-id>"],
    ["name", "general"],
    ["privacy", "private"],
    ["p", "<workspace-id>", "", "workspace"]
  ]
}
```

**Content JSON fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `about` | No | Channel description / topic |

**Privacy values:**

| Value | Description |
|-------|-------------|
| `private` | Only workspace members can see and post in this channel (default) |
| `public` | Visible to anyone, but only members can post |

The channel identifier is **`<workspace-id>:<channel-id>`**, ensuring channels are scoped to their workspace.

**Direct message channels** use the identifier format `dm:<pubkey-a>:<pubkey-b>` (pubkeys sorted lexicographically). DM channels are not published as kind 33103 events — they are implied by the existence of MLS epoch events between two members.

### Kind 33104 — Workspace Index

A **parameterized replaceable** event that lists all channel identifiers within a workspace. Published by the workspace owner or a member with `admin` capability. This event allows clients to discover workspace channels without wildcard `d` tag queries.

```json
{
  "kind": 33104,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["channel", "<workspace-id>:general", "private"],
    ["channel", "<workspace-id>:announcements", "private"],
    ["channel", "<workspace-id>:random", "public"]
  ]
}
```

**Tag format:**

| Tag | Fields | Description |
|-----|--------|-------------|
| `channel` | `<channel-d-tag>`, `<privacy>` | Channel `d` tag identifier and privacy level |

Clients SHOULD fetch the latest kind 33104 event for a workspace to discover available channels, then fetch the corresponding kind 33103 events for channel metadata.

**Verification:** The signer MUST be the workspace owner or hold the `admin` capability in the current membership state.

### Kind 33105 — Workspace Invite

A **regular** event sent from a member with the `invite` capability to a prospective member, inviting them to join.

```json
{
  "kind": 33105,
  "content": "{\"caps\":\"write,read\",\"message\":\"Welcome to the team!\"}",
  "tags": [
    ["p", "<invitee-pubkey>", "<relay-url>"],
    ["d", "<workspace-id>"],
    ["caps", "write,read"],
    ["expires", "<unix-timestamp>"]
  ]
}
```

**Content JSON fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `caps` | Yes | Comma-separated capability tags the invitee will receive upon joining |
| `message` | No | A personal message from the inviter |

**Verification rules:**

1. The event MUST be signed by a pubkey that holds the `invite` or `admin` capability in the latest membership state.
2. The invitee SHOULD verify the invitation by checking the signer's capabilities in the latest membership state before accepting.

### Kind 33106 — Workspace Invite Acceptance

A **regular** event published by an invitee to accept a workspace invitation. References the original invitation event.

```json
{
  "kind": 33106,
  "content": "",
  "tags": [
    ["e", "<kind-33105-event-id>", "<relay-url>", "invite"],
    ["d", "<workspace-id>"]
  ]
}
```

After the invitee publishes this event, a member with `invite` or `admin` capability SHOULD publish a kind 33102 delta event that adds the invitee's pubkey with the specified capabilities.

Clients MAY treat the kind 33106 event as provisional membership, showing the workspace while waiting for the admin to publish the membership delta.

### Kind 33107 — Workspace Capability Change

A **regular** event that records a capability change for a workspace member. Published by a member with the `admin` capability.

```json
{
  "kind": 33107,
  "content": "",
  "tags": [
    ["p", "<subject-pubkey>", "<relay-url>"],
    ["d", "<workspace-id>"],
    ["caps", "create-channel,moderate,write,read"],
    ["previous_caps", "write,read"]
  ]
}
```

This event serves as an audit log entry. The actual capability change takes effect when an admin publishes a kind 33101 snapshot or kind 33102 delta event reflecting the new capabilities.

**Verification:** The signer MUST hold the `admin` capability in the current membership state.

### Kind 33108 — Workspace Keypair Roll

A **regular** event published by a workspace admin when a member's Nostr keypair is rotated. This signals to other members and clients that messages encrypted to the old pubkey should be re-encrypted or re-keyed.

```json
{
  "kind": 33108,
  "content": "",
  "tags": [
    ["p", "<subject-pubkey>", "<relay-url>"],
    ["d", "<workspace-id>"],
    ["previous_pubkey", "<old-hex-pubkey>"],
    ["new_pubkey", "<new-hex-pubkey>"]
  ]
}
```

Clients that encounter this event SHOULD:
1. Update their contact list for the subject pubkey.
2. Attempt MLS key re-exchange for channels involving the subject.
3. Display a notice to the user that the member's key has been rotated.

### Kind 33109 — Workspace Avatar

A **parameterized replaceable** event that sets a workspace's avatar image. Published by the workspace owner or a member with `admin` capability.

```json
{
  "kind": 33109,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["image", "<blossom-url>", "<sha256>"]
  ]
}
```

The `image` tag contains:
- The URL to the avatar image (a Blossom URL, HTTP URL, or any resolvable image URL)
- The SHA-256 hash of the image data for verification

### Kind 33110 — Workspace Affiliation Attestation

A **regular** event signed by the norg key (or workspace owner if no norg key is set) that attests to a member's affiliation with the workspace. This provides a lightweight way for third parties to verify membership without replaying the full delta chain.

```json
{
  "kind": 33110,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["p", "<member-pubkey>", "<relay-url>", "write,read"],
    ["a", "33100:<owner-pubkey>:<workspace-id>"],
    ["image", "<workspace-logo-url>", "<sha256>"],
    ["valid_from", "<unix-timestamp>"],
    ["valid_until", "<unix-timestamp>"]
  ]
}
```

**Tag format:**

| Tag | Fields | Description |
|-----|--------|-------------|
| `d` | `<workspace-id>` | Workspace this attestation belongs to |
| `p` | `<pubkey>`, `<relay>`, `<caps>` | Member pubkey, preferred relay, and their capabilities |
| `a` | `<addr>` | `a` tag reference to the kind 33100 workspace definition event |
| `image` | `<url>`, `<sha256>` | **Required.** Workspace logo URL and its SHA-256 hash for verification |
| `valid_from` | `<timestamp>` | Unix timestamp when this attestation becomes valid |
| `valid_until` | `<timestamp>` | Unix timestamp when this attestation expires |

**Verification rules:**

1. The event MUST be signed by the current norg key (the `norg_pubkey` from the latest kind 33100 event, or the workspace owner's pubkey if `norg_pubkey` is absent).
2. The `valid_until` timestamp MUST be in the future at the time of verification.
3. The `image` tag MUST be present and SHOULD match the workspace logo declared in kind 33100. Clients use this logo when rendering affiliation badges without requiring a separate fetch of the workspace definition. If the hash does not match kind 33100, clients SHOULD warn the user or fall back to the kind 33100 logo.
4. The capabilities listed in the `p` tag SHOULD match the member's capabilities in the current membership state, but this is not strictly required — attestation is a convenience signal, not an authoritative roster.
5. Clients performing **full verification** MUST still check the delta chain. Affiliation attestations are a **lightweight shortcut** for display purposes (badges, profile decorations, etc.) and SHOULD NOT be used as the sole authority for access control decisions.
6. The workspace owner or an admin with `admin` capability SHOULD publish new affiliation attestations when membership changes, and SHOULD publish attestations with short `valid_until` windows (e.g., 7 days) to encourage regular refresh.

### Kind 33111 — Norg Key Rotation

A **regular** event that rotates the norg key for a workspace. Signed by the current norg key (or the workspace owner if no norg key is set). After a valid rotation event, the new norg key becomes authoritative for signing kind 33110 attestations.

```json
{
  "kind": 33111,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["previous_norg", "<old-norg-pubkey>"],
    ["new_norg", "<new-norg-pubkey>"]
  ]
}
```

**Verification rules:**

1. The event MUST be signed by the current norg key (or the workspace owner's pubkey if no norg key has been set yet).
2. The `previous_norg` tag MUST match the norg pubkey currently in effect (from kind 33100 `norg_pubkey`, or the owner's pubkey if absent).
3. After this event is accepted, the workspace owner MUST publish an updated kind 33100 event with the `norg_pubkey` field set to the new norg public key.
4. Clients MUST NOT accept kind 33110 attestations signed by the old norg key after the `created_at` of a valid kind 33111 rotation event.

**Key rotation process:**

1. Generate a new norg keypair.
2. Publish a kind 33111 event signed by the current norg key, referencing the new public key.
3. Publish an updated kind 33100 event with `norg_pubkey` set to the new public key.
4. Publish new kind 33110 attestations for all current members, signed by the new norg key.

If the norg key is compromised, the workspace owner can publish a kind 33111 event signed by their own key (which serves as the root of trust when no norg key is set) to rotate to a fresh norg keypair.

## Member Avatar

User avatars within a workspace context are published as kind 0 (NIP-01) metadata events. A member's `picture` field in their kind 0 metadata event serves as their avatar across all workspaces they belong to. Clients SHOULD resolve and cache this image.

For workspace-scoped avatar administration (where a workspace admin sets a member's avatar within the workspace), the avatar URL is stored server-side and distributed via workspace-specific API endpoints. A future NIP could standardize workspace-scoped profile overrides.

## Norg Identity

The norg key provides a **lightweight affiliation mechanism** that allows members to prove workspace membership without the verifier needing to replay the entire snapshot/delta chain.

### Member Norg Tag

Members indicate their workspace affiliation by including a `norg` tag in their kind 0 (NIP-01) metadata event:

```json
{
  "kind": 0,
  "content": "{\"name\":\"alice\",\"about\":\"Engineer at Acme Corp\",\"picture\":\"https://example.com/alice.jpg\"}",
  "tags": [
    ["norg", "<workspace-id>", "<owner-or-norg-pubkey>"]
  ]
}
```

The `norg` tag contains the workspace identifier and the public key that clients should verify against (the norg pubkey from kind 33100, or the workspace owner's pubkey if no norg key is set).

### Affiliation Verification Flow

When a client encounters a `norg` tag in a user's kind 0 metadata:

1. Look up the kind 33100 workspace definition event for the given workspace-id.
2. Extract the norg pubkey from `norg_pubkey` in content (or fall back to the workspace owner's pubkey).
3. Fetch the most recent kind 33110 affiliation attestation for this member's pubkey.
4. Verify the attestation is signed by the norg key.
5. Check that `valid_until` has not expired.
6. Optionally (for full verification), confirm the member appears in the current membership state by replaying snapshot + deltas.

Clients SHOULD display workspace affiliation (e.g., a badge showing "Verified member of Acme Corp") when steps 1–5 succeed, even without step 6. For access control decisions (channel access, messaging permissions), clients MUST perform full delta-chain verification.

### Norg Key Hierarchy

```
Workspace Owner (kind 33100 signer)
  └── Norg Key (norg_pubkey in kind 33100 content)
        └── Signs kind 33110 affiliation attestations
        └── Signs kind 33111 norg key rotations
```

- If `norg_pubkey` is absent, the workspace owner's pubkey serves as the norg key.
- The norg key can only sign attestations and rotation events — it cannot sign membership snapshots/deltas or any other workspace event kinds.
- The norg key can be rotated at any time via kind 33111, signed by the current norg key (or the owner).

## Relay Privacy (NIP-29 Integration)

Workspaces face an inherent privacy concern: membership rosters and channel content published to open Nostr relays are visible to anyone who queries them. This NIP addresses this concern through integration with **NIP-29 Relay-Based Groups**.

### Overview

NIP-29 defines a model where relays enforce group membership and access control. A relay can restrict which pubkeys can read or write events within a group identified by a group ID. This provides **relay-enforced privacy** that complements the cryptographic verification in workspace events.

### Integration Model

A workspace can designate one or more relays (via `r` tags in the kind 33100 definition event) that implement NIP-29 group logic. On these relays:

1. The workspace `d` tag identifier serves as the NIP-29 group ID in the `h` tag.
2. The relay enforces read/write access using NIP-29 moderation events (`kind 9000`, `kind 9001`).
3. Workspace membership snapshots and deltas (kinds 33101, 33102) are published with an `h` tag matching the workspace ID, making them visible only to group members on NIP-29-aware relays.
4. Channel content events likewise carry the `h` tag for relay-enforced scoping.

```json
{
  "kind": 33101,
  "content": "",
  "tags": [
    ["d", "<workspace-id>"],
    ["h", "<workspace-id>"],
    ["p", "<admin-pubkey>", "<relay-url>", "admin,invite,create-channel"],
    ["p", "<member-pubkey>", "<relay-url>", "write,read"]
  ]
}
```

### Dual-Layer Security

This NIP provides **two complementary layers** of security:

| Layer | Mechanism | Guarantees |
|-------|-----------|-----------|
| Cryptographic | Snapshot/delta signatures, admin-chained trust | Any client can verify membership authenticity independently of relay |
| Relay-enforced | NIP-29 group access control | Non-members cannot discover or read workspace events on compliant relays |

Clients operating on open relays rely solely on cryptographic verification. Clients operating on NIP-29-compliant relays benefit from relay-enforced privacy in addition to cryptographic verification.

### Recommended Practice

Workspace owners SHOULD designate at least one NIP-29-compliant relay in their kind 33100 event's `r` tags when membership privacy is a concern. Open relays MAY still be used for public workspace metadata (kind 33100 definitions, kind 33104 channel indices for public channels) while keeping membership data on private relays.

## MLS Integration

Workspaces use MLS (Messaging Layer Security, RFC 9420) for end-to-end encrypted group messaging, brokered through Nostr events.

**Kind 443 — MLS Epoch** (per NIP-EE, TBD)

MLS epoch events are published to the workspace's relay set and reference the channel identifier in a `d` tag. The epoch event contains the MLS Welcome, Commit, or Proposal message encoded in the event content.

**Channel creation:**

1. When a member with `create-channel` capability creates a channel (kind 33103), they also initialize an MLS group.
2. The MLS KeyPackage for each member is distributed via kind 443 events scoped to the workspace.
3. When a new member joins, the admin distributes an MLS Welcome message via a kind 443 event tagged with the channel's `d` identifier.

**Membership changes:**

When a membership delta or snapshot adds or removes members:
1. For each affected channel, a member with `admin` capability MUST generate MLS Commit messages to add or remove the corresponding MLS group members.
2. These Commit messages are published as kind 443 events tagged with the channel's `d` identifier.
3. Clients process these commits to update their MLS key material and ratchet trees.

**DM channels:**

DM channels between two members use MLS with a group identifier of `dm:<pubkey-a>:<pubkey-b>` (pubkeys sorted lexicographically). No kind 33103 event is needed for DMs — the MLS epoch events themselves define the channel.

## NIP-05 Integration

Workspaces can bind member identities to a NIP-05 domain. When a workspace defines a `nip05_domain` in its kind 33100 event:

1. Each member's NIP-05 identifier follows the pattern: `<nip05_name>@<nip05_domain>`
2. For example, in workspace "Acme Corp" with `nip05_domain: "acme.com"`, member Alice has the identifier `alice@acme.com`.
3. A NIP-05 verification service at `https://acme.com/.well-known/nostr.json?name=alice` SHOULD return the member's pubkey and their NIP-05 address.
4. Clients can verify membership by checking that the pubkey returned by NIP-05 lookup appears in the current membership state (latest snapshot + applied deltas).

This creates a trusted bridge between human-readable identifiers and cryptographic proofs of workspace membership. NIP-05 verification and norg attestation verification (kind 33110) are complementary: NIP-05 binds a human-readable name to a pubkey, while norg attestations bind a pubkey to workspace membership.

## Relay Considerations

### Required Relay Support

Relays that wish to support this NIP MUST:

1. Support parameterized replaceable events (NIP-01, NIP-33) for kinds 33100, 33101, 33103, 33104, and 33109.
2. Support regular events for kinds 33102, 33105, 33106, 33107, 33108, 33110, and 33111.
3. Support the `#d` filter to allow querying by workspace identifier.
4. Support the `#p` filter for membership queries.

### Recommended Queries

**Discover all workspaces:**
```
{kinds: [33100]}
```

**Get current membership snapshot for a workspace:**
```
{kinds: [33101], "#d": ["<workspace-id>"]}
```

**Get membership deltas after a snapshot:**
```
{kinds: [33102], "#d": ["<workspace-id>"], "since": <snapshot-created_at>}
```

**Get workspace index (channel listing):**
```
{kinds: [33104], "#d": ["<workspace-id>"]}
```

**Get all channels in a workspace:**
```
{kinds: [33103], "#d": ["<workspace-id>:*"]}
```

Note: relay support for wildcard `d` tag matching is optional. Clients SHOULD prefer using the kind 33104 workspace index event to discover channel identifiers.

**Get all invites for a user:**
```
{kinds: [33105], "#p": ["<user-pubkey>"]}
```

**Get MLS epochs for a channel:**
```
{kinds: [443], "#d": ["<workspace-id>:<channel-id>"]}
```

**Get affiliation attestations for a member:**
```
{kinds: [33110], "#p": ["<member-pubkey>"]}
```

**Get affiliation attestations for a workspace:**
```
{kinds: [33110], "#d": ["<workspace-id>"]}
```

**Get norg key rotations for a workspace:**
```
{kinds: [33111], "#d": ["<workspace-id>"]}
```

### Federation

A workspace's events can live on multiple relays. The workspace owner specifies preferred relays in the kind 33100 event's `r` tags or in the `relay` field of `p` tags. Members and clients SHOULD publish and subscribe to these relay URLs to ensure they see all workspace events.

```json
{
  "kind": 33100,
  "tags": [
    ["d", "<workspace-id>"],
    ["r", "wss://relay1.example.com"],
    ["r", "wss://relay2.example.com"]
  ]
}
```

## Client Implementation Guidelines

### Discovery

1. Clients discover workspaces through kind 33100 events on their relay set.
2. Clients may also discover workspaces through kind 33105 invite events addressed to the user.
3. Upon discovering a workspace, clients SHOULD fetch the latest kind 33101 snapshot and any subsequent kind 33102 deltas to determine current membership.
4. Clients SHOULD fetch the kind 33104 workspace index to discover available channels.

### Membership Verification

When rendering a workspace or its channels, clients MUST:

1. Fetch the latest kind 33101 snapshot for the workspace.
2. Fetch all kind 33102 deltas with `created_at` after the snapshot.
3. Apply deltas in chronological order to build the current membership state.
4. Verify each event in the chain is signed by either:
   - The workspace owner (creator of kind 33100), OR
   - A pubkey that held the `admin` capability in the state immediately prior to that event.
5. Only show channels and messages to users whose pubkey appears in the current membership state with the `read` capability or higher.

### Capability Evaluation

Clients MUST evaluate capabilities as follows:

1. The workspace owner ALWAYS has all capabilities, regardless of what the membership state says.
2. The initial kind 33101 snapshot MUST be signed by the workspace owner.
3. Each subsequent delta MUST be signed by the owner OR by a pubkey with `admin` in the immediately preceding state.
4. If a member loses the `admin` capability, they can no longer sign valid snapshots or deltas from that point forward.

### Affiliation Verification

Clients encountering a `norg` tag in a user's kind 0 metadata SHOULD:

1. Fetch the kind 33100 workspace definition and extract `norg_pubkey` (or fall back to the owner's pubkey).
2. Fetch the most recent kind 33110 attestation for the member's pubkey.
3. Verify the attestation signature against the norg key.
4. Check that `valid_until` has not expired.
5. Display affiliation status (e.g., a verified badge) if verification succeeds.
6. For access control decisions, perform full membership verification via the snapshot/delta chain rather than relying solely on attestations.

### Message Display

1. Channel messages are published as kind 443 MLS events (or kind 9 kind 42 for plain text, TBD) with a `d` tag matching the channel identifier.
2. Clients filter messages by channel identifier and display them in chronological order.
3. E2EE messages within MLS groups are decrypted using the member's MLS key material.

### Leaving a Workspace

A member who wishes to leave a workspace publishes a kind 33106-like event (or future kind) indicating departure. A member with `admin` capability SHOULD publish a kind 33102 delta removing the departing member. The MLS group for each channel should also be updated with a Commit removing the member's leaf.

## Migration and Portability

A key design goal is that a workspace's entire state can be exported and imported across clients and relays:

1. **Export:** A client can query all events with `#d` tags matching a workspace identifier and produce a complete archive, including the full delta chain.
2. **Import:** A different client can ingest these events and reconstruct the workspace, including all membership history (by replaying the delta chain), channel definitions, and (if keys are available) decrypted messages.
3. **Relay migration:** A workspace owner can publish updated kind 33100 events with new `r` tags to migrate traffic to different relays.
4. **Snapshot re-sealing:** When migrating or after many deltas have accumulated, an admin SHOULD publish a fresh kind 33101 snapshot that consolidates the current state.

## Security Considerations

### Impersonation Resistance

- Workspace membership is verified through the chained signature model: each snapshot and delta must be signed by the workspace owner or a pubkey with `admin` capability in the immediately preceding state. A malicious actor cannot insert themselves into the delta chain without a valid signing key.
- NIP-05 verification provides an additional layer by allowing cross-referencing of human-readable identities.

### Capability Escalation Prevention

- The chain-of-trust model prevents capability escalation: a member can only receive capabilities that an `admin` (or the owner) explicitly grants in a signed delta. A member with `write,read` cannot self-promote to `admin` because they cannot sign a valid delta.
- The initial snapshot MUST be signed by the workspace owner, establishing a root of trust.

### Key Compromise

- If a workspace admin's key is compromised, the attacker can modify membership, capabilities, channels, and settings. Workspace owners (kind 33100 creators) can publish a new kind 33101 snapshot that removes the compromised key's `admin` capability, invalidating any further deltas signed by that key.
- Kind 33108 key roll events provide an audit trail for key rotations.

### Forward Secrecy

- MLS provides forward secrecy for message content. Even if a member's key is compromised, past messages remain encrypted.
- Workspace admins should rotate MLS epoch keys when members are removed (membership deltas should trigger MLS Commit messages).

### Delta Chain Integrity

- Clients MUST validate the full delta chain from the latest snapshot. A delta that does not reference a valid previous event, or is signed by a non-admin in the preceding state, MUST be rejected.
- In the event of conflicting deltas (forks), clients accept the delta with the earlier `created_at`; ties are broken by lexicographic order of event IDs.

### Norg Key Security

- The norg key is a high-value target: compromise allows an attacker to issue fraudulent affiliation attestations. The norg key private key SHOULD be stored securely (e.g., hardware security module, offline storage) and only accessed when publishing attestations or key rotations.
- The norg key can only sign kind 33110 and 33111 events — it has no authority over membership snapshots, deltas, or other workspace operations. This limits the blast radius of a norg key compromise to affiliation claims only.
- If the norg key is compromised, the workspace owner can rotate it by publishing a kind 33111 event signed by their own key (which serves as root of trust). All previously issued attestations become invalid; the owner SHOULD republish attestations for all current members signed by the new norg key.
- Affiliation attestations with short `valid_until` windows limit the window of exploitation even if the norg key is compromised, since expired attestations are rejected by clients.

### Affiliation vs. Authorization

- Kind 33110 affiliation attestations are for **display** (badges, profile decorations, social proof), not for **authorization**. A valid attestation says "the norg key signed this at some point," not "this person currently has these exact capabilities."
- Access control decisions (channel access, message visibility, admin actions) MUST be based on the membership snapshot/delta chain, not on affiliation attestations alone.
- This separation is by design: attestation caching and expiry provide a fast UX path, while the delta chain provides the authoritative ground truth.