<p align="center">
    <a target="_blank" href="https://docs.rs/unbug">
        <img src="https://raw.githubusercontent.com/greymattergames/unbug/main/assets/unbug.svg" width="200" alt="Unbug logo"/>
    </a>
</p>
<h1 align="center">Unbug</h1>

A crate to programmatically invoke debugging breakpoints with helping macros.

These macros are designed to help developers catch errors during debugging sessions that would otherwise be a panic (which may not be desirable in certain contexts) or simply a log message (which may go unnoticed).

This crate's internals are disabled by default, there are shims provided so breakpoints will not be compiled outside of a debugging context. This means that the macros in this crate can be used freely throughout your code without having to conditionally compile them out yourself.

> ### NOTICE
>
> Stable Rust is only supported on x86, x86_64, and ARM64
>
> You must use the `enable` feature of this crate (deactivated by default) to activate the breakpoints. This crate cannot detect the presence of a debugger.

Error messages are logged when used in conjuction with [Tracing](https://github.com/tokio-rs/tracing)

## Examples

# [![VSCode debugging example](https://raw.githubusercontent.com/greymattergames/unbug/master/assets/debug.png)](https://github.com/greymattergames/unbug/blob/master/examples/basic/src/main.rs)

```rust
// trigger the debugger
unbug::breakpoint!();

// Use the tracing_subscriber crate to enable log messages
tracing_subscriber::fmt::init();

for i in 0..5 {
    // ensure! will only trigger the debugger once
    // when the expression argument is false
    unbug::ensure!(false);
    unbug::ensure!(false, "Ensure can take an optional log message");
    unbug::ensure!(false, "{}", i);

    // ensure_always! will trigger the debugger every time
    // when the expression argument is false
    unbug::ensure_always!(i % 2 == 0);

    // fail! pauses and logs an error message
    // will also only trigger once
    unbug::fail!("fail! will continue to log in non-debug builds");

    if i < 3 {
        // fail! and fail_always! can be formatted just like error!
        // from the Tracing crate
        unbug::fail!("{}", i);
    }

    let Some(_out_var) = some_option else {
        unbug::fail_always!("fail_always! will trigger every time");
    };
}

```

## Usage

Prepare your environment for debugging Rust.
> If you are using VSCode you will need the [Rust Analyzer](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer) and [Code LLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)  (Linux/Mac) or the [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) (Windows) extensions. [See Microsoft's Documentation on Rust Debugging in VSCode](https://code.visualstudio.com/docs/languages/rust#_debugging).

__1.__ Create a debug feature in your project that will only be active in the context of a debugger, i.e. not enabled by default.

`Cargo.toml`:
```toml
[features]
default = []
my_debug_feature = [
    "unbug/enable"
]
```

__2.__ Pass your feature flag to cargo during your debug build.

Sample VSCode `.vscode/launch.json` with LLDB (Linux/Mac):
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "LLDB Debug",
            "cargo": {
                "args": [
                    "build",
                    "--bin=my_project",
                    "--package=my_project",
                    "--features=my_debug_feature"
                ],
                "filter": {
                    "name": "my_project",
                    "kind": "bin"
                }
            },
            "args": [],
            "cwd": "${workspaceFolder}",
            "env": {
                "CARGO_MANIFEST_DIR": "${workspaceFolder}"
            }
        }
    ]
}
```

Sample VSCode `.vscode/launch.json` with msvc (Windows):
```json
{
    "version": "0.2.0",
    "configurations": [
		{
            "name": "Windows debug",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceRoot}/target/debug/unbug_basic_example.exe",
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "preLaunchTask": "win_build_debug"
        }
    ]
}
```

and complimentary `.vscode/tasks.json`
```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cargo",
			"command": "build",
			"args": [
				"--bin=my_project",
				"--package=my_project",
				"--features=my_debug_feature"
			],
			"problemMatcher": [
				"$rustc"
			],
			"group": {
				"kind": "build",
				"isDefault": true
			},
			"label": "win_build_debug"
		}
	]
}
```

__3.__ Select the debug launch configuration for your platform and start debugging

In VSCode, open the "Run and Debug" panel from the left sidebar.

launch configurations can be now found in the dropdown menu next to the green "Start Debugging" button.

When debugging is active, controls for resume execution, step-over, and step-out are at the top of the window under the search field.


#### If you are not using x86, x86_64, or ARM64:
Including, but not limited to WASM, RISCV, PowerPC, and ARM32

Nightly Rust and the experimental [`core_intrinsics`](https://doc.rust-lang.org/core/intrinsics/fn.breakpoint.html) feature need to be used.

To Enable Nightly Rust:

You can set a workspace toolchain override by adding a `rust-toolchain.toml` file at the root of your project with the following contents:
```toml
[toolchain]
channel = "nightly"
```

OR you can set cargo to default to nightly globally:
```bash
rustup install nightly
rustup default nightly
```

enable the core_intrinsics feature in the root of your crate (`src/main.rs` or `src/lib.rs`):

`src/main.rs`:
```rust
#![cfg_attr(
    // this configuration will conditionally activate core_intrinsics
    // only when in a dev build and your debug feature is active
    all(
        debug_assertions,
        feature = "my_debug_feature",
    ),
    feature(core_intrinsics),
    // Optionally allow internal_features to suppress the warning
    allow(internal_features),
)]
```


Additonally, debugging may not land on the macro statements themselves. This can have the consequence that the debgger may pause on an internal module. To avoid this, `return` or `continue` immediately following a macro invocation. Alternatively, use your debugger's "step-out" feature until you reenter the scope of your code.

## License

Unbug is free and open source. All code in this repository is dual-licensed under either:

- MIT License ([LICENSE-MIT](/LICENSE-MIT) or <http://opensource.org/licenses/MIT>)
- Apache License, Version 2.0 ([LICENSE-APACHE](/LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)

at your option.
