
# **GSoC 2024 Final Report: Improving Clang-Repl with Out-Of-Process Execution**

### **Introduction**

Hello! I'm Sahil Patidar, and this summer I had the exciting opportunity to
participate in Google Summer of Code (GSoC) 2024. My project revolved around
enhancing Clang-Repl by introducing Out-Of-Process Execution.

Mentors: Vassil Vassilev and Matheus Izvekov

### **Project Overview**

**Clang** is a popular compiler front-end in the **LLVM** project, capable of handling languages like C++. One of its cool tools, **Clang-Repl**, is an interactive C++ interpreter using **Just-In-Time (JIT)** compilation. It lets you write, compile, and run C++ code interactively, making it super handy for learning, quick prototyping, and debugging.

However, Clang-Repl did have a few drawbacks:

1. **High Resource Usage**: Running both Clang-Repl and JIT in the same process used up a lot of system resources.
2. **Instability**: If the user's code crashed, the entire Clang-Repl session would shut down, leading to interruptions.

### **Out-Of-Process Execution**

The solution to these challenges was **Out-Of-Process Execution**—executing user code
in a separate process. This offers two major advantages:

- **Efficient Resource Management**: By isolating code execution, Clang-Repl reduces
  its system resource footprint, crucial for devices with limited memory or processing
  power.
- **Enhanced Stability**: Crashes in user code no longer affect the main Clang-Repl
  session, improving reliability and user experience.

This improvement makes Clang-Repl more reliable, especially on low-power or embedded
systems, and suitable for broader use cases.

---

## **Summary of accomplished tasks**

### **Note: Some of the PRs are still under review, but they are expected to be merged soon.**

#### **Add Out-Of-Process Execution Support for Clang-Repl**  
**PR**: [#110418](https://github.com/llvm/llvm-project/pull/110418)

To implement out-of-process execution in Clang-Repl, I utilized **ORC JIT’s remote execution capabilities** with the `llvm-jitlink-executor`. This enhancement introduced the following key features:

- **New Command Flags**:
  - `--oop-executor`: Initiates the JIT executor in a separate process.
  - `--oop-executor-connect`: Establishes a connection between Clang-Repl and the external process for out-of-process execution.

These newly added flags allow Clang-Repl to leverage the `llvm-jitlink-executor` for executing code in an isolated environment, improving execution separation and flexibility.

---

### **Key ORC JIT Enhancements**

To support Clang-Repl’s out-of-process execution, I contributed several improvements to **ORC JIT**:

#### **Incremental Initializer Execution for Mach-O and ELF**  
[#97441](https://github.com/llvm/llvm-project/pull/97441), [#110406](https://github.com/llvm/llvm-project/pull/110406)

The `dlupdate` function was introduced in the ORC runtime to support incremental execution of new initializers within the REPL environment. Unlike the traditional `dlopen` function, which handles tasks such as code mapping, library reference counting, and initializer execution, `dlupdate` is specifically focused on running only the new initializers. This targeted approach significantly improves efficiency, particularly in interactive environments like `clang-repl`, by reducing unnecessary operations and speeding up the update process.

#### **Push-Request Model for ELF Initializers**  
**PR**: [#102846](https://github.com/llvm/llvm-project/pull/102846)

A push-request model was introduced to manage ELF initializers in the runtime state for each `JITDylib`, similar to the handling of initializers in Mach-O and COFF. Previously, ELF required a fresh request for initializers each time `dlopen` was invoked, but lacked the ability to register, deregister, or retain these initializers. This led to issues when re-running `dlopen`, as initializers would be erased after `rt_getInitializers` was invoked, making subsequent executions impossible.

To address these issues, the following functions were introduced:

- **`__orc_rt_elfnix_register_init_sections`**: This function registers ELF initializers for the `JITDylib`.
- **`__orc_rt_elfnix_register_jitdylib`**: This function registers the `JITDylib` with the ELF runtime state.

With this push-request model, initializers for each `JITDylib` state can now be better tracked and managed. By leveraging Mach-O’s `RecordSectionsTracker`, we ensure that only newly registered initializers are run, significantly improving efficiency and reliability when working with ELF targets in `clang-repl`.

This update plays a key role in enabling out-of-process execution in `clang-repl` on ELF platforms, providing a more effective way to handle incremental execution.

---

### **Additional Improvements**

#### **Issue encountered [[ORC] Fix block dependence calculation in ObjectLinkingLayer.](https://github.com/llvm/llvm-project/commit/896dd322afcc1cf5dc4fa7375dedd55b59001eb4) Resolved By Lang Hames.**

#### **Auto-loading Dynamic Libraries in ORC JIT**  
**PR**: [#109913](https://github.com/llvm/llvm-project/pull/109913) (On-going)

An auto-loading dynamic library feature was added to ORC JIT to enhance the speed of symbol resolution for both loaded and unloaded libraries. A key improvement introduced in this update is the global bloom filter, which helps reduce unnecessary symbol searches by skipping symbols that are unlikely to be present, thereby improving overall performance. 

The updated system operates as follows: when the JIT attempts to resolve a symbol, it first checks the currently loaded libraries. If the symbol is found, its address is retrieved using `dlsym` and stored in the results. If the symbol is not found in the loaded libraries, the search automatically expands to include unloaded libraries.

As the symbol tables of these libraries are scanned, they are added to the global bloom filter. If the search through all possible auto-linkable libraries still does not yield the symbol, the bloom filter result is returned. Additionally, any symbols flagged by the bloom filter as "may-contain" but that do not actually resolve are added to an exclusion set, preventing redundant resolution attempts in the future.

This feature streamlines the symbol resolution process, reducing search overhead and improving performance in the ORC JIT system.


#### **Refactor `dlupdate`**  
**PR**: [#110491](https://github.com/llvm/llvm-project/pull/110491)

This update simplifies the `dlupdate` function by removing the `mode` argument, streamlining the function's interface. The change enhances the clarity and usability of `dlupdate` by reducing unnecessary parameters, improving the overall maintainability of the code.


---

### **Benchmarks: In-Process vs Out-of-Process Execution**

- [Prime Finder](https://gist.github.com/SahilPatidar/4870bf9968b1b0cb3dabcff7281e6135)
- [Fibonacci Sequence](https://gist.github.com/SahilPatidar/2191963e59feb7dfa1314509340f95a1)
- [Matrix Multiplication](https://gist.github.com/SahilPatidar/1df9e219d0f8348bd126f1e01658b3fa)
- [Sorting Algorithms](https://gist.github.com/SahilPatidar/c814634b2f863fc167b8d16b573f88ec)

---

### **Result**

With the implemented changes, `clang-repl` now supports out-of-process execution. This functionality can be invoked using the following command:

```bash
clang-repl --oop-executor=path/to/llvm-jitlink-executor --orc-runtime=path/to/liborc_rt.a
```

This allows for isolated and efficient code execution via the `llvm-jitlink-executor`.


### **Future Work**

- **Crash Recovery and Session Continuation :** 
Investigate and develop ways to enhance crash recovery so that if something goes wrong, the session can seamlessly resume without losing progress. This involves exploring options for an automatic process to restart the executor in the event of a crash.

- **Finalize Auto Library Loading in ORC JIT :**
Wrap up the feature that automatically loads libraries in ORC JIT. This will streamline symbol resolution for both loaded and unloaded dynamic libraries by ensuring that any required dylibs containing symbol definitions are loaded as needed.

### **Conclusion**

Thanks to this project, **Clang-Repl** now supports **out-of-process execution** for
both `ELF` and `Mach-O`, vastly improving its resource efficiency and stability. These
changes make Clang-Repl more robust, especially on low-resource devices.

Looking ahead, I plan to focus on automating library loading and further enhancing
ORC-JIT to optimize Clang-Repl's out-of-process execution.

Thank you for being a part of my **GSoC 2024** journey!

### **Acknowledgements**

I would like to extend my deepest gratitude to Google Summer of Code (GSoC)
for the incredible opportunity to work on this project. A special thanks to
my mentors, Vassil Vassilev and Matheus Izvekov, for their invaluable guidance
and support throughout this journey. I am also immensely grateful to Lang Hames
for their expert insights on ORC-JIT enhancements for `clang-repl`. This experience
has been a pivotal moment in my development, and I am excited to continue
contributing to the open-source community.

---

### **Related Links**

- [LLVM Repository](https://github.com/llvm/llvm-project)
- [Project Description](https://discourse.llvm.org/t/clang-out-of-process-execution-for-clang-repl/68225)
- [My GitHub Profile](https://github.com/SahilPatidar)

---
