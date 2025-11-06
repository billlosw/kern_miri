# (Kern)Miri

For more detail, please visit: https://hackmd.io/bLRtlCH0T9udi9OLGrrXuQ#KernMiri-An-Overview

## How it works

`cargo miri xxx` does not directly generate binary machine code, as is typical. Instead, it uses a small interpreter to execute the **MIR** (**M**id-level **I**ntermediate **R**epresentation). After `cargo` uses `rustc` to generate the MIR from the Rust source code, **Miri** uses the `create_ecx` function (found in `src/eval.rs:eval_entry`) to create an `InterpCx<'tcx, MiriMachine<'tcx>>`.

  * The `InterpCx` is a component from `rustc` used for MIR execution.
  * The **`MiriMachine`** struct defines Miri's specific capabilities for interpreting the MIR.

An example of the execution process:

1.  The **`InterpCx`** reads a MIR instruction, for example, a write to memory: `*ptr = value;`.
2.  The generic `InterpCx` knows it needs to perform a write. Before proceeding, it calls a hook on its `Machine`. In this case, it calls `MiriMachine::before_memory_write`.
3.  The implementation of `before_memory_write` inside `MiriMachine` then performs all of Miri's specific checks:
      * **Borrow Checking**: It consults the `borrow_tracker` to see if the pointer `ptr` has a valid tag and permission to write. If not, it reports an Undefined Behavior (UB) error.
      * **Data Race Detection**: It consults the `data_race` state to check if this write conflicts with reads/writes from other threads.
      * **Alignment Check**: It checks if `ptr` is properly aligned for the type being written.
4.  If all checks pass, `before_memory_write` returns `Ok`, and the `InterpCx` proceeds to perform the actual write in its memory model.

## Diff to KernMiri

**KernMiri** extends Miri by adding a component responsible for **simulating physical memory space**. This includes the use of `PageState` to track the status of a memory page (`Unused`, `Untyped`, or `(TypedKind, usize)`) to further ensure **security and correctness**.

The structure for this simulation is defined in the `PhysicalMemory` struct:

```rust
pub struct PhysicalMemory {
    pub mem: *mut u8,
    pub page_states: Vec<PageState>,
    pub init_masks: BTreeMap<usize, Allocation<Provenance, (), MiriAllocBytes>>,
    pub page_table: Option<PageTable>,
}
```

Standard Miri operates with an abstract graph of memory allocations. Its `AllocId`s are simply numbers from a large, arbitrary address space and have no relation to hardware, physical RAM, or virtual memory layouts.

**KernMiri** extends this with a **hardware simulation layer**, which allows it to interpret a full OS kernel that directly manages **physical memory** and **virtual memory mappings**.

## Shims

Miri can only interpret Rust MIR. When your Rust code calls a function from a C library (such as `malloc` or `exit` from `libc`), Miri cannot execute the original C code. Instead, it intercepts the call and executes its own **Rust implementation** of that function—the **shim**. The main dispatcher for this process is located in `foreign_items.rs`.

The interpreted program **does not have direct access** to the host machine's operating system. Miri uses **Shims** to simulate OS-level functionality via the `RustABI`.

In the context of **KernMiri**, Miri is made to act like **bare-metal hardware** instead of a traditional OS. Shims define the interface between the interpreted kernel and the simulated hardware environment provided by Miri. The `kern_miri_*` functions found in `foreign_items.rs` are custom shims that enable the kernel to perform **privileged operations** like allocating physical memory pages or initializing other CPU cores.

## Provenance

Provenance is defined by the following enum:

```rust
pub enum Provenance {
    Concrete {
        alloc_id: AllocId,
        tag: BorTag, // Borrow Tracker tag.
    },
    Wildcard,
}
```

  * **`Concrete`**: Represents a pointer directly tied to a specific memory allocation and with a known borrowing status. Every time a reference (`&` or `&mut`) is created, Miri assigns it `Concrete` provenance.
      * **`alloc_id`**: Used by Miri to check if the allocation is **alive** (preventing use-after-free errors).
      * **`tag`**: Used for **borrow checking** and tracking the pointer's permissions, ensuring adherence to Rust's aliasing rules.
      
  * **`Wildcard`**: Designed to handle pointers created from integers (e.g., `usize as *const T`). This is necessary because casting a pointer to an integer, manipulating the integer, and casting it back to a pointer (a **pointer-integer roundtrip**) loses the original direct link to the allocation.

      * **Exposure**: When a pointer with `Concrete` provenance is "exposed" (cast to a `usize`), Miri adds its provenance to a set of "exposed" provenances.
      * **Recreation**: When an integer is cast back to a pointer, the new pointer receives `Provenance::Wildcard`.
      * **Access Check**: When a `Wildcard` pointer is used for memory access, Miri does not know which allocation it *should* belong to. Instead, it checks if the access would be valid for **any** of the previously exposed provenances. This serves as an over-approximation of the "angelic choice" semantics, where the operation is considered valid if there is *some* legal way for the access to succeed.

## Miri Machines

The Miri Machine consists of numerous fields that define the interpreter's state and behavior. **KernMiri** extends the base Miri Machine with the following members to simulate a **multi-core kernel environment**:

  * **`cpu_local_alloc_set`**: Tracks allocations that are **per-CPU local**. This ensures that each simulated CPU core gets its own private copy (analogous to a `current_task` pointer) at a distinct memory address.
  * **`thread_map`**: Establishes a connection between the kernel's concept of a "task" and Miri's internal concept of a "thread."
  * **`pt_checker`**: A field used to signal that a Page Table Entry has been modified, enabling related checks, though it may not be fully utilized in all simulation phases.
  * **`record`**: Used to record performance measurements from within the simulation, which is necessary for benchmarking and analysis.

The initialization process for KernMiri differs from standard Miri, specifically in setting up the simulated CPU environment:

```rust
let alloc_addresses = RefCell::new(alloc_addresses::GlobalStateInner::new(config));
let mut threads = ThreadManager::default();
// Sets the base virtual address for the CPU-local data of the first (index 0) CPU.
threads.cpu_local_base[0] = mirch::kernel_code_base_vaddr() + mirch::cpu_local_start_addr();
```

### Implementing Machine Hooks

The **MiriMachine** implements several hooks (methods) that are called by the generic `InterpCx` during execution. KernMiri customizes these hooks for its kernel simulation.

#### Memory Management & Undefined Behavior (UB) Checking

  * **`init_alloc_extra`**: Called when any new memory allocation (`AllocId`) is created (for heap, statics, or stack variables). This hook initializes the allocation's metadata, including the necessary structures for the **`borrow_tracker`** and **`data_race_detactor`**.
  * **`before_memory_deallocation`**: Called before memory is freed. It validates the deallocation request and performs necessary cleanup using the **`borrow_tracker`** and **`data_race_detactor`** to check for UB.
  * **`adjust_global_allocation`**: Called upon the first access to a global static variable. This allows the machine to transform the compiler-provided static data into its specialized format.
      * KernMiri uses this to map the static variable into the **simulated physical memory**: it checks if the address is within the kernel's static data region. If so, it creates a new Miri allocation directly backed by the pseudo-physical memory at that address and copies the contents from the original compiler-provided allocation.
  * **`before_memory_read`/`write`**: Called immediately before a read or write operation to check its legality. These hooks call the **`borrow_tracker`** and **`data_race_detactor`** to check for standard UB violations.
      * KernMiri adds a critical check within `memory_write`: it verifies if the memory being written to is part of a **page table**. If it is, the `pt_checker` field is set to notify the machine that a Page Table Entry (PTE) has been modified, which may trigger subsequent TLB invalidations or related hardware-level checks.

```rust
// KernMiri's page table check added to memory_write hook (simplified logic shown):
// ...
let global_state = machine.alloc_addresses.borrow();
let address = *global_state.base_addr.get(&alloc_id).unwrap() as usize;

// Checks the PageState in the physical memory structure
if let PageState::Typed {page_type, type_size: _} = mirch::physical_mem()
                                                     .page_states[address / mirch::page_size()]
{
    if page_type == TypedKind::PageTable {
        // Notifies the machine of the Page Table modification
        machine.pt_checker = Some(address - address % mirch::PTE_SIZE);
    }
}
```

#### Function Call & Control Flow

  * **`find_mir_or_eval_fn`**: The main function call dispatcher. It determines whether a call targets normal Rust MIR code or an external function that must be handled by Miri's **foreign item shims** (simulation).
  * **`after_stack_push`/`pop`, `init_frame`**: These hooks allow the machine to attach specific data to each stack frame and perform actions when frames are created or destroyed.
      * KernMiri modifies these hooks to save and restore the virtual stack pointer address during function calls.

```rust
// KernMiri's stack management hooks:
fn after_stack_push(...) {
    // ... Pushes the stack pointer.
    let thread = ecx.machine.threads.active_thread_mut();
    let next_stack_addr = *thread.next_stack_addr.borrow();
    thread.stack_addr_records.push(next_stack_addr);
}
fn after_stack_pop(...) {
    // ... Resumes the stack pointer.
    let thread = ecx.machine.threads.active_thread_mut();
    if let Some(next_stack_addr) = thread.stack_addr_records.pop() {
        *thread.next_stack_addr.borrow_mut() = next_stack_addr;
    }
}
```

* **`before_terminator`**: Called before executing any MIR terminator (e.g., `Goto`, `Call`, `Return`). It can trigger Garbage Collection.
  * KernMiri uses this hook to simulate scheduling by adding an `ecx.maybe_switch_cpu` call, which simulates the current thread potentially switching to run on a different virtual CPU core.
* **`before_access_global`, `adjust_alloc_root_pointer`**: Called when a global static is first accessed or a pointer to one is created. These hooks are used to identify special kinds of global statics.
    * Crucially, KernMiri uses these to identify **per-CPU statics** *before* they are accessed. By inspecting the compiler attributes (`link_section`), it identifies statics defined in the `.cpu_local` section and adds their `alloc_id` to the `cpu_local_alloc_set`. This is essential for the memory system to later resolve accesses to the correct per-CPU memory address.

## Thread/Manager

The `Thread` struct in KernMiri is extended with several fields to enable the simulation of hardware-level stack management and context switching:

  * **`stack_addr_records`**: A collection used to mark the stack frames, analogous to the **base pointer (`ebp`)** in x86 architecture. Its primary use is saving and restoring the `next_stack_addr` across function calls.
  * **`next_stack_addr`**: Simulates the **stack pointer (`sp`)** of the current thread, pointing to the next available stack location.
  * **`stack_bottom`**: Stores the lowest valid memory address for the thread's stack. This is used to check for **stack overflow**.

`ThreadManager` is added some new member to support multi-core situation, `active_cpu`, `next_cpu`, `cpu_to_threads`, `next_thread`.

  * **`active_cpu`**: The ID of the currently executing simulated CPU core.
  * **`next_cpu`**: The ID of the next CPU core to be activated (used during scheduling).
  * **`cpu_to_threads`**: A map that connects a CPU ID to the ID of the thread currently running on it.
  * **`next_thread`**: The ID of the next thread selected to run on the `next_cpu`.
  * **`schedule`**: KernMiri significantly modifies this function to implement switching logic for **both the CPU core and the thread**, enabling multi-core simulation.

Yet, in the function `src/concurrency/thread.rs:current_cpu_local_base`, it seems there is a bug:

```rust
pub fn current_cpu_local_base(&self) -> usize {
    self.cpu_local_base[1]
}
```

Not only is this function **hardcoded** like this, but it seems KernMiri is not yet complete in its simulation of the multi-core situation.

