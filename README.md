# Extism JavaScript PDK

> **Note**: This is very experimental. If you are interested in helping or following development, join the [#js-pdk](https://discord.gg/ZACPSVz9) room in our discord channel.

## Overview

This PDK uses [QuickJS](https://bellard.org/quickjs/) and [wizer](https://github.com/bytecodealliance/wizer) to run javascript as an Extism Plug-in.

This is essentially a fork of [Javy](https://github.com/Shopify/javy) by Shopify. We may wish to collaborate and upstream some things to them. For the time being I built this up from scratch using some of their crates, namely quickjs-wasm-rs.

## How it works

This works a little differently than other PDKs. You cannot compile JS to Wasm because it doesn't have an appropriate type system to do this. Something like [Assemblyscript](https://www.assemblyscript.org/) is better suited for this. Instead, we have compiled QuickJS to Wasm. The `extism-js` command we have provided here is a little compiler / wrapper that does a series of things for you:

1. It loads an "engine" Wasm program containing the QuickJS runtime
2. It initializes a QuickJS context
3. It loads your js source code into memory
4. It parses the js source code for exports and generates 1-to-1 proxy export functions in Wasm
5. It freezes and emits the machine state as a new Wasm file at this post-initialized point in time

This new Wasm file can be used just like any other Extism plugin.

## Install the compiler

We now have released binaries. Check the [releases](https://github.com/extism/js-pdk/releases) page for the latest.

> **Note**: Windows is not currently a supported platform.

### Installing on a Mac

Run the script below to install on Mac:

```bash
export TAG=v0.2.0
curl -O "https://github.com/extism/js-pdk/releases/download/$TAG/extism-js-x86_64-macos-$TAG.gz"
gunzip extism-js*.gz
sudo mv extism-js-* /usr/local/bin/extism-js
chmod +x /usr/local/bin/extism-js
```

### Installing on Linux

Run the script below to install on Linux:

```bash
export TAG=v0.2.0
curl -O "https://github.com/extism/js-pdk/releases/download/$TAG/extism-js-x86_64-linux-$TAG.gz"
gunzip extism-js*.gz
sudo mv extism-js-* /usr/local/bin/extism-js
chmod +x /usr/local/bin/extism-js
```


Then run command with no args to see the help:

```
extism-js
error: The following required arguments were not provided:
    <input>

USAGE:
    extism-js <input> -o <output>

For more information try --help
```

> **Note**: If you are using mac, you may need to tell your security system this unsigned binary is fine. If you think this is dangerous, or can't get it to work, see the "compile from source" section below.

Try it on a script file. Name this `script.js`:

```javascript
// script.js

const VOWELS = [
    'a', 'e', 'i', 'o', 'u',
]

function count_vowels() {
    let input = Host.inputString()
    let count = 0
    for (let i = 0; i < input.length; i++) {
        if (VOWELS.includes(input[i].toLowerCase())) {
            count += 1
        }
    }
    Host.outputString(JSON.stringify({count}))
    return 0
}

module.exports = { count_vowels };
```

```bash
extism-js script.js -o count_vowels.wasm
extism call count_vowels.wasm count_vowels --input="Hello World!" --wasi
# => {"count":3}                          
```

## Exports in JS

For the most part, you can write your JS how you want. In order to export a function to Extism you must use this `module.exports = {f1, f2, ..}` syntax:

```javascript

function myFunc() {
    //...
}

module.exports = { myFunc }
```

We will better support module systems in the future but for now we are hard coding the compiler to this one export syntax.

## Compiling the compiler from source

You need the wasi sdk which can be fetched with the makefile:

```
make download-wasi-sdk
```

Then run make to compile the core crate (the engine) and the cli:

```
make
```

```bash
./target/release/extism-js script.js -o out.wasm
extism call out.wasm count_vowels --wasi --input="Hello World Test!"
# => "{\"count\":4}"
```

## Why not use Javy?

Javy, and many other high level language Wasm tools, assume use of the *command pattern*. This is when the Wasm module only exports a main function and communicates with the host through stdin and stdout. With Extism, we have more of a shared library interface. The module exposes multiple entry points through exported functions. Furthermore, Javy has many Javy and Shopify specific things it's doing that we will not need. However, the core idea is the same, and we can possibly contribute by adding support to Javy for non-command-pattern modules. Then separating the Extism PDK specific stuff into another repo.

## What needs to be done?

Implemented so far:

* Host.inputBytes
* Host.inputString
* Host.outputBytes
* Host.outputString
* Var.get
* Var.set
* console.log
* console.error
* throw Error

The above are implemented but need some more validation and resilience built into them. debating whether I should implement the bulk of the code in js or rust. Working on implementing the other pdk methods.

I've got the exports to work, but it's a fragile and complicated solution. Will write it up soon, and maybe it can be replaced with something simpler.
