# WebAssembly (WASM) Development Guide

## Tech Stack

- **Languages:** Rust, C/C++, AssemblyScript
- **Compilation:** Emscripten (C/C++), wasm-pack (Rust), asc (AssemblyScript)
- **Runtime:** Browser (JavaScript), Node.js, WASI (server-side)
- **Build Tools:** wasm-pack, wasm-bindgen, Emscripten
- **Optimization:** wasm-opt, Binaryen

---

## Why WebAssembly?

### Use Cases

- **Performance-critical code:** Image/video processing, physics simulations, cryptography
- **Existing C/C++ libraries:** Port to web (e.g., SQLite, OpenCV)
- **Game engines:** Unity, Unreal Engine
- **Complex calculations:** Scientific computing, data analysis
- **Cross-platform:** Write once, run in browser, Node.js, or standalone

### NOT Ideal For

- Simple DOM manipulation (use JavaScript)
- Network I/O heavy tasks (WASM <-> JS boundary overhead)
- Small, infrequent calculations (startup cost not worth it)

---

## Rust + WebAssembly

### Setup

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Add wasm target
rustup target add wasm32-unknown-unknown

# Install wasm-pack
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
```

### Project Structure

```
my-wasm-project/
├── Cargo.toml
├── src/
│   └── lib.rs
├── www/           # Web frontend
│   ├── index.html
│   ├── index.js
│   └── package.json
└── pkg/           # Generated WASM output
```

### Basic Example

```rust
// Cargo.toml
[package]
name = "my-wasm-project"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"

// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// Performance-critical function
#[wasm_bindgen]
pub fn process_image(data: &[u8]) -> Vec<u8> {
    data.iter()
        .map(|&pixel| pixel.saturating_add(10))
        .collect()
}
```

### Build & Use

```bash
# Build for web
wasm-pack build --target web

# Build for Node.js
wasm-pack build --target nodejs

# Build for bundler (Webpack/Vite)
wasm-pack build --target bundler
```

```javascript
// index.js (ES modules)
import init, { add, greet } from "./pkg/my_wasm_project.js";

async function run() {
  await init(); // Initialize WASM module

  console.log(add(5, 7)); // 12
  console.log(greet("WebAssembly")); // "Hello, WebAssembly!"
}

run();
```

---

## JavaScript Integration

### Passing Complex Data

```rust
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct User {
    pub name: String,
    pub age: u32,
}

#[wasm_bindgen]
pub fn process_user(user_json: &str) -> Result<String, JsValue> {
    let user: User = serde_json::from_str(user_json)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;

    // Process user
    let result = User {
        name: user.name.to_uppercase(),
        age: user.age + 1,
    };

    serde_json::to_string(&result)
        .map_err(|e| JsValue::from_str(&e.to_string()))
}
```

```javascript
const userJson = JSON.stringify({ name: "Alice", age: 30 });
const resultJson = process_user(userJson);
const result = JSON.parse(resultJson);
console.log(result); // { name: "ALICE", age: 31 }
```

### Accessing Web APIs

```rust
use wasm_bindgen::prelude::*;
use web_sys::console;

#[wasm_bindgen]
pub fn log_message(message: &str) {
    console::log_1(&JsValue::from_str(message));
}

// Fetch API
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_data(url: String) -> Result<String, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");

    let request = Request::new_with_str_and_init(&url, &opts)?;

    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;

    let text = JsFuture::from(resp.text()?).await?;
    Ok(text.as_string().unwrap())
}
```

---

## C/C++ + Emscripten

### Setup

```bash
# Install Emscripten SDK
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk
./emsdk install latest
./emsdk activate latest
source ./emsdk_env.sh
```

### Basic Example

```c
// hello.c
#include <emscripten.h>
#include <stdio.h>

EMSCRIPTEN_KEEPALIVE
int add(int a, int b) {
    return a + b;
}

EMSCRIPTEN_KEEPALIVE
void greet(const char* name) {
    printf("Hello, %s!\n", name);
}
```

### Compile

```bash
# Basic compilation
emcc hello.c -o hello.js \
    -s EXPORTED_FUNCTIONS='["_add", "_greet"]' \
    -s EXPORTED_RUNTIME_METHODS='["ccall", "cwrap"]'

# With optimizations
emcc hello.c -o hello.js \
    -s EXPORTED_FUNCTIONS='["_add"]' \
    -s EXPORTED_RUNTIME_METHODS='["ccall"]' \
    -O3 \
    -s WASM=1 \
    -s MODULARIZE=1 \
    -s EXPORT_ES6=1
```

### Use in JavaScript

```javascript
import createModule from "./hello.js";

const Module = await createModule();

// Call C functions
const result = Module.ccall(
  "add", // function name
  "number", // return type
  ["number", "number"], // argument types
  [5, 7], // arguments
);

console.log(result); // 12

// Or use cwrap for repeated calls
const add = Module.cwrap("add", "number", ["number", "number"]);
console.log(add(10, 20)); // 30
```

---

## Performance Optimization

### Memory Management

```rust
// Avoid frequent allocations
#[wasm_bindgen]
pub struct ImageProcessor {
    buffer: Vec<u8>,
}

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(size: usize) -> ImageProcessor {
        ImageProcessor {
            buffer: vec![0; size], // Allocate once
        }
    }

    pub fn process(&mut self, data: &[u8]) -> Vec<u8> {
        // Reuse buffer instead of allocating
        self.buffer[..data.len()].copy_from_slice(data);

        for pixel in &mut self.buffer[..data.len()] {
            *pixel = pixel.saturating_add(10);
        }

        self.buffer[..data.len()].to_vec()
    }
}
```

### SIMD (Single Instruction, Multiple Data)

```rust
// Enable SIMD (experimental)
// Compile with: RUSTFLAGS='-C target-feature=+simd128' wasm-pack build

#[cfg(target_arch = "wasm32")]
use std::arch::wasm32::*;

#[wasm_bindgen]
pub fn add_vectors_simd(a: &[f32], b: &[f32]) -> Vec<f32> {
    let mut result = Vec::with_capacity(a.len());

    unsafe {
        for i in (0..a.len()).step_by(4) {
            let va = v128_load(a.as_ptr().add(i) as *const v128);
            let vb = v128_load(b.as_ptr().add(i) as *const v128);
            let vr = f32x4_add(va, vb);

            let mut tmp = [0f32; 4];
            v128_store(tmp.as_mut_ptr() as *mut v128, vr);
            result.extend_from_slice(&tmp);
        }
    }

    result
}
```

### Optimize Build

```bash
# Rust optimizations
wasm-pack build --release -- --features simd

# Post-process with wasm-opt (from Binaryen)
wasm-opt -O3 -o optimized.wasm pkg/my_wasm_project_bg.wasm

# Size reduction
wasm-pack build --release -- -Z build-std=std,panic_abort
wasm-strip pkg/my_wasm_project_bg.wasm
```

### Benchmark

```rust
// benches/benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use my_wasm_project::process_image;

fn benchmark_process_image(c: &mut Criterion) {
    let data = vec![0u8; 1920 * 1080 * 4]; // 1080p RGBA

    c.bench_function("process_image", |b| {
        b.iter(|| process_image(black_box(&data)))
    });
}

criterion_group!(benches, benchmark_process_image);
criterion_main!(benches);
```

---

## Threading (Web Workers)

### Rust with rayon

```rust
// Cargo.toml
[dependencies]
wasm-bindgen = "0.2"
rayon = "1.8"
wasm-bindgen-rayon = "1.0"

// src/lib.rs
use wasm_bindgen::prelude::*;
use rayon::prelude::*;

#[wasm_bindgen]
pub fn parallel_sum(data: &[i32]) -> i32 {
    data.par_iter().sum()
}
```

```javascript
// main.js
import init, { initThreadPool, parallel_sum } from "./pkg";

async function run() {
  await init();
  await initThreadPool(navigator.hardwareConcurrency);

  const data = new Int32Array(1000000);
  data.fill(1);

  const result = parallel_sum(data);
  console.log(result); // 1000000
}
```

---

## Debugging

### Source Maps

```bash
# Enable debug info
wasm-pack build --dev

# Or with Emscripten
emcc -g4 --source-map-base http://localhost:8080/ hello.c -o hello.js
```

### Console Logging

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);
}

macro_rules! console_log {
    ($($t:tt)*) => (log(&format_args!($($t)*).to_string()))
}

#[wasm_bindgen]
pub fn debug_function() {
    console_log!("Debug: value = {}", 42);
}
```

---

## WASI (WebAssembly System Interface)

### Standalone WASM

```rust
// src/main.rs (not lib.rs)
use std::io::{self, Write};

fn main() {
    println!("Hello from WASI!");

    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();

    writeln!(io::stdout(), "You entered: {}", input.trim()).unwrap();
}
```

```bash
# Build for WASI
cargo build --target wasm32-wasi --release

# Run with wasmtime
wasmtime target/wasm32-wasi/release/my-wasm-project.wasm

# Or with wasmer
wasmer target/wasm32-wasi/release/my-wasm-project.wasm
```

---

## Testing

### Rust Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}

// Run tests
// cargo test
```

### Browser Tests (wasm-pack)

```bash
# Install headless browser
wasm-pack test --headless --chrome
wasm-pack test --headless --firefox
```

---

## Common Patterns

### Loading Indicator

```javascript
import init from "./pkg";

document.getElementById("status").textContent = "Loading WASM...";

init().then(() => {
  document.getElementById("status").textContent = "Ready!";
  // Use WASM functions
});
```

### Progressive Enhancement

```javascript
let wasmSupported = false;

if (typeof WebAssembly === "object") {
  import("./pkg").then((wasm) => {
    wasmSupported = true;
    // Use WASM version
  });
} else {
  // Fallback to JavaScript implementation
}
```

---

## External Resources

- **Rust WASM Book:** https://rustwasm.github.io/docs/book/
- **wasm-bindgen:** https://rustwasm.github.io/wasm-bindgen/
- **Emscripten:** https://emscripten.org/docs/
- **WebAssembly.org:** https://webassembly.org/
- **WASI:** https://wasi.dev/

---

## Common Gotchas

1. **Startup Cost:** WASM initialization has overhead (~50-100ms). Use for long-running tasks.
2. **String Passing:** Expensive due to UTF-8 ↔ UTF-16 conversion. Pass binary data when possible.
3. **GC Types:** No direct DOM access from WASM. Use wasm-bindgen or web-sys.
4. **Memory Growth:** Linear memory can grow but never shrink. Plan capacity carefully.
5. **Debugging:** Source maps required for readable stack traces.
