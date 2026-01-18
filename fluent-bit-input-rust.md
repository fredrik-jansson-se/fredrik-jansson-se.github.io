## Fluent Bit Input Plugin in Rust

An example on how to write an input plugin for [Fluent Bit](https://fluentbit.io/) in Rust (with some C). All the code can be found [here](https://github.com/fredrik-jansson-se/fluent-bit-input-rust).

Fluent assumes input plugins are in a shared library and all examples I found are written in C. For work I had existing code in Rust that consumes logs from a proprietary source that we wanted to ingest in Fluent Bit. Below you'll find the scaffolding for writing a plugin with the bulk of the code in Rust.

## Prerequisites
You need to install:
 1. gcc/clang
 2. cmake
 3. flex
 4. bison
 5. rust

## Code

### C Code

Fluent Bit loads input plugins from a shared library using a convention. Assume the input plugin is named `example`, Fluent assumes the shared library to be named `flb-in_example.so`. From that library it will load a struct describing the plugin called `in_example_plugin`.

I've tried to keep the C code to a absolut minimum.

This struct is defined in [`csrc/in_example_plugin.c`](https://github.com/fredrik-jansson-se/fluent-bit-input-rust/blob/master/csrc/in_example_plugin.c) and looks like
``` c
struct flb_input_plugin in_example_plugin = {
    .name         = "example",
    .description  = "Example log input plugin",
    .event_type   = FLB_INPUT_LOGS,
    .cb_init      = cb_init,
    .cb_pre_run   = NULL,
    .cb_collect   = cb_collect,
    .cb_flush_buf = NULL,
    .config_map   = config_map,
    .cb_exit      = cb_exit,
};
```

**NOTE**, name has to follow the naming convention, i.e. has to be `"example"`.

The `cb_init`, `cb_collect` and `cb_exit` are callback functions called by Fluent Bit. We will actually implement those in Rust, hence, they are just declared in the C code:
``` c
extern int cb_init(struct flb_input_instance *, struct flb_config *, void *);
extern int cb_collect (struct flb_input_instance *, struct flb_config *, void *);
extern int cb_exit (void *, struct flb_config *);
```

The `config_map` describes the configuration parameters the plugin accepts, in our simple example we have a single one that tells Fluent Bit how often to call `cb_collect`.
``` c
static struct flb_config_map config_map[] = {
   {
    FLB_CONFIG_MAP_INT, "interval_sec", "30",
    0, FLB_FALSE, 0,
    "Collect interval."
   },
   {0}
};
```

### Rust Code

C bindings are generated using `bindgen`, please see [build.rs](https://github.com/fredrik-jansson-se/fluent-bit-input-rust/blob/master/build.rs).

All Rust code is compiled into a static library, the shared library is linked from the C and Rust code in the Makefile.

The Rust callback functions has to be declared as `unsafe extern "C"` and also decorated with `#[no_mangle]` to prevent mangling of the function name.

The `cb_init` function is called once by Fluent Bit. It allocates a "context", `FLBContext` that Fluent Bit provides in all callbacks after `cb_init`. The FLBContext is allocated on the heap and the leaked (`into_raw`) and handed over to Fluent Bit. Last, it calls `flb_input_set_collector_time` to indicate the callback interval for cb_collect.

```rust
#[no_mangle]
unsafe extern "C" fn cb_init(
    flb_input_instance: *mut bindings::flb_input_instance,
    flb_config: *mut bindings::flb_config,
    _user: *mut std::os::raw::c_void,
) -> std::os::raw::c_int {
    let ctx = Box::new(FLBContext { collect_cnt: 0 });

    let cfg = check_result!(configure(&ctx, flb_input_instance));

    let ctx_ptr = Box::into_raw(ctx) as *mut c_void;
    bindings::flb_input_set_context(flb_input_instance, ctx_ptr);

    bindings::flb_input_set_collector_time(
        flb_input_instance,
        Some(cb_collect),
        cfg.interval_sec,
        0,
        flb_config,
    );

    0
}
```

At shutdown of Fluent Bit, `cb_exit` is called, to not leak memory, we put the FLBContext back into a Box and drop it.

``` rust
#[no_mangle]
unsafe extern "C" fn cb_exit(
    ctx: *mut std::os::raw::c_void,
    _flb_config: *mut bindings::flb_config,
) -> std::os::raw::c_int {
    // Make sure we drop the CTX allocated in cb_init
    let unboxed_ctx: &mut FLBContext = &mut *(ctx as *mut FLBContext);
    drop(Box::from_raw(unboxed_ctx));
    0
}
```

At the configured interval, Fluent Bit will call `cb_collect` to gather logs. Logs are fed into Fluent using [MessagePack](https://msgpack.org/). I choose to use the `rmp` crate to do the packing in Rust and just hand the packed buffer to Fluent. The code gets the current time and adds a simple record `collect-calls`.

``` rust
/// # Safety
///
/// This function assumes cb_init has been called to initialze ctx
#[no_mangle]
pub unsafe extern "C" fn cb_collect(
    flb_input_instance: *mut bindings::flb_input_instance,
    _flb_config: *mut bindings::flb_config,
    ctx: *mut std::os::raw::c_void,
) -> std::os::raw::c_int {
    let ctx: &mut FLBContext = &mut *(ctx as *mut FLBContext);

    // Increate collect count
    ctx.collect_cnt += 1;

    // Constrult message pack
    // It has the following structure [time-stampe, {a:foo, b:bar, ..}]
    let mut mp = Vec::new();

    // Array with two items
    check_result!(rmp::encode::write_array_len(&mut mp, 2));

    // Encode time

    let now = check_result!(
        std::time::SystemTime::now().duration_since(std::time::SystemTime::UNIX_EPOCH)
    );
    const NS: u128 = 1000000000;
    let now_sec = now.as_nanos() / NS;
    let now_nsec = now.as_nanos() - now_sec * NS;

    // The timestamp is written as 8 bytes, 4 bytes seconds and 4 bytes nanos
    check_result!(rmp::encode::write_ext_meta(&mut mp, 8, 0));
    mp.extend_from_slice(&(now_sec as u32).to_be_bytes());
    mp.extend_from_slice(&(now_nsec as u32).to_be_bytes());

    // Let's add a single record to the record map
    check_result!(rmp::encode::write_map_len(&mut mp, 1));

    // The record will be the number of cb_collect calls
    check_result!(rmp::encode::write_str(&mut mp, "collect-calls"));
    check_result!(rmp::encode::write_u64(&mut mp, ctx.collect_cnt as _));

    let res = bindings::flb_input_chunk_append_raw(
        flb_input_instance,
        std::ptr::null(),
        0,
        mp.as_mut_ptr() as *const c_void,
        mp.len() as u64,
    );

    if res != 0 {
        eprintln!("Failed to store chunk");
    }

    res
}
```

## Testing the Code
The Makefile will download Fluent Bit locally and build it.
``` shell
git clone git@github.com:fredrik-jansson-se/fluent-bit-input-rust.git
cd fluent-bit-input-rust
make all run
...
   Compiling in-example v0.1.0 (/tmp/fluent-bit-input-rust)
    Finished release [optimized] target(s) in 9.25s
cc -s -m64 -O2 -shared -o flb-in_example.so out/in_example_plugin.o target/release/libin_example.a
fluent-bit/build/bin/fluent-bit -vv -e ./flb-in_example.so -i example -o stdout
Fluent Bit v1.8.11
* Copyright (C) 2019-2021 The Fluent Bit Authors
* Copyright (C) 2015-2018 Treasure Data
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2022/01/15 14:33:00] [ info] [engine] started (pid=2355956)
[2022/01/15 14:33:00] [ info] [storage] version=1.1.5, initializing...
[2022/01/15 14:33:00] [ info] [storage] in-memory
[2022/01/15 14:33:00] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
[2022/01/15 14:33:00] [ info] [cmetrics] version=0.2.2
[2022/01/15 14:33:00] [ info] [sp] stream processor started
[0] example.0: [1642253589.973281329, {"collect-calls"=>1}]
[0] example.0: [1642253599.973229065, {"collect-calls"=>2}]
```

And there you see the "logs" generated by our example plugin

## Makefile Deep Dive
The interesting part of the Makefile is 
```make
RSOURCES = $(wildcard src/*rs)
SOURCES = $(wildcard csrc/*c)
OBJECTS = $(patsubst csrc/%,out/%,$(SOURCES:.c=.o))
TARGET = flb-in_example.so

out/%.o: csrc/%.c
        @mkdir -p $(dir $@)
        $(CC) $(CFLAGS) $(INCLUDES) -fpic -c $< -o $@

$(TARGET): $(FLUENT_BIT) $(OBJECTS) $(RSOURCES)
        cargo build --release
        $(CC) -s $(CFLAGS) $(LDFLAGS) -o $@ $(OBJECTS) target/release/libin_example.a
```

Note that the C code is compiled with `-fpic` for going into a shared library. Also note that the object file is directly linked with the Rust static library, this is to make sure the `in_example_plugin` is a public symbol that Fluent Bit can load (`dlsym`).
