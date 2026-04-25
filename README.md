<!--
theme: gaia
class:
 - invert
headingDivider: 2
paginate: true
-->

<!--
_class:
 - lead
 - invert
-->

# WASM in the Kernel: evolving threat detection at runtime


### [Mikhail Katychev - Mimic](https://2026.wasm.io/speakers/mikhail-katychev/) / [Salman Saghafi - Mimic](https://2026.wasm.io/speakers/salman-saghafi/)

No eBPF on Windows? We at Mimic run WASM in the kernel instead! See how we deploy live security patches, extend OS telemetry capabilities, and inject custom logic, all in relative safety and without a single reboot.

When evaluating how to safely extend and observe potential threats to the kernel in the absence of a mature eBPF implementation for Windows, our team at Mimic decided to experiment using the WebAssembly Component Model for running threat detection logic with promising results.

This talk will dissect the major benefits and challenges of distributing, testing, and running threat detection logic as WASM components in Windows kernel. We will discuss novel applications and current pain points in using the Component Model to bridge std (standard library) and no\_std contexts and share the patterns and workarounds we use to coordinate WASM module deployment in the name of cybersecurity.

Some of the practical challenges and accomplishments below will be covered in our talk:

-   interpreting embedded Wasm in the uncharted territory of the OS kernel
-   kernel specific challenges: no-std, panic handling, TLS (Thread Local Storage), and synchronization
-   designing a host environment to safely interact with the kernel using the Wasm Interface Type (WIT)
-   results of our experimentation with various runtimes/interpreters and how we arrived at Wasmtime/Pulley

- https://github.com/mkatychev
- https://github.com/salmans
