# msfs-simconnect-api-wrapper

A JavaScripty wrapper around EvenAR's excellent [node-simconnect](https://github.com/EvenAR/node-simconnect/) with a simplified API.

See [test.js](./test.js) for the basics, which you can run (with MSFS open and sitting on the apron with a plane =) using `npm test`.

## Installation and use

Install with `npm install msfs-simconnect-api-wrapper`.

For examples on how to use this, see the [examples](https://github.com/Pomax/msfs-simconnect-api-wrapper/blob/main/examples) dir.

## API

This API manager has an argless constructor that does nothing other than allocate internal values. In order to work with MSFS, you need to call the `connect` function first, which can take an options object of the following form:

```javascript
{
  autoReconnect: true or false (will try to reconnect to MSFS if the connection gets closed), defaults to `false`.
  retries: positive number or Infinity, defaults to 0.
  retryInterval: positive number, representing number of seconds (not milliseconds) between retries, defaults to 2.
  onConnect: callback function with the node-simconnect handle as its only argument.
  onRetry: callback function with (retries left, retry interval) as its two arguments. This triggers _before_ the next attempt is scheduled.
  onException: callback function with (exceptionName) as argument.
  host: a domain or IP string (e.g. "localhost" or "127.0.0.1", default: "0.0.0.0").
  port: the port at which to contact MSFS (default: 500).
}
```

For example, we can set up an API object and have it start trying to connect, eventually dropping us into the code that knows it has an MSFS connection using the following boilerplate:

```javascript
import { MSFS_API } from "./msfs-api.js";

const api = new MSFS_API();

api.connect({
  retries: Infinity,
  retryInterval: 5,
  onConnect: () => run(),
  onRetry: (_, interval) => {
    console.log(`Connection failed: retrying in ${interval} seconds.`);
  },
});

function run() {
  console.log(`We have an API connection to MSFS!`);
  //
  // ...your code here...
  //
}
```

### Properties

The API has a single property `.connected` which is either `undefined` or `true` and can be used to determine whether the API has a connection to MSFS outside of code that relies on the `onConnect` callback.

### Methods

#### `connect(opts?)`

Sets up a connection to MSFS, see above for an explanation of `opts`. If let unspecified, no retries will be attempted.

#### `on(evtDefinition, handler)`

Starts listening for a specific simconnect event with a specific handler. Returns a corresponding arg-less `off()` function to clean up the listener. See the "System events" section below for details on the event definition.

##### System events (used for on/off handling):

All event names in https://docs.flightsimulator.com/html/Programming_Tools/SimConnect/API_Reference/Events_And_Data/SimConnect_SubscribeToSystemEvent.htm are supported as constants on the `SystemEvents` object, importable alongside MSFS_API:

```javascript
import { SystemEvents, MSFS_API } from "./msfs-api.js";

const api = new MSFS_API();

api.connect({
  retries: Infinity,
  retryInterval: 5,
  onConnect: () => {
    api.on(SystemEvents.PAUSED, () => {
      // ...
    });
  },
});
```

Note that the event names are keys from the `SystemEvents` object, using UPPER_SNAKE_CASE, not strings.

##### Special Events

There are currently two non-simconnect events that can be listened to:

- `AIRPORTS_IN_RANGE`, registers a listener for notifications about airports coming into range of our airplane (or rather, coming into range of "the current Sim bubble", which is all world tiles currently loaded and active in the sim).
- `AIRPORTS_OUT_OF_RANGE`, registers a listener for notifications about airports dropping out of range of our airplane (with the same note as above).

#### `off(evtDefinition, handler)`

Stop listening for a specific simconnect event with a specific handler. You'll typically not need to call this function directly, as you can just use the function that `on` returns. See the "System events" section above for more details on the event definition.

#### `get(...propNames)`

Accepts a list of simvars (with spaces or underscores) and async-returns a key/value pair object with each simvar as key (with spaces replaced by underscores).

##### special (non-simconnect) variables

There are a number of special variables that can only be retrieved using a get call with a single variable name, yielding data that is not serviced by SimConnect's own variables (or data that requires a considerable amount of low-level event handling).

There are currently three variables:

- `ALL_AIRPORTS`, which yields the list of all airports known to MSFS.
- `NEARBY_AIRPORTS`, which yields the list of airports that are currently in range of our airplane (or rather, in range of "the current Sim bubble", which is all world tiles currently loaded and active in the sim).
- `NEARBY_AIRPORTS:NM`, which yields the list of airports within a radius of `NM` nautical miles around the airplane's location.
- `AIRPORT:index`, which yields an airport's information (including runway information), with `index` being the airport's ICAO code.

The first three return arrays of Airport object, the last one returns a single Airport object

Airport objects have the following shape:

```
{
  latitude: number in degrees
  longitude: number in degrees
  altitude: number in feet
  declination: number in degree
  name: airport name as a string with at most 32 characters
  name64: airport name as a string with at most 64 characters
  icao: four character ICAO code for this airport
  region: the ICAO region for this airport as string
  runwayCount: number of runways at this airport
  runways: array of runway objects
}
```

Runway objects have the following shape:

```
{
  latitude: number in degrees, marking the center of the runway
  longitude: number in degrees, marking the center of the runway
  altitude: number in feet
  heading: number in degrees
  length: number in meters
  width: number in meters
  patternAltitude: number in meters
  slope: number in degrees
  slopeTrue: number in degrees
  surface: surface material, as string
  approach: array of runway approaches
}
```

Approaches gave the following shape:

```
{
  designation: runway designation as string
  marking: runway marking, as string (can be a number, or cardinal direction)
  ILS: {
    type: ILS type as string
    icao: ICAO code for this approach's ILS
    region": ICAO region for this approach's ILS
  }
}
```

#### `schedule(handler, interval, ...propNames)`

Sets up a periodic call to `handler` every `interval` milliseconds with the result of `get(...propNames)`. Returns an arg-less `off()` to end the scheduled call.

#### `set(propName, value)`

Accepts a single simvar and the value its should be set to. This will throw "SimVar ... is not settable" when attempting to set the value for a read-only variable.

#### `trigger(triggerName, value?)`

Triggers a simconnect event, with optional value.

### Supported Simvars:

All simvars are supported, barring several simvars with data types for which I need to figure out how to actually deference then, such as LatLonAlt structs, or the (super rare) bool/string combination, as well a any simvar that is officially deprecated, or marked as "legacy, do not use these going forward". If you get an error about an unknown Simvar, look up that variable on the [SimConnect variables list](https://docs.flightsimulator.com/html/Programming_Tools/SimVars/Simulation_Variables.htm) and see if it's either deprecated, or part of a collection that is considered legacy.

- [x] Camera Variables (_not verified_)
- [x] Services Variables (_not verified_)
- [x] Miscellaneous Variables (_not verified_)
- Aircraft SimVars:
  - [x] Aircraft Autopilot/Assistant Variables (_not verified_)
  - [x] Aircraft Brake/Landing Gear Variables (_not verified_)
  - [x] Aircraft Control Variables (_not verified_)
  - [x] Aircraft Electrics Variables (_not verified_)
  - [x] Aircraft Engine Variables (_not verified_)
  - [x] Aircraft Flight Model Variables (_not verified_)
  - [x] Aircraft Fuel Variables (_not verified_)
  - [x] Aircraft Misc. Variables (_not verified_)
  - [x] Aircraft Radio Navigation Variables (_not verified_)
  - [x] Aircraft System Variables (_not verified_)
  - [x] Helicopter Variables (_not verified_)

Further more, all environment variables listed over on https://docs.flightsimulator.com/html/Additional_Information/Reverse_Polish_Notation.htm#EnvironmentVariables are supported as well.

- [ ] WASM guage token variables (see https://github.com/Pomax/msfs-simconnect-api-wrapper/issues/12)

(note: "not verified" means that they've been ported, marked settable where appropriate, pass the test run, but they've not been individually confirmed to be correct with respect to the documentation yet).

### Supported SimEvents:

SimEvents are resolved by key name, so as long as you use a valid key name, you can trigger it.

See https://docs.flightsimulator.com/html/Programming_Tools/Event_IDs/Event_IDs.htm for the full list (there are... a lot).

## Helping out

File an issue if you want to help get this wrapper to 100% simvar and event support!
Even if you just want to help verify a few variables, that's a few variables fewer that I'll need to run through =)
