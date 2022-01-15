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
