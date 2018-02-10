
MQTT-JSON-RPC
=============

[JSON-RPC](http://www.jsonrpc.org/) protocol over [MQTT](http://mqtt.org/) communication.

<p/>
<img src="https://nodei.co/npm/mqtt-json-rpc.png?downloads=true&stars=true" alt=""/>

<p/>
<img src="https://david-dm.org/rse/mqtt-json-rpc.png" alt=""/>

Installation
------------

```shell
$ npm install mqtt mqtt-json-rpc
```

About
-----

This is a small wrapper around the
[MQTT.js](https://www.npmjs.com/package/mqtt) API of
[Node.js](https://nodejs.org/), for Remote Procedure Call (RPC)
communication based on the [JSON-RPC](http://www.jsonrpc.org/)
protocol. It allows a request/response-style communication over
the plain message protocol [MQTT](http://mqtt.org).

Internals
---------

Internally, remote methods are assigned to MQTT topics. When calling a
remote method named `example/hello` with parameters "world" and 42 via...

```js
rpc.call("example/hello", "world", 42).then((result) => {
    ...
})
```

..the following JSON-RPC 2.0 request message is sent to the permanent MQTT
topic `example/hello/request`:

```json
{
    "jsonrpc": "2.0",
    "id":      "d1acc980-0e4e-11e8-98f0-ab5030b47df4:d1db7aa0-0e4e-11e8-b1d9-5f0ab230c0d9",
    "method":  "example/hello",
    "params":  [ "world", 42 ]
}
```

Beforehand, this `example/hello` method should have been registered with...

```js
rpc.register("example/hello", (a1, a2) => {
    return `${a1}:${a2}`
})
```

...and then its result, here `"world:42"`, is then
sent back as the following JSON-RPC 2.0 success response
message to the temporary (client-specific) MQTT topic
`example/hello/response/d1acc980-0e4e-11e8-98f0-ab5030b47df4`:

```json
{
    "jsonrpc": "2.0",
    "id":      "d1acc980-0e4e-11e8-98f0-ab5030b47df4:d1db7aa0-0e4e-11e8-b1d9-5f0ab230c0d9",
    "result":  "world:42"
}
```

The JSON-RPC 2.0 `id` field always consists of `<cid>:<rid>`, where
`<cid>` is the UUID v1 of the MQTT-JSON-RPC instance and `<rid>` is
the UUID v1 of the particular method request. The `<cid>` is used for
sending back the JSON-RPC 2.0 response message to the requestor only.

Usage
-----

### Server:

```js
let mqtt    = require("mqtt")
let mqttRPC = require("./mqtt-json-rpc")

let mqttClient = mqtt.connect("wss://127.0.0.1:8889", { ... })
let rpc = new mqttRPC(mqttClient)

rpc.on("connect", () => {
    rpc.register("example/hello", (a1, a2) => {
        console.log("example/hello: request: ", a1, a2)
        return `${a1}:${a2}`
    })
})
```

### Client:

```js
let mqtt    = require("mqtt")
let mqttRPC = require("./mqtt-json-rpc")

let mqttClient = mqtt.connect("wss://127.0.0.1:8889", { ... })
let rpc = new mqttRPC(mqttClient)

rpc.on("connect", () => {
    rpc.call("example/hello", [ "world", 42 ], (err, data) => {
        console.log("example/hello response: ", data)
        rpc.end()
    })
})
```

Application Programming Interface
---------------------------------

The API of MQTT-JSON-RPC is a superset of the original
[MQTT.js](https://www.npmjs.com/package/mqtt) API because it is just a
wrapper around it with the following additional methods:

- `MQTT-JSON-RPC#register(method: string, callback: (...args: any[]) => any): void`:<br/>
  Register a method. The `method` has to be a valid MQTT topic
  name. The `callback` is called with the `params` passed to
  `MQTT-JSON-RPC#notify()` or `MQTT-JSON-RPC#call()`. For
  `MQTT-JSON-RPC#notify()`, the return value of `callback` will be
  ignored. For `MQTT-JSON-RPC#call()`, the return value of `callback`
  will resolve the promise returned by `MQTT-JSON-RPC#call()`.
  Internally, on the MQTT broker the topic `${method}/request` is
  subscribed.

- `MQTT-JSON-RPC#unregister(method: string): void`:<br/>
  Unregister a previously registered method.
  Internally, on the MQTT broker the topic `${method}/request` is unsubscribed.

- `MQTT-JSON-RPC#notify(method: string, ...params: any[]): void`:<br/>
  Notify a method. The remote `MQTT-JSON-RPC#register()` `callback` is called
  with `params` and its return value silently ignored.

- `MQTT-JSON-RPC#call(method: string, ...params: any[]): Promise`:<br/>
  Call a method. The remote `MQTT-JSON-RPC#register()` `callback` is
  called with `params` and its return value resolves the returned
  `Promise`. If the remote `callback` throws an exception, this rejects
  the returned `Promise`. Internally, on the MQTT broker the topic
  `${method}/response/<cid>` is temporarily subscribed for receiving the
  response.

License
-------

Copyright (c) 2018 Ralf S. Engelschall (http://engelschall.com/)

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

