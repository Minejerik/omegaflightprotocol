# Omegaflight protocol
The protocol used by Omegaflight.

### A lot of documentation already exists with in the protocol buffer files, look in `proto_buffers` for info

#### Basic Ideas
All message-types are split into 3 seconds, common, from-client, and to-client.
Common message-types are used by both to & from client messages.

These have there own package names:
- `common.proto` -> `ofcommon`
- `to_client.proto` -> `oftoclient`
- `from_client.proto` -> `offromclient`

Nearly all communication, except for extreme circumstances, will take place over protobuffers,
as they are fast, and efficient, to transfer. 

Currently, the only scenario in which raw-text is allowed to be transmmited is when an extreme error has occurred on the client side, for example, crashing.

Any raw-text sent over meshtastic by the client should be handled as the body for a high-severity error, which MUST be sent to the user, as in the case of an extreme error as this, danger to the user, or others, may be created, if not dealt with.

##### Despite the fact that, in modern protobuffers, `required` is deprecated, it is used within the protocol. Why?

Because, the issues that were mentioned for its deprecation, are not really applicable here. 
It is assumed that the client and server are running compatible protocol-versions, which gets rid of the issue of possible having a stale listener, on an old, incompatible protocol-version. 
Also, not using `required` can create the issue, in which an error occurs, which could cause a message to be sent, that is missing a very important field, an example would be, if an error occurs when creating a `Location` message, which could cause one of the field to not be sent, without the `required` tag, it would require that the software checks that the packet is complete, which is extra overhead, so by using the `required` tag, despite deprecation, reduces the overhead required, because the deserialization is simply blocked.

#### Style of message names & attributes

All message & enum names should be in Pascal Case,

While attributes should be shortened, and be in all lowercase. 
as in, message -> msg, and location -> loc.
and common attribute names should be used, here is the list, as of now.

- location/Location -> loc
- Message -> msg

Also all enum options should be all upper case.

#### Procedures

##### Take off procedure (manual mode)
This is how take off is triggered in manual mode, it should be trigger by pressing a "Take Off" button on the manual control area of the server's user interface.
- First the home MUST be set, see `oftoclient.SetHome`
- Set the mode to manual by sending `oftoclient.SetMode` with `ModeType` of `MANUAL`
- Then just send a `ofcommon.Command` w/ `CommandType` set as `TAKE_OFF`, this will bring the client to a set altitude AGL, based on client config.

##### Goto location (manual)
This is how movement is done inside of manual mode, this may be changed later. But currently this is used.
This should be sent when the user inputs a location on the user interface while in manual mode.
