# Fluent Bit Input Plugin in Rust

## Prerequisites
You need to install:
 1. gcc/clang
 2. cmake
 3. flex
 4. bison
 5. rust

## Code

Fluent Bit loads input plugins from a shared library using a convention. Assume the input plugin is named `example`, fluent bit assumes the shared library to be named `flb-in_example.so`. From that library it will load a struct describing the plugin called `in_example_plugin`.

This struct is defined in `csrc/in_example_plugin.c` and looks like
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

The `cb_init`, `cb_collect` and `cb_exit` are callback functions called by Fluent Bit. We will actually implement those in Rust. Hence, they are just declared in the C code:
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
