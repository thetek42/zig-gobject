# zig-gobject

Early work-in-progress bindings for GObject-based libraries (such as GTK)
generated using GObject introspection data.

To generate all available bindings using the files under `lib/gir-files`, run
`zig build codegen`. This will generate bindings to the `bindings` directory,
which can be used as a dependency (using the Zig package manager) in other
projects.

To generate bindings only for a subset of available libraries, run the following
command, substituting `Gtk-4.0` with one or more libraries (with files residing
under `lib/gir-files`). Bindings for dependencies will be generated
automatically, so it is not necessary to specify GLib, GObject, etc. in this
example.

```sh
zig build run -- --gir-dir lib/gir-files --extras-dir extras --output-dir src/gir-out Gtk-4.0
```

## Usage

The ideal way of using this library would be through the Zig package manager.
Unfortunately, the package manager has some limitations which make this
impossible right now (https://github.com/ziglang/zig/issues/14719). There is
still vestigial code present in the generated `build.zig` for the bindings from
a time when certain PRs were used to provide this functionality (on my own fork
of Zig); it is intended for this to be adapted to whatever solution is
eventually adopted in the package manager for this issue.

For now, the best way to use the bindings is to add them as a submodule (using
the `bindings` branch of this repository) and use the `addBindingModule`
function exposed by `build.zig`:

```zig
// exe is the compilation step for your applicaton
exe.addModule("gtk", zig_gobject.addBindingModule(b, exe, "gtk-4.0"));
```

There are examples of this pattern in the `examples` and `test` subprojects.

## Examples

There are several examples in the `examples` directory, which is itself a
runnable project (depending on the `bindings` directory as a dependency). To
ensure the bindings are generated and run the example project launcher, run
`zig build run-example`.

## Binding philosophy

zig-gobject is fundamentally a low-level binding with some higher-level
additions ("extras") added in. The first layer (the low-level part) is a direct
translation of information available in GIR (GObject introspection repository)
files, represented and organized in a way that makes sense in Zig while
maintaining transparency of the underlying symbols. Basically, this first layer
can be thought of as an enhanced `zig translate-c`, using GObject introspection
for additional expressiveness (such as pointer nullability).

For example, consider `gtk.Application.new` from the generated GTK 4 bindings,
which is translated as follows:

```zig
extern fn gtk_application_new(p_application_id: ?[*:0]const u8, p_flags: gio.ApplicationFlags) *_Self;
pub const new = gtk_application_new;
```

The `new` function here is merely an alias for the underlying
`gtk_application_new` function, which, thanks to the rich information provided
by GObject introspection, has a more useful signature than what
`zig translate-c` could have provided using the C headers, which would look like
the following:

```zig
extern fn gtk_application_new(application_id: [*c]const u8, flags: GApplicationFlags) [*c]GtkApplication;
```

The key difference here is the use of richer Zig pointer types rather than C
pointers, which offers greater safety. Another very useful translation
enrichment enabled by GObject introspection is in the `gio.ApplicationFlags`
type referenced here, which is translated as follows by zig-gobject:

```zig
/// Flags used to define the behaviour of a #GApplication.
pub const ApplicationFlags = packed struct(c_uint) {
    is_service: bool = false,
    is_launcher: bool = false,
    handles_open: bool = false,
    handles_command_line: bool = false,
    send_environment: bool = false,
    non_unique: bool = false,
    can_override_app_id: bool = false,
    allow_replacement: bool = false,
    replace: bool = false,
    // Padding fields omitted

    const _Self = @This();
    const flags_none = @bitCast(_Self, @as(c_uint, 0));
    const default_flags = @bitCast(_Self, @as(c_uint, 0));
    const is_service = @bitCast(_Self, @as(c_uint, 1));
    const is_launcher = @bitCast(_Self, @as(c_uint, 2));
    const handles_open = @bitCast(_Self, @as(c_uint, 4));
    const handles_command_line = @bitCast(_Self, @as(c_uint, 8));
    const send_environment = @bitCast(_Self, @as(c_uint, 16));
    const non_unique = @bitCast(_Self, @as(c_uint, 32));
    const can_override_app_id = @bitCast(_Self, @as(c_uint, 64));
    const allow_replacement = @bitCast(_Self, @as(c_uint, 128));
    const replace = @bitCast(_Self, @as(c_uint, 256));

    pub const Own = struct{
        extern fn g_application_flags_get_type() usize;
        pub const getType = g_application_flags_get_type;
    };

    pub const Extras = if (@hasDecl(extras, "ApplicationFlags")) extras.ApplicationFlags else struct {};

    pub usingnamespace Own;
    pub usingnamespace Extras;
};
```

The use of `packed struct` here is a more idiomatic pattern in Zig for flags,
allowing us to write code such as this:

```zig
const app = gtk.Application.new("com.example.Example", .{ .handles_open = true });
```

The translation offered by `zig translate-c`, on the other hand, is much more
basic:

```zig
pub const GApplicationFlags = c_uint;
```

This first layer also includes some core-level helpers such as `connect*`
methods for type-safe signal connection.

The second layer (the higher-level part) is the "extras", which can be seen in
`ApplicationFlags` above:

```zig
pub const Extras = if (@hasDecl(extras, "ApplicationFlags")) extras.ApplicationFlags else struct {};

pub usingnamespace Extras;
```

The `extras` referenced here is just `@import("gio-2.0.extras.zig")` (or
`struct {}` if there was no `gio-2.0.extras.zig` provided during translation).
Any `ApplicationFlags` defined in the extras file is mixed in to the generated
`ApplicationFlags` type. In other words, the extras provide additional helpers
on top of what is provided by the underlying libraries, but they do not change
anything about the core bindings.

Occasionally, this approach to binding creation results in conflicting
identifiers, such as `gtk_window_set_focus` and `gtk_root_set_focus` in GTK 4:
since `Gtk.Window` implements `Gtk.Root`, zig-gobject includes all the methods
of `gtk.Root` on `gtk.Window` as a convenience. This means that `setFocus` can
no longer be used directly, because it is duplicated across two different
`usingnamespace`s. To offer a way out of this, zig-gobject groups bindings into
various namespaces before mixing them into the container type, meaning that
either of these variants can be used to disambiguate:

```zig
// Doesn't work due to conflicting bindings:
// window.setFocus(widget);
gtk.Window.OwnMethods(gtk.Window).setFocus(window, widget);
gtk.Root.OwnMethods(gtk.Window).setFocus(window, widget);
```

This is certainly more tedious than if `setFocus` just worked directly, but it
ensures none of the functionality of the underlying libraries is lost in
translation.
