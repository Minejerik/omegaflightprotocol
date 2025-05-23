# Omegaflight protocol
The protocol used by Omegaflight.

### A lot of documentation already exists with in the protocol buffer files, look in `proto_buffers` for info


#### Basic behaviour
The client needs to have a list of `flight blockers`, which are conditions that are currently not set correctly, which are blocking take off.

#### Basic Ideas
All message-types are split into 3 seconds, common, from-client, and to-client.
Common message-types are used by both to & from client messages.

These have there own package names:
- `common.proto` -> `ofcommon`
- `to_client.proto` -> `oftoclient`
- `from_client.proto` -> `offromclient`

(the of is short for Omegaflight)

Every single action, that has a check, such as making sure that a home is set before take off, should be confirmed on both the client on server, just to be sure.

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
If the mission has the same index as another, it should replace that one, if the mission contains no waypoints, the client should delete that mission

##### `oftoclient.GoTo`
First make sure that the current mode is set to manual, it is not, send back `ofcommon.Error` w/ `ErrorType` of `WRONG_MODE`. Make sure that the client is also in flight already, if not, send back `ofcommon.Error` w/ `ErrorType` of `NOT_IN_FLIGHT_CANT_EXECUTE`. Also just make sure that home is set, theoretically it is impossible for the client receive this message, without being in the air or having home set, but it is better to check again. if home is not set, send an error w/ type `NO_HOME_SET`. Once all checks have passed, it is possible to tell the flight controller to fly to the location given, once there, if there is one, perform the action.

##### `oftoclient.SetHome`
This is to set the home of the client, when received, update the flight controller's home with the one received. Also update the internal home. If the client is currently in flight, send back `ofcommon.Error` w/ `ErrorType` of `IN_FLIGHT_CAN_NOT_CHANGE`. A flag should be set, saying that the home has been set, removing another block from flight.

##### `ofcommon.Command`
This might end up being also sent by the client, so it is in `ofcommon` instead of `oftoclient`.
If the type is `TAKE_OFF`, check if there are any active flight blockers, and if there are, send an `Error`  with the appropriate `ErrorType` for the blocker. If the mode is manual, tell the flight controller to rise to a set height AGL, this should be changeable in client config. If the mode is mission, tell the flight controller to start heading towards the first waypoint. If all blockers are removed, and the client starts rising, set a flag saying that the client is currently in flight. Also a `offromclient.Alert` with `AlertType` of `TAKING_OFF` should be sent to the server.
If the type is `LAND_HERE`, tell the flight controller to land at the current location, and once touched-down, send back a `offromclient.Alert` with `AlertType` of `LANDED`. If the type is `LAND_HOME`, tell the flight controller to land at the home position, also send an alert of landed once touched down.
If the type is `MISSION_INDEX_DATA`, gather all of the mission indices from cache/storage, and send them in a `offromclient.MissionIndexTransfer` message to the server.

#### Client Messages (sent)

##### `offromclient.Status`
This is a generic status message, it contains the current location of the client, and, as of now, the battery voltage of the client. This is used by the server to update the UI & add more logging data.

##### `offromclient.Alert`
This may be split into two messages, but as of the current version, it is a single message.
This should be sent anytime one of the alert types occurs, such as a flight controller disconnect, low battery, emergency landing, or a crash. It is also used to communicate non-negative alerts, such as landing or a completed action. The negative messages may be split off into a `Fault` message, but as of now, they are part of `Alert`. 
A location is required, so that the server can log, or update the UI with this information, it also can have a message, this is optional for non-urgent alerts, and required for misc alerts, in order to provide information to the server and user.

##### `offromclient.MissionIndexTransfer`
This message is used to transfer the cached/saved missions to the server, in the form of a list of `MissionIndex` messages. This may change in the future, but for the current protocol version, it is used.
When the client receives an `ofcommon.Command` w/ `CommandType` of `MISSION_INDEX_DATA`, it should send this in return.

##### `offromclient.ReachedWaypoint`
This message is a specific alert for reaching a waypoint, the reason it is not part of the `Alert` message is due to the fact that it requires bespoke data, that no other alert requires.

##### `ofcommon.Error`
This is used to alert the server of a behaviour fault, it is in common, as the server may be able to send this in a future protocol version, in order to inform the client of protocol violation.
The message contains an `ErrorType` which contains the exact type of error that happened. This message is only used as a response, after a behaviour-violating message was sent to the client. It is used to log or update the UI, the server may be able to fix the error itself, but it should still update the user on what is occurring.

##### raw-text
THIS SHOULD ONLY BE SENT IN A WORST-CASE SCENARIO!!
The server will treat this as a high-severity `Alert`, with the text being used as the body for the alert.
This should only be sent if there is something actively going catastrophically wrong on the client-side.
Once this is sent, the server will stop expecting any updates.
This should only really be used when an irrecoverable fault occurs, and if it is possible, the client should tell the flight controller to land either at home, or at the current location. 
In normal operation, this should never be needed.

#### Common Messages
Most of these messages are used types in other messages, they are not to be sent raw.

##### `ofcommon.Location`
Generic location message, contains lat, long, and altitude.
Altitude is in feet.

##### `ofcommon.Waypoint`
A waypoint used in mission mode, used as part of a `Mission` message, contains a location, and a possible action to be performed at that waypoint.

##### `ofcommon.WaypointIndex`
Used to index waypoints as part of a `Mission` message, it just contains the index of the waypoint in the list of waypoints inside of a mission.

##### `ofcommon.Action`
This represent an action to be performed at a waypoint, once it is reached. 
It contains an `ActionType` enum, it also contains pin number and sleep time attributes, which are used for the `PIN_TRIGGER` and `SLEEP` action types respectively.

##### `ofcommon.Mission`
Represents a mission for the client to perform during mission mode.
It contains a list of waypoints, as well as an index, this index is used mostly as an ID for that specific mission. This index does not need to be incremental, and can be any number to the 32-bit integer limit.

##### `ofcommon.MissionIndex`
This message just contains the index of a mission, it used to communicate saved/cached missions to the server.

##### `ofcommon.Command`
See client & server behaviour.
Represents a command, that the receiver must carry out.

##### `ofcommon.Error`
See client & server behaviour.
Just used to represent an error, mostly behaviour errors, but can also be protocol errors. 

#### Examples
These are full, simple, flights, which show the protocol messages sent.

##### Simple Mission Flight
This example creates, and sends a simple mission, which is then flown by the client.

- The server sends `oftoclient.SetHome`, with an `ofcommon.Location` set to the home position
    -   The client must note down the home position
    -   The client must also send the home position to the flight controller
    -   And the client must remove the flight blocker due to not having a home position
- The server sends `oftoclient.SetMode`, with `ModeType` of `MISSION`
    -   This is noted by the client, to change behaviour
    -   Also the client must remove the flight blocker due to not having a mode
- The server sends `oftoclient.TransferMissionData`
    -   The mission is setup as this:
        -   Waypoint 1, location set to point A, at set altitude
            -   No Action, just first position
            -   Should be above/near home position
        -   Waypoint 2, location set to point B, at set altitude
            -   Action should be of type `LAND_HERE`
            -   Once waypoint is reached, client will land
        -   Index
            -   Can be set to any unsigned 32-bit int
            -   For this example will be `42`
    -   Once client receives this message
        -   The mission should be added to the mission cache/storage on client
- The server sends `oftoclient.QueueMission`
    -   The `MissionIndex` should contain the index of the mission
        -   In this case it should be `42`
    -   Once the client receives this, it should:
        -   Make a note of the mission to be flown
        -   Load the first waypoint into memory, so it knows where to go when to fly
        -   Remove the flight blocker due to not having a queued mission
- The server sends `ofcommon.Command`
    -   The `CommandType` should be `TAKE_OFF`
    -   The Client Should:
        -   Verify that no flight blockers are in place
            -   If there are ones, send back an `ofcommon.Error`
                -   With `ErrorType` corresponding to the error.
        -   If no flight blockers exist, tell the flight controller to take off
            -   And tell the flight controller the destination of the first waypoint (Point A)
            -   It should also update the current flight status to flying
- The client sends `offromclient.Alert`
    -   With `AlertType` of `TAKING_OFF`, and no message
    -   The server should update its logs, and update the UI.
    -   The server should update its current client flight status to flying
- The client sends an `offromclient.Status` Message
    -   It should contain the current location of the client
    -   The server should update its logs, and update the UI with this information.
    -   This should be sent every n seconds
- The client reaches Waypoint 1
    -   It should send an `offromclient.ReachedWaypoint` message
        -   `WaypointIndex` would be `0`
            -   As it is the first waypoint in the mission
        -   `MissionIndex` should be `42`
            -   As that is the index of the mission
    -   Server should update its UI with this information, and log it.
    -   The client should wait a few seconds
        -   Before telling the flight controller to fly towards waypoint 2 (Point B)
- The client should send an `offromclient.Status` Message
    -   See above for more info
- The client reaches Waypoint 2
    -   It should send an `offromclient.ReachedWaypoint` message
        -   `WaypointIndex` would be `1`
            -   As it is the second waypoint in the mission
        -   `MissionIndex` should be `42`
            -   As that is the index of the mission
    -   Server should update its UI with this info, aswell as logging it
    -   The client should wait a few seconds
        -   Due to the waypoint having an action, it should perform that action
            -   In this case the action has a type of `LAND_HERE`
            -   The client should tell the flight controller to land here
- Once the client lands
    -   It should send an `offromclient.Alert`
        -   `AlertType` should be `LANDED`
        -   The client should update its current flight status to landed
        -   Once the server receives this, it should do the same.
        -   The server should update its logs, and its UI.

##### Simple Manual Flight
This is a simple manual flight, the beginning is very similar to the mission mode flight
In this example landing will happen through a command, not an action.

- The server sends `oftoclient.SetHome`, with an `ofcommon.Location` set to the home position
    -   The client must note down the home position
    -   The client must also send the home position to the flight controller
    -   And the client must remove the flight blocker due to not having a home position
- The server sends `oftoclient.SetMode`, with `ModeType` of `MANUAL`
    -   This is noted by the client, to change behaviour
    -   Also the client must remove the flight blocker due to not having a mode
        -   Due to being in manual mode, the flight blocker for an unqueued mission does not apply, and should not be added
- The server sends `ofcommon.Command`
    -   The `CommandType` should be `TAKE_OFF`
    -   The Client Should:
        -   Verify that no flight blockers are in place
            -   If there are ones, send back an `ofcommon.Error`
                -   With `ErrorType` corresponding to the error.
        -   If no flight blockers exist
            -   The client should tell the flight controller to fly to a set height AGL
                -   This height should be in the config for the client
            -   It should also update the current flight status to flying
- The client sends `offromclient.Alert`
    -   With `AlertType` of `TAKING_OFF`, and no message
    -   The server should update its logs, and update the UI.
    -   The server should update its current client flight status to flying
- The client sends an `offromclient.Status` Message
    -   It should contain the current location of the client
    -   The server should update its logs, and update the UI with this information.
    -   This should be sent every n seconds
- The server sends an `oftoclient.GoTo`
    -   It contains the location of where the client should go
    -   It can also contain an action
        -   This is another way landing could occur
            -   With an `Action` of type `LAND_HERE`
            -   In this example, a `Command` will be used, instead of this.
            -   But this could work aswell, allowing the user to program in a command
            -   To land once the waypoint is reached, instead of making the user land manually
    -   The client should tell the flight controller to start flying towards this location
- The client sends an `offromclient.Status` Message
    -   It should contain the current location of the client
    -   The server should update its logs, and update the UI with this information.
- The client reaches the location
    -   It should wait until a new `GoTo` message is sent
        -   Or a `Command` with a `CommandType` of land
    -   If an action is added, it should wait a few seconds, before performing the action
- The server sends an `ofcommon.Command`
    -   The `CommandType` should be `LAND_HERE`
    -   This will trigger the landing for the client
    -   The client should
        -   Tell the flight controller to land here
- Once the client lands
    -   It should send an `offromclient.Alert`
        -   `AlertType` should be `LANDED`
        -   The client should update its current flight status to landed
        -   Once the server receives this, it should do the same.
        -   The server should update its logs, and its UI.