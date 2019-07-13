# The Skynet Protocol

## Challenges ##
The Skynet Protocol v4 had many problems that were experienced after it was designed. Solving these problems is the main goal for Protocol v5. Creating a stable and robust messenger is the first aim, security is the second aim and performance is on the third place.

These points will have to change to achieve this:
### 1. Concurrency ###
The UserData will get a revision control system that detects concurrent changes and forces clients to merge data sets before uploading new data.
### 2. Timing independency ###
Checking for updates using a single DateTime is very data efficient, but it is not guaranteed that you will receive all pending messages. Changes are very difficult to track like this and server development gets increasingly difficult. The new revision based system will need to transfer more data but it will be simpler to implement, more versatile and guarantees data integrity. Message edits and deletions will be treated like new messages. The most difficult thing will be the `ViewableContentInfo`. Sending one packet per client and message would result in an inefficient system causing hundreds of separate packets.
### 3. Transactions and retry ###
Accepting a contact request is a complex process requiring many steps. The new protocol has to ensure that changes are only written to the database once all steps succeeded. If this is not the case, the client must be able to retry the action. Therefore changes, especially those done to the UserData must be linked to other packets.
### 4. Cryptography reloaded ###
Although Skynet should be designed to be more secure than other messengers, Protocol v4 was not able to use end-to-end encryption for profile data related content such as profile images or the daystream. Furthermore, we have to check each situation for whether a server-sided validation is enough or we should use a signature algorithm.
Skynet will not offer Perfect Forward Secrecy because we want to keep cloud synchronisation and data backup as our main features, which would not be possible with PFS.
### 5. Sync on demand ###
Downloading all messages on startup wastes a lot of expensive data volume, slows down the client and makes a web integration completely impossible. A new revision based system should allow sync on demand.
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
Using Firebase Cloud Messaging (FCM) requires the server to keep track of the registered devices. Furthermore, tracking devices brings two more important advantages: Devices that were not connected for more than half a year can be forced to reauthenticated. Also, other features like a two factor authentication are then possible.

### Channels ###
With the Skynet protocol v4 we had different implementations for many types of communication channels. This concept made abstraction almost impossible and increased the complexity, especially on the server side.

To avoid this problem we introduce a completely new concept with Platinum Stack: Every communication channel will now be treated as a `Channel` with a unique `ChannelId`. The server does not care which messages are sent over a channel. Data is sent over a `Channel` as a `Message` with a channel-unique, incremental ID. The key exchange for a channel is either done by a contact request or by using the key of an existing private channel.
In order to move away from the old `UserData` packet, which was prone to concurrency and timing errors, we introduce an additional channel and two more types of messages.

##### MessageFlags #####
`MessageFlags.Loopback` indicates that a message is to be delivered only to the other devices of the sender of this message. Messages with the loopback flag are encrypted using the key of the users _loopback channel_.  
Using a loopback flag solves many problems but makes the classical revision system of all clients having the same data impossible. Therefore, the server has to inform the client when message IDs are skipped. With this information, the client can make sure no messages are missing.

`MessageFlags.Unencrypted` marks a message that is not encrypted and should be processed by the server. Data such as blocked contacts and conversations is managed using the loopback channel with this flag.  

##### Loopback Channel #####
This is a special channel whose only member is the creating user. It acts as kind of a clipboard which is sued to synchronize data across devices. Its key is directly derived from the user's password.

##### Account Data Channel #####
This channel contains all public account information like the public keys to avoid individual forwarding to direct channels. All messages have to be sent with `MessageFlags.Unencrypted`. The server automatically adds all contacts to the user's account data channel.

##### Direct Channel #####
The direct channel is used for direct chat between two accounts. There is no longer a _contact request_. If Alice knows the `AccountId` of Bob's account, which she can look up using the search packets, she can simply create a direct channel. The server will automatically inject the public keys for both sides. Once Alice has received Bob's public key, she can derive and cache the channel key. When the server has sent a _DirectChannelUpdate_ packet, Alice can start sending chat messages to Bob.

##### Group Channel #####
Many other scenarios are realized with a _group channel_. This kind of channel is not limited to classical groups, but is also used for sharing profile related data. The key of a group channel is exchanged via the direct channel between the group admin and each member.

##### Dependencies #####
In order to manage key changes, messages almost always have one dependency to another message on the same or a different channel. The only exception is the key change of a group channel, which has one dependency per account. Additionally, this message needs _MessageFlags.NoSenderSync_ to indicate that it's not synchronised across the sender's devices.

##### Profile data #####
Profile data can either be shared publicly via the account's account data channel with _MessageFlags.Unencrypted_ or via a profile data channel, which technically inherit from the group channel. To manage permissions, multiple profile data channels containing different members can be created. The account data channel is unique per account and managed by the server.

### User files ###
The files attached to media messages are transferred using the VSL 1.2+ encrypted file transfer. Authenticated clients can always upload files, and get an Int64 `FileId` from the server. The owner of a file is allowed to reference it in a message. After that, all users in the same channel channel are allowed to reference it in other channels. This saves a lot of bandwidth when files are forwarded.  
When a message is referenced for the first time in a channel and the sender is allowed to use it, this channel is granted permission to share it again. Files without any permissions are deleted by the garbage collector after a certain time. When a message is deleted in a channel that does not hold any other references to this file, the channel looses its permission. When the file has no more permissions, it's instantly deleted by the garbage collector.  
If a client attempts to reference a file that does not exist or that it has no permission for, the client will know this from the error code in the `ChannelMessageResponse` packet.

### Cryptography ###
The security of Skynet is based on the user's password:
1. Password (UTF-8)
2. `AccountKey` (first 32 bytes for HMAC, last 32 bytes for AES)
3. `KeyHash` (32 bytes)

The `AesKey` is derived from the users password with _Argon2id_ using the `SHA256(AccountName)` as salt:
```
AccountKey = Argon2(P: Password, S: SHA256(AccountName), p: 8, T: 64, m: 8192, t: 2, v: {TODO}, y: 2);
KeyHash = SHA256(AccountKey);
```

## The Protocol ##
The protocol section consists of packets, which each have a name, an ID and the specification written in `VPSL`.

### **0x00** ConnectionHandshake ![networkUp] ###
This packet is sent before any other packet to ensure that the client is using the latest version.
`ProtocolVersion` is necessary to catch breaking changes and enables the server to ignore indev versions with the same protocol version.
```vpsl
<Int32 ProtocolVersion><String ApplicationIdentifier><Int32 VersionCode>
```
The application identifier consists of the platform identifier and the package/assembly name e.g. `android/de.vectordata.skynet`.

### **0x01** ConnectionResponse ![networkDown] ###
This is the server's response to _0x00 ConnectionHandshake_. This packet finishes the handshake and leads the way to register, login or restore a session.

```vpsl
<ConnectionState:Byte ConnectionState>
((ConnectionState != Valid)<Int32 LatestVersionCode><String LatestVersion>)
```
```csharp
enum ConnectionState {
    Valid,
    CanUpgrade,
    MustUpgrade
}
```

### **0x02** CreateAccount ![networkUp] ###
This packet is sent when the user registers a new account. After e-mail address verification is completed, the server has to create the loopback channel and send a _PasswordUpdate_ packet as base revision. For further account initialization see _SyncFinished_.
```vpsl
<String AccountName><Byte[32] KeyHash>
```

### **0x03** CreateAccountResponse ![networkDown] ###
```vpsl
<CreateAccountError:Byte ErrorCode>
```
```csharp
enum CreateAccountError {
    Success,
    AccountNameTaken,
    InvalidAccountName
}
```

### **0x04** DeleteAccount ![networkUp] ###
This packet is sent by a client to delete the user's account.
```vpsl
<Byte[32] KeyHash>
```

### **0x05** DeleteAccountResponse ![networkDown] ###
After receiving this packet with `DeleteAccountError.Success`, the server closes the connection.
```vpsl
<DeleteAccountError:Byte ErrorCode>
```
```csharp
enum DeleteAccountError {
    Success,
    InvalidCredentials
}
```

### **0x06** CreateSession ![networkUp] ###
This packet is sent by the client after opening a connection. The `FcmRegistrationToken` can be left empty if it is not necessary (e.g. for a desktop client). The Skynet server will send a Firebase message if a _ChatMessage_ could not be delivered directly to a client. Client applications have to reconnect until they receive a _SyncFinished_ packet. A newly created session does not need to be restored manually.
```vpsl
<String AccountName><Byte[32] KeyHash><String FcmRegistrationToken>
```

### **0x07** CreateSessionResponse ![networkDown] ###
```vpsl
<Int64 AccountId><Int64 SessionId><CreateSessionError:Byte ErrorCode>
```
```csharp
enum CreateSessionError {
    Success,
    InvalidCredentials,
    InvalidFcmRegistrationToken,
    UnconfirmedAccount
}
```

### **0x08** RestoreSession ![networkUp] ###
After initializing a connection, the client can restore a session with this packet. The client sends the server its last message for each channel.
```vpsl
<Int64 AccountId><Byte[32] KeyHash><Int64 SessionId>
{UInt16 Channels <Int64 ChannelId><Int64 LastMessageId>}
```

### **0x09** RestoreSessionResponse ![networkDown] ###
```vpsl
<RestoreSessionError:Byte ErrorCode>
```
```csharp
enum RestoreSessionError {
    Success,
    InvalidCredentials,
    InvalidSession
}
```

### **0x0A** CreateChannel ![networkDuplex] ###
Sent by the server to a client to create a channel. If it is sent by the client with a random `ChannelId` the server assigns one in the _CreateChannelResponse_ packet.
```vpsl
<Int64 ChannelId><ChannelType:Byte ChannelType><Int64 OwnerId>
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
<Int64 TempChannelId><ChannelCreateError:Byte ErrorCode><Int64 ChannelId>
```
```csharp
enum ChannelCreateError {
    Success,
    AlreadyExists,
    InvalidCounterpart,
    Blocked
}
```

### **0x0B** ChannelMessage ![networkDuplex] ###
The ChannelMessage is the most important packet of the new channel-based protocol. It acts as a box for many types of packets that can be sent to a channel. Use `AccountId=0` to indicate a global dependency. The initial value for the version numbers is `1`.
```vpsl
<Byte PacketVersion><Int64 ChannelId>
((ToClient)<Int64 SenderId>)<Int64 MessageId>
((ToClient)<Int64 SkipCount><DateTime DispatchTime>)
<MessageFlags:Byte MessageFlags>
((MessageFlags.FileAttached)<Int64 FileId>)
<Byte ContentPacketId><Byte ContentPacketVersion>
[<Byte[] ContentPacket>((MessageFlags.FileAttached)<Byte[] FileKey>)]
{UInt16 Dependencies <Int64 AccountId>
<Int64 ChannelId><Int64 MessageId>}
```
```csharp
[Flags]
enum MessageFlags {
    None = 0,
    Loopback = 1,
    Unencrypted = 2,
    FileAttached = 4,
    NoSenderSync = 8
}
```

### **0x0C** ChannelMessageResponse ![networkDown] ###
If the client wants to send a new message, it chooses a random negative `MessageId`. The server responds with this packet and assigns a new `MessageId`. The `DispatchTime` specifies when the server has processed this message.
```vpsl
<Int64 ChannelId><Int64 TempMessageId>
<MessageSendError:Byte ErrorCode>
<Int64 MessageId><Int64 SkipCount>
<DateTime DispatchTime>
```
```csharp
enum MessageSendError {
    Success,
    FileNotFound,
    AccessDenied,
    ConcurrentChanges
}
```

### **0x0D** MessageBlock ![networkUp] ###
The message block defines a set of messages as a transaction. This is necessary in some cases, where multiple messages are required for one operation and connection interrupts cannot be handled.
```vpsl
{UInt16 Messages <Byte[] Message>}
```

### **0x0E** RequestMessages ![networkUp] ###
This packet is sent by the client to request channel messages that have not been synchronized yet. To request all missing messages use `RequestCount=0`. After sending all requested messages, the servers sends a _SyncFinshed_ packet.
```vpsl
<Int64 ChannelId><Int64 FirstKnownMessageId><Int64 RequestCount>
```

### **0x0F** SyncFinished ![networkDown] ###
This packet is sent by the server to a client to indicate that all messages have been delivered. If the client has a _WakeLock_ or something like this to prevent the users device from sleeping, it can be released after receiving this message.

```vpsl
// No additional content
```
When a client receives this packet it should ensure that the following tasks have been completed or execute them in this order:
1. PrivateKeys
2. PublicKeys
3. Nickname
4. PersonalMessage

### **0x10** RealTimeMessage ![networkDuplex] ###
The real time message is a volatile and lightweight version of the `ChannelMessage`. It has no dependencies, and as files are handled with depencies in Skynet, a real time message cannot reference a file which makes `MessageFlags.FileAttached` invalid.
```vpsl
<Int64 ChannelId>((ToClient)<Int64 SenderId>)
<MessageFlags:Byte MessageFlags>
<Byte ContentPacketId><Byte[] ContentPacket>
```

### **0x11** SubscribeChannel ![networkUp] ###
The client has to subscribe channels to receive real time messages. Use `ChannelId=0` to subscribe all available channels and `PacketId=0` to subscribe all packets.
```vpsl
<Int64 ChannelId><Byte PacketId>
```

### **0x12** UnsubscribeChannel ![networkUp] ###
The server ends channel subscriptions automatically when the client disconnects, but if the client is running in the background it might also want to end subscriptions.
```vpsl
<Int64 ChannelId><Byte PacketId>
```
---
### Channel messages ###
### **0x13** QueueMailAddressChange ![networkDuplex] ###
This packet is sent by the client to its loopback channel using `MessageFlags.Unencrypted`. A message is sent to the old mail address to notify that it is about to be replaced.
```vpsl
<String NewMailAddress>
```

### **0x14** MailAddress ![networkDown] ###
When the server has verified the client's new mail address, it sends this packet to the client's loopback channel using `MessageFlags.Unencrypted`. If the new mail address is not verified within 24 hours, the address change is cancelled.
```vpsl
<String MailAddress>
```

### **0x15** PasswordUpdate ![networkDuplex] ###
This packet is sent to the loopback channel with `MessageFlags.Unencrypted`. It informs the server and other devices of a password change and acts as a base revision for packets with `MessageFlags.Loopback`. For password changes use a _MessageBlock_ packet with this packet and a _LoopbackKeyNotify_ packet. This packet is generated by the server after an account is created.
```vpsl
@dependency LoopbackKeyNotify // lazy
((ToServer)<Byte[32] OldKeyHash>)<Byte[32] KeyHash>
```

### **0x16** LoopbackKeyNotify ![networkDuplex] ###
This packet is sent to the loopback channel encrypted with the new key which is referenced as dependency. A dependency from the associated _PasswordUpdate_ packet is automatically injected when this packet is received on both server and client side.
```vpsl
<Byte[] Key>
```

### **0x17** PrivateKeys ![networkDuplex] ###
This packet is sent by the client to its loopback channel. It has no direct effect and is only used as a dependency.
```vpsl
<KeyFormat:Byte SignatureKeyFormat><Byte[] SignatureKey>
<KeyFormat:Byte DerivationKeyFormat><Byte[] DerivationKey>
```
```csharp 
public enum KeyFormat {
    // TODO: Add notations
}
```

### **0x18** PublicKeys ![networkDuplex] ###
This packet is sent by the client to its loopback channel using `MessageFlags.Unencrypted` with a mandatory reference to the matching private key as a dependency.  
The server can also inject this packet into a direct channel for the owning account with an additional `MessageFlags.NoSenderSync`.
```vpsl
@dependency PrivateKeys // only to server
<KeyFormat:Byte SignatureKeyFormat><Byte[] SignatureKey>
<KeyFormat:Byte DerivationKeyFormat><Byte[] DerivationKey>
```

### **0x19** KeypairReference ![networkDown] ###
This packet is sent by the server to each client with `MessageFlags.Unencrypted`. It holds dependencies to the users keypair and the counterpart's public key which was injected by the server.
```vpsl
// No additional content
```

### **0x1A** VerifiedKeys ![networkDuplex] ###
This packet is optional and can be used for a QR-code based verification. It saves the SHA-256 over the counterpart's entire _PublicKeys_ packet and is sent by the client to the associated direct channel with `MessageFlags.Loopback`.
```vpsl
<Byte[32] Sha256>
```

### **0x1B** DirectChannelUpdate ![networkDown] ###
This packet acts as base dependency for following channel messages in direct channels. It references two `DerivationKey` packets as dependency and is injected by the server with `MessageFlags.Unencrypted`.
```vpsl
// No additional content
```

### **0x1C** DirectChannelCustomization ![networkDuplex] ###
This packet is sent by the client to the associated direct channel with `MessageFlags.Loopback`.
```vpsl
<String CustomNickname><ImageShape:Byte ProfileImageShape>
```
```csharp
public enum ImageShape {
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
<Int64 ChannelId><Byte[] NewKey><Byte[] HistoryKey>
```

### **0x1E** GroupChannelUpdate ![networkDuplex] ###
To change the group channel key, the admin sends an update to each client in the group via direct channels. To do so, the client sends this packet with `MessageFlags.Unencrypted` and a dependency referencing all direct channel messages. The incremental `GroupRevision` counter is checked by the server and ensures that no concurrent changes are made. Following messages depend on this packet to specify the cryptographic key they use. When used to share profile data, all members should be invisible. The encrypted content uses the `HistoryKey` from the _GroupChannelKeyNotify_ packet if supplied to the matching client. With this key, the client can resolve all former channel and history keys.
If the owner of the group channel deletes its account, the _OwnerId_ is set to `0`.
```vpsl
@dependency GroupChannelKeyNotify // per account
<Int64 GroupRevision>{UInt16 Members
<Int64 AccountId><GroupMemberFlags:Byte Flags>}
[UInt32 ChannelHistory <Byte[] ChannelKey><Byte[] HistoryKey>]
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

### **0x1F** ProfileDataChannelUpdate ###
This packet is sent by a client to one of its profile data channels. It specifies which data is shared over this channel.
```vpsl
// Reserved for permission system in release v1.1+
```

### **0x20** ChatMessage ![networkDuplex] ### 
The client sends a ChatMessage packet to one specific conversation channel. If the user wants to send a file along with the message, the client adds the `MessageFlags.FileAttached` flag. The packet is always encrypted using the conversation key. A message can always have a text and can always quote another message, regardless of the message type. For example, the user could quote a text message and answer with a voice message or send a contact and write some text in the same message. If no message is quoted, set `QuotedMessage` to `0`. Even if a message is quoted, there is only the dependency to a channel base revision.
```vpsl
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
@dependency ChatMessage | DaystreamMessage
// No additional content
```

### **0x23** MessageRead ![networkDuplex] ###
The client sends this packet when the user has viewed a chat message or a daystream to the respective channel. The received confirmation (`0x22`) can omitted if the read confirmation is sent directly. Handling is the same as `0x22 MessageReceived`.
```vpsl
@dependency ChatMessage | DaystreamMessage
// No additional content
```

### **0x24** DaystreamMessage ![networkDuplex] ###
The client sends the DaystreamMessage packet to any profile data channel. It is always encrypted, since daystream posts are only visible to contacts. 
```vpsl
<MessageType:Byte MessageType><String Text>
```

### **0x25** Nickname ![networkDuplex] ###
The Nickname packet is sent by a client to any profile data channel. If sent with `MessageFlags.Unencrypted`, the nickname is publicly visible. Nickname packets always use `PersistenceMode.KeepLast`. 
```vpsl
<String Nickname>
```

### **0x26** PersonalMessage ![networkDuplex] ###
The PersonalMessage packet is sent by a client to any profile data channel. If it is sent with `MessageFlags.Unencrypted`, the profile image is publicly visible. PersonalMessage packets always use `PersistenceMode.KeepLast`. 
```vpsl
<String PersonalMessage>
```

### **0x27** ProfileImage ![networkDuplex] ###
Similar to PersonalMessage packet. This packet uses the `FileId` of its container.
```vpsl
<String Caption>
```

### **0x28** BlockList ![networkDuplex] ###
The blocked content packet is sent by a client to it's loopback channel with `MessageFlags.Unencrypted` and `PersistenceMode.KeepLast`. The server will then filter contact requests or group invites.
```vpsl
{UInt16 BlockedAccounts <Int64 AccountId>}
{UInt16 BlockedConversations <Int64 ChannelId>}
```

### **0x29** DeviceList ![networkDown] ###
The device list packet is sent by the server to the clients loopback channel with `MessageFlags.Unencrypted` and `PersistenceMode.KeepLast`. With this data the client can inform the user after a new login.
```vpsl
{UInt16 Sessions <Int64 SessionId><DateTime CreationTime><String ApplicationIdentifier>}
```

### **0x2A** BackgroundImage ![networkDuplex] ###
This packet is used to change the private background image in a specific conversation. It is sent with `MessageFlags.Loopback` to the affected channel. To set a default background image, send this packet to the loopback channel. For file uploading and selecting, use the file id of the wrapping channel message.
```vpsl
// No additional content
```
---
### Real time messages ###
### **0x2B** OnlineState ![networkDuplex] ###
The online state packet is sent via the user's broadcast channel to all contacts. Later versions will have different channels to handle permissions. When the client is active, `LastActive` specifies the time the client became active.
```vpsl
<OnlineState:Byte OnlineState><DateTime LastActive><Int64 WritingToChannelId>
```
```csharp
enum OnlineState {
    Active,
    Connected,
    Offline
}
```

### **0x2C** DeviceListDetails ![networkUp] ###
The device list details packet contributes real time data to the general DeviceList packet. It is sent by the server with `MessageFlags.Unencrypted`.
```vpsl
{UInt16 SessionDetails <Int64 SessionId><DateTime LastConnected><Int32 LastVersionCode>}
```
---
### On demand packets ###
### **0x2D** SearchAccount ![networkUp] ###
```vpsl
<String Query>
```
### **0x2E** SearchAccountResponse ![networkDown] ###
This packet contains all results of a _SearchAccount_ query and forwards public profile data.
```vpsl
{UInt16 Results
    <Int64 AccountId><String AccountName>
    {UInt16 ForwardedPackets <Byte PacketId><Byte[] PacketContent>}
}
```

### **0x30** FileUpload ![networkUp] ###
Sent by the client when uploading a file to request a file id.
```vpsl
// No additional content
```

### **0x31** FileUploadResponse ![networkDown] ###
The server's response to `0x30 FileUpload`. This packet contains the file id the client needs for uploading a file.
```vpsl
<Int64 FileId>
```

[networkUp]: https://lerchen.net/skynet/static/network-up-36px.png "Only client to server"
[networkDown]: https://lerchen.net/skynet/static/network-down-36px.png "Only server to client"
[networkDuplex]: https://lerchen.net/skynet/static/network-duplex-36px.png "Both client to server and server to client"