# The Skynet Protocol

## Challenges ##
The Skynet Protocol v4 had many problems that were experienced after it was designed. Solving these problems is the main goal for Protocol v5. Creating a stable and robust messenger is the first aim, security is the second aim and performance is on the third place.

These points will have to change to achieve this:
### 1. Concurrency ###
The UserData will get a revision control system that detects concurrent changes and forces clients to merge data sets before uploading new data.
### 2. Timing independency ###
Checking for updates using a single DateTime is very data efficient, but it is not guaranteed that you will receive all pending messages. Changes are very difficult to track like this and server development gets increasingly difficult.
The new revision-based system will need to transfer more data but it will be simpler to implement, more versatile and guarantees data integrity. Message edits and deletions will be treated like new messages. The most difficult thing will be the `ViewableContentInfo`. Sending one packet per client and message would result in an inefficient system causing hundreds of separate packets.
### 3. Transactions and retry ###
Accepting a contact request is a complex process requiring many steps. The new protocol has to ensure that changes are only written to the database once all steps succeeded. If this is not the case, the client must be able to retry the action. Therefore changes, especially those done to the UserData must be linked to other packets.
### 4. Cryptography reloaded ###
Although Skynet should be designed to be more secure than other messengers, Protocol v4 was not able to use end-to-end encryption for profile data related content such as profile images or the daystream. Furthermore, we have to check each situation for whether a server-sided validation is enough or we should use a signature algorithm.
Skynet will not offer Perfect Forward Secrecy because we want to keep cloud synchronization and data backup as our main features, which would not be possible with PFS.
### 5. Sync on demand ###
Downloading all messages on startup wastes a lot of expensive data volume, slows down the client and makes a web integration completely impossible.
A new revision-based system should allow sync on demand.
### 6. Permissions ###
On the current server, anyone can request any file. Although, with the new system, most files will be encrypted and the file IDs are random which makes it more difficult to get a plaintext file, we need to implement permissions. Files will be pinned for single users or groups and unpinned once the associated messages or post was deleted. A cleanup service can then safely delete these files. The server must ensure that only clients with access to a file can share it.
### 7. Structuring ###
- Inconsistent names and ids.
- Echo packets.
- Many packets that don't fulfill their purpose very well.

These problems have to be fixed with Protocol v5

## Concept ##

### Handshake ###
Each connection begins with a handshake. This allows us to inform the user about new application versions, or even enforce an update. Application versions in VSL must only be incremented once the layout of the handshake packet changes. In Skynet however, we will treat each platform differently so that new versions can be released asynchronously for different platforms.

### Device Registration ###
Using Firebase Cloud Messaging (FCM) requires the server to keep track of the registered devices.
Furthermore, tracking devices brings two more important advantages: Devices that were not connected for more than half a year can be forced to reauthenticate. This also allows for other features such as two-factor authentication.

### Channels ###
With the Skynet protocol v4 we had different implementations for many types of communication channels. This concept made abstraction almost impossible and increased the complexity, especially on the server side.

To avoid this problem, we introduce a completely new concept with Platinum Stack: Every communication channel will now be treated as a `Channel` with a unique `ChannelId`.
The server does not care which messages are sent over a channel. Data is sent over a `Channel` as a `Message` with a globally unique incremental ID. Each channel type provides its own mechanism for key exchange.  

##### SkipCount #####
As messages are delivered asynchronously, it might happen that they arrive in a different order than their message IDs. Imagine the following situation in a group with Alice, Bob and Charlie:

1. The last message in this channel has `MessageId = 3`
2. Bob and Charlie send a message at almost the same time.
3. Bob's message has `MessageId = 5`
4. Charlie's message has `MessageId = 6`
5. Charlie's message is delivered to Alice with `SkipCount = 0`
6. Bob's message is delivered to Alice with `SkipCount = 1`

If Alice would lose her connection after step 5, she would never receive Bob's message because her `LastMessageId` is `6`.
Therefore, the server sends a `SkipCount` indicating how many message IDs have been skipped regarding all her channels. With this information Alice can notice that she is missing a message.
If she doesn't receive this message at the next session restore, her client has to request it from the server.

##### MessageFlags #####
`MessageFlags.Loopback` indicates that a message is to be delivered only to the other devices of the sender of this message. Messages with the loopback flag are encrypted using the key of the users _loopback channel_.  
Using a loopback flag solves many problems but makes the classical revision system of all clients having the same data impossible.
Therefore, a client which decides not to synchronize all messages has to request them later from the server which may lead to duplicate messages that have to be ignored.

`MessageFlags.Unencrypted` marks a message that is not encrypted and should be processed by the server. Data such as blocked contacts and conversations is managed using the loopback channel with this flag.  

##### Loopback Channel #####
This is a special channel whose only member is the creating user. It acts as kind of a clipboard which is sued to synchronize data across devices and replaces the old `UserData` packet, which was prone to concurrency and timing errors. Its key is directly derived from the user's password.

##### Account Data Channel #####
This channel contains all public account information like the public keys to avoid individual forwarding to direct channels. All messages have to be sent with `MessageFlags.Unencrypted`. The server automatically adds all contacts to the user's account data channel.

##### Direct Channel #####
The direct channel is used for direct chat between two accounts. There is no longer a _contact request_. If Alice knows the `AccountId` of Bob's account, which she can look up using the search packets, she can simply create a direct channel. The server will automatically inject the public keys for both sides. Once Alice has received Bob's public key, she can derive and cache the channel key. When the server has sent a _DirectChannelUpdate_ packet, Alice can start sending chat messages to Bob.

##### Group Channel #####
Many other scenarios are realized with a _group channel_. This kind of channel is not limited to classical groups, but is also used for sharing profile related data. The key of a group channel is exchanged via the direct channel between the group admin and each member.

##### Dependencies #####
In order to manage key changes and being able to delete messages when they're not needed anymore, messages can have dependencies to multiple messages on the same or a different channel. Encrypted messages with `MessageFlags.Loopback` must reference one `0x15 PasswordUpdate`, messages in direct channels must reference one `0x1B DirectChannelUpdate` and messages in group channels must reference one `0x1E GroupChannelUpdate`. A special case is the key change of a group channel, which has one dependency per account. Additionally, this message needs _MessageFlags.NoSenderSync_ to indicate that it's not synchronized across the sender's devices.

##### Profile data #####
Profile data can either be shared publicly via the account's account data channel with _MessageFlags.Unencrypted_ or via a profile data channel, which technically inherit from the group channel. To manage permissions, multiple profile data channels containing different members can be created. The account data channel is unique per account and managed by the server.

### User files ###
The files attached to media messages are encrypted using `AES-GCM-256` and then uploaded to file transfer servers over HTTPS (see [features.md](features.md)).  
Authenticated clients can always upload files, and get an Int64 `FileId` from the server. The owner of a file is allowed to reference it in a message. After that, all users in the same channel are allowed to reference it in other channels.
This saves a lot of bandwidth when files are forwarded. Clients are encouraged to search for an identical file before uploading as users might want to share the same file at different times to different users.

When a file is referenced for the first time in a channel, all of its members are granted permission to share the file. Files without any permissions are deleted by the garbage collector after a certain time.
When the last message referencing a file is deleted in a channel, the channel members lose their permission to share this file. Once a file that was previously referenced is not accessible to any users anymore, it is instantly deleted by the garbage collector.  
If a client attempts to reference a file that does not exist or that it has no permission for, the client will know this from the status code in the `ChannelMessageResponse` packet.

### Cryptography ###
The security of Skynet is based on the user's password:
1. Password (UTF-8)
2. `AccountKey` (first 32 bytes for HMAC, last 32 bytes for AES)
3. `KeyHash` (32 bytes for authentication)

The `AesKey` is derived from the user's password with _Argon2id_ using the `SHA256(AccountName)` as salt:
```
KeyMaterial = Argon2(P: Password, S: SHA256(AccountName), p: 8, T: 96, m: 8192, t: 2, v: 13, y: 2);
AccountKey = KeyMaterial.Slice(s: 0, l: 64);
KeyHash = KeyMaterial.Slice(s: 64, l: 32);
```

## The Protocol ##
The protocol section consists of packets, which each have a name, an ID and the specification written in `VPSL`.

### ChannelMessage ###
The ChannelMessage used to be the most important packet of the new channel-based protocol.
Now it is no independent packet anymore but rather a specific type of packet. All channel messages (marked with `@message`) inherit its class and behavior.  
Use `AccountId=0` to indicate a global dependency. The initial value for the version number is `1`.  
During the session restore the server will send `SkipCount = -1` which means the client does not have to check that no message is missing.
```vpsl
<Byte PacketVersion><Int64 ChannelId>
((ToClient)<Int64 SenderId>)<Int64 MessageId>
((ToClient)<Int64 SkipCount><DateTime DispatchTime>)
<MessageFlags:Byte MessageFlags>
((MessageFlags.ExternalFile)<Int64 FileId>)
[AesHmac/None:ByteArray PacketContent
    ...
    ((MessageFlags.MediaMessage)<File File>)
]
{UInt16 Dependencies
    <Int64 AccountId><Int64 MessageId>
}
```
```csharp
[Flags]
enum MessageFlags {
    None = 0,
    Loopback = 1,
    Unencrypted = 2,
    NoSenderSync = 4,
    MediaMessage = 8,
    ExternalFile = 16
}
```
```vspl
<ShortString Name>
<DateTime CreationTime>
<DateTime LastWriteTime>
<ShortString ThumbnailContentType>
<ByteArray ThumbnailData>
((MessageFlags.ExternalFile)
    <ShortString ContentType>
    <Int64 Length>
    <Byte[32] Key>
)
```

### **0x00** ConnectionHandshake ![networkUp] ###
This packet is sent before any other packet to ensure that the client is using the latest version.
`ProtocolVersion` is necessary to catch breaking changes and enables the server to ignore indev versions with the same protocol version.
```vpsl
<Int32 ProtocolVersion><ShortString ApplicationIdentifier><Int32 VersionCode>
```
The application identifier consists of the platform identifier and the package/assembly name e.g. `android/de.vectordata.skynet`.

### **0x01** ConnectionResponse ![networkDown] ###
This is the server's response to _0x00 ConnectionHandshake_. This packet finishes the handshake and leads the way to register, login or restore a session.

```vpsl
<ConnectionState:Byte ConnectionState>
((ConnectionState != Valid)<Int32 LatestVersionCode><ShortString LatestVersion>)
```
```csharp
enum ConnectionState {
    Valid,
    CanUpgrade,
    MustUpgrade
}
```

### **0x02** CreateAccount ![networkUp] ###
This packet is sent when the user registers a new account. After e-mail address verification is completed, the server has to create the loopback channel and send a _PasswordUpdate_ packet as base revision. For further account initialization see _SyncFinished_. If this packet is sent referencing an unconfirmed account with the correct password, the confirmation mail is sent again.
```vpsl
<ShortString AccountName><Byte[32] KeyHash>
```

### **0x03** CreateAccountResponse ![networkDown] ###
```vpsl
<CreateAccountStatus:Byte StatusCode>
```
```csharp
enum CreateAccountStatus {
    Success,
    MailResent,
    AccountNameTaken,
    InvalidAccountName
}
```

### **0x04** DeleteAccount ![networkUp] ###
This packet is sent by a client to delete the user's account.
The server will delete only delete personal data that has not been sent to other users. This includes:
- Mail confirmations
- Loopback channel
- Loopback messages
- Blocked accounts
- Blocked conversations
- Account data channel memberships
- Group memberships
- Profile data channel memberships

Furthermore, all direct channels have to be archived and other accounts prevented from opening new channels.

All other messages are kept until they expire or a counterpart's account is deleted, too.
```vpsl
<Byte[32] KeyHash>
```

### **0x05** DeleteAccountResponse ![networkDown] ###
After receiving this packet with `DeleteAccountStatus.Success`, the server closes the connection.
```vpsl
<DeleteAccountStatus:Byte StatusCode>
```
```csharp
enum DeleteAccountStatus {
    Success,
    InvalidCredentials
}
```

### **0x06** CreateSession ![networkUp] ###
This packet is sent by the client after opening a connection. The `FcmRegistrationToken` can be left empty if it is not necessary (e.g. for a desktop client). The Skynet server will send a Firebase message if a _ChatMessage_ could not be delivered directly to a client. Client applications have to reconnect until they receive a _SyncFinished_ packet. A newly created session does not need to be restored manually.
```vpsl
<ShortString AccountName><Byte[32] KeyHash><String FcmRegistrationToken>
```

### **0x07** CreateSessionResponse ![networkDown] ###
```vpsl
<CreateSessionStatus:Byte StatusCode>
<Int64 AccountId><Int64 SessionId>
<Byte[32] SessionToken><String WebToken>
```
```csharp
enum CreateSessionStatus {
    Success,
    InvalidCredentials,
    InvalidFcmRegistrationToken,
    UnconfirmedAccount
}
```

### **0x08** RestoreSession ![networkUp] ###
After initializing a connection, the client can restore a session with this packet. The client sends the server its last message and all known channels. There can only be one client connected to the server with the same _SessionId_. Conflicting clients are kicked on automatically.
```vpsl
<Int64 SessionId><Byte[32] SessionToken>
<Int64 LastMessageId>
{UInt16 Channels
    <Int64 ChannelId>
}
```

### **0x09** RestoreSessionResponse ![networkDown] ###
```vpsl
<RestoreSessionStatus:Byte StatusCode>
```
```csharp
enum RestoreSessionStatus {
    Success,
    InvalidSession
}
```

### **0x0A** CreateChannel ![networkDuplex] ###
Sent by the server to a client to create a channel. If it is sent by the client with a random `ChannelId` the server assigns one in the _CreateChannelResponse_ packet.
```vpsl
<Int64 ChannelId><ChannelType:Byte ChannelType>
((ToClient)<Int64 OwnerId><DateTime CreationTime>)
((ChannelType.Direct)<Int64 CounterpartId>)
```
```csharp
enum ChannelType {
    Loopback,
    AccountData,
    Direct,
    Group,
    ProfileData
}
```

### **0x2F** CreateChannelResponse ![networkDown] ###
The server responds to a _CreateChannel_ packet with this packet. It tells the client about the status and the `ChannelId`.
```vpsl
<Int64 TempChannelId><ChannelCreateStatus:Byte StatusCode>
<Int64 ChannelId><DateTime CreationTime>
```
```csharp
enum ChannelCreateStatus {
    Success,
    AlreadyExists,
    InvalidCounterpart,
    Blocked
}
```

### **0x0D** DeleteChannel ![networkDown] ###
Sent by the server to notify a client that a channel has been deleted permanently from the server. The client should not include this channel in _RestoreSession_ packets anymore, but can keep the data locally.
```vpsl
<Int64 ChannelId>
```

### **0x0C** ChannelMessageResponse ![networkDown] ###
If the client wants to send a new message, it chooses a random negative `MessageId`. The server responds with this packet and assigns a new `MessageId`. The `DispatchTime` specifies when the server has processed this message.
```vpsl
<Int64 ChannelId><Int64 TempMessageId>
<MessageSendStatus:Byte StatusCode>
<Int64 MessageId><Int64 SkipCount>
<DateTime DispatchTime>
```
```csharp
enum MessageSendStatus {
    Success,
    FileNotFound,
    AccessDenied,
    ConcurrentChanges
}
```

### **0x0E** RequestMessages ![networkUp] ###
This packet is sent by the client to request channel messages that have not been synchronized yet.

Use `ChannelId=0` to get messages from arbitrary channels.  
Setting `After` or `Before` to `0` disables the boundry check.

Just like the session restore, this action starts a sync process with _SyncStarted_ and _SyncFinshed_ packets.
It is likely that the client will receive messages from the server that it already has. These messages have to be ignored.
```vpsl
<Int64 ChannelId>
<Int64 After><Int64 Before>
<UInt16 MaxCount>
```

### **0x0B** SyncStarted ![networkDown] ###
This packet is sent by the server to a client to indicate that a sync process has started.
`MinCount` specifies how many channel messages are going to be sent at least. A value of `-1` means that the server did not check the message count in advance. Clients must not consider a sync process to be complete before they receive the respective _SyncFinished_ packet.
```vpsl
<Int32 MinCount>
```

### **0x0F** SyncFinished ![networkDown] ###
This packet is sent by the server to a client to indicate that all messages have been delivered.
If the client has a _WakeLock_ or something like this to prevent the user's device from sleeping, it can be released after receiving this message.
```vpsl
// No additional content
```
If a client receives this packet after session creation or restore, it should ensure that the following tasks have been completed or execute them in this order:
1. PrivateKeys
2. PublicKeys
3. Nickname
4. Bio

---
### Channel messages ###
### **0x13** QueueMailAddressChange ![networkDuplex] ###
This packet is sent by the client to its loopback channel using `MessageFlags.Unencrypted`. A message is sent to the old mail address to notify that it is about to be replaced.
```vpsl
@message Loopback Unencrypted
<ShortString NewMailAddress>
```

### **0x14** MailAddress ![networkDown] ###
When the server has verified the client's new mail address, it sends this packet to the client's account data channel.
If the new mail address is not verified within 24 hours, the address change is cancelled.
```vpsl
@message Unencrypted
<ShortString MailAddress>
```

### **0x15** PasswordUpdate ![networkDuplex] ###
This packet is sent to the loopback channel with `MessageFlags.Unencrypted`. It informs the server and other devices of a password change and acts as a base revision for packets with `MessageFlags.Loopback`.
The previous key is stored encrypted in the `KeyHistory` encrypted with the new key. New devices need this information to decrypt messages that were encrypted with previous passwords.
This packet is generated by the server for new accounts with zero bytes as `PreviousKeyHash` and an empty `KeyHistory` field.
```vpsl
@message Loopback Unencrypted
<Byte[32] PreviousKeyHash><Byte[32] KeyHash>
[AesHmac:ByteArray KeyHistory
    <Byte[64] PreviousKey>
]
```

### **0x17** PrivateKeys ![networkDuplex] ###
This packet is sent by the client to its loopback channel. It has no direct effect and is only used as a dependency.
```vpsl
@message Loopback
<KeyFormat:Byte SignatureKeyFormat><ByteArray SignatureKey>
<KeyFormat:Byte DerivationKeyFormat><ByteArray DerivationKey>
```
```csharp 
enum KeyFormat {
    BouncyCastle = 0,
}
```

### **0x18** PublicKeys ![networkDuplex] ###
This packet is sent by the client to its account data channel with a mandatory reference to the matching private key as a dependency.
```vpsl
@message Unencrypted
@dependency PrivateKeys // only to server
<KeyFormat:Byte SignatureKeyFormat><ByteArray SignatureKey>
<KeyFormat:Byte DerivationKeyFormat><ByteArray DerivationKey>
```

### **0x1A** VerifiedKeys ![networkDuplex] ###
This packet is optional and can be used for a QR-code based verification. It saves the SHA-256 over the counterpart's entire _PublicKeys_ packet and is sent by the client to the associated direct channel with `MessageFlags.Loopback`.
```vpsl
@message Loopback
<Byte[32] Sha256>
```

### **0x1B** DirectChannelUpdate ![networkDown] ###
This packet is injected by the server and acts as base dependency for following channel messages in direct channels.
It references a keypair for two perspectives and thereby two `PrivateKeys` packets and two `PublicKeys` packets.
```vpsl
@message Unencrypted
@dependency PrivateKeys & PublicKeys // Alice's private and Bob's public key, specific for Alice's account
@dependency PrivateKeys & PublicKeys // Bob's private and Alice's public key, specific for Bob's account
// No additional content
```

### **0x1C** DirectChannelCustomization ![networkDuplex] ###
This packet is sent by the client to the associated direct channel with `MessageFlags.Loopback`.
```vpsl
@message Loopback
<ShortString CustomNickname><ImageShape:Byte ProfileImageShape>
```
```csharp
enum ImageShape {
    Square,
    Squircle,
    Circle,
    Star,
    Heart
}
```

### **0x1D** GroupChannelKeyNotify ![networkDuplex] ###
This packet is sent over a direct channel from a group admin to all clients. Because the admin's devices do not need all of these messages, the client should use `MessageFlags.NoSenderSync`. This packet notifies all members about a group channel key change, which becomes active when an administrator client sends a _GroupChannelUpdate_ packet.
```vpsl
@message NoSenderSync
<Int64 GroupChannelId><Byte[64] NewKey><Byte[64] HistoryKey>
```

### **0x1E** GroupChannelUpdate ![networkDuplex] ###
To change the group channel key, the admin sends an update to each client in the group via direct channels. To do so, the client sends this packet with `MessageFlags.Unencrypted` and a dependency referencing all direct channel messages. The incremental `GroupRevision` counter is checked by the server and ensures that no concurrent changes are made. Following messages depend on this packet to specify the cryptographic key they use. When used to share profile data, all members should be invisible. The encrypted content uses the `HistoryKey` from the _GroupChannelKeyNotify_ packet if supplied to the matching client. With this key, the client can resolve all former channel and history keys.
```vpsl
@message Unencrypted
@dependency GroupChannelKeyNotify // per account
<Int64 GroupRevision>
{UInt16 Members
    <Int64 AccountId><GroupMemberFlags:Byte Flags>
}
[AesHmac:ByteArray KeyHistory
    <Byte[64] ChannelKey><Byte[64] HistoryKey>
]
```
```csharp
[Flags]
enum GroupMemberFlags {
    None = 0,
    Administrator = 1,
    NoContent = 2,     // Prohibit sending of chat messages
    NoMetadata = 4,    // Prohibit sending of receive and read confirmations
    Invisible = 8      // User is invisible for other group members
}
```

### **0x19** ArchiveChannel ![networkDuplex] ###
This packet changes a channel's archive mode which is necessary to avoid sending messages to a deleted or blocked channel.
Only direct channels can be blocked which is done by the server. In contrast to deleted channels, clients can still send messages with `MessageFlags.Loopback` like _DirectChannelCustomization_ as well as _GroupChannelKeyNotify_ packets over blocked channels.
Direct channels can only be deleted by deleting one of the accounts involved, whereas group channels can be deleted by their administrators by sending this packet with `ArchiveMode.Deleted`.
```vpsl
@message Unencrypted
<ArchiveMode:Byte ArchiveMode>
```
```csharp
enum ArchiveMode {
    None,
    Blocked,
    Deleted
}
```

### **0x20** ChatMessage ![networkDuplex] ### 
The client sends a ChatMessage packet to one specific conversation channel.
Please look at the documentation of _ChannelMessage_ to see how to send media messages.
Chat messages are always encrypted using the conversation key.
A message can always have a text and can always quote another message, regardless of the message type. For example, the user could quote a text message and answer with a voice message or send a contact and write some text in the same message.
If no message is quoted, set `QuotedMessage` to `0`. Even if a message is quoted, there is only the dependency to a channel base revision.
```vpsl
@message [MediaMessage] [ExternalFile]
<MessageType:Byte MessageType><String Text><Int64 QuotedMessage>
```
```csharp
enum MessageType {
    Plaintext,
    Image,
    Video,
    File,
    Location,
    Audio,
    Contact
}
```

### **0x21** MessageOverride ![networkDuplex] ###
To edit or delete a message or a daystream entry, the client sends a MessageOverride packet to the respective conversation channel with a dependency to the respective _ChatMessage_ or _DaystreamMessage_.
```vpsl
@message
@dependency ChatMessage | DaystreamMessage
<OverrideAction:Byte Action>((OverrideAction == Edit)<String NewText>)
```
```csharp
enum OverrideAction {
    Edit,
    Delete
}
```

### **0x22** MessageReceived ![networkDuplex] ###
The client sends this packet when it has received a chat message or a daystream to the respective channel with `MessageFlags.Unencrypted`. The `MessageId` is taken from the dependency (no dependency on _ChannelKeyChange_).
```vpsl
@message Unencrypted
@dependency ChatMessage | DaystreamMessage
// No additional content
```

### **0x23** MessageRead ![networkDuplex] ###
The client sends this packet when the user has viewed a chat message or a daystream to the respective channel. The received confirmation (`0x22`) can omitted if the read confirmation is sent directly. Handling is the same as `0x22 MessageReceived`.
```vpsl
@message Unencrypted
@dependency ChatMessage | DaystreamMessage
// No additional content
```

### **0x24** DaystreamMessage ![networkDuplex] ###
The client sends the DaystreamMessage packet to any profile data channel. It is always encrypted, since daystream posts are only visible to contacts. 
```vpsl
@message [MediaMessage] [ExternalFile]
<MessageType:Byte MessageType><String Text>
```

### **0x25** Nickname ![networkDuplex] ###
The Nickname packet is sent by a client to any profile data channel. If sent with `MessageFlags.Unencrypted`, the nickname is publicly visible. Nickname packets always use `PersistenceMode.KeepLast`. 
```vpsl
@message [Unencrypted]
<ShortString Nickname>
```

### **0x26** Bio ![networkDuplex] ###
The Bio packet is sent by a client to any profile data channel. If it is sent with `MessageFlags.Unencrypted`, the profile image is publicly visible. Bio packets always use `PersistenceMode.KeepLast`. 
```vpsl
@message [Unencrypted]
<String Bio>
```

### **0x27** ProfileImage ![networkDuplex] ###
Similar to Bio packet. This packet uses the `FileId` of its container.
```vpsl
@message [Unencrypted] [MediaMessage] [ExternalFile]
<String Caption>
```

### **0x28** BlockList ![networkDuplex] ###
The blocked content packet is sent by a client to its loopback channel with `MessageFlags.Unencrypted` and `PersistenceMode.KeepLast`.
The server will then filter contact requests or group invites.
```vpsl
@message Loopback Unencrypted
{UInt16 BlockedAccounts
    <Int64 AccountId>
}
{UInt16 BlockedConversations
    <Int64 ChannelId>
}
```

### **0x29** DeviceList ![networkDown] ###
The device list packet is sent by the server to the client's loopback channel with `MessageFlags.Unencrypted` and `PersistenceMode.KeepLast`.
With this data the client can inform the user after a new login.
```vpsl
@message Loopback Unencrypted
{UInt16 Sessions
    <Int64 SessionId><DateTime CreationTime><ShortString ApplicationIdentifier>
}
```

### **0x2A** BackgroundImage ![networkDuplex] ###
This packet is used to change the private background image in a specific conversation. It is sent with `MessageFlags.Loopback` to the affected channel. To set a default background image, send this packet to the loopback channel. For file uploading and selecting, use the file id of the wrapping channel message.
```vpsl
@message Loopback [MediaMessage] [ExternalFile]
// No additional content
```

### **0x2B** OnlineState ![networkDown] ###
This packet is injected by the server with `MessageFlags.Unencrypted` and `MessageFlags.NoSenderSync` in the user's account data channel. Later versions will have different channels to handle permissions.
```vpsl
@message Unencrypted NoSenderSync
<OnlineState:Byte OnlineState>((OnlineState.Inactive)<DateTime LastActive>)
```
```csharp
// Might be extended to a bitmask later
enum OnlineState {
    Inactive,
    Active
}
```
---
### Online state ###
### **0x2C** ChannelAction ![networkDown] ###
The server sends this packet to all channel members where a client starts or stops typing, audio recording, etc.
```vpsl
<Int64 ChannelId><Int64 AccountId><ChannelAction:Byte Action>
```
```csharp
enum ChannelAction {
    None,
    Typing,
    RecordingAudio
}
```

### **0x34** SetClientState ![networkUp] ###
This packet informs the server about the current client state.
```vpsl
<OnlineState:Byte OnlineState><ChannelAction:Byte Action>((!ChannelAction.None)<Int64 ChannelId>)
```
---
### On demand packets ###
### **0x2D** SearchAccount ![networkUp] ###
Clients can use this packet to search for accounts in order to start a new conversation. The query may include the mail address or the public nickname of the sought account.
```vpsl
<ShortString Query>
```

### **0x2E** SearchAccountResponse ![networkDown] ###
This packet contains all results of a _SearchAccount_ query. The server starts a sync to forward public profile data of every account asynchronously. These channel messages are forwarded with `ChannelId = 0` and their original `OwnerId`.
```vpsl
{UInt16 Results
    <Int64 AccountId><ShortString AccountName>
}
```

### **0x32** DeviceListRequest ![networkUp] ###
Sent by the client to request device list details.
```vpsl
// No additional content
```

### **0x33** DeviceListResponse ![networkDown] ###
This packet contributes real time data to the general _DeviceList_ packet.
```vpsl
{UInt16 SessionDetails
    <Int64 SessionId><DateTime LastConnected><Int32 LastVersionCode>
}
```

---
### Unassigned Packet IDs
- **0x10** RealTimeMessage
- **0x11** SubscribeChannel
- **0x12** UnsubscribeChannel
- **0x16** LoopbackKeyNotify
- **0x1F** ProfileDataChannelUpdate
- **0x30** FileUpload
- **0x31** FileUploadResponse


[networkUp]: https://lerchen.net/skynet/static/network-up-36px.png "Only client to server"
[networkDown]: https://lerchen.net/skynet/static/network-down-36px.png "Only server to client"
[networkDuplex]: https://lerchen.net/skynet/static/network-duplex-36px.png "Both client to server and server to client"
