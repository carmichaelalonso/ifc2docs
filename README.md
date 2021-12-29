> This document has been written by [@likeablegeek](https://www.likeablegeek.com) to extend the documentation for the [Infinite Flight Connect API v2](https://infiniteflight.com/guide/developer-reference/connect-api/version-2) as a contribution to the Infinite Flight developer community. They may be freely shared under the [Creative Commons Attribution 4.0 International (CC BY 4.0) license](https://creativecommons.org/licenses/by/4.0/).

# Connect API v2

This version of the API was first included in Infinite Flight 19.4.

## About the API

The Connect API v2 is designed to provide a high-performance way to interrogate aircraft and Infinite Flight states, manipulate those states and issue commands to Infinite Flight. The API is designed to be highly responsive so that it can be used for a variety of purposes including controlling aspects of Infinite Flight in "real time" while flights are in progress and creating applications which can respond rapidly to changes in aircraft state.

For this reason, a traditional HTTP-based REST API is not appropriate -- it involves sending larger amounts of data across the network and includes overhead for establishing and negotiating HTTP connects ill-suited to this type of application.

Instead, the Connect API v2 is a [TCP socket](https://www.easytechjunkie.com/what-is-a-tcpip-socket.htm) API. A TCP socket is a network-level mechanism used for direct communication between two devices on a TCP/IP network. It allows for a raw form of communication betwen devices where any series of bytes can be sent from one device or another.

By doing this, the Connect API v2 is able to use a compact, terse form of communication between the client and Infinite Flight device maximising the data that can be exchanged in the least time possible.

In simple terms, what this means is you send commands or requests to the API as a series of bytes which are structured as outline below. Where the API has to return data, it will similarly return a series of bytes which then need to be "decoded" to obtain the actual information being sent.

## Enabling the API

You can enable or disable the Connect API in Infinite Flight. Before trying to connect to an Infinite Flight device using the API, go to `Settings > General` in Infinite Flight and ensure that `Enable Infinite Flight Connect` is checked.

## Connecting to the API

To connect to the Connect API v2, you need to make a TCP socket connection to your Infinite Flight device on port `10112` using the IP address of the device.

How you do this depends on the language/platform you are using to connect to the API.

Common examples include:

* JavaScript (Node): Node's [`net` module](https://nodejs.org/api/net.html)
* C#: [TcpClient](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcpclient?view=net-6.0)
* Python: [`socket` module](https://docs.python.org/3/library/socket.html)
* Swift: [SwiftSocket](https://github.com/swiftsocket/SwiftSocket)
* Kotlin: [`java.net.socket`](https://sylhare.github.io/2020/04/07/Kotlin-tcp-socket-example.html)

### Finding an Infinite Flight device

If you don't know the IP address of the device you want to connect to, you can discover existing Infinite Flight devices on the same local network using UDP. Infinite Flight broadcasts [UDP](https://www.cloudflare.com/en-gb/learning/ddos/glossary/user-datagram-protocol-udp/) packets on port `15000` which provide the IP address of the device among other details. An example UDP broadcast from Infinite Flight looks like this:

```
["State": Playing, "Port": 10111, "DeviceID": iPad7, "Aircraft": Cessna 172,
"Version": 19.4.7354.25209, "DeviceName": Thomas’s iPad,
"Addresses":("fe80::1c79:baf4:f9f1:dd59%3", "192.168.1.26"),
"Livery": Civil Air Patrol]
```

From this information you can extract IPv4 or IPv6 addresses for the device as well as information about the version of Infinite Flight in use, the current aircraft and livery, and the type of device. The port indicated will be `10111` which is the port for the [Connect API v1](https://infiniteflight.com/guide/developer-reference/connect-api/version-1). **To connect to the v2 API, connect to port `10112` as mentioned above in "Connecting to the API".**

## Using the API

The Connect API v2 offers two mechanisms to interact with Infinite Flight:

* **States**: States are used to retrieve or set specific aspects of a flight, aircraft's or Infinite Flight's current active configuration. These are broken down into several groupings:
  * `aircraft`: Everything from the aircraft's altitude, heading, bank and pitch to the position of flaps, the aircraft's livery, and autopilot settings.
  * `infiniteflight`: Settings related to the state of Infinite Flight itself such as the current camera views and angles as well as the current version of Infinite Flight being used.
  * `api_joystick`: A set of states related to joystick support in Infinite Flight.
  * Miscellaneous: Miscellaneous other states related to the environment (such as wind speed), the simulator itself (such as current length of flight) and other one-off states.
* **Commands**: Commands are used to replicate actions typically taken in the Infinite Flight user interface such as toggling the parking brakes, moving the camera, starting and stopping engines, moving flaps, raising landing gear and more.

### The API Manifest

Each aircraft offers a different set of states and commands so before using the states or commands it is important to first obtain the manifest from the API after connecting.

The manifest is a long  text string containing a series of entries -- one per command or state. Each entry is separated by a newline character.

Each entry contains three fields separated by a comma:

* A numeric ID (32-bit integer) for the state or command which is used to get or set a state or execute a command through the API. These numbers can differ by aircraft.
* A 32-bit integer indicating the data type used by the state (see "[Data Types](#data-types)" below). For commands, this field will be `-1` rather than an integer indicating a data type.
* The name of the command or state - a human-friendly way to understand the comamnd's intent. These names will be consistent between aircraft even if the specific set of states and commands available might differ.

The following is an extract from a typical manifest illustrating this format for several states:

```
632,4,aircraft/0/flightplan/route\n
539,2,aircraft/0/groundspeed\n
548,2,aircraft/0/heading_magnetic\n
556,0,aircraft/0/is_on_ground\n
554,3,aircraft/0/latitude\n
555,3,aircraft/0/longitude\n
...
```

> *In this example, the text is split after each newline (`\n`) for readability -- the actual text won't have have an additional line break after the newline character.*

As an example, if we take the first entry, these are the details of the state:

* Numeric ID: `632`
* Data type: `4` (See "[Data Types](#data-types)" below to understand what type this refers to -- in this case a string)
* Command name: `aircraft/0/flightplan/route`

What's notable is that all state names take a format of a series of terms separated by forward slash (`/`) characters. In this case we can derive that this indicates:

* `aircraft`: The state refers to the aircraft itself
* `flightplan`: The state refers to the current flightplan
* `route`: The state refers to the route defined in the flightplan

The following sample illustrates the way commands will appear in the manifest:

```
> It is important to note that currently there are commands which cannot be used -- specifically commands which require data to be passed to them. To illustrate, the command `commands/ParkingBrakes` is a simple toggle: issue the command and the parking brakes switch between off and on. But, other commands clearly don't work that way. For instance, the command `commands/FlightPlan.AddWaypoints` requires a series of waypoints to be provided -- but there is no mechanism in the API to do that currently which renders the commands effectively non-functional at this time. it is expected that in the future Infinite Flight will enable these commands.
s/AutoStart\n
1048628,-1,commands/BeaconLights\n
1048613,-1,commands/Brakes\n
...
```

What's notable is that there are some key factors which distinguish commands from states in the manifest:

1. The command ID will be larger than 1,000,000 -- unlike states which have command IDs which are all smaller numbers
2. The data type will be specified as `-1` -- which is not an actual data type
3. The names will all take the format `commands/...` -- no states will have names which start with `commands`

> It is important to note that currently there are commands which cannot be used -- specifically commands which require data to be passed to them. To illustrate, the command `commands/ParkingBrakes` is a simple toggle: issue the command and the parking brakes switch between off and on. But, other commands clearly don't work that way. For instance, the command `commands/FlightPlan.AddWaypoints` requires a series of waypoints to be provided -- but there is no mechanism in the API to do that currently which renders the commands effectively non-functional at this time. it is expected that in the future Infinite Flight will enable these commands.

### The Structure of API Requests

All requests to the API take a common form -- a series of bytes broken down as follows:

* A 32-bit integer indicating the numeric ID of the state or command being requested.
* A one-byte boolean value indicating if the request includes and data being sent: `true` (represented as `1`) indicating that the state/command is followed by data being sent, or `false` (represented as `0`) indicating that there is no data to follow.

In practice this means that if you are getting a state or executing a command from the manifest, you will send the numeric ID followed by a `0`. If you are setting a state you will send the numeric ID followed by a `1` followed by some data.

For example, referring to the sample manifest examples above, if you wanted to get the current aircraft latitude you need to use the `aircraft/0/latitude` which has a numeric ID of `554`. This means sending `554` as a 32-bit integer followed by a single byte `0`.

This means sending the following five bytes:

```
2A 02 00 00 00
```

When the API returns data it will be returned as a series of bytes as follows:

* A 32-bit integer indicating the numeric ID of the state being sent.
* The one-byte boolean `false` (represented as `0`).
* A series of one or more bytes containing the data being sent for the state (the specific length and format of the data will depend on the data type for the command per "[Data Types](#data-types)" below).

It is important to realise that using a TCP socket means the dialog with the API is not a sequential send-receive. When you use HTTP to fetch a REST API endpoint, for instance, the HTTP transaction is a full round-trip:

1. Connect to the API endpoint by HTTP
2. Send the request
3. Receive the response
4. Disconnect

But with a TCP socket the request we send is disconnected from the response. It is entirely possible for two responses to be sent back-to-back before any response is sent by the API meaning we need to be associate the response to the relevant request at the client end. Because the API provides numeric ID of the state being returned, this allows is to manage and associate our requests with responses from the API if that is required at the client.

> This series of bytes is represented in hexadecimal notation each from `00` to `FF`.

Here, the first four bytes represent a 32-bit integer (in little endian notation -- more on that later in "[Data Types](#data-types)"). This means the actual 32-bit integer in hexadecimal is `0000022A` and `22A` hexadecimal is `554` in decimal notation. The fifth byte represents the `0` or `false` marker.

More specific examples are provided below as we discuss how to get and set states and run commands.

### Obtaining the Manifest

Typically, the first thing you will do once you successfully connect to the TCP socket is obtain the manifest.

This is done by sending the special command `-1` followed by `false` (or `0`) to indicate that no data is being sent with the request. The `-1` command is the API's command for requesting the manifest.

In practice this means sending this series of bytes:

```
ff ff ff ff 00
```

Here, `ff ff ff ff` is the 32-bit little endian hexadecimal representation of `-1` and the fifth byte is the `0` marker.

The API will return the manifest in this way:

* `-1` to indicate the API is returning manifest data, followed by
* The manifest data itself broken down as follows:
  * A 32-bit integer indicating the total length of the manifest
  * The manifest string itself broken down into two parts:
    * A 32-bit integer indicating the length of the string itself in bytes
    * The string data itself as a series of bytes

As an example, the following is the first 50 bytes of a manifest returned by the API:

```
ff ff ff ff 13 b7 00 00 0f b7
00 00 35 31 35 2c 32 2c 61 69
72 63 72 61 66 74 2f 30 2f 73
79 73 74 65 6d 73 2f 6e 61 76
5f 73 6f 75 72 63 65 73 2f 61
64 66 2f 32 2f 64 69 73 74 61
...
```

If we break this down, the data is structured like this:

* `ff ff ff ff`: This is the representation of the integer `-1` indicating this is a response to the request for the manifest.
* `13 b7 00 00`: This is the size of the total manifest data as a 32-bit integer -- in this case the hexadecimal number `0000b713` indicates the total manifest data to follow is 46,867 bytes.
* `0f b7 00 00`: This is the first part of the manifest data: a 32-bit integer indicating the length of the manifest string which follows. In this case the hexadecimal number `0000b70f` which indicates the length of manifest string is 46,863 bytes.
* `35 31 35 2c 32 2c 61 69 72 63 72 61 66 74 2f 30 2f 73 ...`: The start of the actual bytes of the manifest string. This series of bytes are the hexadeciaml representations of ASCII characters -- in this case these bytes are `515,2,aircraft/0/s ...` which is clearly the first part of an entry in the manifest.

An important point is that you will not receive the entire manifest in one large message. You will likely receive multiple messages so will need to append these messages to the end of the manifest data until you have received the entire string length indicated in the third integer (`0f b7 00 00` in this example).

How you do this will differ based on the specific language and platform you are using but the basic logic holds: continue to append messages to the end of the manifest until you have received the entire indicated length.

### Data Types

As discussed previously, states in the Manifest each have an associated data type indicated by a 32-bit integer value. There are six such data types:

<table>
<thead>
<tr>
<th>Integer</th><th>Type</th><th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td data-label="Integer">0</td><td data-label="Type">Boolean</td><td>Single byte representing `true` with `01` and `false` with `00`.</td></tr>
<tr><td data-label="Integer">1</td><td data-label="Type">Integer (32-bit)</td><td>32-bit Integer represented as four bytes in little endian format (see "<a href="#little-endian">Little-Endian</a>" below). Can store numbers in the range -2,147,483,648 to 2,147,483,647.</td></tr>
<tr><td data-label="Integer">2</td><td data-label="Type">Float</td><td>Floating point number represented as four bytes. Can represent 6 to 7 decimal digits.</td></tr>
<tr><td data-label="Integer">3</td><td data-label="Type">Double</td><td>Floating point number represented as eight bytes. Can represent 15 decimal digits.</td></tr>
<tr><td data-label="Integer">4</td><td data-label="Type">String</td><td>String represented as a 32-bit integer (four bytes) indicating the length of the string in bytes followed by the string itself as a series of bytes.</td></tr>
<tr><td data-label="Integer">5</td><td data-label="Type">Long</td><td>64-bit integer represented as eight bytes in little endian format (see "<a href="#little-endian">Little-Endian</a>" below). Can store numbers in the range -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807.</tr>
</tbody>
</table>

#### Little-Endian

*[Endianness](https://en.wikipedia.org/wiki/Endianness)* refers to the way in which data is ordered when represented digitally. Taken in the context of hexadecimal representation of numbers, numbers can be *big-endian* (or BE) or *little-endian* (or LE).

Consider this hexadecimal number representing the 32-bit integer 1,210,590:

```
001278DE
```

*Big-endian* means the bytes are represented left-to-right from the most significant byte to least significant as in:

```
00 12 78 DE
```

By comparison, ~little-endian* reverses this order to:

```
DE 78 12 00
```

The Connect v2 API represents numbers in little-endian format so it is important when sending requests to represent states or commands as 32-bit *little-endian* integers. For instance, if state you are requesting has the numeric ID `535`, then its hexadecimal value is `217` which as a 32-bit *little-endian* integer is represented by:

```
17 02 00 00
```

Similarly, if the command you are requesting has the numeric ID `1048616`, then its hecadecimal value is `100028` which as a 32-bit *little-endian* integer is represented by:

```
28 00 10 00
```

All numbers use little-endian including *Float,*, *Double,* and *Long* data types.

### Retrieving States from the API

To fetch a state from the API you need to send a `GetState` request as follows:

* A 32-bit integer indicating the numeric ID of the state being sent.
* The one-byte boolean `false` (represented as `0`).

For instance, let's assume our manifest contains the following state:

```
522,4,aircraft/0/livery
```

This indicates the `aircraft/0/livery` state has a numeric ID of `522` and returns a `String` data type.

If we want to retrieve this state we would send the following:

```
0a 02 00 00 00
```

This breaks down as:

* `0a 02 00 00`: The 32-bit little endian integer representation of `522` which has the hexadecimal value `20A`.
* `00`: A single byte representing `false`.

When the API responds it will return a response which looks like this:

```
0a 02 00 00 0e 00 00 00 0a 00 00 00 41 65 72 20 4c 69 6e 67 75 73
```

This breaks down as:

* `0a 02 00 00`: The 32-bit little-endian integer representation of the state's numeric ID `522`.
* `0e 00 00 00`: The length of the data being returned -- in this case the hexadecimal value `0000000e` is `14` indicating the data is 14-bytes long.
* `0a 00 00 00 41 65 72 20 ....`: The actual data returned for the state. Since this state is a string, the data breaks down into two parts:
  * `0a 00 00 00`: A 32-bit little-endian integer indicating the length of the string -- in this case `0000000a` ia `10` indicating the string is 10-bytes long.
  * `41 65 72 20 4c 69 6e 67 75 73`: The 10-byte string as a series of one-byte characters which represents `Aer Lingus` -- the name of the livery of the aircraft in this case.

This structure of request and response works regardless of the data type in question.

Let's look at an example with a 32-bit integer data type:

```
622,1,aircraft/0/systems/flaps/state
```

If we want to retrieve this state we would send the following:

```
6e 02 00 00 00
```

This breaks down as:

* `63 02 00 00`: The 32-bit little endian integer representation of `622` which has the hexadecimal value `26E`.
* `00`: A single byte representing `false`.

When the API responds it will return a response which looks like this:

```
02 00 00 04 00 00 00 00 00 00 00>
```

This breaks down as:

* `6e 02 00 00`: The 32-bit little-endian integer representation of the state's numeric ID `622`.
* `04 00 00 00`: The length of the data being returned -- in this case the hexadecimal value `00000004` is `4` indicating the data is 4-bytes long since 32-bit integers are represented as four bytes.
* `00 00 00 00`: The actual data returned for the state. Since this state is a 32-bit integer, the value being returned is `0`.

### Setting States with the API

It is possible to set states -- assigning new values to them -- through the API by sending a `SetState` request as outline below.

However, not all states can be set and the manifest offers no indication of which states can be set and which can't. The only way you can determine this is trial-and-error and some common sense.

For instance, a state such as `aircraft/0/livery` can't be set -- it represents the actual livery on the aircraft. Similarly, the state `aircraft/0/latitude` can't be set as it reflects the actual location of the aircraft at a point in time.

By contrast, a state such as `aircraft/0/systems/flaps/state` can be set to change the flap position by setting the state to the number representing the intended flap state (which will differ by aircraft).

To illustrate this, let's set the flap state to `1` using the API. To do this, we need to generate a `SetState` request we send:

* The 32-bit integer representing the numeric ID of the state
* A boolean value of `true` represented by a one-byte value `1`
* The actual data itself -- in this case the number `1` as a 32-bit little-endian integer

As with our previous example retrieving the same state, if the numeric ID of the state is `622` then we would send the following to set the state to `1`:

```
6e 02 00 00 01 01 00 00 00
```

This breaks down as:

* `6e 02 00 00`: The 32-bit little-endian integer representation of the state's numeric ID `622`.
* `01`: A single byte representing `true`.
* `01 00 00 00`: The 32-bit integer representation of the value we want to set for the state (in this case, `1`).

As with retrieving states, the way we represent the value being set will follow the way each [Data Type](#data-types) is represented.

For instance, we can try setting the following state which has a `String` data type:

```
605,4,aircraft/0/systems/comm_radios/com_1/atc_name
```

If we wanted to set this value to `Bob the Pilot`, we would send the following request to the API:

```
5d 02 00 00 01 0d 00 00 00 42 6f 62 20 74 68 65 20 50 69 6c 6f 74
```

This breaks down as:

* `5d 02 00 00`: The 32-bit little-endian integer representation of the state's numeric ID `605`.
* `01`: A single byte representing `true`.
* `0d 00 00 00 42 6f 62 20 ...`: The string data broken down into two parts:
  * `0d 00 00 00`: A 32-bit little-endian integer indicating the length of the string -- in this case `0000000d` ia `13` indicating the string is 13-bytes long (the number of characters in `Bob the Pilot`).
  * `42 6f 62 20 74 68 65 20 50 69 6c 6f 74`: The 13-byte string as a series of one-byte characters which represents `Bob the Pilot`.

After sending a request to set a state, the API will not provide any response to indicate the state was set successfully. The only way to be sure if the request succeeded is to subsequently [retrieve the state from the API](#retrieving-states-from-the-api).

### Running Commands through the API

To execute a command through the API you need to send a `RucCommand` request as follows:

* A 32-bit integer indicating the numeric ID of the command being executed.
* The one-byte boolean `false` (represented as `0`).

For instance, let's assume our manifest contains the following command:

```
1048614,-1,commands/ParkingBrakes
```

This indicates the `commands/ParkingBrakes` state has a numeric ID of `1048614`.

If we want to execute this command we would send the following:

```
26 00 10 00 00
```

This breaks down as:

* `26 00 10 00`: The 32-bit little endian integer representation of `1048614` which has the hexadecimal value `100026`.
* `00`: A single byte representing `false`.

This will execue the command and toggle the parking brakes on/off and the user will see this change in Infinite Flight.

As with setting a state, after sending a request to execute a command, the API will not provide any response to indicate the command was executed successfully. The only way to be sure if the request succeeded is to subsequently [retrieve an appropriate state from the API](#retrieving-states-from-the-api) or to visually confirm in Infinite Flight itself.

For instance, if you toggle the parking brakes state as illustrated above you might retrieve the `aircraft/0/systems/parking_brake/state` state to verify the command succeeded.

As mentioned earlier in "[The API Manifest](#the-api-manifest)", currently there are commands returned in the manifest which cannot be used because they required data to be passed to them. The API currently provides no mechanism to do this which renders there comamnds effectively non-functional at this time. It is expected that in the future Infinite Flight will enable these commands.

For example, the command `commands/ParkingBrakes` is a simple toggle: issue the command and the parking brakes switch between off and on. But, other commands clearly don't work that way. For instance, the command `commands/FlightPlan.AddWaypoints` requires a series of waypoints to be provided. The latter cannot be done at this time.
