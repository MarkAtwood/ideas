---
title: JMAP for Chat
abbrev: JMAP Chat
docname: draft-atwood-jmap-chat-00
category: exp
stream: independent

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC8174:
  RFC8620:

informative:
  RFC8621:
  RFC9420:
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

--- abstract

This document defines JMAP for Chat, a JMAP capability ({{RFC8620}}) for direct and group text messaging. It supports two deployment topologies: a mailbox-per-user model in which each user operates their own server, and a relay model in which a shared server routes end-to-end encrypted messages.

The specification defines the `urn:jmap:chat:1` capability; sixteen data types (Attachment, Endpoint, MessageAction, Mention, MessageRevision, Reaction, Contact, ChatMember, Chat, Message, SpaceRole, SpaceMember, Category, ChannelPermission, Space, and SpaceInvite); JMAP owner methods for each top-level type; and five server-to-server methods (Peer/deliver, Peer/receipt, Peer/typing, Peer/retract, Peer/groupUpdate).

The protocol covers the feature set common to contemporary messaging systems: group chat with membership roles, message reactions, editing, deletion, threading, @mentions, typing indicators, read receipts per participant, presence, pinned messages, per-chat notification settings, sender-controlled message expiry, and burn-on-read. Spaces provide a named multi-channel container with a role-based permission system, analogous to what other systems call a server, workspace, or team.

Note: `urn:jmap:chat:1` is a provisional capability URI. If this specification is adopted by the IETF JMAP working group, the URI will be updated to `urn:ietf:params:jmap:chat`.

--- middle

# Introduction

JMAP {{RFC8620}} defines a JSON-based protocol for accessing and mutating application data. The core protocol is intentionally generic; application semantics are expressed through capability URIs declared in the JMAP Session object. {{RFC8621}} defines JMAP for Mail. This document defines an analogous capability for real-time chat.

## Deployment Topologies

This specification accommodates two primary deployment topologies.

In the **mailbox-per-user** model, each participant runs their own JMAP server (a "mailbox") that stores only their own messages. There is no central server, no central message store, and no central operator. Mailboxes exchange messages directly with each other over a secure transport using the Peer/* server-to-server methods defined in {{peer-methods}}.

In the **relay** model, a shared server routes messages between clients. The Peer/* methods are implemented by the relay rather than by individual user-controlled mailboxes. Relay deployments are designed to handle only opaque ciphertext; see {{e2ee}} for the normative requirements.

Both topologies are fully compatible with this specification. Transport, identity, and encryption choices are confined to the deployment layer.

## Authentication Model

Authentication is handled entirely at the transport layer. The protocol requires only that the authentication layer provide a stable, opaque user identity string for each connection. How that identity is established — overlay network membership, mutual TLS, bearer tokens, or any other mechanism — is outside the scope of this document. Authorization rules derived from this identity are defined in {{authorization}}.

## Reference Implementations

Two reference implementations informed this specification: Kith, a mailbox-per-user daemon, and nie, an E2EE relay system.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout.

The following terms are specific to this document:

Mailbox:
: A JMAP server instance serving exactly one user.

Owner:
: The user whose data a mailbox stores and serves.

Peer:
: Another mailbox server communicating with this mailbox.

id / userId:
: A Contact's `id` is the stable, opaque identity string provided by the authentication layer for that user. These two terms are intentionally equivalent in this protocol: Contact.id IS the userId. There is no separate identity namespace. Servers MUST set Contact.id to the userId string obtained from the authentication layer.

# The urn:jmap:chat:1 Capability {#capability}

The `urn:jmap:chat:1` capability is advertised in the JMAP Session object at both the top-level `capabilities` key and within each account's `accountCapabilities` map.

## Session-Level Capability Object

The value of `capabilities["urn:jmap:chat:1"]` is a JSON object with the following fields:

`maxBodyBytes` (UnsignedInt):
: Maximum UTF-8 byte length of a Message `body`. Servers MUST reject messages exceeding this limit with `invalidArguments`.

`maxAttachmentBytes` (UnsignedInt):
: Maximum size in bytes of a single attachment blob.

`maxAttachmentsPerMessage` (UnsignedInt):
: Maximum number of attachments per message.

`maxGroupMembers` (UnsignedInt):
: Maximum number of members in a group Chat, including the owner.

`maxSpaceMembers` (UnsignedInt):
: Maximum number of members in a Space, including the owner.

`maxRolesPerSpace` (UnsignedInt):
: Maximum number of named roles per Space.

`maxChannelsPerSpace` (UnsignedInt):
: Maximum number of channel Chats per Space.

`maxCategoriesPerSpace` (UnsignedInt):
: Maximum number of categories per Space.

`supportedBodyTypes` (String[]):
: MIME types accepted in `bodyType`. MUST include `"text/plain"`. End-to-end encrypted deployments SHOULD also include an appropriate encrypted-content type such as `"application/mls-ciphertext"`.

`supportsThreads` (Boolean):
: Whether this server supports the optional thread model defined in {{threads}}.

## Account-Level Capability Object

The value of `accountCapabilities["urn:jmap:chat:1"]` is a JSON object with the following field:

`role` (String):
: Either `"owner"` or `"peer"`.

## Session Object Extensions

Servers MAY include in the Session object:

`ownerUserId` (String):
: The id of the mailbox owner (equals the owner's Contact.id on any peer server that has recorded this mailbox as a contact).

`ownerLogin` (String):
: A human-readable login name for the mailbox owner.

`ownerDirectAddress` (String, optional):
: A deployment-specific address at which the owner's client may be reachable directly. Peers that probe `/.well-known/jmap` SHOULD store this value as the `directAddress` on the corresponding Contact record.

`ownerEndpoints` (Endpoint[], optional):
: The owner's advertised out-of-band capability endpoints (see {{endpoint}}). Peers that probe `/.well-known/jmap` SHOULD merge these into the `endpoints` field of the corresponding Contact record.

# Data Types

Data types are defined in dependency order: embedded sub-types precede the types that contain them.

## Attachment {#attachment}

An Attachment carries metadata for a file blob associated with a Message.

`blobId` (String):
: Opaque server-assigned blob identifier.

`filename` (String):
: Original filename. MUST NOT contain `/`, `\`, or null bytes.

`contentType` (String):
: Valid MIME type string.

`size` (UnsignedInt):
: Blob size in bytes. Servers MUST verify against actual content.

`sha256` (String):
: Lowercase hex SHA-256 of blob content. Servers SHOULD verify.

## Endpoint {#endpoint}

An Endpoint advertises an out-of-band capability reachable at a URI. Endpoints appear on Contact records and in Session objects as persistent capability advertisements. The `type` field uses an extensible URI namespace; clients MUST silently ignore Endpoint records whose `type` they do not recognize.

`type` (String):
: A URI identifying the capability type. Well-known values defined by this document:

  - `"urn:jmap:chat:cap:vtc"` — video/voice teleconference. `uri` is a signaling or room URL (e.g., a WebRTC signaling endpoint, a Jitsi room URL, a SIP URI).
  - `"urn:jmap:chat:cap:payment"` — payment receiving endpoint. `uri` is a payment URI (e.g., `lightning:...`, `zcash:...`, `monero:...`, `bitcoin:...`).
  - `"urn:jmap:chat:cap:blob"` — out-of-band file transfer endpoint. `uri` is a base URL for fetching or uploading blobs outside the JMAP blob mechanism.

  Other type URIs MAY be defined by deployments or future documents. The `urn:jmap:chat:cap:` prefix is reserved for types defined in JMAP Chat specifications.

`uri` (String):
: The endpoint URI. Format is type-specific. Peer-supplied; MUST be treated as untrusted.

`label` (String, optional):
: Human-readable label for this endpoint (e.g., `"Personal Jitsi"`, `"Zcash address"`).

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys. Examples by type:

  - `vtc`: `{"protocol": "webrtc", "roomName": "...", "password": "..."}`
  - `payment`: `{"network": "lightning", "currency": "BTC"}`
  - `blob`: `{"maxBytes": 10485760}`

## MessageAction {#message-action}

A MessageAction is a per-message out-of-band action invitation carried in a Message and in `Peer/deliver`. It signals that a message is associated with an out-of-band interaction — a video call invitation, a payment request, a file available outside the blob channel, etc. Servers MUST store and forward MessageAction records without inspection or transformation. Clients MUST NOT act on a MessageAction automatically; all OOB actions require explicit user initiation.

`type` (String):
: Same URI namespace as Endpoint `type` (see {{endpoint}}). Clients MUST ignore actions whose `type` they do not recognize.

`uri` (String):
: The action URI. Peer-supplied; MUST be treated as untrusted.

`label` (String, optional):
: Human-readable label for the action (e.g., `"Join call"`, `"Pay $5"`, `"Download file"`).

`expiresAt` (UTCDate, optional):
: Time after which the action is no longer valid. Clients SHOULD visually indicate expired actions. Servers MUST NOT enforce expiry on stored actions; enforcement is the OOB system's responsibility.

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys.

## Mention {#mention}

A Mention identifies a user referenced within a message body.

`id` (String):
: The Contact.id (userId) of the mentioned participant.

`offset` (UnsignedInt):
: Byte offset into `body` where the mention text begins.

`length` (UnsignedInt):
: Byte length of the mention text. Servers MUST reject a mention where `offset + length` exceeds the byte length of `body`.

## MessageRevision {#message-revision}

A MessageRevision records one historical version of a Message body.

`body` (String):
: The prior body text.

`bodyType` (String):
: The prior MIME type.

`editedAt` (UTCDate):
: The time this version was superseded by an edit.

## Reaction {#reaction}

A Reaction is an emoji response to a Message.

`id` (String, immutable, server-set):
: A ULID assigned by the receiving server when the reaction is stored. For reactions composed by the owner, equals `senderReactionId`.

`senderReactionId` (String, immutable):
: The sender-assigned ULID for this reaction, carried in `Peer/deliver` `reactionUpdate`. Used to identify a reaction for removal across the mailbox boundary.

`emoji` (String):
: A non-empty string identifying the reaction. Typically a Unicode emoji sequence or a deployment-defined token.

`customEmojiId` (String, optional):
: The id of a Space-scoped custom emoji. When present, `emoji` SHOULD contain a fallback representation (e.g., the emoji name) for clients that do not support custom emoji.

`senderId` (String):
: `"self"` for the owner's reaction, or a Contact.id.

`sentAt` (UTCDate):
: Time the reaction was added.

## Contact {#contact}

A Contact represents a remote user known to this mailbox. A Contact's `id` is exactly the userId provided by the authentication layer: it is the single, global identity key for that user within this deployment.

`id` (String, immutable, server-set):
: The userId provided by the authentication layer. Servers MUST set this to the verified identity string and MUST NOT assign a different value.

`login` (String, server-set):
: Human-readable login name. MUST NOT be empty. Typically email-shaped but the format is deployment-specific.

`displayName` (String, optional):
: Human-readable display name. MAY be absent or empty. Clients SHOULD fall back to `login`, then `id`.

`serverUrl` (String):
: Base HTTPS URL of this contact's mailbox. Used for outbound delivery and for probing `/.well-known/jmap`.

`firstSeenAt` (UTCDate, server-set):
: Time this contact was first recorded.

`lastSeenAt` (UTCDate, server-set):
: Time of most recent interaction with this contact's mailbox.

`presence` (String, server-set):
: Last known presence state: `"online"`, `"away"`, `"offline"`, or `"unknown"`. Updated on a best-effort basis.

`lastActiveAt` (UTCDate, optional, server-set):
: Time the contact was last observed to be active.

`directAddress` (String, optional):
: A deployment-specific address at which this contact's client may be reachable directly, without routing through their mailbox. The format and semantics are deployment-defined (examples: a Tailscale node name, a WebRTC signaling URI, an IP:port). This field is a hint only: senders MAY attempt delivery to this address when both parties are online and the deployment supports it, but MUST fall back to the standard mailbox path on any failure. This field has no effect on message storage or multi-device sync. Servers populate this field from the `ownerDirectAddress` advertised in the contact's `/.well-known/jmap` Session object.

`endpoints` (Endpoint[], optional):
: Out-of-band capability endpoints advertised by this contact. Servers populate this field from the contact's `ownerEndpoints` at `/.well-known/jmap`. Clients MAY use these to initiate video calls, send payments, or fetch files outside the JMAP message channel. Clients MUST NOT act on these values automatically without explicit user intent.

`blocked` (Boolean):
: When `true`, messages from this contact are silently dropped by this mailbox, including messages arriving in group chats. Default: `false`.

## ChatMember {#chat-member}

A ChatMember describes one participant in a group Chat. The `id` field is the participant's Contact.id.

`id` (String):
: The participant's Contact.id / userId.

`role` (String):
: Either `"admin"` or `"member"`. Admins may add and remove members and update group chat metadata. The creator is automatically assigned the `"admin"` role.

`joinedAt` (UTCDate):
: Time this participant joined the chat.

`invitedBy` (String, optional):
: The Contact.id of the member who added this participant.

## Chat {#chat}

A Chat is a conversation between two or more participants. Three kinds are defined: `"direct"` (one-to-one), `"group"` (multi-party), and `"channel"` (a channel within a Space). Fields whose applicability is restricted to one or two kinds are labeled accordingly; unlabeled fields apply to all kinds.

### Chat ID Assignment {#chat-id}

All Chat IDs are ULIDs {{ULID}} assigned by the creating server at the moment the chat is created. IDs are opaque and stable for the lifetime of the chat.

For a **direct chat**, the creating server is the one whose owner sends the first message. Before assigning a new chatId, the server MUST check whether a direct chat with the relevant contactId already exists locally. If one exists, the server MUST use the existing chatId rather than creating a new one. When a `Peer/deliver` arrives for a direct chat with an unknown chatId, the receiving server creates a new Chat record with that chatId and sets `contactId` to the sender.

For a **group chat**, the creating server assigns the chatId and distributes it to all initial members via `Peer/groupUpdate` ({{peer-groupupdate}}) before any messages are sent.

**Channel** Chats are created as part of a Space via the `addChannels` patch key in `Space/set` ({{space-set}}). Their chatId is assigned by the server at that time.

### Chat Object Fields

`id` (String, immutable, server-set):
: A ULID assigned per {{chat-id}}.

`kind` (String, immutable):
: `"direct"`, `"group"`, or `"channel"`. Channel Chats do not carry a `members` field; membership and access control are determined by the containing Space.

`contactId` (String, immutable):
: **Direct chats only.** The Contact.id of the other participant.

`name` (String):
: **Group and channel Chats.** Display name. Required at creation for group chats; mutable by admins. For channel chats, mutable by members with `"manage_channels"` permission.

`description` (String, optional):
: **Group chats only.** Short description. Mutable by admins.

`avatarBlobId` (String, optional):
: **Group chats only.** blobId of the group avatar image. Mutable by admins.

`members` (ChatMember[]):
: **Group chats only.** Full membership list including the owner.

`spaceId` (String, immutable):
: **Channel Chats only.** The id of the containing Space.

`categoryId` (String, optional):
: **Channel Chats only.** The Category id within the Space. Absent if this channel is uncategorized.

`position` (UnsignedInt, optional):
: **Channel Chats only.** Sort order within the category (or among uncategorized channels). Lower values appear first.

`topic` (String, optional):
: **Channel Chats only.** Short description shown in the channel header. Mutable by members with `"manage_channels"` permission.

`slowModeSeconds` (UnsignedInt, optional):
: **Channel Chats only.** When non-zero, each member MUST wait at least this many seconds between messages in this channel. Members with `"manage_channels"` permission are exempt. Enforced server-side; clients SHOULD display a countdown. Default: `0`.

`permissionOverrides` (ChannelPermission[]):
: **Channel Chats only.** Per-channel permission overrides for specific roles or members. Evaluated after Space-level role permissions per {{space-permissions}}. Only members with `"manage_channels"` permission may modify this list. Empty by default.

`createdAt` (UTCDate, immutable, server-set):
: Time this chat was first recorded on this mailbox.

`lastMessageAt` (UTCDate, optional, server-set):
: Received time of the most recent message.

`unreadCount` (UnsignedInt, server-set):
: Messages received since the owner last read this chat.

`pinnedMessageIds` (String[]):
: Ordered list of pinned Message ids, most-recently-pinned first. For group chats, only admins may modify this list. For direct chats, the owner may modify it freely. Empty by default.

`muted` (Boolean):
: When `true`, push notifications for this chat are suppressed. Owner-side preference; not shared with peers. Default: `false`.

`muteUntil` (UTCDate, optional):
: Muting expires at this time. Servers SHOULD clear `muted` and `muteUntil` automatically when the time passes.

`messageExpirySeconds` (UnsignedInt, optional):
: A local expiry policy. When set and non-zero, messages in this chat older than this many seconds are deleted by this mailbox. Each mailbox enforces its own policy independently. This is a local setting, not a bilateral negotiated commitment.

## Message {#message}

A Message is a single transmission within a Chat.

### Message IDs {#message-ids}

Message IDs are ULIDs {{ULID}}, assigned by the **receiving** mailbox at storage time. ULIDs are lexicographically ordered by time, enabling ordered retrieval without a separate sort field.

The **sender-assigned ULID** (`senderMsgId`) is set by the originating mailbox and carried in `Peer/deliver`. The receiving mailbox stores both its own `id` and the `senderMsgId`. Servers MUST maintain a durable index of `senderMsgId` values per chat to support idempotent delivery, `Peer/retract` lookup, and resolution of `replyTo` / `threadRootId` references. If a `senderMsgId` is seen again for the same chat, the server MAY silently discard the duplicate.

### Message Object Fields

`id` (String, immutable, server-set):
: Receiver-assigned ULID. Used in all client-facing references.

`senderMsgId` (String, immutable, server-set):
: The sender-assigned ULID carried in `Peer/deliver`. Equals `id` for messages composed by the owner.

`chatId` (String, immutable):
: ID of the containing Chat.

`senderId` (String, immutable, server-set):
: `"self"` for owner-composed messages; the sender's Contact.id for inbound messages, as verified by the authentication layer.

`body` (String):
: Message content. When `bodyType` is `"text/plain"` or another plaintext type, `body` MUST be valid UTF-8 text. When `bodyType` indicates an end-to-end encrypted payload (e.g., `"application/mls-ciphertext"`), `body` contains ciphertext encoded as a base64url string; servers MUST store and forward it without inspection or transformation. Cleared to empty string when the message is deleted.

`bodyType` (String):
: MIME type of `body`. MUST be in `supportedBodyTypes`.

`attachments` (Attachment[]):
: File attachments. Cleared to empty array when deleted.

`mentions` (Mention[]):
: Structured @mention annotations. Empty by default.

`actions` (MessageAction[]):
: Out-of-band action invitations associated with this message. Empty by default. Servers MUST store and forward these without inspection.

`reactions` (Reaction[]):
: Emoji reactions. Empty by default.

`replyTo` (String, optional):
: The receiver-assigned `id` of the Message this replies to. Servers MUST validate that this ID exists in the same Chat before storing.

`threadRootId` (String, optional):
: The receiver-assigned `id` of the thread root message. Only meaningful when `supportsThreads` is `true`. See {{threads}}.

`replyCount` (UnsignedInt, server-set):
: Count of messages in this chat with `replyTo` equal to this message's `id`. Present only when `supportsThreads` is `true`.

`sentAt` (UTCDate):
: Sender's claimed composition time. Peer-supplied; MUST be treated as untrusted and MUST NOT be used for ordering.

`receivedAt` (UTCDate, immutable, server-set):
: Time this mailbox stored the message. Authoritative for ordering and expiry calculations.

`senderExpiresAt` (UTCDate, optional, immutable):
: Sender-set hard-deletion deadline. When present, servers MUST permanently delete this message — removing the row entirely, not leaving a tombstone — at or before this time. A hard-deleted message appears in the `destroyed` list of subsequent `Message/changes` responses, not `updated`. Receiving servers MUST honor this field regardless of local `messageExpirySeconds` policy; whichever deadline arrives first takes effect. Servers MUST NOT use this field for ordering. Servers MUST reject a `senderExpiresAt` value that is already in the past at delivery time with `invalidArguments`. After hard deletion, stored attachment blobs referenced by the message SHOULD also be purged.

`burnOnRead` (Boolean, optional, immutable):
: When `true`, the receiving server MUST permanently hard-delete this message (row removal, not tombstone) immediately after setting `readAt`. Applies only to the receiving mailbox; the sender's own copy is not affected. In E2EE relay deployments, the relay cannot observe read events; the bridge or client layer MUST enforce this rule after receiving the read acknowledgement from the owner.

`deliveryState` (String, server-set):
: `"pending"`, `"delivered"`, `"failed"`, or `"received"`. For group chats, reflects aggregate state across all recipients; see `deliveryReceipts` for per-recipient detail.

`deliveryReceipts` (Object, optional, server-set):
: For group chats, a JSON object mapping each non-owner participant's Contact.id to `{"deliveredAt": <UTCDate-or-null>, "readAt": <UTCDate-or-null>}`. Present only when `senderId` is `"self"`.

`deliveredAt` (UTCDate, optional, server-set):
: Time the first outbound delivery was acknowledged.

`readAt` (UTCDate, optional, server-set):
: Time the owner acknowledged reading this message.

`editedAt` (UTCDate, optional, server-set):
: Time of the most recent edit.

`editHistory` (MessageRevision[], optional, server-set):
: Prior versions, oldest first. Servers MAY limit the number of retained revisions.

`deletedAt` (UTCDate, optional, server-set):
: Time the message was deleted. When set, `body` is empty and `attachments` is empty. The record is retained as a tombstone unless a hard-delete rule applies.

`deletedForAll` (Boolean, optional, server-set):
: `true` when deletion was propagated to all participants via `Peer/retract`.

## SpaceRole {#space-role}

A SpaceRole is a named set of permissions within a Space. Roles are ordered by `position`; higher position values outrank lower ones. The implicit `@everyone` role (position 0) is held by all Space members and is not included in the `roles` array.

`id` (String, immutable, server-set):
: A ULID assigned by the server.

`name` (String):
: Display name of the role.

`color` (String, optional):
: Hex color string (e.g., `"#5865f2"`). Clients MAY use this to visually distinguish role holders.

`permissions` (String[]):
: Named permissions this role grants. Defined permission names:

  - `"view"` — see the channel
  - `"send"` — send messages
  - `"pin"` — pin messages
  - `"manage_channels"` — create, edit, delete, and reorder channels
  - `"manage_members"` — kick members, edit nicknames
  - `"manage_roles"` — create and edit roles below own highest role
  - `"manage_space"` — edit Space name, description, and icon
  - `"ban"` — ban and unban members
  - `"mention_all"` — use Space-wide @mentions

  Servers MUST ignore unrecognized permission names.

`position` (UnsignedInt):
: Role hierarchy position. No two roles in a Space SHOULD share the same value.

## SpaceMember {#space-member}

A SpaceMember describes one participant in a Space.

`id` (String):
: The participant's Contact.id.

`roleIds` (String[]):
: SpaceRole ids held by this member. Order is not significant. An empty list means the member holds only the `@everyone` role.

`nick` (String, optional):
: Space-specific display name override. MAY be absent; clients SHOULD fall back to Contact `displayName`, then `login`.

`joinedAt` (UTCDate):
: Time this member joined the Space.

## Category {#category}

A Category is a named grouping of channels within a Space.

`id` (String, immutable, server-set):
: A ULID assigned by the server.

`name` (String):
: Display name of the category.

`position` (UnsignedInt):
: Sort order among categories. Lower values appear first.

`channelIds` (String[]):
: Ordered list of channel Chat ids in this category.

## ChannelPermission {#channel-permission}

A ChannelPermission record overrides Space-level role permissions for a specific channel, for a specific role or member. The evaluation order for these overrides is defined in {{space-permissions}}.

`targetId` (String):
: A SpaceRole id or a SpaceMember Contact.id.

`targetType` (String):
: `"role"` or `"member"`.

`allow` (String[]):
: Permissions explicitly granted in this channel, overriding the Space-level role defaults.

`deny` (String[]):
: Permissions explicitly denied in this channel, overriding the Space-level role defaults.

## Space {#space}

A Space is a named container for channel Chats, members, roles, and categories. It corresponds to what other systems call a server, workspace, or team.

`id` (String, immutable, server-set):
: A ULID assigned by the server.

`name` (String):
: Display name of the Space.

`description` (String, optional):
: Short description. Mutable by members with `"manage_space"` permission.

`iconBlobId` (String, optional):
: blobId of the Space icon. Mutable by members with `"manage_space"` permission.

`roles` (SpaceRole[]):
: Named roles defined for this Space, ordered by `position` descending. Does not include the implicit `@everyone` role.

`members` (SpaceMember[]):
: Full membership list including the owner.

`categories` (Category[]):
: Categories, ordered by `position`.

`uncategorizedChannelIds` (String[]):
: Ordered list of channel Chat ids not assigned to any category.

`createdAt` (UTCDate, immutable, server-set):
: Time this Space was created.

## SpaceInvite {#space-invite}

A SpaceInvite grants a new member access to a Space via a shared invite code.

`id` (String, immutable, server-set):
: The invite code. Opaque short string.

`spaceId` (String, immutable):
: The Space this invite grants access to.

`defaultChannelId` (String, optional):
: Chat id of the channel to highlight when a new member arrives.

`createdBy` (String, immutable, server-set):
: Contact.id of the member who created this invite.

`expiresAt` (UTCDate, optional):
: Expiry time. Servers MUST reject redemption after this time.

`maxUses` (UnsignedInt, optional):
: Maximum redemption count. Servers MUST reject redemption when `uses` equals `maxUses`.

`uses` (UnsignedInt, server-set):
: Number of times this invite has been redeemed.

`createdAt` (UTCDate, immutable, server-set):
: Time this invite was created.

# Methods

## Contact Methods

Contacts are created automatically when a peer delivers a message or a group update names a new participant. Owner clients may not create or destroy contacts directly.

### Contact/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### Contact/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### Contact/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`create` and `destroy` are not supported; both MUST return `forbidden`.

`update` supports: `blocked`, `displayName`.

### Contact/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties: `blocked` (Boolean), `presence` (String).
Sort properties: `lastSeenAt`, `login`, `lastActiveAt`. When sorting by `lastActiveAt`, contacts for which the field is absent sort last.

## Chat Methods

### Chat/get

Standard JMAP `/get`.

### Chat/changes

Standard JMAP `/changes`.

### Chat/set

Standard JMAP `/set`.

#### Creating a Direct Chat

`create` with `kind: "direct"` accepts:

`contactId` (String, required):
: Contact.id of the other participant. If a direct Chat with this contactId already exists, the server MUST return it in `updated` rather than creating a duplicate. Otherwise the server assigns a new ULID per {{chat-id}}.

#### Creating a Group Chat

`create` with `kind: "group"` accepts:

`name` (String, required):
: Display name of the group.

`memberIds` (String[], required):
: Contact.ids of initial non-owner members. Total membership including the owner MUST NOT exceed `maxGroupMembers`.

Optional at creation: `description` (String), `avatarBlobId` (String), `messageExpirySeconds` (UnsignedInt).

The server assigns the chatId (a ULID), sets the owner as an admin member, and MUST send `Peer/groupUpdate` to each initial member before any messages are sent.

#### Updating a Chat

`update` supports the following patch keys for all chat kinds: `muted`, `muteUntil`, `pinnedMessageIds`, `messageExpirySeconds`.

For group chats, admins additionally may update: `name`, `description`, `avatarBlobId`.

Member list changes use the following patch keys (all require admin role):

`addMembers` (Object[]):
: Each entry: `id` (String, Contact.id) and optional `role` (String, default `"member"`). Total membership after addition MUST NOT exceed `maxGroupMembers`. The server MUST send `Peer/groupUpdate` to all current members.

`removeMembers` (String[]):
: Contact.ids to remove. The server MUST send `Peer/groupUpdate` to all remaining members and to the removed members.

`updateMemberRoles` (Object[]):
: Each entry: `id` (String) and `role` (String). The server MUST send `Peer/groupUpdate` to all members.

### Chat/query

Standard JMAP `/query`.

Filter properties: `kind` (String), `muted` (Boolean).
Default sort: `lastMessageAt` descending; chats without messages sort last.

## Message Methods

### Message/get

Standard JMAP `/get`.

### Message/changes

Standard JMAP `/changes`.

### Message/set

Standard JMAP `/set`.

#### Creating a Message

`create` accepts:

`chatId` (String, required), `body` (String, required), `bodyType` (String, required), `sentAt` (UTCDate, required).

Optional: `attachments` (Attachment[]), `mentions` (Mention[]), `actions` (MessageAction[]), `replyTo` (String), `threadRootId` (String), `senderExpiresAt` (UTCDate), `burnOnRead` (Boolean).

The server sets `id`, `senderMsgId`, `senderId`, `receivedAt`, `deliveryState`, and delivery timestamp fields, then enqueues the message for outbound delivery.

#### Editing a Message

`update` with changed `body`, `bodyType`, and/or `mentions`, on a message where `senderId` is `"self"` and `deletedAt` is absent.

The server MUST:

1. Push a MessageRevision onto `editHistory` with the current `body`, `bodyType`, and current server time as `editedAt`.
2. Replace `body` and `bodyType` with the submitted values.
3. Set `editedAt` to the current server time.
4. Send `Peer/deliver` carrying an `edit` payload to all recipients (see {{peer-deliver}}).

#### Adding and Removing Reactions

`update` with `addReaction` or `removeReaction`:

`addReaction` (Object):
: `{"emoji": <String>, "customEmojiId": <String, optional>}`. The server assigns a ULID as both `id` and `senderReactionId`, appends a Reaction with `senderId: "self"` and current time, and returns the new reaction `id` in the response. The server MUST propagate via `Peer/deliver` `reactionUpdate` payload.

`removeReaction` (Object):
: `{"id": <String>}`. Removes the owner's reaction with this local `id`. The server MUST propagate via `Peer/deliver` `reactionUpdate` payload using the reaction's `senderReactionId`.

#### Deleting a Message

`update` with `deletedAt: <timestamp>`.

- If `deletedForAll: true` is also set, the server MUST send `Peer/retract` to all participants before marking the local record. Servers MUST reject `deletedForAll: true` for messages where `senderId` is not `"self"`.
- Otherwise, deletion is local only: `body` and `attachments` are cleared on this mailbox with no peer notification.

#### Marking as Read

`update` with `readAt: <UTCDate>`.

### Message/query

Standard JMAP `/query`.

All requests MUST include a `chatId` filter.

Additional filter properties:

`text` (String, optional):
: Full-text search over `body`. Servers that do not support full-text search MUST return `unsupportedFilter`.

`threadRootId` (String, optional):
: Return only messages in this thread. Valid only when `supportsThreads` is `true`; otherwise servers MUST return `unsupportedFilter`.

`hasAttachment` (Boolean, optional):
: Filter to messages with or without attachments.

`hasMention` (Boolean, optional):
: Filter to messages that mention the owner (owner's Contact.id appears in `mentions`).

Default sort: `receivedAt` ascending.

### Message/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## Space Methods

### Space/get

Standard JMAP `/get`.

### Space/changes

Standard JMAP `/changes`. The Space object is returned in full on each change; the sub-arrays `roles`, `members`, and `categories` are included in every Space object and reflect the current state.

### Space/set {#space-set}

Standard JMAP `/set`.

#### Creating a Space

`create` accepts:

`name` (String, required).

Optional: `description` (String), `iconBlobId` (String).

The server assigns a ULID, sets the caller as the owner with all permissions, and returns the new Space.

#### Updating a Space

`update` supports the following patch keys:

`name`, `description`, `iconBlobId`:
: Metadata fields. Require `"manage_space"` permission.

`addRoles` (Object[]):
: Each entry: `name` (String), `permissions` (String[]), `position` (UnsignedInt), and optionally `color` (String). Server assigns ULIDs. Total MUST NOT exceed `maxRolesPerSpace`. Requires `"manage_roles"`. Members may only add roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this.

`removeRoles` (String[]):
: SpaceRole ids to remove. Members holding only removed roles are demoted to `@everyone`. Requires `"manage_roles"`.

`updateRoles` (Object[]):
: Each entry: `id` (String) and any of `name`, `color`, `permissions`, `position`. Requires `"manage_roles"`. Members may only modify roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this.

`addMembers` (Object[]):
: Each entry: `id` (Contact.id, String) and optional `roleIds` (String[]). Total MUST NOT exceed `maxSpaceMembers`. Requires `"manage_members"`.

`removeMembers` (String[]):
: Contact.ids to remove. Requires `"manage_members"`. The owner cannot be removed.

`updateMembers` (Object[]):
: Each entry: `id` (String) and any of `roleIds`, `nick`. Role changes require `"manage_roles"`.

`addChannels` (Object[]):
: Each entry: `name` (String, required), optional `categoryId` (String), `position` (UnsignedInt), and `topic` (String). The server creates a Chat record of `kind: "channel"` with `spaceId` set to this Space's id and assigns a ULID as the chatId. Total channel count MUST NOT exceed `maxChannelsPerSpace`. Requires `"manage_channels"`.

`removeChannels` (String[]):
: Channel Chat ids to remove. Cascades to all Messages in those channels. Requires `"manage_channels"`.

`updateChannels` (Object[]):
: Each entry: `id` (String, channel Chat id) and any of `name`, `topic`, `categoryId`, `position`, `slowModeSeconds`, `permissionOverrides`. Requires `"manage_channels"`.

`addCategories` (Object[]):
: Each entry: `name` (String), optional `position` (UnsignedInt) and `channelIds` (String[]). Server assigns ULIDs. Total MUST NOT exceed `maxCategoriesPerSpace`. Requires `"manage_channels"`.

`removeCategories` (String[]):
: Category ids to remove. Channels in removed categories move to `uncategorizedChannelIds`. Requires `"manage_channels"`.

`updateCategories` (Object[]):
: Each entry: `id` (String) and any of `name`, `position`, `channelIds`. Requires `"manage_channels"`.

#### Destroying a Space

Cascades to all channel Chats and their Messages. Hard-deletes all records; no tombstones are retained. Requires owner identity.

### Space/query

Standard JMAP `/query`.

Filter properties: `name` (String, substring match).
Default sort: `name` ascending.

### Space/join

Allows a caller to join a Space by redeeming an invite code.

Method name: `Space/join`

Request: `accountId` (String), `inviteCode` (String).

The server MUST resolve `inviteCode` to a SpaceInvite record, verify the invite has not expired and has not reached `maxUses`, and then atomically increment `uses` and add the caller to the Space's member list. The `uses` increment and membership insertion MUST be performed within a single atomic operation so that concurrent redemptions cannot exceed `maxUses`. The caller is assigned no roles beyond `@everyone` unless the invite specifies otherwise.

Response: `accountId` (String), `spaceId` (String).

## SpaceInvite Methods

### SpaceInvite/get

Standard JMAP `/get`. Only members of the Space may retrieve its invites. Members with `"manage_members"` permission see all invites; other members see only invites they created.

### SpaceInvite/set

Standard JMAP `/set`.

`create` accepts: `spaceId` (String, required), `defaultChannelId` (String, optional), `expiresAt` (UTCDate, optional), `maxUses` (UnsignedInt, optional). Requires `"manage_members"` permission.

`destroy` revokes an invite. Requires `"manage_members"` permission or ownership of the invite.

`update` is not supported.

# Thread Model {#threads}

Servers advertising `supportsThreads: true` support structured conversation threads.

A thread is the set of Messages sharing a common `threadRootId`. The root message has `threadRootId` absent.

Thread root assignment rules:

- A message with no `replyTo` is a potential thread root. `threadRootId` MUST be absent.
- A message replying to a thread root (the referenced message has no `threadRootId`): set `threadRootId` to the value of `replyTo`.
- A message replying to a non-root message: set `threadRootId` to the referenced message's `threadRootId`.

Clients MUST follow these rules. Servers SHOULD validate them and MAY correct `threadRootId` if the client supplies an incorrect value.

`Message/query` with a `threadRootId` filter returns all messages in that thread. The `replyCount` field on each message gives the count of direct replies.

# Server-to-Server Methods {#peer-methods}

The following methods are used between mailbox servers only. Callers without the `"peer"` role MUST receive `forbiddenMethod`.

## Peer/deliver {#peer-deliver}

Delivers a new message, an edit, or a reaction update from a remote mailbox. Exactly one of `message`, `edit`, or `reactionUpdate` MUST be present.

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`chatId` (String):
: The Chat ID. For direct chats, if the receiving server already has a chat with this contactId, it MUST verify that `chatId` matches the stored chatId. For group and channel chats, the receiver MUST verify this value matches the chatId of a known chat of which the sender is a current member.

`senderUserId` (String):
: The sender's id (Contact.id / userId). MUST match the identity provided by the authentication layer. The receiver MUST compare these and MUST reject the request if they differ.

`chatKind` (String):
: `"direct"`, `"group"`, or `"channel"`. Informs which chatId verification procedure applies.

`message` (Object, optional):
: A new message to deliver. Fields:

  - `senderMsgId` (String) — Sender-assigned ULID. Idempotency key.
  - `body` (String) — Validated against `maxBodyBytes`.
  - `bodyType` (String) — Validated against `supportedBodyTypes`.
  - `sentAt` (UTCDate) — Stored as-is; not used for ordering.
  - `attachments` (Object[]) — Each carries Attachment fields plus `fetchUrl` (String): the URL from which the receiver fetches the blob.
  - `mentions` (Mention[], optional).
  - `actions` (MessageAction[], optional) — Servers MUST store and forward without inspection.
  - `replyTo` (String, optional) — Sender's own `senderMsgId` of the referenced message. Receiver resolves to local `id` via the `senderMsgId` index.
  - `threadRootId` (String, optional) — Sender's own `senderMsgId` of the thread root. Receiver resolves similarly.
  - `senderExpiresAt` (UTCDate, optional) — Hard-deletion deadline. Servers MUST reject a value that is already in the past at delivery time with `invalidArguments`.
  - `burnOnRead` (Boolean, optional) — Hard-delete on first read.

`edit` (Object, optional):
: An edit to an existing message. Fields:

  - `senderMsgId` (String) — Identifies the message to edit via the `senderMsgId` index.
  - `body` (String) — New body; validated against `maxBodyBytes`.
  - `bodyType` (String) — Validated against `supportedBodyTypes`.
  - `editedAt` (UTCDate) — Claimed edit time; stored as-is.
  - `mentions` (Mention[], optional).

  The receiver MUST verify the sender is the original sender of the identified message before applying the edit.

`reactionUpdate` (Object, optional):
: A reaction change on an existing message. Fields:

  - `senderMsgId` (String) — Identifies the target message.
  - `senderReactionId` (String) — Sender-assigned ULID for this reaction. For `"add"`, the receiver stores this as the reaction's `senderReactionId`. For `"remove"`, identifies which reaction to delete via the `senderReactionId` index.
  - `emoji` (String) — Non-empty reaction value.
  - `customEmojiId` (String, optional) — Space-scoped custom emoji id, if applicable.
  - `action` (String) — `"add"` or `"remove"`.
  - `sentAt` (UTCDate).

### New Message Validation

Before storing a new message, the server MUST in order:

1. Verify caller identity via the authentication layer.
2. Confirm `senderUserId` matches the verified identity.
3. For direct chats: if a chat with this sender already exists, confirm `chatId` matches its stored id; otherwise create a new Chat record with this chatId and contactId. For group and channel chats: confirm `chatId` matches a known chat and the sender is a current member.
4. Confirm the sender is not blocked by the owner.
5. Validate `body` byte length against `maxBodyBytes`. For plaintext `bodyType` values, also validate UTF-8 encoding. For encrypted `bodyType` values (e.g., `"application/mls-ciphertext"`), body is opaque; servers MUST NOT parse or transform it beyond byte-length checking.
6. Validate `bodyType` against `supportedBodyTypes`.
7. Validate each attachment `filename`, `contentType`, and `size`.
8. Fetch each attachment blob from `fetchUrl`; verify byte count against `size` and content against `sha256`.
9. Validate each mention `offset + length` against body byte length.
10. If `senderExpiresAt` is present, confirm it is strictly in the future; reject with `invalidArguments` if not. Schedule hard deletion at that time. If `burnOnRead` is `true`, register a trigger to hard-delete the message when `readAt` is set.

Failure at any step MUST result in rejection with no data stored.

### Edit and Reaction Validation

For `edit` payloads: apply steps 1–4 above, validate `body` and `bodyType` as in steps 5–6, then verify the identified message exists and `senderUserId` matches its recorded sender.

For `reactionUpdate` payloads: apply steps 1–4, validate `emoji` as a non-empty string, verify the identified message exists.

### Response

`accountId` (String), `receivedMsgId` (String — the receiver's assigned ULID for a new message, or the local `id` of the edited or reacted-to message), `receivedAt` (UTCDate).

## Peer/receipt

Notifies the sending mailbox that a message was stored and/or read.

Method name: `Peer/receipt`

Request: `accountId` (String), `senderMsgId` (String — the sender-assigned ULID), `deliveredAt` (UTCDate, optional — time of storage), `readAt` (UTCDate, optional — time the recipient read the message), `readerUserId` (String — the Contact.id of the acknowledging user, for group receipt aggregation).

Response: `accountId` (String).

## Peer/typing

Notifies a remote mailbox that the owner is or is not typing in a chat.

Method name: `Peer/typing`

Request: `accountId` (String), `chatId` (String), `senderUserId` (String, MUST match authenticated identity), `typing` (Boolean).

Response: `accountId` (String).

The receiving server MUST NOT store this event. It MUST forward a `typing` push event (see {{push}}) to the owner's connected clients. Servers MUST rate-limit inbound `Peer/typing` calls per peer.

## Peer/retract

Requests that a remote mailbox tombstone a specific message.

Method name: `Peer/retract`

Request: `accountId` (String), `chatId` (String), `senderUserId` (String, MUST match authenticated identity), `senderMsgId` (String — the sender-assigned ULID of the message to retract).

The receiving server MUST look up the message by `senderMsgId` within `chatId` and MUST verify that the stored message's `senderId` matches `senderUserId` before applying the tombstone. On success, `body` and `attachments` are cleared, `deletedAt` is set to the current time, and `deletedForAll` is set to `true`.

Response: `accountId` (String), `retractedAt` (UTCDate).

## Peer/groupUpdate {#peer-groupupdate}

Notifies participant mailboxes of a new group chat or a membership or metadata change to an existing one.

Method name: `Peer/groupUpdate`

Request arguments:

`accountId` (String), `chatId` (String — the group chat ULID), `senderUserId` (String, MUST match authenticated identity and MUST be an admin of the group on the sending server).

`action` (String):
: One of:
  - `"create"` — Initial group creation. Carries the full initial state.
  - `"addMembers"` — Members were added.
  - `"removeMembers"` — Members were removed.
  - `"updateRoles"` — Member roles changed.
  - `"updateMetadata"` — `name`, `description`, or `avatarBlobId` changed.

`members` (ChatMember[], required for `"create"`):
: Full membership list at the time of this update.

`addedMembers` (ChatMember[], for `"addMembers"`):
: Newly added members.

`removedMemberIds` (String[], for `"removeMembers"`):
: Contact.ids of removed members.

`updatedRoles` (Object[], for `"updateRoles"`):
: Each entry: `id` (String), `role` (String).

`name` (String, optional), `description` (String, optional), `avatarBlobId` (String, optional):
: Updated metadata fields (any combination, for `"updateMetadata"` or `"create"`).

The receiving server MUST verify `senderUserId` is authenticated and is an admin of this group (or, for `"create"`, is the initial creator). On success, the receiving server updates its local Chat record accordingly.

Response: `accountId` (String).

# Push Notifications {#push}

Servers MUST support the EventSource mechanism defined in {{RFC8620}} Section 7.3.

## State-Change Events

~~~
event: state
data: {"changed":{"<accountId>":{"Message":"<s>","Chat":"<s>","Contact":"<s>"}}}
~~~

Clients SHOULD call the corresponding `/changes` method upon receipt. On `cannotCalculateChanges`, fall back to `/get`.

## Typing Events

~~~
event: typing
data: {"chatId":"<id>","senderId":"<contact-id>","typing":<bool>}
~~~

Not stored; carries no state token.

## Presence Events

~~~
event: presence
data: {"contactId":"<id>","presence":"<state>","lastActiveAt":"<ts>"}
~~~

# Blob Storage {#blobs}

Standard JMAP blob upload and download per {{RFC8620}} Section 6, using the `uploadUrl` and `downloadUrl` Session templates.

## Upload

`POST <uploadUrl>` with the blob as the request body.

Response (HTTP 200) — this document extends the standard RFC 8620 upload response with the `sha256` field:

~~~json
{
  "blobId":  "<id>",
  "type":    "<mime-type>",
  "size":    <bytes>,
  "sha256":  "<lowercase-hex>"
}
~~~

## Download

`GET <downloadUrl>` with placeholders percent-encoded.

# Outbox and Delivery {#outbox}

Outbound messages MUST be queued in a persistent outbox before the first delivery attempt. Servers MUST retry with exponential backoff.

For group chats, the sender delivers independently to each participant's mailbox and tracks per-recipient state in `deliveryReceipts`. The aggregate `deliveryState` advances to `"delivered"` when all participants have acknowledged.

A message whose `senderMsgId` is already known for the given chat MAY be silently discarded by the receiver.

# Authorization {#authorization}

Authentication is handled at the transport layer as described in {{introduction}}. The protocol derives access control from the stable, opaque id provided per connection:

- **Owner** (identity equals the mailbox owner's id): all methods.
- **Peer** (identity equals a known contact's id): `Peer/deliver`, `Peer/receipt`, `Peer/typing`, `Peer/retract`, and `Peer/groupUpdate` only.
- **Other**: HTTP 401.

# Space Permission Resolution {#space-permissions}

When determining whether a member may perform an action in a channel Chat, servers MUST evaluate permissions in this order:

1. The Space owner always has all permissions; skip remaining steps.
2. Compute the union of `permissions` across all SpaceRoles held by the member, including the implicit `@everyone` role.
3. Apply `deny` entries from any ChannelPermission records matching the member's roles (`targetType: "role"`), in ascending position order.
4. Apply `allow` entries from the same role-targeted records, in ascending position order.
5. Apply `deny` entries from any ChannelPermission record matching the member directly (`targetType: "member"`).
6. Apply `allow` entries from the same member-targeted record.

Servers MUST perform this resolution server-side. Clients MUST NOT be trusted to assert their own permissions.

Role hierarchy enforcement: members may only create or modify SpaceRoles whose `position` is strictly less than their own highest-position role. Servers MUST reject role management operations that violate this constraint.

# Security Considerations {#security}

## Identity Verification

`senderUserId` in all Peer/* methods is caller-supplied and MUST be treated as untrusted. The server MUST obtain the verified identity from its own authentication layer independently and MUST compare before any storage or action. Verification MUST precede all effects.

## Input Validation

All peer-supplied fields are attacker-controlled. Servers MUST validate:

- `body`: validate UTF-8 for plaintext body types; enforce `maxBodyBytes`.
- `bodyType`: validate against `supportedBodyTypes`.
- `filename`: reject values containing `/`, `\`, or null bytes.
- `contentType`: reject syntactically invalid MIME values.
- `size`: verify against actual blob byte count after fetch.
- `sha256`: verify against actual blob content after fetch.
- `sentAt`, `editedAt`: store as-is; never use for ordering or expiry.
- `chatId`: verify per {{chat-id}} — confirm match with stored id or verify membership; reject mismatches.
- `emoji`: validate as a non-empty string; enforce an implementation-defined maximum byte length to prevent denial of service.
- `mentions`: reject any entry where `offset + length` exceeds body byte length.

## Blob Fetch and SSRF {#blob-fetch-and-ssrf}

When fetching attachment blobs from peer-supplied `fetchUrl` values, servers MUST restrict outbound connections to the known peer address space. Unrestricted fetches are an SSRF vector.

## Denial of Service

Enforce `maxBodyBytes` and `maxAttachmentBytes` at parse time, before any fetch or storage. Enforce `maxAttachmentsPerMessage`, `maxGroupMembers`, and `maxSpaceMembers` at creation and update time. Rate-limit `Peer/typing` per peer.

## Chat ID Integrity

Chat IDs are server-assigned ULIDs. Security against cross-conversation injection relies on sender authentication and chat membership verification, not on ID derivation.

For direct chats, the receiving server MUST confirm that the incoming chatId matches the chatId already associated with the sending contact (if one exists). This prevents a sender from injecting messages into a chat ID that belongs to a different conversation.

For group and channel chats, servers MUST confirm the sender is a current member of the identified chat before accepting any `Peer/deliver` or `Peer/groupUpdate`.

## Retract Authorization

`Peer/retract` MUST verify via the `senderMsgId` index that `senderUserId` matches the original sender before applying any tombstone.

## Group Admin Verification

`Peer/groupUpdate` MUST verify that `senderUserId` holds an admin role in the named group on the receiving server's local record before applying membership or metadata changes.

## Direct Address Hints

`directAddress` and `ownerDirectAddress` are peer-supplied values and MUST be treated as untrusted. Implementations that use them MUST apply the same authentication requirements to the direct path as to the mailbox path, and MUST NOT use them as fetch targets in contexts analogous to the blob-fetch SSRF risk described in {{blob-fetch-and-ssrf}}. Senders MUST NOT treat delivery to a `directAddress` as a substitute for mailbox delivery; the mailbox path MUST still be used to ensure message persistence and multi-device visibility.

## Blocked Contacts

Messages from a contact whose `blocked` field is `true` are silently dropped regardless of whether they arrive in a direct chat or a group chat context.

## Out-of-Band Endpoints and Actions

`Contact.endpoints`, `Session.ownerEndpoints`, and `Message.actions` carry peer-supplied URIs and MUST be treated as untrusted at every level:

- Clients MUST NOT fetch or connect to any OOB URI automatically. All OOB interactions require explicit user initiation.
- Payment URIs (`urn:jmap:chat:cap:payment`) MUST be validated by the client wallet before any funds are transferred. Servers MUST NOT inspect or act on payment URI values.
- VTC URIs (`urn:jmap:chat:cap:vtc`) MUST NOT be opened without user consent; auto-joining a call is a privacy violation.
- Blob/file URIs (`urn:jmap:chat:cap:blob`) used for OOB fetch are subject to the same SSRF constraints as attachment `fetchUrl` values (see {{blob-fetch-and-ssrf}}); servers that fetch from peer-supplied blob endpoints MUST restrict connections to the known peer address space.
- `metadata` values are peer-supplied and MUST be ignored if they do not conform to the expected shape for the known `type`.
- Unknown `type` URIs MUST be silently ignored by both clients and servers.

## End-to-End Encrypted Deployments {#e2ee}

In relay deployments, the relay routes Peer/* messages but MUST NOT have access to plaintext message content. Implementations MUST ensure:

- The `body` field carries ciphertext only; plaintext MUST never be transmitted to the relay in an encrypted deployment.
- The relay is architecturally excluded from the encryption key schedule (e.g., by using MLS {{RFC9420}} or a similar protocol that does not involve the relay in key agreement).
- Servers MUST NOT reject or transform `body` based on content when `bodyType` indicates an encrypted type.
- Metadata visible to the relay — sender id, recipient id, timestamp, and body size — remains an information-leakage surface. Deployments requiring metadata privacy SHOULD apply message padding and cover traffic at the transport layer; those techniques are outside the scope of this document.

# IANA Considerations

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:jmap:chat:1`

Intended Use:
: common

Change Controller:
: Mark Atwood (mark@reviewcommit.com)

Reference:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.

Note: this URI is provisional. If this specification is adopted by the IETF JMAP working group, it will be updated to `urn:ietf:params:jmap:chat`.

--- back

# Acknowledgements

The author thanks the JMAP working group for {{RFC8620}}.
