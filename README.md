# Frida-JS [![Build Status](https://github.com/httptoolkit/frida-js/workflows/CI/badge.svg)](https://github.com/httptoolkit/frida-js/actions) [![Available on NPM](https://img.shields.io/npm/v/frida-js.svg)](https://npmjs.com/package/frida-js)

> _Part of [HTTP Toolkit](https://httptoolkit.com): powerful tools for building, testing & debugging HTTP(S)_

Pure-JS bindings to control Frida from node.js &amp; browsers.

This module provides access to Frida, without bundling Frida itself. That means no native binaries included, no compilation required, and no heavyweight files (the entire library is under 10KB). This works by making WebSocket connections (supported in Frida v15+) directly to an existing Frida instance elsewhere.

This is particularly useful in mobile device scenarios, as you can now run Frida purely on the device (as an Android/iOS Frida server instance, or using Frida-Gadget embedded in a specific application) and connect to it through using this library as a tiny client in Node.js or a browser elsewhere.

## Getting Started

```bash
npm install frida-js
```

First, you'll need a v15+ Frida instance to connect to. If you don't have this already, you'll first want to download, extract & run the `frida-server` release for your platform from https://github.com/frida/frida/releases/latest. In future Frida-JS will provide an API to do this automatically on demand.

Frida-JS supports both local & remote Frida instances, but doesn't do automated tunnelling over USB or ADB, and so mobile devices will require setup to expose the Frida port (27042) so that it's accessible to this client.

Once you have a running accessible Frida server, to use Frida-JS you first call the exported `connect()` method and wait for the returned promise to get a FridaClient, and then call the methods there to query the available targets and hook them (full API listing below). For example:

```javascript
import { connect } from 'frida-js';

const fridaClient = await connect();

await fridaClient.spawnWithScript(
    '/usr/bin/your-target-bin',
    ['some', 'arguments'],
    `
        const targetFn = DebugSymbol.fromName('a_target_function');

        // Hook the target function and replace the argument
        Interceptor.attach(ptr(targetFn.address), {
            onEnter(args) {
                // Modify your target functions args
            },
            onLeave(retval) {
                // Or return value
            }
        });
    `
);
```

To connect to a remote instance (such as a mobile device) you can pass options to `connect()`, such as `connect({ host: 'localhost:27042' })`.

See the full API reference below for more details of the Frida APIs exposed, or see the test suite for a selection of working examples, and fixtures to test against.

## Frida Caveats

There are a few general Frida issues you might commonly run into while using this library, documented here for easier troubleshooting:

* On Linux, to attach to non-child processes with Frida, you will need to set `ptrace_scope` to `0`. You can do so with this command:
    ```bash
    sudo sysctl kernel.yama.ptrace_scope=0
    ```
* To use Frida in Docker, you will need to disable seccomp and enable PTRACE capabilities, like so:
    ```bash
    docker run --cap-add=SYS_PTRACE --security-opt seccomp=unconfined <...>
    ```
    Alternatively, in some scenarios you might want to just set `--privileged` instead.

## API reference

### `FridaJS.connect([options])`

Connects to a local Frida server, and returns a promise for a FridaClient.

Options must be an object containing the connection parameters. `host` is the only parameter currently supported, and must be set to the hostname and (optional) port string for the target Frida instance. If not set, it defaults to `localhost:27042`.

### `fridaClient.queryMetadata()`

Returns a promise for an object containing the exposed system parameters of the target Frida server. For example, on Linux this might look like:

```json
{
  "arch": "x64",
  "os": { "version": "22.10", "id": "ubuntu", "name": "Ubuntu" },
  "platform": "linux",
  "name": "your-system-hostname",
  "access": "full"
}
```

The exact parameters returned may vary and will depend on the specific target system.

### `fridaClient.enumerateProcesses()`

Returns a promise for an array of `[pid: number, processName: string]` pairs. You can use this to query the currently running processes that can be targeted on your local machine.


### `fridaClient.injectIntoProcess(pid: number, script: string)`

Injects a given Frida script into a target process, specified by PID. Returns a promise that will resolve once the script has been successfully injected.

### `fridaClient.injectIntoNodeJSProcess(pid: number, script: string)`

Injects real JavaScript into Node.JS processes specifically. Rather than requiring a full Frida script, this takes any normal JS script, and conveniently wraps it with a script to inject it into the V8 event loop for you, so you can just write JS and run it in a target directly.

### `fridaClient.spawnWithScript(command: string, args: string[], script: string)`

Takes a command to run and arguments, launches the process via Frida, injects the given Frida script before it starts, and then resumes the process.

### `fridaClient.disconnect()`

When you're done, it's polite to disconnect from Frida. This returns a promise which resolves once the WebSocket connection has been closed.