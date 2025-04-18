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

(the of is short for Omegaflight)

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

#### Procedures & Messages

More info for each message can be found in the comments inside of the message.
This is mostly how a client or server should handle every possible procedure or message.
All the information is how stock OF will be using these messages & procedures.
Any client or server that wishes to have compatibility with stock OF, should follow these procedures.

#### Server Messages (received)

How the sever should handle received messages.

##### `offromclient.Status`
This message from the client is just a generic status message, sent every n time steps.
This should be used to update the server's information about the client, which should be used to update the UI.

##### `offromclient.Alert`
This message is an alert from the client, it contains the location and alert type.
Based on the alert type, seen in the comments in the file, a certain severity of notification should be sent to the UI/user.
It may contain a message, based on the alert type, the server can show the message no matter the type, but the client should not sent a message for certain types of alert, and can be assumed, in stock OF, that it won't send one. 
The location included can be shown on a map, if one is included in the UI, but it can also just be used for logging purposes.

##### `offromclient.MissionIndexTransfer`
This is a transfer of all of the saved messages on the client, it contains just a list of mission indices.
May be removed/deprecated in the future, but as of now, is the method of sharing missions on the client, it is requested by sending `ofcommon.Command` w/ the `CommandType` set as `MISSION_INDEX_DATA`

##### `offromclient.ReachedWaypoint`
This contains the mission and waypoint index of the waypoint just reached, can be used for updating UI.

##### `ofcommon.Error`
This is only being sent by the client, but may be sent by the server in the future, that is why it is in common.
This contains an error, this is like `offromclient.Alert` but is more specific towards errors, containing an enum of possible errors.
This can be shown to the user, but is not required, as it is possible to fix some of the errors automatically on the server side, it is still a good idea to at least inform the user that an error did occur though, and it should be added to a log file.

##### raw-text
In the case that raw text, as in, a message that is not a protobuffer is sent, it should be treated as an alert of top severity. This was created so that the serialization can be skipped, in case of emergency.
Once a raw-text message is sent, regular updates from the client should not be expected, as a major issue may have occurred.

#### Server Messages (sent)

How the server should send messages.

##### `oftoclient.SetMode`
This message is used to set the mode of the client, either manual or mission mode.
It contains only a single enum, which contains the mode type.
This needs to be set before a flight takes places, in order to setup correct behaviour from client

##### `oftoclient.QueueMission`
This is needed to tell the client which mission should be executed, this needs to be sent before taking off when in mission mode.
It requires the mission index, which can be obtained by requesting all saved mission data from the client

##### `oftoclient.TransferMissionData`
This is sent to the client w/ a mission, used to transfer data about a mission to the client, contains only a mission message.

##### `oftoclient.GoTo`
May be replaced in the future w/ waypoints, but is currently used to tell the client where to fly while in manual mode.
It requires a location and an action, the action is optional, but will be able to set in stock OF.

##### `oftoclient.SetHome`
IMPORTANT, needs to be set before mode is even set, this tells the client where to land if all goes wrong.
This just requires a location for the home inside of the message.

##### `ofcommon.Command`
This may end up being used by both client and server in the future, but is currently only being used by the sever to command the client. It requires just a type, information about what each type of command is given in the comments around the message.

#### Client Messages (received)

This is how the client should react to sent messages from server.

##### `oftoclient.SetMode`
The client should simply take note of the current operating mode, set from this message.
Based on the mode, the behaviour of flight should change, will be noted later.
Should be blocked during flight, send back `ofcommon.Error` w/ `ErrorType` of `IN_FLIGHT_CAN_NOT_CHANGE` if flight mode is set to be changed.

##### `oftoclient.QueueMission`
The client should use the mission index part of this message, to set the currently used mission.
If the mission does not exist, send back a `ofcommon.Error` w/ `ErrorType` of `MISSION_DOESNT_EXIST`.
Make sure to set a flag that the mission has been queued, so that it can be clear that this is not a flight blocker.
The mission should be loaded from mission cache/storage, based on index, ignore mission name, that is only for UI.

##### `oftoclient.TransferMissionData`
This is the server sending a new mission to the client, it contains the mission, which contains a list of waypoints for the client to go to.
The new mission should be saved to mission storage/cache, it could be saved as a binary file, as a raw protobuf, and then the filename added to a database. That works but other solutions can be used.

##### `oftoclient.GoTo`
First make sure that the current mode is set to manual, it is not, send back `ofcommon.Error` w/ `ErrorType` of `WRONG_MODE`. Make sure that the client is also in flight already, if not, send back `ofcommon.Error` w/ `ErrorType` of `NOT_IN_FLIGHT_CANT_EXECUTE`. Also just make sure that home is set, theoretically it is impossible for the client receive this message, without being in the air or having home set, but it is better to check again. if home is not set, send an error w/ type `NO_HOME_SET`. Once all checks have passed, it is possible to tell the flight controller to fly to the location given, once there, if there is one, perform the action.

##### `oftoclient.SetHome`
This is to set the home of the client, when received, update the flight controller's home with the one received. Also update the internal home. If the client is currently in flight, send back `ofcommon.Error` w/ `ErrorType` of `IN_FLIGHT_CAN_NOT_CHANGE`. A flag should be set, saying that the home has been set, removing another block from flight.

##### `ofcommon.Command`
This might end up being also sent by the client, so it is in `ofcommon` instead of `oftoclient`.
If the type is `TAKE_OFF`, check if there are any active flight blockers, and if there are, send an `Error`  with the appropriate `ErrorType` for the blocker. If the mode is manual, tell the flight controller to rise to a set height AGL, this should be changeable in client config. If the mode is mission, tell the flight controller to start heading towards the first waypoint. If all blockers are removed, and the client starts rising, set a flag saying that the client is currently in flight. Also a `offromclient.Alert` with `AlertType` of `TAKING_OFF` should be sent to the server.
If the type is `LAND_HERE`, tell the flight controller to land at the current location, and once touched-down, send back a `offromclient.Alert` with `AlertType` of `LANDED`. If the type is `LAND_HOME`, tell the flight controller to land at the home position, also send an alert of landed once touched down.
If the type is `MISSION_INDEX_DATA`, gather all of the mission indices from cache/storage, and send them in a `offromclient.MissionIndexTransfer` message to the server.



#### Procedures

These are full actions that are done, like taking off, these go into less detail, but explain how a full actions should be done.

##### Take off procedure (manual mode)
This is how take off is triggered in manual mode, it should be trigger by pressing a "Take Off" button on the manual control area of the server's user interface.
- First the home MUST be set, see `oftoclient.SetHome`
- Set the mode to manual by sending `oftoclient.SetMode` with `ModeType` of `MANUAL`
- Then just send a `ofcommon.Command` w/ `CommandType` set as `TAKE_OFF`, this will bring the client to a set altitude AGL, based on client config.

##### Goto location (manual)
This is how movement is done inside of manual mode, this may be changed later. But currently this is used.
This should be sent when the user inputs a location on the user interface while in manual mode.
