+++
title = 'Load testing Socket.IO servers using msgpack-parser with Gatling'
date = 2023-11-18T14:39:05+02:00
draft = false
showtoc = true
tocopen = false
tags = ['testing','gatling','socket-io','load-testing','message-pack']
+++

In this post I will show how we can connect with Gatling to a Socket.IO server which uses a custom parser using the MessagePack serialization format.

This is the second post about Socket.IO testing with Gatling.
In the first post:

- [Load testing Socket IO with Gatling](/posts/gatling-socketio)

I showed an example of connecting to a Socket.IO server using the default parser which uses text messages.

## What is MessagePack?

MessagePack is a binary serialization format. It has the advantage that payloads are usually smaller especially if they contain many numbers. The disadvantage is that Socket.IO messages serialized with MessagePack are harder to debug in the Network tab of the browser.

## The msgpack Socket.IO server

The server is the same as in the previous post with the only difference being the added customer parser option.

```js
const { Server } = require("socket.io");
const cors = require("cors");
const customParser = require("@skgdev/socket.io-msgpack-javascript");

const io = new Server({
  transports: ["websocket", "polling"],
  cors: {
    origin: "*",
    methods: ["GET", "POST"],
  },
  parser: customParser.build({
    encoder: {
      ignoreUndefined: true,
    },
  }),
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

io.listen(5555);
```

## The simulation scenario

I will use the same example that I had in my previous post. For quick reference here is the Gatling scenario using the default Socket.IO parser.

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

Exchanged messages were in a text format, `"40"` for connecting, `"42[....]"` for sending and receiving message frames and `"41"` for disconnecting.

Initially, I thought that I needed to use Message Pack in order to decode these text messages, but I found out that I was wrong (after several attempts and several hours).

## Required changes

### Sending and receiving data

According to the Socket.IO documentation https://socket.io/docs/v4/socket-io-protocol/#sending-and-receiving-data the exchange of data has the following payload:

```
{ type: EVENT, namespace: "/", data: ["foo"] }
```

So the text representation (above) is the result of the default parser encoding this payload.

I actually discovered that the payload description is not exactly correct. By debugging (this time using a msg-pack client I wrote, since Postman does not support binary messages), I found out that the namespace is actually represented with the `nsp` key in the payload. So the payload for a send event looks something like:

```json
{ "type": 2, "nsp": "/", "data": ["message", "Hi"] }
```

Of course the namespace and the data can be different. `"/"` is the main (default) namespace. The data array can hold many elements. Typically, the first element is the event name (that the server is listening to with `socket.on("message", ...`) and the rest are the exchanged data.

### Serialization with MessagePack for Java

In order to send binary serialized messages on the wire, I used the **[MessagePack for Java](https://github.com/msgpack/msgpack-java)** library.

The easiest way to create the JSON payload and decode it with MessagePack is to create a POJO for the payload:

```java
class Packet {
  public int type;
  public String nsp;
  public List<String> data;

  public Packet() {
  }

  public Packet(int type, String nsp, List<String> data) {
    this.type = type;
    this.nsp = nsp;
    this.data = data;
  }
}
```

then instantiate an `objectMapper`:

```java
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());
```

and serialize as bytes:

```java
byte[] packSendBytes;

try {
  packSendBytes = objectMapper.writeValueAsBytes(
      new Packet(2, "/", Arrays.asList("message", "hi")));
} catch (JsonProcessingException e) {
  e.printStackTrace();
}
```

Then, in the simulation, I used `sendBytes` (instead of `sendText`) to send the `byte[]` array:

```java
      .exec(ws("Say hi").sendBytes(packSendBytes)
```

### Connecting and disconnecting

The same `Packet` POJO class worked for the connection also:

```java
byte[] packConnectBytes;
// surround with try/catch
packConnectBytes = objectMapper.writeValueAsBytes(
    new Packet(0, "/", Arrays.asList()));
```

Unfortunately, it did not work for the disconnection message. The server did not except the _extra_ `data` field. I had to create a different POJO `DisconnectPacket` for this case. (I thought to use inheritance but ...)

```java
class DisconnectPacket {
  public int type;
  public String nsp;

  public DisconnectPacket(String nsp) {
    type = 1;
    this.nsp = nsp;
  }
}
```

```java
byte[] packDisconnectBytes;
// surround with try/catch
packDisconnectBytes = objectMapper.writeValueAsBytes(
    new DisconnectPacket("/"));
```

### Checking for the server message

The simulation should check that after a client's message to the `message` event, the server emits back a message to the `broadcast` event.

```java
  .exec(ws("Say hi")
      .sendBytes(packSendBytes)
      .await(30).on(
          ws.checkBinaryMessage("checkMessage")
              .check(checkBroadcastEventSaveMessage)))
```

Note the use of `checkBinaryMessage` instead of `checkTextMessage`.

Creating this check involved writing longer and a bit more complex code so I decided to put it in a separate `CheckBuilder checkBroadcastEventSaveMessage` variable. I also had to write my (first) Gatling [`transform`](https://gatling.io/docs/gatling/reference/current/core/check/#transforming) function to perform the check:

```java
  CheckBuilder checkBroadcastEventSaveMessage = bodyBytes()
      .transform(bytes -> {
        // process the received bytes array
        //    decode the data into a Packet POJO
        // check that the client sent to the `broadcast` event
        // and return the message
      })
      .saveAs("response");
```

It turned out a bit long but it is not difficult to understand:

```java
  CheckBuilder checkBroadcastEventSaveMessage = bodyBytes()
      .transform(bytes -> {
        Packet packet = null;

        try {
          packet = objectMapper.readValue(bytes, Packet.class);
        } catch (IOException e) {
          throw new RuntimeException(
              "Failed to deserialize incoming bytes to Packet object. ", e);
        }

        String eventName = packet.data.get(0);

        if (!eventName.equals("broadcast")) {
          throw new RuntimeException(
              "Expected broadcast event, got " + eventName + " instead");
        }

        String message = packet.data.get(1);
        return message;
      })
      .saveAs("response");
```

## The complete Gatling Simulation

If I put everything together, we get the following class:

```java
public class SimpleBinSocketIOSimulation extends Simulation {

  HttpProtocolBuilder httpProtocol = http
      .wsBaseUrl("ws://localhost:5555")
      .wsReconnect()
      .wsMaxReconnects(5)
      .wsAutoReplySocketIo4();

  byte[] packConnectBytes;
  byte[] packSendBytes;
  byte[] packDisconnectBytes;

  // Instantiate ObjectMapper for MessagePack
  ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

  /**
   * POJO for the packet object that is sent serialized to the server. Same packet
   * is deserialized from the server.
   */

  static class Packet {
    public int type;
    public String nsp;
    public List<String> data;

    public Packet() {
    }

    public Packet(int type, String nsp, List<String> data) {
      this.type = type;
      this.nsp = nsp;
      this.data = data;
    }
  }

  /**
   * A different POJO is needed for the disconnect packet because the server does
   * not expect the data field and responds with an error if it is present.
   */
  static class DisconnectPacket {
    public int type;
    public String nsp;

    public DisconnectPacket(String nsp) {
      type = 1;
      this.nsp = nsp;
    }
  }

  {
    try {
      packConnectBytes = objectMapper.writeValueAsBytes(
          new Packet(0, "/", Arrays.asList()));
      packSendBytes = objectMapper.writeValueAsBytes(
          new Packet(2, "/", Arrays.asList("message", "hi")));
      packDisconnectBytes = objectMapper.writeValueAsBytes(
          new DisconnectPacket("/"));
    } catch (JsonProcessingException e) {
      e.printStackTrace();
    }
  }

  CheckBuilder checkBroadcastEventSaveMessage = bodyBytes()
      .transform(bytes -> {
        Packet packet = null;

        try {
          packet = objectMapper.readValue(bytes, Packet.class);
        } catch (IOException e) {
          throw new RuntimeException(
              "Failed to deserialize incoming bytes to Packet object. ", e);
        }

        String eventName = packet.data.get(0);

        if (!eventName.equals("broadcast")) {
          throw new RuntimeException(
              "Expected broadcast event, got " + eventName + " instead");
        }

        String message = packet.data.get(1);
        return message;
      }).saveAs("response");

  ScenarioBuilder scene = scenario("WebSocket")
      .exec(ws("Connect WS")
          .connect("/socket.io/?EIO=4&transport=websocket"))
      .exec(ws("Connect to Socket.IO").sendBytes(packConnectBytes))
      .pause(1)
      .exec(ws("Say hi")
          .sendBytes(packSendBytes)
          .await(30).on(
              ws.checkBinaryMessage("checkMessage")
                  .check(checkBroadcastEventSaveMessage)))

      .exec(session -> {
        System.out.println("response: " + session.get("response"));
        return session;
      })
      .exec(ws("Disconnect from Socket.IO").sendBytes(packDisconnectBytes))
      .exec(ws("Close WS").close());

  {

    setUp(scene.injectOpen(atOnceUsers(1))).protocols(httpProtocol);
  }
}
```

Hopefully, this will be useful for somebody experimenting with Gatling and Socket.IO servers using MessagePack. It was definitely fun for me to complete this small project and write this post.

All this work is part of a project in which I am building a complex Gatling simulation. Probably more things will come out of this project and I will find more opportunities to write and share a post.

Thanks for reading!

## Disclaimer

The presented code is not optimal and certainly nor reusable. In a real project I would define the POJO classes in different files and I would create reusable abstractions for the serialization/deserialization and the exchange of messages. I wanted to keep the example very basic and have everything in a single class so that it can easily be copied and reproduced.

## Links

- https://msgpack.org/
- https://socket.io/docs/v4/custom-parser/#the-msgpack-parser
- https://github.com/msgpack/msgpack-java
- https://gatling.io/docs/gatling/
