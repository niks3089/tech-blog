---
title: "Unikernels and Solana: A Hypothetical Exploration"
date: 2024-08-18T00:00:00+08:00
description: "A what-if about using Unikernel as Solana VM"
tags: ["solana"]
type: post
weight: 25
showTableOfContents: true
---

# Unikernels and Solana: A Hypothetical Exploration

## Introduction

I’ve always enjoyed pondering the "what-ifs" of technology and one area that has fascinated me for years is the concept of unikernels. Back in 2017-2018, I spent a lot of my free time researching [unikernels](http://unikernel.org/). The idea of tailoring an application with only the necessary components of the kernel to create a lightweight, highly optimized library operating system (libOS) struck me as incredibly innovative.

For those of you who might not be familiar with unikernels, let me explain.

## What Are Unikernels?

Unikernels are specialized, single-address-space machine images constructed using library operating systems. Essentially, they bundle your application with just the minimal set of operating system functions required to run it, resulting in a lightweight and highly optimized kernel. This kernel can be deployed directly on hypervisors or in cloud environments, offering significant performance and security benefits due to the reduced overhead.

During my research, I had the opportunity to work on a couple of unikernel projects:

- **Solo5:** A [unikernel](https://github.com/Solo5/solo5) in C that we built to run as a user-space process or be containerized and deployed in cloud environments.
- **IncludeOS:** A C++ [unikernel](https://github.com/includeos/IncludeOS) where you could literally `#include <OS>` into your application, allowing it to run as bare metal or on top of another unikernel like Solo5.

In fact, we even published a paper on this topic, which you can find [here](https://dl.acm.org/doi/10.1145/3267809.3267845). One particularly interesting use case for unikernels is in Serverless, where we could drastically reduce cold start times by loading in just a few milliseconds.

## Connecting Unikernels with Solana

You might be wondering, what does any of this have to do with Solana? Well, let’s dive into that.

### Solana’s BPF Model

Solana uses a virtual machine and execution environment called BPF (Berkeley Packet Filter), specifically adapted for running smart contracts on Solana. This environment is designed for:

- **High Performance:** Optimized for fast execution on Solana's validator nodes.
- **Security:** Sandbox isolation ensures that programs can't harm the network, even if they’re buggy or malicious.
- **Flexibility:** Supports multiple languages via a WebAssembly-like interface, although Rust is most commonly used.

### How eBPF Works as a Virtual Machine

eBPF (extended Berkeley Packet Filter) started as a technology for filtering network packets in the Linux kernel, but it has evolved into a tool for running sandboxed programs in various contexts, including network filtering, performance monitoring and security enforcement.

#### Bytecode Execution
- eBPF programs are compiled into bytecode, a set of low-level instructions similar to assembly language.
- This bytecode is loaded into the kernel via system calls and executed by an eBPF interpreter within the kernel.

#### JIT Compilation
- To boost performance, the eBPF VM can Just-In-Time (JIT) compile bytecode into native machine code, allowing for faster execution.

#### Sandboxing
- eBPF programs run in a sandboxed environment, isolated from the kernel and user space, ensuring safety and correctness.

Here's an example to disassemble and print eBPF instructions

```rust
// Import the necessary crates
extern crate elf;
extern crate solana_rbpf;
use std::path::PathBuf;
use solana_rbpf::{
    elf::Executable,
    program::{BuiltinProgram, FunctionRegistry, SBPFVersion},
    static_analysis::Analysis,
    vm::TestContextObject,
};
use std::sync::Arc;

// Disassemble and print the eBPF program instructions
fn disassemble_program(program: &[u8]) {
    let executable = Executable::<TestContextObject>::from_text_bytes(
        program,
        Arc::new(BuiltinProgram::new_mock()),
        SBPFVersion::V2,
        FunctionRegistry::default(),
    )
    .unwrap();
    let analysis = Analysis::from_executable(&executable).unwrap();

    println!("Disassembled eBPF Program:");
    for (index, insn) in analysis.instructions.iter().enumerate() {
        let desc = analysis.disassemble_instruction(insn);
        println!("{:04}: {}", index, desc);
    }
}

// Load a program from an object file and disassemble it
fn main() {
    // Path to the ELF file containing the eBPF program
    let filename = "examples/load_elf__block_a_port.o";

    let path = PathBuf::from(filename);
    let file = match elf::File::open_path(path) {
        Ok(f) => f,
        Err(e) => panic!("Error: {:?}", e),
    };

    // Extract the .classifier section, which contains the eBPF bytecode
    let text_scn = match file.get_section(".classifier") {
        Some(s) => s,
        None => panic!("Failed to look up .classifier section"),
    };

    let prog = &text_scn.data;

    // Disassemble and print the program
    disassemble_program(prog);
}
``` 

### Some Challenges with JIT Compilation

Just-In-Time (JIT) compilation is a powerful technique used to optimize code at runtime, but it comes with several challenges

#### 1. Startup Performance Impact
- **Compilation Overhead**: JIT compilers compile code at runtime, which can cause delays during the initial execution of a program, leading to slower startup times compared to ahead-of-time (AOT) compiled code.
- **Warm-up Time**: The code may run slower initially until the JIT compiler has had time to optimize it. During this "warm-up" period, performance can be less predictable.

#### 2. Memory Usage
- **Increased Memory Footprint**: JIT compilation requires memory for storing both the JIT compiler itself and the generated machine code. This can lead to higher memory consumption compared to AOT compilation, where the machine code is generated once and reused.
- **Garbage Collection Interactions**: JIT compilers often run in environments with garbage collection, which can add additional pressure on memory management systems, potentially leading to performance bottlenecks.

#### 3. Unpredictable Performance
- **Unpredictable Latency**: Because JIT compilation happens at runtime, the performance can be unpredictable. For instance, a critical section of code might be interrupted by a JIT compilation process, causing a sudden increase in execution time.
- **Variable Optimization Levels**: The level of optimization applied by a JIT compiler can vary depending on runtime conditions, leading to inconsistent performance across different runs or environments.

#### 4. Code Size Inflation
- **Code Bloat**: JIT compilers might generate more machine code than necessary due to conservative optimization strategies or the need to handle various edge cases. This can lead to code bloat and increased cache pressure.


With above limitation in mind, lets dive in.

### A Hypothetical Solana Unikernel

Is there a Rust based Unikernel? Yes, there's [hermit](https://github.com/hermit-os/hermit-rs). Now, imagine a scenario where Solana programs are not just compiled as eBPF bytecode to be executed in the Sealevel runtime, but instead are packaged as unikernels. How would this look?

#### Conceptual Overview

In a Solana + Unikernel setup:

- **Compilation:** Each Solana program would be compiled along with a unikernel like Hermit, creating a single binary.
- **Execution:** The execution of Solana smart contracts would occur within this unikernel and as a userspace process, which provides the necessary runtime environment, replacing the need for eBPF. We will use [uhyve](https://github.com/hermit-os/hermit-playground?tab=readme-ov-file#uhyve---a-lightweight-hypervisor), A hypervisior to start our unikernels as userspace process within a KVM-accelerated virtual machine
- **Customization:** Hermit would need to be customized to support Solana-specific features like account management, transaction validation and state persistence.

### Next Steps: End-to-End Flow

In the envisioned Solana + Unikernel setup, where each Solana smart contract is compiled with as a Unikernel, the process extends beyond merely creating and executing these contracts. The real challenge lies in managing the entire lifecycle of these unikernel-based smart contracts, ensuring their seamless execution, state management, and orchestration across the network. The end-to-end workflow begins with the compilation of a Solana smart contract into a binary along with the Hermit unikernel, producing a standalone unikernel binary that encapsulates the smart contract and the minimal runtime needed to execute it. This binary is then deployed to a Solana node or a cluster of nodes, where it runs as a userspace process within the unikernel. During execution, the unikernel interacts with Solana's state, handling tasks like account management, transaction validation, and state persistence. To effectively manage these processes, a robust orchestration layer is essential. The orchestrator is responsible for coordinating the deployment of unikernel binaries across the network, managing the lifecycle of these unikernels by starting, stopping, and scaling them as needed, and ensuring they have the necessary resources to operate efficiently. Additionally, the orchestrator plays a critical role in monitoring the health and performance of each unikernel, collecting real-time status information, and synchronizing state changes across the network to maintain consistency and integrity in the blockchain.

#### Compilation

When a developer submits a Solana smart contract, it is first compiled into a binary along with the Hermit unikernel. This compilation produces a standalone unikernel binary that encapsulates the smart contract and the minimal runtime needed to execute it.
Deployment: The compiled binary is then deployed to a Solana node or a cluster of nodes. Each node is capable of executing the unikernel binary in a controlled environment, such as a virtual machine (VM) or a container, ensuring isolation and security.
Execution:

#### Runtime Execution 
The deployed unikernel runs as a userspace process on the node. The Hermit unikernel provides the necessary runtime environment, executing the smart contract logic while handling low-level operations like memory management, input/output, and networking.

#### State Management
During execution, the smart contract interacts with Solana's state, including account balances, transaction validation, and other blockchain-specific operations. The unikernel must be equipped to handle these operations efficiently, ensuring that the state is correctly updated and persisted.
Orchestration:

#### Orchestrator
Given that each smart contract is now a self-contained unikernel binary, there needs to be a robust orchestration layer to manage these unikernels. The orchestrator will be responsible for:
- Deployment Coordination: Deploying and distributing unikernel binaries across the Solana network.
- Lifecycle Management: Starting, stopping, and scaling unikernel instances as needed based on the network's demands.
- Resource Allocation: Ensuring that each unikernel has the necessary resources (CPU, memory, storage) to execute efficiently.
- Health Monitoring: Continuously monitoring the health and performance of each unikernel instance, ensuring that they are functioning correctly
- Status Collection: The orchestrator will collect real-time status information from each running unikernel. This includes metrics like CPU usage, memory consumption, execution time, and any errors or exceptions that occur during execution.
- State Synchronization: Since each unikernel may modify the blockchain state, it’s crucial to synchronize this state across the network. The orchestrator will ensure that state changes are propagated and validated across all relevant nodes, maintaining consistency and integrity in the blockchain
- Transaction Finalization: Once a transaction is processed by a unikernel, the result is sent back to the Solana network for finalization. The orchestrator ensures that the results are consistent and that consensus is reached among nodes.
- State Persistence: After finalization, the updated state is persisted on the blockchain. The orchestrator helps coordinate this process, ensuring that all nodes in the network have a consistent view of the blockchain state.

### Advantages and Challenges

#### Advantages

- **Performance:** Unikernels are highly optimized, with minimal overhead, potentially leading to faster execution of Solana programs.
- **Security:** With no full OS, the attack surface is significantly reduced, enhancing security.
- **Isolation:** Each Solana program could be compiled into its own unikernel, providing strong isolation from other programs.

#### Challenges

- **Integration Complexity:** Solana’s current model relies on a multi-program execution environment. Moving to unikernels would require significant changes to how programs interact with the blockchain and each other.
- **State Management:** Handling persistent state across multiple unikernels would be complex, especially in a distributed ledger context like Solana.
- **Tooling:** The Solana developer ecosystem, including deployment, debugging and monitoring tools, would need significant modifications to support unikernel-based execution.

### Managing State in a Unikernel-Based Solana

Let’s explore how state persistence could be managed if each Solana program were running in its own unikernel.

#### Distributed State Management

- **Local State:** Each unikernel would manage its own local state, including account data and other necessary program states. This state would need to be persisted within the unikernel and synchronized with other unikernels.
- **Global State Synchronization:** To maintain consistency across the network, a mechanism would be needed to synchronize state changes between unikernels. This could involve a distributed state management system that coordinates state updates across unikernels running on different nodes.

#### State Persistence Mechanism

- **Persistent Storage Integration:** Each unikernel could be integrated with a persistent storage backend, such as a distributed key-value store or a blockchain-based storage layer, allowing it to persist its local state across reboots or crashes.
- **State Replication:** To ensure fault tolerance, state changes would need to be replicated across nodes, potentially using distributed consensus protocols like Raft or Paxos.
- **State Checkpointing:** Unikernels could periodically checkpoint their state to persistent storage, providing recovery points in case of failure. These checkpoints would also need to be synchronized across the network.

#### State Coordination Layer

- **Coordination Service:** A dedicated service could manage the interaction between unikernels, handling transaction ordering, state synchronization and conflict resolution.
- **Event-Driven State Updates:** Unikernels could be designed to respond to state change events triggered by the coordination service, ensuring that state updates are applied consistently across the network.

### Conclusion

In this hypothetical scenario, each Solana program would run within its own unikernel, offering potential performance and security benefits. However, the complexity of managing state persistence, synchronization and consistency across multiple unikernels would pose significant challenges, particularly in a distributed ledger environment like Solana.

While the idea is intriguing and offers some unique advantages, it would require substantial architectural changes and innovation to address the challenges of state management and integration with Solana’s existing infrastructure. Whether or not this approach could be practical or superior to the current model is a fascinating question that would need further exploration and experimentation.

