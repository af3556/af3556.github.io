---
layout: post
title: 'Shelly Scripting - Timers'
categories: [Home Automation]
tags: [shelly]
---

This post follows on from [the intro posts on Shelly scripting]({% post_url 2025-02-13-shelly-scripting-part1 %}) and takes a look at timers. Timers are one of the two primary ways any Shelly script can Get Stuff Done (the other being via event/notification handlers). Two practical examples are considered:

1. posting [webhook](https://www.google.com/search?q=webhook) (HTTP POST/GET) notifications to [ntfy.sh](https://ntfy.sh)
2. waiting on an asynchronous RPC (calling `Shelly.GetConfig` and waiting for the result)

## Do You Have A Minute?

Shelly provides a [Timer object](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures#timer) that will schedule a given callback to be called after a given delay, either once or repeatedly, with a granularity on the delay or interval on the order of a few milliseconds. It can also pass along some data to the callback. As of 1.0 of the [Script Language](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/), a script can have a [maximum of 5 timers](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/#resource-limits).

> Aside: the Shelly Script "language version" appears to be managed by Shelly independently of firmware releases and/or devices; however the availability of functionality mentioned in the language documentation varies with firmware (and/or devices). The doc tags _some_ of the API calls with a minimum firmware version (e.g. "Since version 1.5.0") but most are not labelled, including a number of classes/methods that are unavailable in 1.4.4. Attempting to use these - e.g. referencing `Timer.getInfo()` - results in a runtime error. It would be helpful if Shelly properly documented the API for each firmware release.
{: .prompt-tip }
> Regarding webhooks: Shelly firmware 1.5 appears to have introduced a [`Webhook`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Webhook/) service that allows a script to fire off webhook calls when certain events occur and certain conditions are true. This looks promising (I don't yet have a 1.5 device) but it also has some limitations: seems you can't ask `Webhook` to send an ad-hoc message, nor can you tell if it succeeded or failed, nor can you control the HTTP request type or set headers (e.g. for authentication). Depending on your application this may or may not be a problem for you. Regardless, rate limiting notifications serves as a good example for Timers, so let's carry on.
{: .prompt-info }

## Timers, huh, what are they good for?

Simple applications for a timer are to cycle a switch or periodically fetch or upload data.

Here's about the simplest example, simply writing a log message every 1s:

```javascript
function _timerCallback(userdata) {
  console.log('_timerCallback', userdata);
}

var _timerHandle;
function startTimer() {
  _timerHandle = Timer.set(1000, true, _timerCallback, 'important data');
  console.log('startTimer', _timerHandle);
}

startTimer();
```

Output ([prefix is a timestamp MM:SS]({% post_url 2025-02-13-shelly-scripting-part1 %}#filtering-logs)):

```text
27:13 startTimer 1
27:14 _timerCallback important data
27:15 _timerCallback important data
27:16 _timerCallback important data
27:17 _timerCallback important data
```

Notes:

- the timer period is in milliseconds
- the second argument controls one-shot (`false`) or repeating (`true`)
- some (fixed) user data can be passed

Aside: passing a negative value for interval is undefined in the documentation; experiments indicate the timer triggers immediately:

```javascript
for (var n=1; n < 6; n++) { // max 5 timers
  var t = Math.pow(-2, n);
  Timer.set(t*500, false, function(i){console.log(i)}, t);
}

/* output:
29:31 -32
29:31 -8
29:31 -2
29:35 4
29:47 16
*/
```

## How Fast?

The Pro4PM appears to be able to run timer callbacks down to around an interval of 8ms (~120 callbacks/second):

```javascript
var n = 0;
function _timerCallback() {
  n++;
}

var t;
var nPrev = 0;
function _logCallback() {
  // Shelly.getUptimeMs() is not available in 1.4.4; uptime is in seconds
  var dtavg = (Shelly.getComponentStatus('Sys').uptime - t)/n;
  console.log('n=' + n + '(' + (n-nPrev) + ') dtavg=' + (dtavg*1000) + 'ms');
  nPrev = n;
}

function startTimer() {
  Timer.set(1, true, _timerCallback);   // count as fast as we can
  Timer.set(1000, true, _logCallback);  // report in every 1s
}

t = Shelly.getComponentStatus('Sys').uptime;
startTimer();
```

Output (prefix is a timestamp MM:SS):

```text
41:56 n=126(126) dtavg=7.93650793650ms
41:57 n=255(129) dtavg=7.84313725490ms
41:58 n=385(130) dtavg=7.79220779220ms
41:59 n=513(128) dtavg=7.79727095516ms
42:00 n=614(101) dtavg=8.14332247557ms
42:01 n=733(119) dtavg=8.18553888130ms
42:02 n=852(119) dtavg=8.21596244131ms
42:03 n=971(119) dtavg=8.23892893923ms
```

## DIY Repeating Timers

A periodic timer is indeed a useful thing; however sometimes it can be useful to more closely control exactly when each timer event occurs and/or change the interval as the script runs. The example below runs the timer at half the rate in the second half of every minute; you could do something similar to slow down a process at night vs day for example.

```javascript
function _timerCallback() {
  var unixtime = Shelly.getComponentStatus('Sys').unixtime;
  _timerHandle = Timer.set(unixtime % 60 < 30 ? 1000 : 2000, false, _timerCallback);  // non-repeating
  console.log('_timerCallback', _timerHandle);
}

var _timerHandle;
function startTimer() {
  _timerHandle = Timer.set(1000, false, _timerCallback);  // non-repeating
  console.log('startTimer', _timerHandle);
}

startTimer();
```

Output (prefix is a timestamp MM:SS):

```text
56:26 startTimer 1
56:27 _timerCallback 2
56:28 _timerCallback 3
56:29 _timerCallback 4
56:30 _timerCallback 5
56:32 _timerCallback 6
56:34 _timerCallback 7
56:36 _timerCallback 8
56:38 _timerCallback 9
```

This mechanism is used below in the second version of the notification rate limiting script, where the Timer is tweaked to give the minimal delay between messages and stopped altogether when not needed.

## Rate Limiting Notifications

Let's move on to a non-trivial example: the goal is to post a notification to a HTTP server (a "webhook"), but to make sure that we don't flood the server with too many messages at once. In addition we'll remove any duplicate messages.

The queuing logic adds a bit of extra code but it's well worth it to not get spammed by a runaway script ;-).

> In general it is preferable for processing of notifications (de-duplication, rate-limiting, prioritisation, etc) to be handled by the notification service. That approach avoids having to implement this processing at each source and allows reuse of the same pipeline for all notification sources. However not all notification services support such a thing, [ntfy.sh](https://ntfy.sh/) doesn't.
{: .prompt-info }

```javascript
/* Shelly script to demonstrate using Timers for rate limited notifications to
a HTTP endpoint (webhook).

Device: Pro4PM 1.4.4| 679fcca9
*/

// ntfy.sh is a neat, simple HTTP-based pub-sub notification service
var CONFIG = {
  httpNtfyURL: 'https://ntfy.sh/',  // topic will be appended
  httpNtfyHeaders: {
    // any static headers required by the webhook, incl. auth if required
    'Title': 'Ping!',
    'Tags': 'smiley'
  }
}

/*** queue ***/

// rate limit HTTP notifications to (approximately) the given interval
// the queue length here should be very small if you want to eliminate any
// potential for spamming the notification service
var _notifyQueue = {
  queue: [],          // queued messages
  maxSize: 5,         // limit the size of the queue
  interval: 20        // interval (seconds)
}

// dequeue one notification; intended to be called via a periodic Timer
function _notifyWrite() {
  if (_notifyQueue.queue.length > 0) {
    // Shelly doesn't do array.shift (!), splice instead
    postNotification(_notifyQueue.queue.splice(0, 1)[0]);
  }
}

var _lastNotificationMessage;
function _notify() {
  if (!CONFIG.httpNtfyURL) return;
  // suppress duplicates; can't just peek at the queue as it may have been
  // serviced already
  var message = arguments.join(' ');
  if (message === _lastNotificationMessage) return;
  _lastNotificationMessage = message;

  if (_notifyQueue.queue.length < _notifyQueue.maxSize) {
    _notifyQueue.queue.push(message);
  } else {
    console.log('_notify overflow:', message);
  }
}

/*** end queue ***/

function _callbackLogError(result, errorCode, errorMessage) {
  if (errorCode != 0) {
    console.log('call failed: ', errorCode, errorMessage);
  }
}

function postNotification(message) {
  var params = {
    'method': 'POST',
    'url': CONFIG.httpNtfyURL,
    'headers': CONFIG.httpNtfyHeaders,
    'body': message,
    'timeout': 5
  }

  // have to use HTTP.Request to specify headers
  console.log('postNotification [' + JSON.stringify(message) + ']');
  Shelly.call("HTTP.Request", params, _callbackLogError);
}

function updateStatus(notifyStatus) {
  _notify('update from', notifyStatus.name);
}

function init() {
  if (CONFIG.httpNtfyURL) {
    CONFIG.httpNtfyURL += Shelly.getDeviceInfo().id || 'shelly';
    console.log('httpNtfyURL ', CONFIG.httpNtfyURL);
    // set a repeating timer to service the notification queue at regular intervals
    Timer.set(_notifyQueue.interval * 1000, true, _notifyWrite);
  }
  _notify('script start');
  _notify('script start');  // duplicate; will be ignored

  // pile on a bunch of notifications - should hit queue overflow from
  // maxSize+1 on; i.e. the first one above will have been sent immediately
  // as the queue was empty and rate limit not hit; the next maxSize will be
  // queued up
  for (var i = 1; i < _notifyQueue.maxSize+2; i++) {
    _notify('start', i);
  }

  Shelly.addStatusHandler(updateStatus);
}

init();
```

The above delivers on the stated goals however it has a few drawbacks:

- notifications are delayed by up to the queue service (Timer) period
- they dribble out one at a time at a fixed interval; it might be preferable to allow a burst of notifications before rate limiting
- when too many notifications are invoked and the queue is full new notifications are discarded; for many use cases including mine this is undesirable - the queue should make room for new messages by discarding the oldest ones first
  - for example, a rapid sequence of toggling a switch output may result in the last notification being sent not matching up with the actual final state of the switch; it's important to note the notifications have no guarantees on delivery, or ordering, etc, but we can at least try
  - might also be a good idea to include a sequence number in the message so any missing notifications can be detected

Let's add the bells and whistles: the version below removes the single periodic timer started in `init()` and replaces it with a series of one-shot timers that
are scheduled "on-demand". The "demand" is whenever a notification message is queued up, the writer will be pinged to see if it's able to be sent immediately.
If the new message is too close on the tail of the last message, a timer is set to send it later. When no notifications are pending, there are no timers running.

> When developing/debugging it's tempting to sprinkle print (`console.log()`) statements everywhere; that can backfire on Shelly due to the platform [rate limiting the number of log messages]({% post_url 2025-02-13-shelly-scripting-part1 %}#consolelog-rate-limiting) (due to finite message queue size) and _silently_ discarding any excess. Ironically the script we're working on here is implementing exactly the same behaviour albeit for different reasons - with the notable exception that queue overflows are logged ;-)
>
> See [the Shelly Scripting intro post]({% post_url 2025-02-13-shelly-scripting-part1 %}#take-4---going-to-town) on how to use a periodic queue to avoid the Shelly `console.log()` rate limiting and ensure you get all your log messages - or at least get a warning when they're dropped. That code is pretty much the same as the above fixed period rate limiting but with a low timer interval, unlike webhook notifications that have a high interval.
{: .prompt-warning }

```javascript
/* Shelly script to demonstrate using Timers for rate limited notifications to
a HTTP endpoint (webhook).

Device: Pro4PM 1.4.4| 679fcca9
*/

// ntfy.sh is a neat, simple HTTP-based pub-sub notification service
var CONFIG = {
  httpNtfyURL: 'https://ntfy.sh/',  // topic will be appended
  httpNtfyHeaders: {
    // any static headers required by the webhook, incl. auth if required
    'Title': 'Ping!',
    'Tags': 'smiley'
  }
}

/*** queue ***/

// rate limit HTTP notifications to (approximately) the given interval
// the queue length here should be very small if you want to eliminate any
// potential for spamming the notification service
var _notifyQueue = {
  queue: [],          // queued messages
  maxSize: 5,         // limit the size of the queue
  interval: 20        // minimum interval (seconds)
}

// dequeue one notification and if necessary schedule the next one-shot timer
var _lastNotificationTime = 0;
var _timerHandle = null;
function _notifyWrite(byTimer) {
  // aside: this function can be called _either_ as a Timer callback or directly
  // via _notify() (a "flush"); ES5 is single threaded so only one _or_ the
  // other should be in play at any given invocation
  // it's unclear what guarantees Shelly's Timer provides around timing of a
  // callback: it may well be possible that a callback is called prematurely
  // i.e. where _notifyQueue.interval - (t - _lastNotificationTime) > 0
  // to sidestep that entire problem the byTimer argument is used and set true
  // (only) when called via Timer

  // there are no console.log() calls in the 'fallthrough' path here; _notify()
  // (and hence _notifyWrite()) may well be called quite frequently; be aware
  // that some logs will be discarded if console.log() is called "too
  // frequently"

  // use uptime and not unixtime; the latter won't be available without NTP
  var t = Shelly.getComponentStatus('Sys').uptime;
  // how long is left to wait?
  var remaining = _notifyQueue.interval - (t - _lastNotificationTime);

  if (byTimer || remaining < 0) { // go, go gadget
    //console.log('_notifyWrite by', (byTimer ? 'timer' : 'flush:' + remaining),
    //  'qlen=' + _notifyQueue.queue.length);
    // there shouldn't be a situation where we get here with an empty queue
    // but handle it anyway
    if (_notifyQueue.queue.length > 0) {
      postNotification(_notifyQueue.queue.splice(0, 1)[0]);
      _lastNotificationTime = t;
      remaining = _notifyQueue.interval;
    }
  }

  if (byTimer) _timerHandle = null; // the timer that called us is an ex-timer
  if (_notifyQueue.queue.length == 0) {
    // queue's empty, no need for another callback
    if (_timerHandle) Timer.clear(_timerHandle);
  } else if (!_timerHandle) {
    // there's more to come yet no timer in play -> schedule the next callback
    // time remaining should be in the range [0, _notifyQueue.interval]
    var interval = Math.max(0, Math.min(remaining, _notifyQueue.interval));
    //console.log('_notifyWrite', interval, 'qlen=' + _notifyQueue.queue.length);
    _timerHandle = Timer.set(interval * 1000, false, _notifyWrite, true);
  }
}

var _lastNotificationMessage;
var _notificationSequenceNumber = 0;
function _notify() {
  if (!CONFIG.httpNtfyURL) return;
  // suppress duplicates; can't just peek at the queue as it may have been
  // serviced already
  var message = arguments.join(' ');
  if (message === _lastNotificationMessage) return;
  _lastNotificationMessage = message;

  // discard oldest messages to make room for new ones
  if (_notifyQueue.queue.length >= _notifyQueue.maxSize) {
    // delete the 0'th message, (attempt to) report it
    var m = _notifyQueue.queue.splice(0, 1)[0];
    console.log('_notify overflow:', JSON.stringify(m));
  }
  _notifyQueue.queue.push(message + '\n' + _notificationSequenceNumber++);
  _notifyWrite(); // service the queue if possible (attempt a write 'asap')
}

/*** end queue ***/


function _callbackLogError(result, errorCode, errorMessage) {
  if (errorCode != 0) {
    console.log('call failed: ', errorCode, errorMessage);
  }
}

function postNotification(message) {
  var params = {
    'method': 'POST',
    'url': CONFIG.httpNtfyURL,
    'headers': CONFIG.httpNtfyHeaders,
    'body': message,
    'timeout': 5
  }

  // have to use HTTP.Request to specify headers
  console.log('postNotification [' + JSON.stringify(message) + ']');
  Shelly.call("HTTP.Request", params, _callbackLogError);
}

function updateStatus(notifyStatus) {
  // a 'script started' status update will be delivered immediately and then
  // heartbeat and other ad-hoc updates from then on
  // - these will all be discarded until the notification queue catches up

  // heartbeats are every 60s with a separate notification for every switch
  // (i.e. four in rapid succession for a Pro4PM); this will trip up
  // console.log() rate limiting (in conjunction with the other log calls), so
  // skip them totally
  var delta = notifyStatus.delta;
  if (delta && delta.aenergy) return;
  console.log('update from', notifyStatus.name + ':' + notifyStatus.id);

  // include (switch) output state if present
  _notify('update from', notifyStatus.name + ':' + notifyStatus.id +
    (delta && delta.output !== undefined ? ' (' + (delta.output ? 'on' : 'off') + ')' : ''));
}

function init() {
  if (CONFIG.httpNtfyURL) {
    CONFIG.httpNtfyURL += Shelly.getDeviceInfo().id || 'shelly';
    console.log('httpNtfyURL ', CONFIG.httpNtfyURL);
  }
  _notify('script start');
  _notify('script start');  // duplicate; will be ignored

  // pile on a bunch of notifications - should hit queue overflow from
  // maxSize+1 on; i.e. the first one above will have been sent immediately
  // as the queue was empty and rate limit not hit; the next maxSize will be
  // queued up
  for (var i = 1; i < _notifyQueue.maxSize+2; i++) {
    _notify('start', i);
  }

  Shelly.addStatusHandler(updateStatus);
}

init();
```

## Making The Asynchronous, Synchronous

Shelly is all about asynchronous flow: callbacks and timers. Sometimes its desirable to coordinate separate asynchronous parts of a script, to block or wait for an asynchronous activity to complete. More modern versions of Javascript accomplish this coordination via Promises and async/await functions, however neither are available in the Shelly environment. Not to worry, Timers can be used to poll instead (polling is less efficient than many other approaches, but it's all we have here).

The example below shows one way of waiting on an asynchronous call to complete before proceeding with other parts of the script. Specifically, the `Shelly.GetConfig` RPC is not available as a synchronous version (unlike how the `Shelly.getDeviceInfo()` method wraps the `Shelly.GetDeviceInfo` RPC). So in order to acquire say [the device name](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Sys#configuration), the script below executes the asynchronous RPC and then polls for the result via a timer (absent any "backoff"). In practice, for this RPC, the call returns very quickly - but that would not always be the case, for example if calling an external system.

The process is split into two parts:

1. stage 1 - make the asynchronous call and poll for it to complete
2. stage 2 - called by stage 1; execute the code that depends on the async call

The two stages can communicate via a shared global variable. In the example below,  `getDeviceConfigThenInit()` is the stage 1, `init()` the stage 2, and they communicate via `deviceConfig`.

Note that stage 1 does not block; the first call just fires off the RPC and sets up a timer to poll for the result. It's feasible that the RPC could have completed by the time the timer is first set; in this example that's not a problem though [race conditions](https://www.google.com/search?q=race+conditions) are always something to keep in mind when working with asynchronous processes.

```javascript
var deviceConfig;

function _callbackGetConfig(result, errorCode, errorMessage) {
  if (errorCode == 0) {
    deviceConfig = result;
    console.log('_callbackGetConfig:', deviceConfig.sys.device.name);
  } else {
    console.log('call failed: ', errorCode, errorMessage);
  }
}

function getDeviceConfigThenInit(poll) {
  if (!poll) {  // kick-off
    console.log('getDeviceConfigThenInit call');
    Shelly.call("Shelly.GetConfig", null, _callbackGetConfig);
  }
  if (deviceConfig === undefined) {  // waiting, waiting...
    console.log('getDeviceConfigThenInit setting timer');
    Timer.set(100, false, getDeviceConfigThenInit, true); // non-repeating, userdata: poll=true
  } else {
    console.log('getDeviceConfigThenInit done');
    init();
  }
}

function init() {
  console.log('init',
      deviceConfig === undefined ? 'failed to get deviceConfig' : 
        'got deviceConfig: ' + deviceConfig.sys.device.name);
  // carry on with rest of the code that uses deviceConfig
}

getDeviceConfigThenInit();  // this will invoke init()
```

The main problem with the above code is that it will 'block' forever if the RPC fails to call the callback; that's not something reasonably expected from the Shelly environment. Even when calling external services via Shelly's [`HTTP`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/HTTP/), [`Webhook`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Webhook/) and related, they all have timeouts, etc that ensure the callback is called. If somehow the above turns out to be incorrect (a Shelly bug), a counter could be used to abort the polling.

Stylistically it would be preferable to pass the stage 2 function as a parameter to stage 1, however attempting to do that in the Shelly environment barfs, e.g.:

```javascript
function getDeviceConfig(callback) {
  // ...
  callback(); // barf here
}
function init() { /* ... */ }
getDeviceConfig(init);
```

Results in an `Uncaught Error: Function "callback" not found!` when the callback is invoked.
