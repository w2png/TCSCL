# TCSCL (Turing Complete Systems for Compute Languages)
**TCSCL** is a low-level, Turing complete programming language designed to run on OpenCL-compatible GPUs. It offers a minimalist syntax for performing fundamental operations such as arithmetic, logical functions, and loops, while allowing users to define and call functions.

## Key Features
- **GPU-Optimized**: Runs on OpenCL 1.2+ compatible GPUs for high-performance computing.
- **Modular Compiler**: Easily extendable with new instructions for future growth.

---

## How to Program in TCSCL (Turing Complete Systems for Compute Language) v1.3

### Introduction

Welcome to TCSCL! Think of TCSCL as a specialized language designed to directly instruct a Graphics Processing Unit (GPU) to perform calculations, especially repetitive ones, very quickly. It's like a custom assembly language for a virtual machine that runs on your GPU.

The C++ code you have is the **compiler and runtime** for TCSCL. You write your instructions in a `.tcscl` file, and this C++ program takes that file, prepares it, sends it to the GPU, executes it, and retrieves the results.

This guide will explain how to write the code that goes *inside* the `.tcscl` file.

### Core Concepts

1.  **Registers:**
    *   Imagine the GPU has a large set of temporary storage slots called **registers**. In TCSCL, there are 4096 registers, numbered 0 to 4095.
    *   Each register can hold *either* an integer (whole number) *or* a floating-point number (a number with a decimal point). The system uses the same memory space for both, so how you use it (e.g., with `OP_ADD_INT` vs `OP_ADD_FLOAT`) determines how the bits are interpreted.
    *   You'll use registers constantly to store values, perform calculations, and pass data between instructions. We often refer to them like `R0`, `R1`, `R6`, etc., corresponding to register indices 0, 1, 6.

2.  **Instructions (Opcodes):**
    *   Instructions are the commands you give to the virtual machine. Each instruction has a name (like `OP_LOAD_INT`, `OP_ADD_FLOAT`, `OP_HALT`) called an **opcode**.
    *   Most instructions are followed by **arguments**, which are the data the instruction needs to work with. Arguments are separated by commas. These can be register numbers, immediate (literal) values, or special IDs (like function IDs, label IDs, or list IDs).

3.  **Data Types:**
    *   **Integer (`int`):** Whole numbers (e.g., 5, -10, 0). Used with `_INT` instructions.
    *   **Floating-Point (`float`):** Numbers with decimal points (e.g., 3.14, -0.5, 8.0). Used with `_FLOAT` instructions.
    *   **`float_as_int()`:** Because the program code itself is stored as integers, you *cannot* directly write `3.14f` in your `.tcscl` code. Instead, when you need to load a literal float value, you use the special notation `float_as_int(value)` in the C++ vector that defines your program (or conceptually, when writing your code). The C++ compiler translates this into the correct integer bit pattern representing that float. For example, to load 3.14 into Register 6, you'd conceptually write: `OP_LOAD_FLOAT, 6, float_as_int(3.14f)`.

4.  **Program Flow:**
    *   By default, instructions execute one after another, top to bottom.
    *   You can change this flow using **jump** instructions (`OP_JUMP`, `OP_JUMP_IF_ZERO`), **call** instructions (`OP_CALL`, `OP_RET`), and **conditional** blocks (`OP_IF`, `OP_ELSE`, `OP_ENDIF`).

### Structure of a `.tcscl` File

Your actual TCSCL code isn't written directly *in* the C++ file (except for the example). You would typically create a text file (e.g., `my_program.tcscl`) and then modify the C++ host code to load and parse *that* file into the `host_program_source` vector.

However, based on the example structure provided in the prompt, a `.tcscl` file conventionally looks like this:

1.  **Constants (Optional, C++ side):**
    *   You can define constants in C++ (like `const int DEF_SE_LOSS = 1;`) to give meaningful names to numbers (like function IDs). This makes your TCSCL code easier to read.

2.  **Comments:**
    *   Lines starting with `//` are comments. They are ignored by the compiler and are used to explain the code.

3.  **Program Details Block (Convention):**
    *   Often starts with `// Program Details [` and ends with `// ] End Program Details`.
    *   Used for high-level comments about the program, like conventions used (e.g., the `FLOAT_ID_THRESHOLD`).

4.  **Cache Block (Convention, "Initialization"):**
    *   Often starts with `// Cache [` and ends with `// ] End Cache`.
    *   **Important:** This isn't a special hardware cache. It's just a conventional place at the *beginning* of the instruction list where you often put `OP_LOAD_INT` or `OP_LOAD_FLOAT` instructions to initialize registers with default values (like 0 or 0.0f) before the main logic starts. It's just regular TCSCL code.

5.  **Program Block (Main Execution):**
    *   Often starts with `// Program [` and ends with `// ] End Program`.
    *   This is the main part of your program that runs first on the GPU (in a single thread).
    *   It *must* eventually end with an `OP_HALT` instruction to signal the end of the main execution phase.

6.  **Function Definitions Block:**
    *   Often starts with `// Function Definitions [` and ends with `// ] End Function Definitions`.
    *   This is where you define reusable blocks of code called functions.
    *   Each function starts with `OP_FUNC_DEF, function_id` and ends with `OP_RET`.
    *   Functions defined here can be called from the main `Program` block or even from other functions using `OP_CALL`.
    *   **Crucially:** After the main `Program` block hits `OP_HALT`, *all* functions defined here are automatically scheduled to run **in parallel**, each function instance potentially running on a different GPU core.

### Core Instruction Overview

Here's a breakdown of the available instructions (opcodes) by category:

**Data Movement**

*   `OP_LOAD_INT, dest_reg, integer_value`
    *   Loads an immediate integer value into a register.
    *   Example: `OP_LOAD_INT, 1, 100` (Put 100 into R1)
*   `OP_LOAD_FLOAT, dest_reg, float_as_int(float_value)`
    *   Loads an immediate float value into a register (remember `float_as_int`).
    *   Example: `OP_LOAD_FLOAT, 6, float_as_int(9.81f)` (Put 9.81 into R6)
*   `OP_MOV, dest_reg, source_reg`
    *   Copies the value (bits) from `source_reg` to `dest_reg`.
    *   Example: `OP_MOV, 2, 1` (Copy value from R1 into R2)
*   `OP_POP_REG, reg_id`
    *   Sets the specified register `reg_id` to 0 (integer zero). Useful for clearing.
    *   Example: `OP_POP_REG, 5` (Set R5 to 0)
*   `OP_ROUND, destIntReg, srcFloatReg`
    *   Sets the specified register `srcFloatReg` to be rounded. Useful for List-Loading in Integers.
    *   Example: `OP_ROUND, 1, 9` (Round R9 and save to R1)

**Arithmetic**

*   `OP_ADD_INT, dest_reg, src_reg1, src_reg2` (Integer Addition: R_dest = R_src1 + R_src2)
*   `OP_SUB_INT, dest_reg, src_reg1, src_reg2` (Integer Subtraction: R_dest = R_src1 - R_src2)
*   `OP_MUL_INT, dest_reg, src_reg1, src_reg2` (Integer Multiplication: R_dest = R_src1 * R_src2)
*   `OP_DIV_INT, dest_reg, src_reg1, src_reg2` (Integer Division: R_dest = R_src1 / R_src2, Result is 0 if R_src2 is 0)
*   `OP_ADD_FLOAT, dest_reg, src_reg1, src_reg2` (Float Addition)
*   `OP_SUB_FLOAT, dest_reg, src_reg1, src_reg2` (Float Subtraction)
*   `OP_MUL_FLOAT, dest_reg, src_reg1, src_reg2` (Float Multiplication)
*   `OP_DIV_FLOAT, dest_reg, src_reg1, src_reg2` (Float Division, Result is undefined/NaN/Inf if R_src2 is 0.0)
    *   Example: `OP_ADD_FLOAT, 8, 6, 7` (R8 = R6 + R7, treating values as floats)

**Logical & Bitwise**

*   `OP_AND, dest_reg, src_reg1, src_reg2` (Logical AND: R_dest = (R_src1 != 0) && (R_src2 != 0) ? 1 : 0)
*   `OP_OR, dest_reg, src_reg1, src_reg2` (Logical OR: R_dest = (R_src1 != 0) || (R_src2 != 0) ? 1 : 0)
*   `OP_NOT, dest_reg, source_reg` (Logical NOT: R_dest = (R_source == 0) ? 1 : 0)
*   `OP_XOR, dest_reg, src_reg1, src_reg2` (Bitwise XOR: R_dest = R_src1 ^ R_src2, integer bits are XORed)
    *   Example: `OP_AND, 3, 1, 2` (If R1 and R2 are both non-zero, set R3 to 1, else set R3 to 0)

**Control Flow**

*   `OP_LABEL, label_id`
    *   Defines a named location in the code. `label_id` is an integer you choose. This instruction itself does nothing during execution but marks a spot.
    *   Example: `OP_LABEL, 10` (Marks this spot as Label 10)
*   `OP_JUMP, label_id`
    *   Unconditionally jumps execution to the instruction immediately *after* the `OP_LABEL` with the matching `label_id`.
    *   Example: `OP_JUMP, 10` (Execution continues after `OP_LABEL, 10`)
*   `OP_JUMP_IF_ZERO, condition_reg, label_id`
    *   Jumps to the instruction after `OP_LABEL, label_id` *only if* the integer value in `condition_reg` is exactly 0. Otherwise, continues to the next instruction.
    *   Example: `OP_JUMP_IF_ZERO, 3, 10` (If R3 contains 0, jump to Label 10)
*   `OP_FUNC_DEF, function_id`
    *   Defines the start of a function block. `function_id` is an integer you choose (often using a C++ `const int`). Execution doesn't start here unless called or run in parallel.
    *   Example: `OP_FUNC_DEF, 1` (Starts the definition of Function 1)
*   `OP_CALL, function_id`
    *   Pauses execution at the current point, jumps to the start of the function defined with `OP_FUNC_DEF, function_id`, executes that function until `OP_RET`, and then returns to the instruction *after* the `OP_CALL`. Uses a stack to handle nested calls.
    *   Example: `OP_CALL, 1` (Execute Function 1, then come back here)
*   `OP_RET`
    *   Marks the end of a function's execution. Returns control to wherever the function was called from (using `OP_CALL`). If executed outside a function called via `OP_CALL` (e.g., during parallel execution), it terminates that specific execution thread.
*   `OP_IF, condition_reg`
    *   Starts a conditional block. If the integer value in `condition_reg` is *not* 0 (i.e., true), the instructions between `OP_IF` and the next `OP_ELSE` or `OP_ENDIF` are executed. Otherwise, they are skipped.
*   `OP_ELSE`
    *   Used within an `OP_IF`/`OP_ENDIF` block. If the original `OP_IF` condition was false (0), the instructions between `OP_ELSE` and `OP_ENDIF` are executed.
*   `OP_ENDIF`
    *   Marks the end of an `OP_IF` or `OP_IF`/`OP_ELSE` block. Execution continues normally after this point.
    *   Example:
        ```tcscl
        OP_LOAD_INT, 1, 5
        OP_LOAD_INT, 2, 0
        OP_IF, 1             // R1 is 5 (not 0), so execute this block
        OP_OUTPUT_INT, 1, 1  // This will run
        OP_ELSE              // Skip this block
        OP_OUTPUT_INT, 2, 2  // This will NOT run
        OP_ENDIF             // End of conditional
        OP_OUTPUT_INT, 1, 3  // This runs regardless
        ```
*   `OP_HALT`
    *   Stops the execution of the *main program thread*. This is essential. After `OP_HALT`, the parallel function execution phase begins (if functions are defined). In a parallel function thread, `OP_HALT` stops just that thread.
*   `OP_NOP`
    *   No Operation. Does nothing, just takes up a cycle and moves to the next instruction. Sometimes useful for debugging or padding.

**Comparison**

These instructions compare two registers and store the result (1 for true, 0 for false) as an integer in the destination register.

*   `OP_EQ_INT, dest_reg, src_reg1, src_reg2` (Equal Int: R_dest = (R_src1 == R_src2) ? 1 : 0)
*   `OP_NEQ_INT, dest_reg, src_reg1, src_reg2` (Not Equal Int)
*   `OP_GT_INT, dest_reg, src_reg1, src_reg2` (Greater Than Int)
*   `OP_LT_INT, dest_reg, src_reg1, src_reg2` (Less Than Int)
*   `OP_GTE_INT, dest_reg, src_reg1, src_reg2` (Greater Than or Equal Int)
*   `OP_LTE_INT, dest_reg, src_reg1, src_reg2` (Less Than or Equal Int)
*   `OP_EQ_FLOAT, dest_reg, src_reg1, src_reg2` (Equal Float)
*   `OP_NEQ_FLOAT, dest_reg, src_reg1, src_reg2` (Not Equal Float)
*   `OP_GT_FLOAT, dest_reg, src_reg1, src_reg2` (Greater Than Float)
*   `OP_LT_FLOAT, dest_reg, src_reg1, src_reg2` (Less Than Float)
*   `OP_GTE_FLOAT, dest_reg, src_reg1, src_reg2` (Greater Than or Equal Float)
*   `OP_LTE_FLOAT, dest_reg, src_reg1, src_reg2` (Less Than or Equal Float)
    *   Example: `OP_GT_FLOAT, 3, 6, 7` (Set R3 to 1 if R6 > R7, else set R3 to 0, comparing as floats)

**Output**

*   `OP_OUTPUT_INT, source_reg, output_id`
    *   Sends the integer value from `source_reg` to the host (CPU) associated with the integer `output_id`.
*   `OP_OUTPUT_FLOAT, source_reg, output_id`
    *   Sends the float value from `source_reg` to the host associated with the integer `output_id`.
    *   **Convention:** The C++ host code example uses a threshold (e.g., `output_id >= 10000`) to distinguish float outputs from integer outputs when writing the final `gpuOut.bin` file. You should follow the convention set in the host code.
    *   Example: `OP_OUTPUT_FLOAT, 8, 10001` (Output the float value in R8 with ID 10001)

**List Operations (for dynamic arrays of floats)**

These operate on dynamic lists stored on the GPU. Each list has an ID (0 to MAX_LISTS-1).

*   `OP_FLOAT_LIST, list_id`
    *   Declares that a specific `list_id` is intended to be used. Needs to be called before using the list ID with other list operations (unless loaded with `OP_LOAD_LNST`).
    *   Example: `OP_FLOAT_LIST, 0` (Declare list 0 as usable)
*   `OP_ADD_LIST_ITEM, list_id, value_reg`
    *   Appends the float value currently in `value_reg` to the end of the list specified by `list_id`. Handles resizing automatically (up to `MAX_LIST_SIZE`).
    *   Example: `OP_ADD_LIST_ITEM, 0, 6` (Add the float from R6 to the end of list 0)
*   `OP_READ_LIST_ITEM, dest_reg, list_id, index_reg`
    *   Reads the float item from the list `list_id` at the position specified by the *integer* value in `index_reg`. Stores the float value in `dest_reg`. If the index is invalid (out of bounds), `dest_reg` is set to 0.0f.
    *   Example: `OP_LOAD_INT, 1, 5` (Put index 5 into R1) -> `OP_READ_LIST_ITEM, 7, 0, 1` (Read item at index 5 from list 0 into R7)
*   `OP_REMOVE_LIST_ITEM, list_id, index_reg`
    *   Removes the item from list `list_id` at the position specified by the *integer* value in `index_reg`. It does this efficiently by swapping the last item into the removed item's slot and reducing the list size. Order is not preserved. Does nothing if the index is invalid.
    *   Example: `OP_LOAD_INT, 1, 2` (Put index 2 into R1) -> `OP_REMOVE_LIST_ITEM, 0, 1` (Remove item at index 2 from list 0)
*   `OP_EDIT_LIST_ITEM, list_id, index_reg, value_reg`
    *   Overwrites the float item in list `list_id` at the position specified by the *integer* value in `index_reg` with the float value currently in `value_reg`. Does nothing if the index is invalid.
    *   Example: `OP_LOAD_INT, 1, 0` (Put index 0 into R1) -> `OP_LOAD_FLOAT, 9, float_as_int(99.9f)` (Put value 99.9 into R9) -> `OP_EDIT_LIST_ITEM, 0, 1, 9` (Set item at index 0 in list 0 to the value from R9)
*   `OP_OUTPUT_LIST, list_id, user_output_id`
    *   Sends a special marker pair to the host output. The host C++ code interprets this marker pair, reads the entire content of the specified `list_id` from GPU memory, formats it as a string (e.g., `[1.0, 2.5, 3.0]`), and associates it with the `user_output_id` in the final `gpuOut.bin` file.
    *   Example: `OP_OUTPUT_LIST, 0, 20001` (Tell the host to read list 0 and output it with ID 20001)

**Input Loading (List Data)**

*   `OP_LOAD_LNST, load_id, dest_list_id`
    *   **Host-Side Operation:** This instruction tells the *host C++ code* (before the GPU kernel starts) to look inside the `gpuIn.bin` file for an entry matching `load_id`. If found, the host reads the list of floats associated with that `load_id` and copies it directly into the GPU memory allocated for the list identified by `dest_list_id`. It also marks `dest_list_id` as existing and sets its size.
    *   **GPU-Side Operation:** On the GPU itself, this instruction acts like a `NOP` (it does nothing and just advances the program counter). The actual data loading happened before the GPU code started running.
    *   **`gpuIn.bin` Format:** A text file where each line defines a list: `load_id: [float1, float2, float3, ...]` (e.g., `10: [1.2, 3.4, -5.6]`). Comments start with `#`.
    *   Example: `OP_LOAD_LNST, 10, 1` (Before GPU execution, the host looks for ID 10 in `gpuIn.bin` and loads its data into GPU list 1).

**Random Numbers**

*   `OP_RAND_M, dest_reg_float`
    *   Generates a pseudo-random floating-point number between 0.0 and 1.0 (inclusive) and stores it in `dest_reg_float`. Each execution thread (main or parallel function) has its own independent random number sequence.
    *   Example: `OP_RAND_M, 10` (Put a random float between 0.0 and 1.0 into R10)

### Execution Model: Main vs. Parallel Functions

1.  **Main Program Execution:**
    *   The code in the `// Program [...]` block runs first.
    *   It runs on a *single* GPU thread/core.
    *   It continues until it hits `OP_HALT`.

2.  **Parallel Function Execution:**
    *   *After* the main program hits `OP_HALT`, the system looks at all functions defined using `OP_FUNC_DEF`.
    *   It launches *each defined function* as a separate, parallel task. If you defined 5 functions, it attempts to run 5 function instances simultaneously on different GPU cores (hardware permitting).
    *   These functions operate on the *final state* of the registers left by the main program.
    *   They also operate on the *shared* list data.
    *   **Concurrency Warning:** Because registers and lists are shared, if multiple parallel functions try to write to the *same* register or modify the *same* list concurrently without careful coordination, you can get **race conditions** (unpredictable results depending on which thread finishes writing last). List operations like `ADD` and `REMOVE` use atomic operations internally for basic safety (preventing two threads from getting the same index), but complex interactions or edits to the *same* item can still be problematic. Reads from registers/lists are generally safer but might read stale data if another thread modifies it concurrently. Design parallel functions to work on independent data as much as possible.

### Example Walkthrough (`DEF_SE_LOSS` function)

Let's break down the example from the prompt:

```tcscl
// Define a constant for the function ID (done in C++)
const int DEF_SE_LOSS = 1;

// Program Details [...] (Comments)

// Cache [...] ("Initialization")
OP_LOAD_INT, 1, 0,      // R1 = 0 (integer)
OP_LOAD_INT, 2, 0,      // R2 = 0
// ... other initializations ...
OP_LOAD_FLOAT, 6, float_as_int(0.0f), // R6 = 0.0 (float)
OP_LOAD_FLOAT, 7, float_as_int(0.0f), // R7 = 0.0
OP_LOAD_FLOAT, 8, float_as_int(0.0f), // R8 = 0.0
OP_LOAD_FLOAT, 9, float_as_int(0.0f), // R9 = 0.0
OP_LOAD_FLOAT, 10, float_as_int(0.0f),// R10 = 0.0
// ] End Cache

// Program [...] (Main execution block)
OP_LOAD_FLOAT, 6, float_as_int(8.5f),  // Load 'true value' 8.5 into R6
OP_LOAD_FLOAT, 7, float_as_int(7.341f), // Load 'predicted value' 7.341 into R7
OP_CALL, DEF_SE_LOSS,                 // Call function with ID 1 (DEF_SE_LOSS)
                                      // Execution jumps to the function, R6 and R7 hold inputs
                                      // ... function executes ...
                                      // Function returns, R8 now holds the result
OP_OUTPUT_FLOAT, 8, 10001,            // Output the result in R8 with float ID 10001

OP_HALT,                              // End main program execution. Now parallel functions run.
// ] End Program

// Function Definitions [...]
OP_FUNC_DEF, DEF_SE_LOSS,             // Define function 1. Comment says: Args R6, R7, Result R8
OP_SUB_FLOAT, 9, 6, 7,                // R9 = R6 - R7 (Calculate the difference)
OP_MUL_FLOAT, 8, 9, 9,                // R8 = R9 * R9 (Square the difference -> Squared Error)
OP_RET,                               // Return from function. Execution goes back after OP_CALL
// ] End Function Definitions
```

**Execution Flow:**

1.  Registers R1-R10 are initialized (mostly to 0 or 0.0f).
2.  R6 is set to 8.5f.
3.  R7 is set to 7.341f.
4.  `OP_CALL, 1` is executed. The program jumps to `OP_FUNC_DEF, 1`.
5.  Inside the function: `OP_SUB_FLOAT` calculates `8.5 - 7.341` and stores it in R9.
6.  `OP_MUL_FLOAT` calculates `R9 * R9` (the squared error) and stores it in R8.
7.  `OP_RET` is executed. The program returns to the instruction after `OP_CALL`.
8.  `OP_OUTPUT_FLOAT` sends the value now in R8 (the calculated squared error) to the host with ID 10001.
9.  `OP_HALT` is executed. The main program stops.
10. The system sees `OP_FUNC_DEF, 1` exists. It launches function 1 again, in parallel (though in this simple case, it just runs again, recalculating the same result based on the final values in R6/R7, likely writing to R8/R9 again, and potentially outputting again if the function included output instructions). If other functions were defined, they would also run in parallel now.

### Tips for Beginners

*   **Start Simple:** Write small programs first to understand individual instructions.
*   **Comment Your Code:** Explain *why* you are doing something, not just *what* the instruction does. Note register usage (e.g., `// R6: input value, R7: weight`).
*   **Initialize Registers:** Use the "Cache" section convention to set registers to known default values.
*   **Be Careful with Types:** Use `_INT` opcodes for integers and `_FLOAT` opcodes for floats. Remember `float_as_int()` for loading literal floats.
*   **Plan Register Use:** Think about which registers will hold inputs, intermediate values, and outputs, especially for functions.
*   **Test Incrementally:** Build your program piece by piece and test it.
*   **Understand Parallelism:** Be mindful of shared registers and lists when functions run in parallel. Avoid race conditions.
*   **Use Output for Debugging:** Use `OP_OUTPUT_INT` and `OP_OUTPUT_FLOAT` with distinct IDs to check intermediate values during development.

---
