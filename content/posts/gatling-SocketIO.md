+++
title = 'Load testing Socket IO with Gatling'
date = 2023-11-12T16:08:41+02:00
draft = false
showtoc = true
tocopen = false
tags = ['testing','Gatling','Socket.IO','load-testing']
+++

This article will hopefully save you some of the trouble I had, when I tried to use Gatling for Socket.IO sockets. Gatling has support for web sockets but one has to do a lot of research and workarounds in order to make it work with Socket.IO sockets.

# What are Gatling and Socket.IO?

Gatling is a powerful load-testing solution for applications, APIs, and microservices (https://gatling.io/).

Socket.IO is a library that enables low-latency, bidirectional and event-based communication between a client and a server (https://socket.io/docs/v4/). The Socket.IO connection can be established with different low-level transports and one of them are web sockets. Socket.IO uses the web socket protocol as one of the underlying protocols for bidirectional communication.

# Load testing Socket.IO

Back to our problem...

## The Socket.IO server

I prefer to always work with minimal examples when I test something new. I want to avoid the complexity of a large problem getting into the way of understanding how to solve the technical issues. That's why the working example is very simple:

- The Socket.IO server responds to the special `connection` event.
- On a successful connection:
  - the socket accepts messages with the `message` event and the message content. On the reception of the message it logs the message and then emits the message using a `broadcast` event.
  - Finally, the socket accepts the special `disconnect` event with a log message.

This is the implementation of the server in JavaScript:

```js
const { Server } = require("socket.io");
const cors = require("cors");

const io = new Server({
  transports: ["websocket", "polling"],
  cors: {
    origin: "*",
    methods: ["GET", "POST"],
  },
});

io.on("connection", (socket) => {
  console.log("a user connected to the socket.io server: " + socket.id);

  socket.on("message", (msg) => {
    console.log("message: " + msg);
    io.emit("broadcast", "they say: " + msg);
  });

  socket.on("disconnect", () => {
    console.log("user disconnected: " + socket.id);
  });
});

io.listen(3000);
```

The only required dependencies are:

```sh
npm i socket.io cors
```

Back to Gatling:

## Gatling websocket simulation

Following the Gatling documentation for Web Sockets: https://gatling.io/docs/gatling/reference/current/http/websocket/ I created a simple simulation:

```java
package computerdatabase;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;

public class WebsocketExampleSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
            .wsBaseUrl("ws://localhost:3000");

    ScenarioBuilder scene = scenario("WebSocket")
            .exec(ws("Connect WS").connect("/"))
            .pause(1)
            .exec(ws("Say hi")
                    .sendText("message Hi")
                    .await(30).on(
                            ws.checkTextMessage("checkMessage")
                                    .check(regex(".*they say*"))))
            .pause(1)
            .exec(ws("Close WS").close());

    {
        setUp(scene.injectOpen(
                atOnceUsers(1))
                .protocols(httpProtocol));
    }
}

```

... but nothing works the first time:

```
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=0      KO=2     )
> Connect WS                                               (OK=0      KO=1     )
> Say hi                                                   (OK=0      KO=1     )
---- Errors --------------------------------------------------------------------
> j.i.IOException: Premature close                                    1 (33.33%)
> Client issued text frame but server has closed the WebSocket a      1 (33.33%)
nd max reconnects is reached
> Close WS: Client issued close order but WebSocket was already       1 (33.33%)
crashed: j.i.IOException: Premature close
...
```

Well in this particular case, nothing worked after several hours of all kinds of attempts (which I will not present) until I managed to figure out all the pieces of the puzzle. And this is the reason I want to share with you my findings.

## Testing with Postman

To make sure that my server is working properly I used Postman and I created a new Socket.IO connection.

> Note
>
> Well, in reality I did many other things before trying out Postman and this is one of the major mistakes that I have done. I should have used a trusted tool to make sure that my server is working instead of trying things around.

When I created a new Socket.IO client in Postman I could successfully connect to the Socket.IO server, send and receive messages. But the Gatling simulation could not even connect to the server.

## Gatling URL for Socket.IO connections

So my first investigation involved finding out how to connect to the Socket.IO server. Searching for examples, I discovered a question in a forum (https://community.gatling.io/t/checking-socket-io-messages/7350) using the syntax:

```java
.exec(ws("Connect WS").connect("/socket.io/?EIO=4&transport=websocket"))
```

This time the Gatling output was:

```
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=2      KO=1     )
> Connect WS                                               (OK=1      KO=0     )
> Say hi                                                   (OK=1      KO=0     )
> checkMessage                                             (OK=0      KO=1     )
---- Errors --------------------------------------------------------------------
> WebSocket crashed while waiting for check:                          1 (50.00%)
> Close WS: Client issued close order but WebSocket was already       1 (50.00%)
crashed:
```

Gatling reported that `Connect WS` received an `OK` response. Even the `Say hi` received and `OK` response. But on the server side there was no log message for the connection or the reception of the message.

## Logging the Socket.IO server connection with Postman as a client

This is the part that took me the longest to figure out and I managed to solve it only after turning on the debug messages on the server side and trying the successful connection and transfer of messages with Postman.

On a terminal, I started the server with:

```sh
DEBUG="*" node server.js
```

```
  socket.io:server initializing namespace / +0ms
  socket.io:server creating http server and binding to 3000 +2ms
  socket.io:server creating engine.io instance with opts {"transports":["websocket","polling"],"cors":{"origin":"*","methods":["GET","POST"]},"cleanupEmptyChildNamespaces":false,"path":"/socket.io"} +3ms
  socket.io:server attaching client serving req handler +2ms
```

And then I connected with Postman. The debug messages are:

```
  engine applying middleware n°1 +0ms
  engine writing headers: {"Access-Control-Allow-Origin":"*"} +2ms
  engine handshaking client "_9SxGdR_2ReMDY5bAAAA" +3ms
  engine:transport readyState updated from undefined to open (websocket) +0ms
  engine:socket readyState updated from undefined to opening +0ms
  engine:socket readyState updated from opening to open +1ms
  engine:socket sending packet "open" ({"sid":"_9SxGdR_2ReMDY5bAAAA","upgrades":[],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}) +0ms
  engine:socket flushing buffer to transport +0ms
  engine:ws writing "0{"sid":"_9SxGdR_2ReMDY5bAAAA","upgrades":[],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}" +0ms
  engine:transport setting request +3ms
  socket.io:server incoming connection with id _9SxGdR_2ReMDY5bAAAA +1m
  engine:ws received "40" +2ms
  engine:socket received packet message +4ms
  socket.io-parser decoded 0 as {"type":0,"nsp":"/"} +0ms
  socket.io:client connecting to namespace / +0ms
  socket.io:namespace adding socket to nsp / +0ms
  socket.io:socket socket connected - writing packet +0ms
  socket.io:socket join room JnYOfDWvpdm1zrCEAAAB +0ms
  socket.io-parser encoding packet {"type":0,"data":{"sid":"JnYOfDWvpdm1zrCEAAAB"},"nsp":"/"} +2ms
  socket.io-parser encoded {"type":0,"data":{"sid":"JnYOfDWvpdm1zrCEAAAB"},"nsp":"/"} as 0{"sid":"JnYOfDWvpdm1zrCEAAAB"} +1ms
  engine:socket sending packet "message" (0{"sid":"JnYOfDWvpdm1zrCEAAAB"}) +3ms
  engine:socket flushing buffer to transport +0ms
  engine:ws writing "40{"sid":"JnYOfDWvpdm1zrCEAAAB"}" +4ms
a user connected to the socket.io server: JnYOfDWvpdm1zrCEAAAB
```

At the bottom I could see the log message from my server, indicating a successful connection. The interesting messages are those starting with `engine:ws`:

```
engine:ws writing "0{"sid":"_9SxGdR_2ReMDY5bAAAA","upgrades":[],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}" +0ms

engine:ws received "40" +2ms

engine:ws writing "40{"sid":"JnYOfDWvpdm1zrCEAAAB"}" +4ms
```

## Gatling: connecting to a Socket.IO server

So my guess was that the first interaction from Postman received a web-socket response with the number `0` and an object with the connection info. Even more interestingly, the websocket then received a `40` from Postman. And the answer was `40{"sid":"JnYOfDWvpdm1zrCEAAAB"}`.

At that moment I tested sending the message `40` in Gatling after the connect.

```java
    .exec(ws("Connect WS").connect("/socket.io/?EIO=4&transport=websocket"))
    .exec(ws("Connect to SocketIO").sendText("40"))
```

This time I also received the log message from the server on connection:

```
a user connected to the socket.io server: EDHRJJJhtwDADx9hAAAD
```

That was progress!

## Logging the Socket.IO messages with Postman as a client

I then had to see what happens with the other messages. So back to Postman, I sent a message `Hi` (with the default event name `message`). The output of the debug was:

```
  engine:ws received "42["message","Hi"]" +20s
  engine:socket received packet message +20s
  socket.io-parser decoded 2["message","Hi"] as {"type":2,"nsp":"/","data":["message","Hi"]} +5m
  socket.io:socket got packet {"type":2,"nsp":"/","data":["message","Hi"]} +5m
  socket.io:socket emitting event ["message","Hi"] +0ms
  socket.io:socket dispatching an event ["message","Hi"] +1ms
message: Hi
```

The last line was from the log from the server. Observing:

```
engine:ws received "42["message","Hi"]"
```

## Gatling: sending Socket.IO messages

I imitated the format in my message in Gatling:

```java
        .exec(ws("Say hi")
                .sendText("42[\"message\",\"Hi\"]")
```

Finally! I could see the server receiving my message and additionally no errors in the simulation:

```
message: Hi
```

```
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=5      KO=0     )
> Connect WS                                               (OK=1      KO=0     )
> Connect to SocketIO                                      (OK=1      KO=0     )
> Say hi                                                   (OK=1      KO=0     )
> checkMessage                                             (OK=1      KO=0     )
> Close WS                                                 (OK=1      KO=0     )
```

So what are these numbers? According to the Socket.IO documentation:

```
4 => the Engine.IO message type (version 4)
0 => the Socket.IO CONNECT type
2 => the Socket.IO EVENT type
```

So, in 40, 4 is for EIO=4 and 0 for connect. In 42, 4 is for EIO=4 and 2 for an event.

The next thing I wanted to solve was to receive the `broadcast` message sent by:

```js
socket.on("message", (msg) => {
  console.log("message: " + msg);
  io.emit("broadcast", "they say: " + msg);
});
```

Back to the debug log:

```
message: Hi
  socket.io-parser encoding packet {"type":2,"data":["broadcast","they say: Hi"],"nsp":"/"} +0ms
  socket.io-parser encoded {"type":2,"data":["broadcast","they say: Hi"],"nsp":"/"} as 2["broadcast","they say: Hi"] +0ms
  engine:socket sending packet "message" (2["broadcast","they say: Hi"]) +0ms
  engine:socket flushing buffer to transport +0ms
  engine:socket sending packet "message" (2["broadcast","they say: Hi"]) +0ms
  engine:socket flushing buffer to transport +0ms

```

So the message is emitted as:

```
2["broadcast","they say: Hi"]
```

## Gatling: receiving server messages

Actually it is emitted as `42["broadcast","they say: Hi"]`. I found this by adding in Gatling:

```java
            .exec(ws("Say hi")
                    .sendText("42[\"message\",\"Hi\"]")
                    .await(30).on(
                            ws.checkTextMessage("checkMessage")
                                    .check(regex(".*")
                                            .saveAs("response"))))
            .exec(session -> {
                System.out.println("response: " + session.get("response"));
                return session;
            })
```

```
response: 42["broadcast","they say: Hi"]
```

So, I could extract with a regular expression the exact response after the `broadcast` event.

```java
            .exec(ws("Say hi")
                    .sendText("42[\"message\",\"Hi\"]")
                    .await(30).on(
                            ws.checkTextMessage("checkMessage")
                                    .check(regex(".*broadcast...([^\"]*)")
                                            .saveAs("response"))))
```

## Closing the Socket.IO connection

A minor detail, but I noticed that closing the Socket.IO connection (with Postman) involved yet another text message:

```
  engine:ws received "41" +18s
  engine:socket received packet message +18s
  socket.io-parser decoded 1 as {"type":1,"nsp":"/"} +5m
  socket.io:socket got packet {"type":1,"nsp":"/"} +5m
  socket.io:socket got disconnect packet +0ms
  socket.io:socket closing socket - reason client namespace disconnect +0ms
user disconnected: JnYOfDWvpdm1zrCEAAAB
  engine:transport readyState updated from open to closed (websocket) +5m
  engine:socket readyState updated from open to closed +1ms
  socket.io:client client close with reason transport close +5m
```

So to properly disconnect, I have to send the text `41`:

```java
            .exec(ws("Disconnect from Socket.IO").sendText("41"))
            .exec(ws("Close WS").close())
```

## Complete simulation scenario

With this addition, my debug log when executing with Gatling was identical with the debug log when executing with Postman. Mystery revealed. My Gatling scenario looks like this:

```java
    ScenarioBuilder scene = scenario("WebSocket")
            .exec(ws("Connect WS").connect("/socket.io/?EIO=4&transport=websocket"))
            .exec(ws("Connect to Socket.IO").sendText("40"))
            .pause(1)
            .exec(ws("Say hi")
                    .sendText("42[\"message\",\"Hi\"]")
                    .await(30).on(
                            ws.checkTextMessage("checkMessage")
                                    .check(regex(".*broadcast...([^\"]*)")
                                            .saveAs("response"))))
            .exec(session -> {
                System.out.println("response: " + session.get("response"));
                return session;
            })
            .exec(ws("Disconnect from Socket.IO").sendText("41"))
            .exec(ws("Close WS").close());
```

The Socket.IO server sends ping text messages to check if the client is still alive. Thankfully, this is already implemented in Gatling with `wsAutoReplySocketIo4` which automatically replies to a ping TEXT frame with the corresponding pong TEXT frame:

```java
    HttpProtocolBuilder httpProtocol = http
            .wsBaseUrl("ws://localhost:3000")
            .wsAutoReplySocketIo4();
```

Hopefully, this article will save some time for some of you.

## Some useful links

- https://socket.io/docs/v4/socket-io-protocol/
- https://socket.io/docs/v4/troubleshooting-connection-issues/
