---
layout: post
title: 'Shelly Scripting Part 2 - Practical Example: "Underpower Off"'
categories: [Home Automation]
tags: [shelly]
---

This post is about creating a Shelly script to implement a mechanism to turn off an output of a [Shelly Pro4PM](https://kb.shelly.cloud/knowledge-base/shelly-pro-4pm-v2) when the load power draw drops below a given threshold for a certain time. One potential use case could be a "safety interlock" for a machine where it is desired to also turn off the outlet once the machine itself has been turned off / stopped, i.e. so that it can't start up again without instructing the Shelly switch to re-enable the output.

> Aside: Shelly has built-in controls for over- and under-voltage and over-current (but not under-current); the former also has an "auto-recovery" mode but unfortunately has no delay options. i.e. a series of voltage excursions will result in the Shelly output cycling for each; it may be preferable to "wait it out" until the voltage has stabilised for a time; this script could be modified to do that.
{: .prompt-tip }

This is part 2 of "Shelly Scripting", introduced in [part 1]({% post_url 2025-02-13-shelly-scripting-part1 %}).

## The Use Case

This script was created to turn a regular pressure-controlled water pump into a "one-shot" pump - one that turns off and stays off shortly after the high pressure level is reached. This is being used to transfer water from one tank to another, where the destination tank has a float valve that closes when it is full. The goal is to (manually at this point) turn the pump on when required and then not have to keep a close eye on it from there on. The water pump has a built-in pressure switch that turns the pump off when a certain water pressure is reached and back on when pressure falls below a lower threshold. This is ideal behaviour to feed a water tap (spigot, faucet, outlet), however for my use case (transferring water between tanks) once the pressure cutoff is reached [the pump's job is done](https://www.youtube.com/watch?v=Kmu_UVgk2ZA&t=367s) until restarted at some much later time. Ideally the water would stay pressurised for arbitrarily long periods of time and thus the pump's lower pressure limit never reached and the pump would stay off of its own accord - however it turns out real-world one-way / non-return valves do not always behave as they are named and the pump cycles every 10-20 minutes.

A simple off-timer could be used to shut the pump power off after some fixed time, however the time taken for the pump to do its work can vary quite substantially and the pump may either be cut short or will needlessly cycle once it hits its pressure limit. Also, where's the fun in that? Instead, this script is used to monitor the pump's energy use and when it drops to some low value (a proxy for the pump having completed its work for now), turns power to the pump off.

## Some Shelly background

The following is not well documented by Shelly, it has been gleaned from various posts on the [Shelly forums](https://community.shelly.cloud) as well as inference from observation of the Pro4PM (firmware `1.4.4| 679fcca9`). YMMV.

Broadly speaking, there are two approaches to monitoring load power on the Pro4PM: by keeping an eye on notification messages as they are occur, or by polling the device periodically for status updates. The former seems like a more efficient approach.

### Notifications: Event and Status

Shelly provides two 'Notify' mechanisms: [NotifyStatus and NotifyEvent](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures#shellyaddeventhandler-and-shellyaddstatushandler). It's rather unclear why you'd use one over the other. For example, here's the NotifyEvent and NotifyStatus messages sent to an eventHandler and a statusHandler for the same action (turning on an output via the web UI):

```json
eventHandler {"component":"switch:0","name":"switch","id":0,"now":1739795317.49988508224,"info":{"component":"switch:0","id":0,"event":"toggle","state":true,"ts":1739795317.5}}
statusHandler {"component":"switch:0","name":"switch","id":0,"delta":{"id":0,"output":true,"source":"WS_in"}}
```

Oddness such as the following occurs with a NotifyEvent, where two near-identical messages are posted, but the first has an incorrect `apower` value (actual load is 0, nothing is connected):

```json
{"component":"switch:2","name":"switch","id":2,"now":1739795959.21503114700,"info":{"component":"switch:2","id":2,"event":"power_update","apower":-1.1,"ts":1739795959.22000002861}}
{"component":"switch:2","name":"switch","id":2,"now":1739795960.21493196487,"info":{"component":"switch:2","id":2,"event":"power_update","apower":0,"ts":1739795960.21000003814}}
```

Anecdotally, the NotifyStatus messages are far more useful.

### Status Notifications

The Pro4PM posts different Status notifications at various times, including:

  1. periodic (aka heartbeat) notifications every minute for every power measuring component, regardless of whether its output is on or off. This is four notifications/minute for the 4-channel Pro4PM.
      - these periodic notifications contain a `delta.aenergy` object
  2. "output state change" notifications: posted any time the output is switched on/off
     - these contain a `delta.output` value
  3. "load change" notifications: posted any time the measured load changes, for the specific component
     - these contain a `delta.apower` (instantaneous power out (W)) and/or a `delta.current` (instantaneous current (A)) value

For the sake of brevity I'll refer to each of the above by the fields that are present, e.g. `aenergy`, `output` and `apower/current` respectively.

### `aenergy` Notifications

> I didn't end up using `aenergy` notifications, covering here for reference.
{: .prompt-info }

The periodic `aenergy` notifications report accumulated energy usage since last reset (`total`, in Wh) and the incremental energy usage for each of the last three minutes in units of mWh. This appears to be a a subset of that reported by [Switch.GetStatus](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Switch/#status).

The `by_minute` values are the change in `total` over each of the previous three minutes, in mWh; `total` was incremented in minute preceding `minute_ts` by `by_minute[0]/1000`. The _average_ power over that minute is given by `P_avg = by_minute[0] * 60/1000`.

Aside: the `by_minute` values appear to be a moving average of some fashion: step changes in load are reflected in decaying/appreciating values.

`minute_ts` timestamp is documented as the timestamp of `by_minute[0]`.

`total` appears to be reset only on device init or explicit counter reset - unclear what the overflow conditions are.

Here's an example periodic status with a load pulling around 33.5W (i.e. by_minute = 33.5Wh*1000/60 = ~600mWh)

```json
{
  "component": "switch:1",
  "name": "switch",
  "id": 1,
  "delta": {
    "id": 1,
    "aenergy": {
      "by_minute": [595.232, 600.569, 604.704],
      "minute_ts": 1739187120,  // seconds
      "total": 199.418          // Wh
    },
    "ret_aenergy": {
      "by_minute": [0, 0, 0],
      "minute_ts": 1739187120,
      "total": 0
    }
  }
}
```

The initial implementation of this script naively attempted to avoid keeping track of time by considering that if all three `by_minute` values are below the threshold then the load has been idle on average for >3 minutes, and thus used `max(by_minute)` to assess the 3-minute load history. This approach fails in practice due to initialisation / startup issues: if the load is switched on, but doesn't draw any power, Shelly will dutifully and correctly report `by_minute = [0,0,0]` and so the output immediately turned off. The same will also happen if the load happens to be turned on a fraction of a second before the next `aenergy` update (i.e. where, even though power is being drawn, the energy consumed is still approximately 0 over the minute). A secondary issue is if you want to assess load over time periods other than 3 minutes.

### `apower/current` Notifications

`apower/current` notifications are sent on load changes _and_ switch output state changes. In the latter case, power may be 0, i.e. switch is on but load isn't drawing any current. Shelly sometimes includes `apower`, sometimes `current` and sometimes both. There appears to be a minimum reporting interval of around 6-7s (i.e. consecutive `apower` notifications are at least 6-7s apart). It's unclear exactly what the resolution is for power changes, some experimentation for a fan (max. 34W) suggests that whilst the resolution is around 0.1W, a change of 1W or more is required to trigger a notification. Even somewhat gradual changes appear to be reported once they cross the 1W delta from the last `apower` notification.

> This script uses the `apower` notifications, these alone are sufficient to determine "is the load idle". The only slight hiccup is that these notifications do not include a timestamp so that needs to be tracked separately.
{: .prompt-tip }

### Example Notifications

Here are example event-based `apower` and `output` notifications (note: these do not have a timestamp, and they only occur when a change happens):

Switch on, via web UI:
```json
{"component":"switch:1","name":"switch","id":1,"delta":{"id":1,"output":true,"source":"WS_in"}}
```

Load power draw changes:
```json
{"component":"switch:1","name":"switch","id":1,"delta":{"id":1,"apower":41.5,"current":0.175}}
{"component":"switch:1","name":"switch","id":1,"delta":{"id":1,"apower":39.3}}
{"component":"switch:1","name":"switch","id":1,"delta":{"id":1,"current":0.168}}
```

Switch off via web UI (unclear why `apower`,`current`,`pf` are included, always 0):
```json
{"component":"switch:1","name":"switch","id":1,"delta":{"id":1,"apower":0,"current":0,"output":false,"pf":0,"source":"WS_in"}}
```

## Script Design

A [state machine](https://www.espruino.com/StateMachine) approach often leads to clean, easy to understand code for controlling real-world systems. The initial design here was to model the Shelly switch by a simple state machine: off/on/idle. However there is a very important distinction for Shelly scripts: these scripts are not the only thing controlling the device: state changes often occur outside of the script. e.g. a user or schedule turning an output on or off, overload, etc. Taking a full state machine approach introduces unnecessary complexity in attempting to track the actual device state.

This script takes a simpler approach:

1. tracking switch state in a global object that is updated as each new piece of information arrives via the various Shelly notifications
   - including recording the time of entering 'idle state' (required for a timeout)
2. turning the output off when the idle state and timeout conditions are met

## One More Thing...

A runtime error will stop the script and Shelly won't automatically restart it (except [on boot, if the script is configured to do so](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Script/#configuration)). One way a script can crash is by deferencing a non-existent property, e.g. something like this will crash the script when a `delta` notification arrives with only `apower/current` data ("Uncaught Error: Cannot read property 'total' of undefined"):

```javascript
function statusHandler(notifyStatus) {
  var total = notifyStatus.delta.aenergy.total; // boom
  //...
}
Shelly.addStatusHandler(statusHandler);
```

In ES6 you can use the ['optional chaining' (`?.`) operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining), e.g. `notifyStatus?.delta?.aenergy?.total`, but for ES5 it's a bit more fiddly. A one-liner could be used:

```javascript
function _get(obj, path) {
  return path.split('.').reduce((o, p) -> (typeof o === 'undefined' || o === null ? o : o[p]), obj);
}
function statusHandler(notifyStatus) {
  var total = _get(notifyStatus, 'delta.aenergy.total');  // can't crash me
  //...
}
Shelly.addStatusHandler(statusHandler);
```

However Shelly's Array object has been neutered of the `.reduce()` function. Instead a bulkier implementation is needed (refer `_get()` in the script).

The above is only really a problem when attempting to access >1 level deep properties, which the final version of the script below does not do. However I consider the `_get()` implementation as "good practice" boilerplate, and am not aware of any drawbacks.

## The Script

And now the big reveal ([canonical version](https://github.com/af3556/shelly/blob/main/underpower-off.js)):

```javascript
/* Shelly script to turn off an output when the load power is below a given
threshold for a given time period.

Device: Pro4PM 1.4.4| 679fcca9

*/

// configure these as desired:
// - switch IDs are 0-based (i.e. 0-3 for the Pro4PM) though they're labelled on
//   the device as 1-4
var CONFIG = {
  switchId: 1,    // switch to monitor
  threshold: 10,  // idle threshold (Watts)
  timeout: 30,    // timeout timeout (seconds) (rounded up to heartbeat timeout)
  log: true       // enable/disable logging
}



// notifications for switch state changes (e.g. on/off, power) arrive
// independently and asynchronously; the state machine logic is greatly
// simplified by having all the necessary inputs in the one place/time
//
// this object is used to accumulate, via each Shelly notification, a complete
// view of the device's actual state as at the last change time
// - an alternative approach could just query all the necessary bits every
//   callback, but where's the fun in that #efficiency
var switchState = {
  output: null,   // last known switch output State
  apower: 0,      // last known `apower` reading
  timer: 0        // timestamp of last on or idle transition
}

var currentTime = 0;  // not every notification has a timestamp, have to DIY


function _defined(v) {
  return v !== undefined && v !== null;
}

// helper to avoid barfing on a TypeError when object properties are missing
function _get(obj, path) {
  var parts = path.split('.');
  var current = obj;

  for (var i = 0; i < parts.length; i++) {
    if (current && current[parts[i]] !== undefined && current[parts[i]] !== null) {
      current = current[parts[i]];
    } else {
      return undefined;
    }
  }
  return current;
}

function _log() {
  // Shelly doesn't support the spread operator `...`
  // workaround: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments
  if (CONFIG.log) console.log('[underpower-off]', arguments.join(' '));
}

function _callback(result, errorCode, errorMessage) {
  if (errorCode != 0) {
    // not _log: always report actual errors
    console.log('call failed: ', errorCode, errorMessage);
  }
}

function _getSwitchTimestamp() {
  // not every notification include a timestamp (`delta`'s don't)
  // https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Sys#status
  currentTime = Shelly.getComponentStatus('Sys').unixtime;
}

// 'init' switch state when the script is starting up with no or constant load
function _getSwitchState() {
  var status = Shelly.getComponentStatus('Switch', CONFIG.switchId);
  _log('_getSwitchState status=', JSON.stringify(status));
  switchState.output = status.output;
  switchState.apower = status.apower;
  switchState.timer = currentTime;
}

// update switch state with current output state (on/off)
function _updateSwitchOutput(notifyStatus) {
  var output = _get(notifyStatus, 'delta.output');
  if (!_defined(output)) return;  // not a delta.output update
  _log('_updateSwitchOutput output=', JSON.stringify(output));
  // reset the timer when turning on ('on/off edge transition')
  // !== true is not necessarily === false (e.g. on init, where output is null);
  // just want to determine a _change_
  if (output === true && switchState.output !== output) {
    _log('_updateSwitchOutput reset timer');
    switchState.timer = currentTime;
  }
  switchState.output = output;
}

// update switch state with current power
function _updateSwitchPower(notifyStatus) {
  // `delta.apower` notifications are sent on load changes _and_ switch output
  // state changes (even when power remains 0)
  var apower = _get(notifyStatus, 'delta.apower');
  if (!_defined(apower)) return;  // not a delta.apower update
  _log('_updateSwitchPower apower=', JSON.stringify(apower));
  // reset the timer on power idle edge transition; when going from not-idle to
  // idle
  var idlePrev = _isPowerIdle();
  switchState.apower = apower;
  if (idlePrev === false && _isPowerIdle() !== idlePrev) {
    _log('_updateSwitchPower reset timer');
    switchState.timer = currentTime;
  }
}

function _isTimeExpired()
{
  return currentTime - switchState.timer > CONFIG.timeout;
}
function _isPowerIdle() {
  return switchState.apower < CONFIG.threshold;
}


function statusHandler(notifyStatus) {
  // only interested in notifications regarding the specific switch
  if (notifyStatus.component !== 'switch:' + CONFIG.switchId) return;
  //_log(JSON.stringify(notifyStatus));

  _getSwitchTimestamp();
  // the notification will be _one of_: an `output` notification, an `apower`
  // notification, or 'something else'
  // - some notifications may include both switch `output` and `apower` info
  //   (e.g. when a switch is turned on), this could be leveraged to eliminate
  //   some processing but for the sake of simplicity we'll KISS
  _updateSwitchPower(notifyStatus);
  _updateSwitchOutput(notifyStatus);

  switch (switchState.output) { // JS switch uses strict equality
    case true:  // on
      _log('on p=', switchState.apower, ' dt=', currentTime - switchState.timer);
      if (_isPowerIdle() && _isTimeExpired()) {
        _log('idle, timer expired: turning off');
        Shelly.call('Switch.Set', { id: CONFIG.switchId, on: false }, _callback);
      }
    break;
    case false: // off; nothing to do
      break;
    default:
      // when the script starts up with a constant load output (incl. 0), we won't
      // see any `delta.output` or `delta.apower` notifications (only
      // hearbeats), have to "manually" get the current state
      // this should happen only once; no need to invoke on every iteration
      _getSwitchState();
  }
  _log(JSON.stringify(switchState));
}

Shelly.addStatusHandler(statusHandler);
*/

```
