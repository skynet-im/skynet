# Transport Protocol

## Introduction

Skynet has been using _VSL_ as transport protocol since the beginning of internal version 4. However, maintaining this library becomes increasingly time-consuming and the file transfer is not scalable at all.

The solution for these problems will be a simple transport protocol that offers Packet boundries and is build on TLS.
An optimized version of the Skynet Server could use a special proxy for TLS offloading written in C/C++ to achieve further performance improvements compared to VSL.

## Binary format

Just like VSL, the new Skynet transport protocol uses little endian as network byte order.

### General packet

```vspl
<Int32 PacketMeta>...
```

```
Raw bytes (little endian)

| 2B | FF | FF | 00 |
| ID |    Length    |
```

The maximum packet size is limited to 2^24-1 Bytes (16 MiB).

```csharp
void Read()
{
    int packetMeta = buffer.ReadInt32();
    byte packetId = (byte)(packetMeta & 0x000000FF);
    int contentLength = packetMeta >> 8;
}

void Write(byte packetId, int contentLength)
{
    buffer.WriteInt32(contentLength << 8 | packetId);
}
```

### PacketBuffer
User content always uses little endian byte order and has different length markers. Strings are always UTF-8 encoded.

`ShortString` and `ShortByteArray` are length prefixed with an `UInt8`.
`String` and `ByteArray` are length prefixed with an `UInt16`.  
`LongString` and `LongByteArray` are prefixed with an `Int32`.

## File transfer

Unlike VSL the new transport protocol will not offer a full end-to-end encrypted file transfer. Small files (≤ 50 KiB) can be stored inside the channel message packet. Large files are uploaded over HTTPS to a mirror server and can be retrieved on the same way.

> The domain names are examples and might be different in production.

### Upload

1. Client requests file upload URL  
`PUT https://api.skynet.app/filetransfer/upload?session={sessionId}&token={webToken}`
2. Skynet server responds  
`307 Temporary Redirect`  
`Location: ...`  
`X-Skynet-FileId: {fileId}`
3. Client uploads file  
`PUT https://cdn.skynet.app/filetransfer/{fileId}?token={signedToken}`
4. CDN Server notifies Skynet server on success  
`POST https://api.skynet.app/filetransfer/{fileId}/completed?token={secretToken}`
5. CDN Server responds client
`201 Created`

### Continue aborted upload

1. Client request status  
`HEAD https://cdn.skynet.app/filetransfer/{fileId}?token={signedToken}`
2. Client uploads rest of the file with `Content-Range`  
`PATCH https://cdn.skynet.app/filetransfer/{fileId}?token={signedToken}`
3. CDN Server notifies Skynet server on success  
`POST https://api.skynet.app/filetransfer/{fileId}/completed?token={secretToken}`
4. CDN Server responds client
`204 No Content`

### Download file
1. Client requests file download URL  
`GET https://api.skynet.app/filetransfer/{fileId}?session={sessionId}&token={webToken}`
2. Skynet server responds  
`307 Temporary Redirect`  
`Location: ...`
3. Client downloads file possibly with `Content-Range`  
`GET https://cdn.skynet.app/filetransfer/{fileId}?token={signedToken}`
4. CDN Server responds client  
`200 OK`
