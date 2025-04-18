# omegaflightprotocol
The protocol buffers + protocol description for Omegaflight

Read `PROTOCOL.md` for protocol description.

This is the protocol used between an Omegaflight control server, and an Omegaflight client.

The protocol uses `Protocol Buffers` for data seralization, and as a schema, and it uses `LoRa` based open-source mesh-networking protocol, `Meshtastic`, for long-range, low power, communication.


Pre compiled protobufs will be in the releases.
But to build it yourself use this command:
```
protoc --python_out=./temp ./ofprotobufs/offromclient.proto ./ofprotobufs/oftoclient.proto ./ofprotobufs/ofcommon.proto
```