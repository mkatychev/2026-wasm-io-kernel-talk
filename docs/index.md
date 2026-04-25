Wasm in the Kernel
====

<!-- 20 mins slides, 5 mins demo, 5 mins Q&A -->

Evolving Threat Detection at Runtime


<!--
welcome to our presentations "Evolving Threat Detection at Runtime"
I am Mikhail and this is my co-presenter Salman
We are going to talk about how we at Mimic use WebAssembly
to dynamically update behavior in the windows kernel
-->


---

Mimic Image

<!-- Transition: -->
<!-- let me tell you a little about Mimic -->
---



<!-- ~ 2 minutes -->
<!-- Mikhail -->

## Mimic

<!-- we are a member of the BytecodeAlliance that specializes in... -->
- System level threat detection and mitigation
<!-- the technology we are developing allows us to rapidly react to ongoing attacks from adversaries -->
- Reconfigure and redeploy during ongoing attacks
- Modifying system in production is risky
<!-- - Modifying it often is even riskier -->

<!-- Transition: -->
<!-- We use Wasm to mitigate some of this risk through sandboxing -->

---

<!-- ~ 2 minutes -->
<!-- Mikhail -->

## WebAssembly at Mimic

<!-- Our product monitors system events from the core of the operating system,
the kernel, to the cloud to detect threats. -->
- Process system events from kernel to cloud
<!-- and we use Wasm for safe customization and dynamic updates -->
- Wasm-centric product
  <!-- Wasm allows us to customize the product to meet the individual needs of our customer's environment -->
  - Environment customization
  <!-- We use Wasm to rapidly configure and update our product -->
  - Dynamic updates

<!-- we do this by leveraging the Component Model at mimic -->
- Component Model
<!-- we built a modular ecosystem using the Component Model for our product -->
  - Help build and maintain an ecosystem
  <!-- Virtualization with the Componen Model allows us to reuse code across platforms -->
  - Virtualization for cross-platform development

<!-- Transition: -->
<!-- One of the places we deploy this kind of dynamic logic is in the operating system's Kernel -->

---

<!-- ~ 1 minute -->
<!-- Salman -->

## Kernel Programming

- ... is hard!
  - Low-level system API
  - Asynchronous environment
  - Minor bugs cause full system crashes
<!-- eBPF allows safe customization and dynamic updates in the Linux kernel -->
- eBPF (extended Berkeley Packet Filter) for Linux kernel
  - Change kernel's behavior without restart
  - Restricted logic for safety
<!-- But in Windows... -->
- eBPF for Windows is under development
   microsoft/ebpf-for-windows

<!-- Transition: -->
<!-- So... -->

---
<!-- 2 minutes -->
<!-- Salman -->

## Wasm in Windows Kernel

<!-- our own safe space inside a hostile kernel environment -->
- Update code in kernel without restart
- Sandboxing for safety guarantees
  <!-- restriction v.s. sandboxing -->
- Policy-driven execution
  - Resource limiting
  - Fine-grained access control
  <!-- we have full control on a running instance of Wasm -->

<!-- Transition: -->
<!-- Kernel programming is notoriously hard -->
<!-- getting Wasm working in the kernel wasn't easy -->

---

<!-- ~ 1 minute -->
<!-- Mikhail -->

## Challenges

<!-- first, we don't have access to all the tools we get in the standard library -->
* No standard library (`#[no_std]`)
<!-- second ... you can't run random wasn binaries in the Kernel
<!-- only code that has been reviewed and signed by microsoft can run with standard security policies -->
<!-- hypervisor kernel mode integrity windows -->
* No executable memory due to kernel security policies
<!-- and ... that means we also can't to JIT compilation -->
* No Just in Time (JIT) compilation

<!-- Transition: -->
<!-- thus, we decided to interpret Wasm -->

---

<!-- 2 minutes -->
<!-- Mikhail -->

## Interpreting Wasm
<!-- constrained environment -->
<!-- we wanted Wasm interpreters that supported no_std environment -->
* TinyWasm, Wasmi, and Wasmtime with Pulley
  <!-- Pulley, by the way is an optimized bytecode format and interpreter for Wasmtime -->

<!-- these were the high level results of OUR micro-benchmarking, Wasmi was the fastest one -->

* Wasmi is ~10x slower than Wasmtime with JIT
<!-- Interpreting was an magnitude slower for OUR micro-benchmarks than JIT -->
* Wasmi is ~20-50% faster than Wasmtime with Pulley
<!-- interpreted Wasmi is faster -->

<!-- Transition: -->
<!-- BUT we still chose Wasmtime+Pulley instead of Wasmi -->

---

<!-- Salman -->

<!-- ~ 1 minute -->
## Component Model

- Secure and modular ecosystem
  <!-- capability-based Compatibility -->
  <!-- technology independence is important for building ecosystems -->
- Compatibility with user-space ecosystem
  <!-- compatibility enables workload migration -->
- Wasmtime was the only viable option
<!-- Component Model is difficult to adopt -->
<!-- we wish there were more implementations that handled the Component Model -->

<!-- Transition: -->
<!-- So how do we make wasmtime run in the kernel... -->

---

<!-- 2 minutes -->
<!-- Salman -->

<!-- Technical Details -->

<!-- ?? We should expand the following sections. -->

<!-- We embed wasmtime as a minifilter in the kernel -->

## Kernel Embedding

- Run as a minifilter driver
  <!-- TODO should be a keynote and have a simpler bullet point -->
  <!-- - minifilters provide a way for developers to monitor and modify file system operations -->
  <!--   without needing to interact directly with lower-level file system drivers -->
   - Intercept and process I/O events
   - High-level API

- Host functions interact with kernel (no WASI)
  <!-- - we are not supporting Wasi but we are implementing the host functions in the kernel -->
  <!-- - the host functions interact with the kernel and need to be fully tested -->
  <!-- this "unsafe" portion of our code -->

<!-- we compile modules in user space -->
- Compile to Pulley in user space
  <!-- what we are deploying is not standard wasm, it's optimized pulley bytecode -->

---

<!-- 2 minutes -->
<!-- Salman -->

## Host implementation

<!-- When you are in no_std you have to implement the C API  -->
- Enforce `no_std` in Wasmtime
  *  bytecodealliance/wasmtime#11152
<!-- Wasmtime was still pulling in the standard library with no_std -->
<!-- talk about "respect `std` PR" #11152 -->
<!-- shoutout to Alex C for the quick turnaround -->

- IRQL (Interrupt Request Level) aware error handling
<!-- IRQLs limits what you can and cannot do based on your level -->
<!-- performing wrong operation at wrong IRQL causes BSoD -->
- Implement host primitives through C API
  * Memory mapping via heap allocation
  * Structured Exception Handling instead of panic unwinding
  * Custom TLS (Thread Local Storage )


---

<!-- Transition: -->
<!-- we wanted our hosts to run multiple Wasm components at once -->

<!-- 2 minutes -->
<!-- Mikhail -->

## Multithreading

- Run concurrent Wasm components
<!-- when we tried spawning multiple worker threads, wasmtime panicking on contention -->
<!-- Wasmtime needed the std to provide synchronization primitives to avoid contention -->
* Wasmtime `no_std` assumed single threaded host
  * Panic on contention
<!-- we modified Wasmtime to allow the host to provide synchronization primitives through C API -->
* Host provided synchronization primitives
  *  bytecodealliance/wasmtime#11836

<!-- Transition: -->
<!-- We also had to implement our own thread-local storage (TLS) -->

---

<!-- 2 minutes -->
<!-- Mikhail -->

<!-- In computer programming, thread-local storage (TLS) is a lock-free memory management method -->
<!-- that gives each thread its own copy of a global variable.
<!-- When the thread exits, its copied variables are destroyed -->
## Thread-Local Storage

* The standard library usually implements TLS for you
<!-- Solution -->
* Custom TLS implementation
  <!-- we use stack pinning to ensure that allocated memory does not shift when thread scopes are passed around -->
  <!-- and that their IDs stay relevant even after leaving the current scope -->
  * Stack pinned thread scopes and IDs
<!-- we use a registry pattern to link threads to their parents and the resources they own  -->
* Thread-scope registry pattern
  * Per-thread stacks of values
<!-- ...now we can can run multiple components at once -->

<!-- Transition: -->
<!-- What does this get us -->

---

<!-- 1 minutes -->
<!-- Salman -->

## Wrapping up

<!-- we allowed WebAssembly to handle dynamic updates in the kernel without restart -->

- Safe kernel development with Wasm
  <!-- This allows us and our customers to safely write logic for the kernel -->
  <!-- it meets our business use cases -->
- Dynamic sandboxing vs. static verification
  <!-- logic restriction v.s. sandboxing -->
- Be aware of performance cost
  <!-- Interpreting Wasm comes at a cost -->
  <!-- Wasm solution should not be used on the hot path -->


<!-- Transition: -->
<!-- We will now show you a demo to see our solution in practice... -->

---
