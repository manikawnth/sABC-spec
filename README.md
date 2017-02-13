# sABC-spec
simple Asynchronous Bi-directional Communication Protocol<br>

## Overview
A highly-opinionated transport agnostic protocol to communicate asynchronously between client-server
<br>
It comprises of two basic mechanisms
 - Connection mechanism
 - Messaging mechanism

## Actors

 ### Client-server
 A client initiates a connection.<br>
 A server authenticates, accpets a connection and sends a unique session-id for that connection <br>
 Both the client and server communicates using the session-id.
  - Each active session id should be used by exactly one connection handler

 ### Message
 A message is just one or more **message frames**<br>
 Messages have restrictions imposed by server:
  - Each message is limited by its size (command + header + body + null) crossing which the server should truncate and mark end of the message<br>
    In short, the null message frame portion (`\r\n\r\n\x00`) or size limit which ever is met first should markt the end of message

 ### Message Frame
 A messsage frame is one frame sent on the network by using underlying protocol like TCP or web-sockets.<br>
 It consists of 4 portions

  - **COMMAND** - An upper-cased command which identifies the type of the message frame
  - **Headers** - List of key-value paires delimited by double colon (::) which carry crucial information of the connection and the frame
  - **Body** - Message frame Body
  - **null** - optional Null byte marking end of the message (not the message frame)

  Command and Headers are separated by one CR-LF (`\r\n`).<br>
  Headers and body are separated by two CR-LFs (`\r\n\r\n`).<br>
  null preceded by two CR-LFs mark end of the message.

*This is a valid frame*

    COMMAND
    header-key1::headerValue1
    header-key2::headerValue2

    bodybodybodybody
    bodybodybodybody

    ^@

*Although useless, this is also a valid frame*

    COMMAND
    header-key::headerValue

    ^@

*This is also a valid frame just that the message is not complete yet*

    COMMAND
    header-key::headerValue

    bodybodybodybody
    bodybodybodybody

*But this is an **invalid** frame since headers are not present*
    COMMAND

    bodybodybodybody
    bodybodybodybody

    ^@

#### Commands
Valid commands are:
 - **CONNECT** issued by client
 - **CONNECTED** issued by server
 - **DISCONNECT** issued by client
 - **DISCONNECTING** issued by server
 - **MESSAGE** issued by both server and clients
 - **ERROR** to transmit any errors

#### Headers
Valid headers are listed in the below table. Rules and restrictions can be observed while examining the individual commands

| Header              | Command                                    | Mandatory? |
|---------------------|--------------------------------------------|:----------:|
| client-id           | CONNECT                                    |     NO     |
| client-passcode     | CONNECT                                    |     NO     |
| session-id          | All except CONNECT                         |     YES    |
| session-expiry      | CONNECTED                                  |     NO     |
| msg-id / ref-msg-id | MESSAGE                                    |     YES    |
| send-only           | MESSAGE                                    |     NO     |
| msg-more            | MESSAGE                                    |     NO     |

## Connection mechanism

### Connection Initiation
Connection initiation happens only from client

*Client*

        CONNECT
        client-id::23450-678-aedc
        client-passcode::Password@123

        ^@
 NOTES:
 - The headers client-id, client-passcode should be used for authentication on the server
 - Omit those optional headers if no authentication is needed

*Server*

        CONNECTED
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        session-expiry::Tue, 01 Jun 2017 21:47:38 GMT
NOTES:
 - session-id can be anything unqiue that an active server could generate to identify a connection session. This is a named alias for client connection handler
 - session-id is a mandatory header on all commands except `CONNECT`
 - session-expirty is an optional header which identifies the expiry time of the session
 - After the expiry time is reached, the server should disconnect the client 

### Connection Termination
Connection can be terminated by either server or client

Client can request server to gracefully terminate the session

*Client*

        DISCONNECT
        session-id::35a4cf3a-0a75-414b-81e8-69b9b4075c4b

NOTES:
 - session-id is passed to request Server to disconnect the client
 - This is a graceful termination and server should process the earlier or in-flight messages for the client before disconnecting the client
 - After this command is received, server should stop accepting any further messages from the client 

 Client / Server both can terminate the session informing the other party

 *Client* / *Server*

        DISCONNECTING
        session-id::abcdef123456@#

 NOTES:
  - This could be an ugly termination of connection
  - The disconnecting party sends the disconnecting command and terminates the Connection
  - However, it is not guaranteed (and shouldn't be) that the other party sees this command 

## Messaging mechanism

MESSAGE is a command to send a message between two peers.<br> 
Message can be sent from either client or server.<br>
Since the messaging is asynchronous all the messages have to be tagged with msg-id. 
And it is the responsibility of the message sending party to uniquely tag the message id.

There are 2 flavors of messaging 
 - Messages where responses / acknowledgements are expected
 - Messagess which are send-only and delivery guarantee is not bothered

**Messsags can span across multiple message frames**<br>
**Maximum message-size should be implemented by server to limit the size of the message**

### Messaging types

Sending a response:

*Peer*

        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::000001

        Hola

        ^@

*Other peer*

        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        ref-msg-id::000001

        Mundo!!!

        ^@

Sending an acknowledgement is no different than sending a response 

*Peer*

        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::123123

        Hello

        ^@

*Other peer*

        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        ref-msg-id::123123

        OK

        ^@

Send-only messages

*Client/Server*

        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::888999
        send-only::yes

        OK

        ^@

NOTES:
 - message-id or ref-message-id header should be present on every message 
 - message-id represents the originating message where as ref-message-id represents the response of the original message
 - send-only (`yes`) represents a send only message. ref-messsage-id shouldn't be used in this case 
 - If not a send-only message, server should respond back with a response or an acknowledgement based on how server is implemented


### Multi-framed message

Message can be wired on multiple frames.<br>
Either a null character in the message body marks the end of the message OR <br>
reaching the message size limit marks the end of the message

#### Multi-framed message with the null character

*Frame1*
        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::123456
        msg-more::yes

        This is first part of the message. Second
        part 

*Frame2*
        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::123456
        msg-more::yes

        is on the way. See I can have ^@ within the
        message but shouldn't precede with two CR-LFs.
        Finally the third

*Frame3*
        MESSAGE
        session-id::NaTPOgp1QUuB6Gm5tAdcSw
        msg-id::123456

        part marks the end of the
        message

        ^@

The peeer that receives the frames should accumulate them to form a complete message.<br>
The message body formed by the peer should read as follows:
        This is first part of the message. Second
        part is on the way. See I can have ^@ within the
        message but shouldn't precede with two CR-LFs.
        Finally the thirdpart marks the end of the
        message

NOTES:
 - msg-id should be consistent across the frames of a single message
 - msg-more should be set to `yes` if there is something more to be sent on next frame 
 - msg-more has **no significance** on the end frame (where frame has ^@ section)
 - If the frame has no ^@ section and no msg-more, then it is marked as Error frame and should be discarded

#### Multi-framed message with message-size limit on it.

Let's assume a peer has message size limit of **300** bytes on it<br>
The size includes all the command, headers, body and the null sections of every frame.

In this size limit case, the message that is formed by the peer from the above example looks like 

    MESSAGE
    session-id::NaTPOgp1QUuB6Gm5tAdcSw
    msg-id::123456
    msg-more::yes

    This is first part of the message. Second
    part MESSAGE
    session-id::NaTPOgp1QUuB6Gm5tAdcSw
    msg-id::123456
    msg-more::yes

    is on the way. See I can have ^@ within the
    message but shouldn't precede with two CR-LFs.
    Finally t

So, the message body that is formed is as follows:

    This is first part of the message. Second
    part is on the way. See I can have ^@ within the
    message but shouldn't precede with two CR-LFs.
    Finally t

#### Error message frames

Error messages are special frames to communicate errors.<br> 
Unlike regular messages, the errror messages should be sent in a single frame

*Example*

        ERROR
        error-code::403

        Access Forbidden

        ^@

NOTES:
 - There are no specific rules for the headers or message body incase of error frames.
 - One way is to send the error-code in header and the error description as part of body.

## Rules in a nut-shell

**Command Rules**

1. Valid commands are `CONNECT` , `CONNECTED` , `DISCONNECT` , `DISCONNECTING` , `MESSAGE` , `ERROR`
2. Only clients can issue `CONNECT` and only servers can issue `CONNECTED`
3. Only clients can issue `DISCONNECT` but both client and server can issue `DISCONNECTING`
4. Both client and server can send messages using `MESSAGE` command 
5. Although uncommon for clients to send, both client and server can send Error messages using `ERROR` command

**Header Rules**

1. Valid headers which are reserved (cannot be used otherwise) are
  - client-id
  - client-passcode
  - session-id
  - session-expiry
  - msg-id
  - ref-msg-id
  - send-only
  - msg-more
  - error-code
2. `client-id` is mandatory on `CONNECT` command. Should represent the client identity
3. `client-passcode` is optional on `CONNECT` command. Should be used for authentication alogn with `client-id`
4. `session-id` is mandatory on all commands except `CONNECT`. Should be first populated as part of `CONNETED` command 
5. `session-expiry` is an optional header returned on `CONNECTED` command. Represents the *upto* session validty time
6. `msg-id` is mandatory header on `MESSAGE` command unless there is a `ref-msg-id` header. 
It represents the unique id of the message in a session to communicate asynchronously.
In case of multi-framed message scenario, this identifier is also used to concatenate the message frame bodies to form a single message body.
7. `ref-msg-id` is mandatory header on `MESSAGE` command unless there is a `msg-id`. 
It represents a response to the message sent with id `msg-id`
In case of multi-framed message (response) scenario, this identifier is also used to concatenate the message frame bodies to form a single message body.
8. `send-only` is an optional header on `MESSAGE` command which represents that the message is send-only and no response is expected.
In case of multi-framed message with message-id, it is sufficient to have `send-only` populated on the first frame.
It can have value `yes` to reprsent sendonly message. All other values represent *no*
9. `msg-more` is an optinal header on `MESSAGE` command which conveys the peer that there are more message frames being sent as part of single message
It can havge a value `yes` . All other values represent `no`
10. `error-code` is an optional *conventional* header that should only be transmitted on `ERROR` commands. 
It represents the an error. If it is an error on a specific message, a `message-id` could be accompanied with.
But it is purely upto the implementer.

**Encoding rules**

1. All the messages should be encoded in *UTF-8* encoding
2. Headers cannot have `::` in their values