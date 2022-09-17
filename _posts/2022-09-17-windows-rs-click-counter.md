## windows-rs mouse click counter


In this chapter I'll explain how to use the `windows-rs` which is the Win32 API crate for Rust to write a click counter, with help of the Windows Virtual-Key Codes, Hooks and then compile it from WSL2 to a Windows executable using Cross-compilation.

---

### First Steps

Architecture:
- Rust 1.62.1
- windows-rs 0.39.0
- once_cell 1.13.1
- WSL2 with Ubuntu 20.04
- Target OS Windows 10 21H2

#### Add Toolchain (msvc) to Rust

```console
rustup target add x86_64-pc-windows-msvc
```

Once it's installed all the compiler needs to Cross-compile to Windows, Proceed to install the `Microsoft Windows SDKs` `xwin`, this utily packages the `Microsoft CRT Standard Library` that will be possible to the Cross-compilation.

```console
cargo install xwin
```

Proceed to accept the Microsoft License, this will download all the required files from Microsoft servers, and select an output folder.

```console
xwin --accept-license 1 splat --output /xwin
```

#### Add Linker to Cargo (msvc)

Go to `.cargo/.cargo.toml` (or create that file if it doesn't exists) and add those lines (install `lld` linker before).

```toml
[target.x86_64-pc-windows-msvc]
linker = "lld"
rustflags = [
  "-Lnative=/xwin/crt/lib/x86_64",
  "-Lnative=/xwin/sdk/lib/um/x86_64",
  "-Lnative=/xwin/sdk/lib/ucrt/x86_64"
]
```

### The code

First of all, create an empty cargo binary project, then add the dependencies and windows dependencies, this project use three `windows-rs` imports, `"Win32_Foundation"`, `"Win32_System_LibraryLoader"`, and `"Win32_UI_WindowsAndMessaging"`,

```toml
[dependencies]
once_cell = "1.13.1"

[dependencies.windows]
version = "0.39.0"
features = [
    "Win32_Foundation",
    "Win32_System_LibraryLoader",
    "Win32_UI_WindowsAndMessaging",
]
```

Also, import once_cell `Lazy` struct to have a global static value, `Arc` to have thread-safe reference-counting pointers, `AtomicU64` to have an integer type which can be safely shared between threads, `io` module to print to the standard output, and finally, the `thread` module to spawn a new OS thread.

```rust
use windows::{
  core::*,
  Win32::Foundation::*,
  Win32::System::LibraryLoader::GetModuleHandleA,
  Win32::UI::WindowsAndMessaging::*,
};
use once_cell::sync::Lazy;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::io::{self, Write};
use std::thread;
```

Initiante a new const that will hold a Lazy Atomically Reference Counted with an `Unsigned 64-bits Atomic Integer` starting at 0.

```rust
static COUNT: Lazy<Arc<AtomicU64>> = Lazy::new(|| Arc::new(AtomicU64::new(0)));
```

Inside a main function get an instance of `GetModuleHandleA`, It will only be necessary to use the console, therefore, skip the process of `CreateWindowExA` and instead of that set a Windows Hook that will receive a `WH_MOUSE_LL`, a None so far, later it will be changed to a `Callback`, `HINSTANCE` and an Unsigned 32-bits Interger with the value of 0, then dispatch a Message. If the hook fails, `UnhookWindowsHookEx` will unmont the current Hook.

If there are any struggle with this, read both the [windows-rs](https://github.com/microsoft/windows-rs) documentation and [Windows Api](https://docs.microsoft.com/en-us/windows/win32/apiindex/windows-api-list) documentation.

```rust
fn main() -> Result<()> {
  unsafe {
    let instance = GetModuleHandleA(None)?;
    debug_assert!(instance.0 != 0);

    let k_hook = SetWindowsHookExA(
      WH_MOUSE_LL,
      None,
      HINSTANCE::default(),
      0,
    );

    let mut message = MSG::default();
    
    while GetMessageA(&mut message, HWND::default(), 0, 0).into() {
      DispatchMessageA(&message);
    }

    if k_hook.is_err() {
      UnhookWindowsHookEx(k_hook.unwrap());
    }

    Ok(())
  }
}
```

Compile to make sure everything is working fine.

![cmd wsl2](https://i.imgur.com/psQBefi.jpg)

The last step is create and external funcion, why an external? because `SetWindowsHookExA` receives an external `__stdcall`, then, to solve this problem, according to [The Rust Reference](https://doc.rust-lang.org/reference/items/external-blocks.html#abi) this will call a `__stdcall` linking to the `Windows API` itself.

Proceed to create an extern fn (the name is irrelevant) `m_callback`, which will work every time a `WH_MOUSE_LL` event occurs, so it will call the callback funcion every time.

This callback function will receive three params:
- `ncode`: A code the hook procedure uses to determine how to process the message. If nCode is less than zero, the hook procedure must pass the message to the CallNextHookEx function without further processing and should return the value returned by CallNextHookEx. This parameter can only be a HC_ACTION with the value of 0.
- `wparam`: The identifier of the mouse message. This parameter can be one of the following messages: WM_LBUTTONDOWN, WM_LBUTTONUP, WM_MOUSEMOVE, WM_MOUSEWHEEL, WM_MOUSEHWHEEL, WM_RBUTTONDOWN, or WM_RBUTTONUP. This hook only will need `WM_LBUTTONUP`.
- `lparam`: A pointer that ontains information about a low-level mouse input event.

```rust
unsafe extern "system" fn m_callback(
  ncode: i32,
  wparam: WPARAM,
  lparam: LPARAM,
) -> LRESULT {
  if wparam.0 as u32 == WM_LBUTTONUP && ncode as u32 == HC_ACTION {
    let pma_coordinate = *(lparam.0 as *const u16);
    dbg!(pma_coordinate);

    let builder = thread::Builder::new()
      .name("Click counter".into());

    let val = Arc::clone(&COUNT);

    let handle = builder.spawn(move || {
      val.fetch_add(1, Ordering::SeqCst);

      let text = format!("Click number: {:?} \n", val);
      let stdout = io::stdout();
      let mut handle = stdout.lock();

      handle.write_all(text.as_bytes()).unwrap();
  
      let _ = io::stdout().flush();
      assert_eq!(thread::current().name(), Some("Click counter"));
    });

    handle.unwrap().join().unwrap();
  }
  CallNextHookEx(HHOOK::default(), ncode, wparam, lparam)
}
```

The first verification is check if the identifier of mouse message is equal to `WM_LBUTTONUP`, as the ncode is equal to a `HC_ACTION`.

Cast `lparam.0` to a raw pointer to get the x- and y-coordinates of the cursor, in [per-monitor-aware](https://learn.microsoft.com/en-us/windows/win32/api/shellscalingapi/ne-shellscalingapi-process_dpi_awareness) screen coordinates. So debug it.

```rust
let pma_coordinate = *(lparam.0 as *const u16);
dbg!(pma_coordinate);
```

Create a new thread builder that will increase the `Unsigned 64-bits Atomic Integer` with the help of an `Arc pointer clone`, inside this spawned thread it will be increased an `Atomically Reference Counted` using `fetch_add` method, increasing in `1` every time when a click event occurs and passing an `Atomic Memory Orderings` that sequentially will incresease the `Unsigned 64-bits Atomic Integer` and returning the last value. Then using the io modulue, locking it to avoid interleave between data and flush the current buffer to the stdout output. Finishing joining that spawned thread to the main thead and finally the counter will increase.

```rust
let builder = thread::Builder::new()
  .name("Click counter".into());

let val = Arc::clone(&COUNT);

let handle = builder.spawn(move || {
  val.fetch_add(1, Ordering::SeqCst);

  let text = format!("Click number: {:?} \n", val);
  let stdout = io::stdout();
  let mut handle = stdout.lock();

  handle.write_all(text.as_bytes()).unwrap();
  
  let _ = io::stdout().flush();
  assert_eq!(thread::current().name(), Some("Click counter"));
});

handle.unwrap().join().unwrap();
```

Change the `SetWindowsHookExA` in the main function with a None to use this new unsafe extern function callback.

```rust
let k_hook = SetWindowsHookExA(
  WH_MOUSE_LL,
  Some(m_callback),
  HINSTANCE::default(),
  0,
);
```

To end, return a `CallNextHookEx` to passes the information inside the hook to the next hook procedure in the current hook chain.

```rust
CallNextHookEx(HHOOK::default(), ncode, wparam, lparam)
```

After being compiled to `x86_64-pc-windows-msvc`, the click counter will work as expected.

```console
cargo build --release --target=x86_64-pc-windows-msvc
```

![end](https://i.imgur.com/7MZOLzk.gif)

### Epilogue

`windows-rs` provides a good Win32API abstraction for handling hooks. Another good approach would be to use a GUI to print the number of clicks instead of using the console.
