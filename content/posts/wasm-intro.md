+++
date = '2025-09-21T08:44:34+03:00'
title = 'WebAssembly interaction with a Python Host'
draft = false
tags = ["wasm", "python", "rust"]
summary = "Running Web Assembly modules in Python can be tricky, due to weak wasm tools support in python ecosystem"
+++

WebAssembly (wasm) is a young technology, but a very promising one.

In this article i'll show how to run wasm modules from python code. When i did it for the first time it was challenging due to lack of information and weak wasm support in python ecosystem.

The basic wasm workflow consists of two parts: **host** and **guest**.

- **Host** is the environment that runs wasm and controls its execution (e.g. a browser or a runtime like [wasmtime](https://docs.wasmtime.dev/)).

- **Guest** is the wasm module itself, compiled from Rust, C, Wat, etc).

Guest can export / import functions. But for function calling only simple types like `bool` / `i32` arguments can be used. (note: actually `bool` arguments is converting to `i32` too)

If a function needs more complex data (like `strings`, `structs`, `vectors`), it must be transferred through **linear memory**.
# wasm memory

Host <-> guest interaction works through wasm linear memory.

Both host and guest has full access to wasm instance memory. For example:
1. Host wants to send data to the instance, it writes the data into the instance’s memory at some offset.
2. Then the host calls a wasm function, passing an `i32` pointer to that data.
3. The wasm code can then read from memory starting at that pointer.

# component model
In Jan 2024 Bytecode Alliance introduced [Component Model](https://component-model.bytecodealliance.org/) - a new way to structure wasm modules.

It defines a higher-level interface for interoperability between languages and runtimes. With the component model, modules can talk in terms of complex types instead of raw pointers.

This makes wasm much more practical for multi-language ecosystems.

# wit
The [wasm interface type](https://component-model.bytecodealliance.org/design/wit.html) language is used to define interfaces for interaction between host and guest.

with `wit` we can do e.g
```wit
package xyz:machine;

interface machine-interface {
    type machine-id = u64;

    record point {
        x: u32,
        y: u32,
    }

    record run-result {
	    machine-id: machine-id,
	    message: string,
    }

    get-u32: func() -> u32;
    get-str: func() -> string;
    get-vec: func() -> list<u8>;
    run: func(machine: machine-id, start: point, destination: point) -> run-result;
    count-symbols: func(s: string) -> u32;
}

world machine {
    export machine-interface;
}
```

With this approach developers can avoid implementing interfaces manually and just generate host / guest parts code with such tools as `wit-bindgen`, `wasmtime-bindgen` etc

# the problem in python
But... Python does not support these codegen tools and component model at all

This means on python side implementation must be manually implemented, even if `wit` is used in module side to simplify module implementation.

Let's say wasm module is written in rust with wit-bindgen


# rust side (guest)
```rust
wit_bindgen::generate!({
    path: "wit",
    world: "machine",
});

use crate::exports::xyz::machine::machine_interface::{Guest, MachineId, Point, RunResult};

struct Machine;

impl Guest for Machine {
    fn get_u32() -> u32 {
        123
    }

    fn get_str() -> String {
        String::from("simple string to return")
    }

    fn get_vec() -> Vec<u8> {
        return vec![5, 4, 3, 2, 1, 10];
    }

    fn run(machine: MachineId, start: Point, destination: Point) -> RunResult {
        RunResult {
            machine_id: 1,
            message: format!("machine {machine} run from {start:?} to {destination:?}"),
        }
    }

    fn count_symbols(s: String) -> u32 {
        s.chars().count() as u32
    }
}

export!(Machine);
```

# python side (host)
## simple argument and return value
```python
import struct
from wasmtime import loader, Config, Engine, Store, Module, Linker

engine = Engine()
store = Store(engine)
linker = Linker(engine)

wasm_intro = Module.from_file(engine, "wasm_intro.wasm")
instance = linker.instantiate(store, wasm_intro)
exports = instance.exports(store)
memory = exports['memory']

print([e for e in exports])
# ['memory', 'xyz:machine/machine-interface#get-u32', 'xyz:machine/machine-interface#get-str', 'cabi_post_xyz:machine/machine-interface#get-str', 'xyz:machine/machine-interface#get-vec', 'cabi_post_xyz:machine/machine-interface#get-vec', 'xyz:machine/machine-interface#run', 'cabi_post_xyz:machine/machine-interface#run', 'xyz:machine/machine-interface#count-symbols', 'cabi_realloc']

# call function
get_u32 = exports['xyz:machine/machine-interface#get-u32']
print(get_u32(store)) # 123
```

As you probably notice, every `xyz:machine/*` function has `cabi_post_xyz:machine/*` pair.
That second function must be called in order to clear memory after data returned from function is handled.

This was an easy interface,  now let's write host code for functions with more complex  interfaces.

## complex return value (string / vector)

Here `cabi_post*` need to be used to clean memory:
```python
# string
get_str = exports['xyz:machine/machine-interface#get-str']
cabi_post_get_str = exports['cabi_post_xyz:machine/machine-interface#get-str']

def parse_string(memory, store, ptr) -> str:
    STR_SIZE = 8
    string_struct = memory.read(store, ptr, ptr + STR_SIZE)
    str_ptr, str_len = struct.unpack('<II', string_struct)
    str_bytes = memory.read(store, str_ptr, str_ptr + str_len)
    return str_bytes.decode()

str_ptr = get_str(store)
print(parse_string(memory, store, str_ptr)) # "simple string to return"


# vector of u8
get_vec = exports['xyz:machine/machine-interface#get-vec']
cabi_post_get_vec = exports['cabi_post_xyz:machine/machine-interface#get-vec']

def parse_u8_vec(memory, store, ptr) -> list[int]:
    U8_SIZE = 1
    bytes = memory.read(store, ptr, ptr + 8)
    data_ptr, length = struct.unpack('<II', bytes)
    print(data_ptr, length)
    result = []

    next_elem_ptr = data_ptr
    for _ in range(length):
        u8_bytes = memory.read(store, next_elem_ptr, next_elem_ptr + U8_SIZE)
        number = struct.unpack('<B', u8_bytes)[0]
        result.append(number)
        next_elem_ptr += U8_SIZE

    return result

vec_ptr = get_vec(store)
print(parse_u8_vec(memory, store, vec_ptr)) # [5, 4, 3, 2, 1, 10]
```

## complex argument

In order to send complex data, e.g string to wasm module it is required to write data to memory. But where it should be written?

`wit_bindgen` provides `cabi_realloc` function that can be used to allocate memory, and then it returns offset in linear memory. Definition is something like `cabi_realloc(old_ptr:i32, old_size:i32, align:i32, new_size:i32) -> i32`

```python
count_symbols = exports['xyz:machine/machine-interface#count-symbols']
cabi_realloc = exports['cabi_realloc']

encoded_str = "string with symbols (~25)".encode()
ptr = cabi_realloc(store, 0, 0, 4, len(encoded_str))
memory.write(store, encoded_str, ptr)  # write to instance memory
count = count_symbols(store, ptr, len(encoded_str))  # note ptr is offset of the string in instance memory
print(count)  # 25
```

Notice that there is no need to clean up string data written into instance memory from the host side. When `count_symbols` is called, wasm module must manage this memory itself and will free it when it’s no longer needed.

# conclusion
Despite the fact that wasm is promising for interaction between different subsystems, so far, tool support for languages other than rust may be quite limited. But as discussed before, basically many things are easy to do manually.

Wasm modules also seem promising as a more lightweight replacement of containers.


# additional
There are also a bunch of useful CLI tools:
- `wasm-objdump` - inspect wasm binaries. Useful for debugging or understanding module.
- `wasm2wat` - converts a `.wasm` binary into human-readable WAT (WebAssembly Text format).
- `wit-bindgen` - generates host/guest bindings from `.wit` definitions.
