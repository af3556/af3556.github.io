---
layout: post
title: Shelly Scripting Part 1 - Intro
categories: [Home Automation]
tags: [shelly]
---

[Shelly](https://www.shelly.com/) - nee Allterco - produce a variety of home automation devices. One notable feature is on-device scripting: devices host an embedded Javascript interpreter that opens up all kinds of useful functionality. This post is part one of two that are gp from 'first contact' with a Shelly through to having a script that provides bespoke control of a load.

[Part 2]({% post_url 2025-02-13-shelly-scripting-part2 %}) covers a practical / real-world example.

The nominal audience for these posts is someone who has some programming background, but not in Javascript or [asynchronous programming](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Async_JS/Introducing). I'd welcome any constructive feedback via [GH Discussions](https://github.com/af3556/af3556.github.io/discussions).

## Why on-device scripting?

There exist myriad home-automation platforms that can be used to deliver sophisticated functionality in various ways. Shelly supports all the common options, plus one:

1. a [cloud service](https://control.shelly.cloud/)
2. self-hosted home automation (MQTT and everything downstream of that, and more recently, Matter)
3. on-device automation (e.g. timers, schedules)
4. on-device scripting

On-prem self-hosted home automation systems can be a great option to avoid dependencies on cloud services (and/or an Internet connection), but on-device scripting has even fewer dependencies: only the device itself!

In terms of reliability and minimal complexity on-device scripting is easily the best choice for complex device-focused behaviours - functionality that is beyond simple built-in automation timers and schedules, yet does not require significant coordination with other devices or services nor any sort of user interface (though both are possible via on-device scripting if you really need to).

## Overview

One of the Shelly's best features is the embedded Javascript (JS) interpreter. As of the 'gen2' devices, Shelly are using a modified version of the [Espruino](https://www.espruino.com/) interpreter targeted at embedded systems; earlier generations used [mJS](https://github.com/cesanta/mjs). JS as a language has matured over time, with the [Ecma standards group](https://en.wikipedia.org/wiki/Ecma_International) publishing the ECMAScript standard. Shelly's interpreter targets ECMAScript 5 (ES5, ca. 2009), and as such does not support the more modern parts of JS. Moreover some parts of ES5 (such as Array.reduce), and the [standard Espruino libraries](https://www.espruino.com/Reference) such as `E.init()`, are not provided. To figure out what is and is not available, the best bet is to review the [Espruino ES5 feature list](https://www.espruino.com/Features#es5) and then just watch out for syntax errors when developing code. For example, neither classes nor `use strict` are supported by Espruino ES5, so Shelly doesn't have them either. `[].reduce` is only supported by official Espruino boards, so again, not on Shelly. One successful strategy when attempting to use more modern JS code on Shelly is to ask your favourite LLM: "how can I do implement the following code on ES5: `<insert code>`".

A third party [Shelly Forge](https://github.com/mslavov/shelly-forge) project claims to provide a more modern TypeScript framework, amongst other things, that in turn is compiled down to an Shelly/ES5-compatible script; I have not tried it.

This doc is written in the context of a [Shelly Pro 4PM v2](https://kb.shelly.cloud/knowledge-base/shelly-pro-4pm-v2)(a 'gen3' device), using only the Shelly environment.

### Remote Procedure Call (RPC) Application Programming Interface (API)

The scripting environment relies heavily on Shelly's [Remote Procedure Call (RPC) based Application Programming Interface (API)](https://shelly-api-docs.shelly.cloud/gen2/) rather than provide a JS library that wraps the device's functionality. i.e. you interact with the device (mostly) by RPC rather than calling JS methods (as you would for an [Espruino](https://www.espruino.com/Reference#software) device).

Shelly provide 'ok' [API documentation](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures) and [a library of sample scripts](https://github.com/ALLTERCO/shelly-script-examples) of varying quality; it can be a steep learning curve if you're unfamiliar with the various parts: Javascript, RPCs, and so on. Cue this guide.

### Notifications, not `loop()`

If you're familiar with embedded system environments such as the Arduino, be aware the Shelly scripting environment is all about non-blocking callbacks; there is no `main()` or `loop()`, and no `delay()`. Instead, a Shelly script generally will be [notification](https://shelly-api-docs.shelly.cloud/gen2/General/Notifications/) based: a script registers one or more Status and/or Event handlers, and Shelly will call those handlers at the appropriate times providing relevant information in the notification (e.g. switch output state). A Status message is provided in response to a request and also periodically (without an explicit request) via a "heartbeat" notification (every 60s (unclear if that's configurable)). An example Status message is an energy meter load power update reporting the load current and/or power. The device hasn't materially changed state, rather an attribute has changed value. A regular Status notification will only include data that has changed; your code may need to deal with "missing" fields that haven't changed.

An Event is an asynchronous notification sent when the device state changes, e.g. button pressed. There can be conceptual overlap between an Event and Status update, e.g. the above energy status notification is only sent when the load energy consumption changes (or on a 60s interval), arguably both are "events". Practically, the [API](https://shelly-api-docs.shelly.cloud/gen2/General/Notifications/) defines what is an Event and what is a Status notification.

### Caveats

A Shelly script is simply a task running alongside all the other tasks in Shelly's "real time" operating system (RTOS) and has to share resources with those other tasks. Shelly provides some "guardrails" to try and prevent a script from negatively impacting the device, and it'll kill your script if it's misbehaving.

One notable [resource limit](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/#resource-limits) is on the number of nested [anonymous functions](https://www.javascripttutorial.net/javascript-anonymous-functions/). JS is commonly used in web apps where - much as with Shelly - it's (traditionally) all about callbacks. Anonymous functions are a common JS idiom that can (in some cases) lead to more concise, readable code. It's not uncommon to have anonymous functions end up calling other anonymous functions - however each invocation consumes resources. For a browser on a modern PC, not a problem - but Shelly warn that ["with more than 2 or 3 levels the device crashes"](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/#limited-levels-of-nested-anonymous-functions). The good news is that it's usually straightforward to structure your code to avoid anonymous functions - see the example Shelly provide in the above doc.

> [`var` vs `let`](https://www.google.com/search?q=javascript+let+var): though nominally only available in ES6, [Espruino parses](https://www.espruino.com/Features) `let` however treats it as `var`. The two have different behaviours in ES6, regarding scoping of the declared variables. It seems sensible to use (only) `var` to avoid misleading behaviour (`let` actually behaving as `var`) and/or avoid bugs if/when Shelly moves on to ES6.
{: .prompt-tip }

### Conventions and Best Practices

Code outside of a function is executed each time the script is started. It is good practice to arrange scripts such that there is one "entry point", often called `init` or `setup`. Shelly script convention is also to put all script "configuration parameters" into a global `CONFIG` object, at the top of the script file.

e.g.

rather than:

```javascript
var switchId = 2;
var someOtherParameter = 42;

// ... declare functions
function doSomeSetupWork () { /* ... */ }

//// all this should be moved to a function:
console.log("Starting ...")
// ... do some set up things
doSomeSetupWork();
// ...
// ... code to register a handler
////
```

preferred:

```javascript
const CONFIG = {
  switchId: 2,
  someOtherParameter: 42
}

// ... declare functions
function _doSomeSetupWork () { /* ... */ }

function init() {
  console.log("Starting ...")
  // ... do some set up things
  _doSomeSetupWork();
  // ...
  // ... code to register a handler
}
init();
```

With regard to matters of style: where you can't follow [Mozilla's style guide](https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Writing_style_guide/Code_style_guide/JavaScript) (e.g. ES5-constraints), all I can say is "at least be consistent".

In terms of naming functions: a convention I like to use is prefixing functions that are intended to be 'private' (as in, don't form part of any interface to, or invoked by, some outside party) with an underscore `_`. In the above, `init()` is intended to be part of "the interface", whereas `_doSomeSetupWork()` only exists as an implementation detail.

### Executing and Debugging

Scripts can be created, edited and executed via the web UI, or managed via the API - though strictly speaking, you can only manage via the API, the web UI just uses the API in a more convenient fashion. One hiccup with the API is that it can only accept a limited amount of data per RPC call; Shelly don't actually define this number but they do provide [a Python script](https://github.com/ALLTERCO/shelly-script-examples/blob/main/tools/put_script.py) that 'chunks' the upload into a maximum of 1024 octets at a time.

The web UI is certainly convenient when getting started; when using the web UI the script is stopped and started on each save. The web UI also provides a limited Read-Eval-Print-Loop (REPL)-like 'console' that you can use to execute code quasi-interactively. The 'console' is very clunky - for example it does not echo back the command you entered, and entering a bad command (e.g. creating a syntax error) will stop the script from running.

One important part of writing code is being able to observe the script's operation, and the simplest form of that is a [liberal application of print statements](https://blog.startifact.com/posts/print-debugging/). Shelly provides the ubiquitous JS `console.log` - but where is the 'console' here? Shelly directs output to 'console' to a websocket; the web UI can connect to this ("enable websocket debug"), or you can use separate tools to view these over the network. I recommend [websocat](https://github.com/vi/websocat).

Another debugging technique is to call individual functions within your script via RPC, see [the example in the doc](https://shelly-api-docs.shelly.cloud/gen2/Scripts/Tutorial#step-7-create-a-function-that-replaces-strings). For example, you might write a debug function that returns script global variable values, and call that remotely.

#### Log Format

The 'raw' log is a stream of individual JSON objects of the form:

```shell
% websocat ws://${SHELLY}/debug/log
{"ts":1740133629.559, "level":2, "data":"<message>", "fd":1}
```

- `ts`: timestamp
- `level`: log level? usually 2
- `fd`: appears to be related to the Shelly process that generated the message (file descriptor?)
- `data`: log message, see below

The `data` field is of no fixed form: messages generated from Shelly itself are mostly free-form text, for example when starting a script via the web UI: `shos_rpc_inst.c:243     script.start via WS_in 192.168.43.1:52617`. In some cases a stringified JSON object is included, what appears to be a debug message of the form `$filename:$lineno $jsonstring`; refer the `shelly_notification` examples below. These all appear to have a `fd` of 1; script `console.log()` messages are >1. Messages from different scripts have their own distinct `fd`, but it's unclear when/how the `fd` is generated/assigned so it's an unreliable identifier over the medium/long term.

#### `console.log()` message escaping and splitting

Shelly's `console.log()` does not behave identically to more common implementations (e.g. in the browser). One of the differences is that messages generated via `console.log()` are split into separate log messages at the 128 character mark. e.g.

```javascript
var s = '1234567890';
console.log(s + s + s + s + s + s + s + s + s + s + s + s + s + s); // 140 chars
```

results in

```shell
{"ts":1740136531.466,"level":2,"data":"12345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678","fd":101}
{"ts":1740136531.468,"level":2,"data":"901234567890","fd":101}
```

This can be annoying when dumping largish JSON objects (which will expand even further by escaping special characters). Shelly generated messages are not split.

#### Filtering Logs

The Shelly logs are rather noisy (at least for the Pro4PM); the following is with no scripts running (firmware `1.4.4| 679fcca9`):

```shell
% websocat ws://${SHELLY}/debug/log | jq --raw-output --compact-output
{"ts":1739582438.840,"level":2,"data":"shelly_debug.cpp:236    Streaming logs to 192.168.118.215:51161","fd":1}
{"ts":1739582460.123,"level":2,"data":"shelly_notification:162 Status change of switch:2: {\"id\":2,\"aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000},\"ret_aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000}}","fd":1}
{"ts":1739582460.126,"level":2,"data":"shelly_notification:162 Status change of switch:0: {\"id\":0,\"aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000},\"ret_aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000}}","fd":1}
{"ts":1739582460.129,"level":2,"data":"shelly_notification:162 Status change of switch:1: {\"id\":1,\"aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":306.890},\"ret_aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000}}","fd":1}
{"ts":1739582460.132,"level":2,"data":"shelly_notification:162 Status change of switch:3: {\"id\":3,\"aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000},\"ret_aenergy\":{\"by_minute\":[0.000,0.000,0.000],\"minute_ts\":1739582460,\"total\":0.000}}","fd":1}
```

A "heartbeat" event is is posted every minute with energy data, one for each channel (by this measure the Pro4PM has four hearts). The above appear to be debug messages for these "hearbeats"; presumably they were to be removed before release; even if they do hang around over firmware updates the `shelly_notification:162` string is likely to change when Shelly update their firmware. It would be unreliable to depend on these messages. Fortunately these events are also sent to script `Shelly.addStatusHandler()` handlers, so it's easy to log your own as needed (well, except for the 128-character message splitting, as above).

Various other messages are emitted when the Shelly is going about its business:

```shell
{"ts":1740133629.559, "level":2, "data":"shos_rpc_inst.c:243     switch.toggle via WS_in 192.168.43.1:52144", "fd":1}
{"ts":1740133629.594, "level":2, "data":"shelly_notification:162 Status change of switch:3: {\"id\":3,\"output\":true,\"source\":\"WS_in\"}", "fd":1}
```

[`jq`](https://jqlang.org/) is very helpful to filter out the Shelly cruft (`fd=1`) and show only the abbreviated timestamp and `.data` payload for script messages:

```shell
% websocat ws://${SHELLY}/debug/log | jq --raw-output 'select(.fd != 1) | [(.ts | strftime("%M:%S")), .data]| @tsv'
```

On the jq expression `select(.fd != 1) | [(.ts | strftime("%M:%S")), .data]| @tsv`: selects only records that are not from Shelly itself (match `.fd != 1`), then selects only the formatted timestamp and data fields, emitting them as tab-separated strings. jq is definitely one of those tools that is easier to read than write.

To achieve a similar filter in a script handler you have to ignore any messages you're not interested in, e.g. in this example ignore messages that don't report a `delta.apower`:

```javascript
  function statusHandler(notifyStatus) {
    if(typeof notifyStatus.delta.apower !== 'undefined') return;
    ...
```

## Artificial Example

The script below is intended to periodically increment a global variable if the current second is less than the :30s mark, otherwise decrement the variable.

The Shelly [`Timer`](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures/#timer) object is used to create a periodic timer; `Timer.Set` will call the provided callback function when the timer goes off. In this first example, the callback is an anonymous function.

The anonymous function invokes the [`Sys.GetStatus`](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Sys#sysgetstatus) RPC call, which returns the current time in seconds (since 1970, aka `unixtime`). That invocation via `Shelly.call` expects a callback and again an anonymous function is used. This anonymous function copies the RPC result's `unixtime` field over to the `unixtime` global variable (global variables always being in scope).

```javascript
count = 0;
unixtime = 0;

function startTimer() {
  Timer.set(6*1000,  // duration is in milliseconds
    true,
    function () {
      Shelly.call('Sys.GetStatus', {}, function (result) {unixtime = result.unixtime});
      if (unixtime % 60 < 30) {
        count++;
      } else {
        count--;
      }
      console.log('unixtime%60=', unixtime%60, 'count=', count);
    }
  );
}

startTimer();
```

### Bugs

There are a multitude of problems with the above implementation, not least of which is a bug where it lags behind the actual time. Here is the console output, with the log message timestamp in MM:SS on the left:

```text
57:18   unixtime%60= 0 count= 1
57:24   unixtime%60= 18 count= 2
57:30   unixtime%60= 24 count= 3
57:36   unixtime%60= 30 count= 2
57:42   unixtime%60= 36 count= 1
57:48   unixtime%60= 42 count= 0
```

Note the `unixtime%60`, which is the 00-59 seconds component of current time, lags behind the actual time, and the value on the first call is 0 (the initial value when declared). What's going on here is a race condition: the `Shelly.call` is asynchronous, that is the script executes that function - dutifully firing off an RPC call - and then immediately carries on with the next instruction, in this case the `if (unixtime%60 < 30)` statement. The actual RPC call (likely) hasn't finished yet and the callback has not been called, and thus the result not yet copied over to the global `unixtime`. The asynchronous `Shelly.call` does not wait for the RPC to complete, it's a "fire and forget".

Dealing with these kinds of issues is challenging and the best approach is to simply avoid them by structuring your code differently, or to use synchronous calls instead. Both are demonstrated below.

Other issues with the above include assuming errors aren't a thing, and having anonymous functions nested two-deep (Timer callback > Shelly.call callback). Frankly, I've always found trying to read code that has anonymous functions spread over half dozen or more lines, end often ending in a spray of closing braces and brackets, unreadable.

### Take 2 - Anonymous -> Named

So we can do better:

```javascript
var count = 0;

function getStatusCallback(result) {
  if (result.unixtime % 60 < 30) {
    count++;
  } else {
    count--;
  }
  console.log('unixtime%60=', result.unixtime%60, 'count=', count);
}

function startTimer() {
  Timer.set(6*1000,
    true,
    function () {
      Shelly.call('Sys.GetStatus', {}, getStatusCallback);
    },
  );
}

startTimer();
```

```text
59:52   unixtime%60= 52 count= -1
59:58   unixtime%60= 58 count= -2
00:04   unixtime%60= 4 count= -1
00:10   unixtime%60= 10 count= 0
00:16   unixtime%60= 16 count= 1
```

Much better - now in sync as expected (i.e. the `unixtime%60` reported in `getStatusCallback()` actually lines up with the wallclock seconds). The script now has only one anonymous function (ergo not nested), has eliminated a global variable, and is easier to read. There are still a few problems, such as still ignoring errors.

### Take 3 - Asynchronous -> Synchronous

The 'take 2' script chains two asynchronous calls: the anonymous function callback from `Timer.set()` does nothing else but fire off an RPC `Shelly.call('Sys.GetStatus')`, in turn that at returns the device `Status` to `getStatusCallback()` which does "the work". This isn't a problem per se - and indeed the `Timer` call is well and good (i.e. no [busy waits](https://en.wikipedia.org/wiki/Busy_waiting)!), though as it happens the second asynchronous call is more simply implemented as a synchronous call, i.e. replace the `Sys.GetStatus` RPC call + callback with a straight-up `Shelly.getComponentStatus('Sys')`. Whilst we're there, we should also handle the situation where the clock [may not yet be synced](https://shelly-api-docs.shelly.cloud/gen2/ComponentsAndServices/Sys#status).

Rather than try and dump all that in an anonymous function, we'll put this code in a named function and pass that to the `Timer.set()` callback.

```javascript
var count = 0;

function incrementTimerCallback() {
  var unixtime = Shelly.getComponentStatus('Sys').unixtime;
  // unless you've time-travelled back to 1 Jan 1970, a valid unixtime should
  // be a number > 0 (i.e. a JS "true" value); anything else is an error
  if (!unixtime) {
    // possibly because the clock hasn't sync'd, or other reasons
    console.log('WARN failed to get unixtime', unixtime);
    return;
  }
  if (unixtime % 60 < 30) {
    count++;
  } else {
    count--;
  }
  console.log('unixtime%60=', unixtime%60, 'count=', count);
}

function startTimer() {
 Timer.set(6*1000, true, incrementTimerCallback);
}

startTimer();
```

The improvements in this version:

- handles exceptional conditions that may reasonably arise (i.e. not having time sync)
- avoids anonymous functions completely
- is easier to follow (imnsho)

#### Don't Get Caught Out When Things Fail

Regarding the test for `unixtime`: it's always a good idea to test return values from Shelly calls for validity first before charging ahead and attempting to use return object values; e.g. `Shelly.getComponentStatus("switch", 0)` could return an object, or could also return other values (e.g. `null` if the switch index is out of range for the device). Recall that Shell will kill your script on exceptions, including attempting to dereference a property that doesn't exist ('Uncaught ReferenceError').

In the above situation it's sufficient to simply test if `unixtime` is "true", as all the "false" values are either Shelly errors or an invalid timestamp (`null`, `undefined` and `0`). More generally separate tests for "Shelly error" and "is the return value as expected" are best.

On a related note some RPC calls - specifically [Notifications](https://shelly-api-docs.shelly.cloud/gen2/General/Notifications) - will return objects where "certain status keys will only exist in certain situations", and it appears unpredictable in at least some cases (e.g. switch `output` and `apower` info), see the script in the next post for one way to handle this.

### Take 4 - Going to Town

The previous version is not easy to configure; changing the period or increment means reading the whole script and editing parameters in the middle of code. Let's move everything to a 'CONFIG' object. And why not allow for multiple timers, and sprinkle in some logging (that can be turned off when not needed)?

A couple of iterations later, and we have this:

```javascript
// Manage a counter that increments or decrements based on the current second.
// - if current second is <30, increment; otherwise decrement

var CONFIG = {
  log: true,          // enable/disable informational/debug log messages
  timers: [
      { period: 6 },  // use defaults for increment and initialCount
      { period: 10, increment: 2, initialCount: 100 }
  ]
}


function _log() {
  // include a tag in the log messages for easier filtering
  if (CONFIG.log) console.log('[thecount]', arguments.join(' '));
}

// factory function to create timer objects (ES5: no class)
function createTimerObject(id, period, increment, initialCount) {
  return {
    id: id,
    period: period || 1,
    increment: increment || 1,
    count: initialCount || 0
  };
}

function incrementTimerCallback(timerObject) {
  //_log("incrementTimerCallback", JSON.stringify(timerObject));
  var unixtime = Shelly.getComponentStatus('Sys').unixtime;
  // unless you've time-travelled back to 1 Jan 1970, a valid unixtime should
  // be a number > 0 (i.e. a JS "true" value); anything else is an error
  if (!unixtime) {
    // possibly because the clock hasn't sync'd, or other reasons
    // not _log as this should always be reported
    console.log('WARN failed to get unixtime', unixtime);
    return;
  }
  if (unixtime % 60 < 30) {
    timerObject.count += timerObject.increment;
  } else {
    timerObject.count -= timerObject.increment;
  }
  _log('incrementTimerCallback', 'sec=' + unixtime%60, 'timer:' + timerObject.id,
    'count=' + timerObject.count);
}

function startTimer(timerObject) {
  _log("startTimer", JSON.stringify(timerObject));
  Timer.set(timerObject.period*1000, true, incrementTimerCallback, timerObject);
}

function init() {
  _log("init");
  for (const i in CONFIG.timers) {  // for...in iterates over indices of an array
    var t = CONFIG.timers[i];
    startTimer(createTimerObject(i, t.period, t.increment, t.initialCount));
  }
}

init();
```

Regarding the `_log()` function: the JS spread operator `...` would normally be used to handle the variable number of arguments to `_log()`, however Shelly doesn't support it; instead [`arguments`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments) is used.

What's left? Plenty of course:

- some more introductory documentation would be good, including how to use the script and example use cases
- unit tests!

But that's not for today.

## Next - A Real Example

[Part 2]({% post_url 2025-02-13-shelly-scripting-part2 %}) covers a practical / real-world example.
