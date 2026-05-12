# Dart Live

The Dart VM, compiled to WebAssembly, running in your browser, with stateful
hot reload on the web.

**Live:** https://modulovalue.github.io/dart-live/

## What this is

A single-page app, no server. The browser runs:

| component | wasm size | role |
|---|---:|---|
| `dart_il.wasm` | 9.7 MB | Dart VM (emcc, ARM simulator) |
| `dart_cfe.wasm` | 2.4 MB | Common front-end (dart2wasm), Dart to kernel |
| `dart_analyzer.wasm` | 2.5 MB | Dart analyzer (dart2wasm) |
| `vm_platform.dill` | 8.3 MB | Platform kernel (dart:core, dart:async, ...) |
| `dart_sdk.sum` | 3.2 MB | Analyzer SDK summary |

Total: about 26 MB uncompressed, 7.6 MB gzipped.

## Pipeline

1. You type Dart in Monaco.
2. Browser JS calls `dartCompile(source, vm_platform.dill)`, and the CFE
   wasm produces kernel bytes.
3. JS hands those bytes to `dart_il_run`, which has the VM wasm execute
   them through the ARM simulator.
4. `print` calls in Dart go through an `EM_ASM` bridge back to JS, which
   updates the DOM.
5. **Run** orphans the prior isolate and creates a fresh one, so state
   resets.
6. **Hot Reload** calls `IsolateGroup::ReloadKernel` in place. Top-level
   fields keep their values, but the library code is swapped to the new
   kernel.
7. **Invoke** (the inline CodeLens above every zero-arg top-level function)
   calls that function on the live isolate via `Dart_Invoke`, bypassing
   entry-point pragmas via `--no-verify-entry-points`.
8. `Future.delayed` actually waits wall-clock time. `_embedderSleep`
   forwards to `emscripten_sleep` via Asyncify, so the JS event loop
   stays responsive.
9. The analyzer runs in parallel against `dart_sdk.sum` and feeds Monaco
   via `setModelMarkers`.

## Expert mode

The toggle in the toolbar reveals two extra CodeLens buttons above every
top-level function:

- `IL`, a text dump of the unoptimized flow graph from `Dart_CompileAll`.
- `IL Graph`, the same flow graph rendered with `@viz-js/viz`, one
  Graphviz `digraph` per compiler phase, with phase tabs across the top.

## Samples

Four starred samples exercise the headline feature (stateful hot reload
plus invoke): iterative pi via the Leibniz series, Conway's Game of Life,
bouncing 2D particles, and a counter. Hot Reload preserves the running
state. Change the step batch, the gravity, the neighborhood rule, then
press Hot Reload and keep going from where you were.

## License

This repository contains a built artifact bundle. Build inputs and patches
live in a separate working tree (Dart SDK plus an emcc-built embedder).
