# Backend Handlers

Handlers define externally handled application, including:
 * Format of the uplink and downlink messages
 * Data fields forwared via the backend *Connectors*
 * Retransmission logic for confirmed downlinks

Each Handler may be linked with one or more backend *Connectors*, which handle
the communication towards the backend server.

The Handler will process every uplink frame and forward it to the backend. It
will also process every downlink request received from the backend.

In addition to uplink frames the backend can receive device related events:
 * when a device **joined**
 * when a confirmed frame was **delivered**
 * when a confirmed frame was **lost**
 * for a connection **test**


## Administration
![alt tag](https://raw.githubusercontent.com/gotthardp/lorawan-server/master/doc/images/admin-handler.png)

To create a new handler you need to set:
 - **Application** name
 - **Uplink Fields** that will be forwarded to the backend Connector
 - **Parse Uplink** function to extract additional data fields from the uplink frame
 - **Parse Event** function to amend event data fields
 - **Build Downlink** function to create a downlink frame based on backend data fields
 - **D/L Expires** defines when the downlinks may be dropped.
   - **Never** means that:
     * All class A downlinks for a device will be queued and eventually delivered.
     * All confirmed downlinks will be retransmitted until acknowledged, even
       when a new downlink is sent.
   - **When Superseded** means that:
     * Only the most recent class A downlink will be scheduled for delivery.
       Superseded downlinks will be dropped.
     * Unacknowledged downlinks will be dropped when a new downlink (either
       class A or C) is sent.

**Test** button can be used to send a `test` event to all connections associated with this
handler.

**Connectors** related to this Handler are displayed for your convenience. The table
lists all backend connectors with the same *Application* name.


## Fields

### Uplink

Depending on the *Uplink Fields* settings the server sends to backend
applications the following fields:

  Field      | Type        | Usage  | Meaning
 ------------|-------------|--------|----------------------------------------------
  app        | String      | U J DL | Application (Handler) name.
  devaddr    | Hex String  | U J DL | DevAddr of the active node.
  deveui     | Hex String  | U J DL | DevEUI of the device.
  appargs    | Any         | U J DL | Application arguments for this node.
  battery    | Integer     | U      | Most recent battery level reported by the device.
  fcnt       | Integer     | U      | Received frame sequence number.
  port       | Integer     | U      | LoRaWAN port number.
  data       | Hex String  | U      | Raw application payload, encoded as a hexadecimal string.
  datetime   | ISO 8601    | U J DL | Timestamp using the server clock.
  freq       | Number      | U      | RX central frequency in MHz (unsigned float, Hz precision).
  datr       | String      | U      | LoRa datarate identifier (eg. "SF12BW500").
  codr       | String      | U      | LoRa ECC coding rate identifier (usually "4/5").
  best_gw    | Object      | U      | Gateway with the strongest reception.
  all_gw     | Object List | U      | List of all gateways that received the frame.
  receipt    | Any         | DL     | Custom data sent along the confirmed downlink.

The fields 'U' can be included in Uplink messages, fields 'J' in Join events,
fields 'DL' in Delivered and Lost events.

The Gateway object included in *best_gw* and *all_gw* has the following fields:

  Field      | Type        | Explanation
 ------------|-------------|-------------------------------------------------------------
  mac        | Hex String  | MAC address of the gateway that received the frame.
  rxq        | Object      | Indicators of the reception quality as indicated in the `rxpk` structure by the gateway (see Section 4 of the [packet_forwarder protocol](https://github.com/Lora-net/packet_forwarder/blob/master/PROTOCOL.TXT).
  rxq.lsnr   | Number      | LoRa uplink SNR ratio in dB (signed float, 0.1 dB precision)
  rxq.rssi   | Number      | RSSI in dBm (signed integer, 1 dB precision)
  rxq.tmst   | Number      | Internal timestamp of "RX finished" event (32b unsigned) used for response scheduling; it doesn't indicate any calendar date.

For example:
```json
    {"devaddr":"11223344", "port":2, "fcnt":58, "data":"0125D50B020BA23645F1A90BDDEE0004",
        "shall_reply":false, "last_lost":false,
        "rxq":{"lsnr":9.2,"rssi":-53,"tmst":3127868932,"codr":"4/5","datr":"SF12BW125","freq":868.3}}
```

### Downlink

The client may send back to the server the following fields:

  Field       | Type        | Explanation
 -------------|-------------|-------------------------------------------------------------
  app         | String      | Application (Handler) name.
  deveui      | Hex String  | DevEUI of the device.
  devaddr     | Hex String  | DevAddr of the active node.
  port        | Integer     | LoRaWAN port number. If not specified for Class A, the port number of last uplink will be used. Mandatory for Class C.
  time        | ISO 8601    | Requested downlink time or `immediately` (for class C devices only).
  data        | Hex String  | Raw application payload, encoded as a hexadecimal string.
  confirmed   | Boolean     | Whether the message shall be confirmed (false by default).
  pending     | Boolean     | Whether the application has more to send (false by default).
  receipt     | Any         | Custom data to receive in the in Delivered and Lost events.

For example:
```json
    {"devaddr":"11223344", "data":"0026BF08BD03CD35000000000000FFFF", "confirmed":true}
```
Or (for class C devices only):
```json
    {"data":"00", "port":2, "time":"2017-03-04T21:05:30.2000"}
    {"data":"00", "port":2, "time":"immediately"}
```


## Parse Uplink

The *Parse Uplink* is an Erlang function that converts binary data to custom
data fields and can extend (or even amend) the *Uplink Fields*. It shall be a
[Fun Expression](http://erlang.org/doc/reference_manual/expressions.html#funs)
with two parameters, which matches the
[binary data](http://erlang.org/doc/programming_examples/bit_syntax.html)
and returns an
[Erlang representation of JSON](https://github.com/talentdeficit/jsx#json---erlang-mapping).

For example:

```erlang
fun(Fields, <<LED, Press:16, Temp:16, AltBar:16, Batt, Lat:24, Lon:24, AltGps:16>>) ->
  Fields#{led => LED, pressure => Press, temp => Temp/100, alt_bar => AltBar, batt => Batt}
end.
```

The `<<A, B, C>>` is a binary pattern, where A, B, C are "variables" corresponding
to the values encoded in the binary. Erlang matches the incoming binary data against
this pattern and fills the "variables" with the values in the binary. Here are some
examples:
 * `<<A>>` matches 1 value, 1 byte long.
 * `<<A, B>>` matches 2 values, each 1 byte long.
 * `<<A:16>>` matches 1 unsigned int value, 2 bytes long in big-endian
 * `<<A:16/little-signed-integer>>` matches 1 signed int value, 2 byes long in little-endian
 * `<<A:2/binary>>` matches an array of 2 bytes

To match a variable sized array of bytes you can do:

```erlang
fun(Fields, <<Count, Data:Count/binary>>) ->
  Fields#{data => binary_to_list(Data)}
end.
```

The `#{name1 => A, name2 => B, name3 => C}` creates a `fields` attribute with
the JSON `{"name1":A, "name2":B, "name3":C}`.

## Parse Event

The *Parse Event* is an Erlang function that converts event name to custom
data fields and can extend (or even amend) the *Uplink Fields*.

To generate events like `{"devaddr":"00112233", "event":"joined"}` you can write:

```erlang
fun(Vars, Event) ->
  Vars#{event => Event}
end.
```

Alternatively, to generate `{"joined":{"devaddr":"00112233"}}` write:

```erlang
fun(Vars, Event) ->
  #{Event => Vars}
end.
```

## Build Downlink

*Build Downlink* works in the opposite direction. It takes the data fields and
constructs the binary payload. It shall be a
[Fun Expression](http://erlang.org/doc/reference_manual/expressions.html#funs)
with two parameters, which gets an
[Erlang representation of JSON](https://github.com/talentdeficit/jsx#json---erlang-mapping)
and returns
[binary data](http://erlang.org/doc/programming_examples/bit_syntax.html).
If you send `{"fields":{"led":1}}`, you can have a function like this:

```erlang
fun(#{led := LED}) ->
  <<LED>>
end.
```

The `#{name1 := A, name2 := B, name3 := C}` matches the `fields` attribute containing
a JSON structure `{"name1":A, "name2":B, "name3":C}`. The order is not significant,
but all fields are mandatory.

The binary is then built using similar approach as the pattern matching
explained above. For example, `<<A, B, C>>` builds a binary of three 1-byte integers.

To build a variable sized array you can do:

```erlang
fun(#{data := Data}) ->
  <<(length(Data)), (list_to_binary(Data))/binary>>
end.
```