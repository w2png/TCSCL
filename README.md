# TCSCL (Turing Complete Systems for Compute Languages)
**TCSCL** is a low-level, Turing complete programming language designed to run on OpenCL-compatible GPUs. It offers a minimalist syntax for performing fundamental operations such as arithmetic, logical functions, and loops, while allowing users to define and call functions.

## Key Features
- **GPU-Optimized**: Runs on OpenCL 1.2+ compatible GPUs for high-performance computing.
- **Modular Compiler**: Easily extendable with new instructions for future growth.

---
Okay, adventurers! Get ready to dive into the wonderful world of **TCSCL (Turing Complete Systems for Compute Language)**, your friendly neighborhood language for programming a virtual machine that runs right on your powerful GPU! 🎉

Think of it like building with digital LEGOs, but these LEGOs crunch numbers super fast using your graphics card. TCSCL (we're talking version **pre-1.3.5** here) lets you write low-level instructions to control this virtual machine. It's powerful, a bit quirky, and a whole lot of fun once you get the hang of it!

---

## 🚀 Getting Started: Your First `.tcscl` File!

TCSCL programs live in plain text files, and we like to give them the `.tcscl` extension just to keep things tidy. Inside, the structure is usually organized into a few key sections, marked by helpful comments (`//` starts a comment, ignored by the machine!).

Here's the typical layout:

```tcscl
// CONSTANTS (Optional, but super helpful!)
// Define friendly names for numbers (like register IDs or function IDs) here.
const int MY_COOL_REGISTER = 0;
const int MY_AWESOME_FUNCTION = 10;
// ] // Marks the end of this section (just for human eyes!)

// REGISTERS [ (Optional, but good for setting initial values)
// Here you can pre-load registers with starting values before the main program runs.
// It's like setting up your tools before you start building!
// Use OP_LOAD_INT or OP_LOAD_FLOAT.
OP_LOAD_FLOAT, MY_COOL_REGISTER, float_as_int(3.14159f), // Put Pi in register 0
// ]

// PROGRAM
// This is the MAIN part of your code! Execution starts here.
// Put your sequence of instructions (opcodes and their arguments) here.
OP_CALL, MY_AWESOME_FUNCTION, // Example: Call a function
OP_OUTPUT_FLOAT, MY_COOL_REGISTER, 10000, // Output the value in register 0 with ID 10000
OP_HALT, // VERY important! Tells the machine to stop.
// ]

// FUNCTION DEFINITIONS (Optional)
// Define reusable blocks of code (functions) here.
// They only run when you explicitly CALL them.
OP_FUNC_DEF, MY_AWESOME_FUNCTION, // Start defining function 10
  OP_ADD_FLOAT, MY_COOL_REGISTER, MY_COOL_REGISTER, MY_COOL_REGISTER, // Add register 0 to itself
OP_RET, // Return from the function back to where it was called
// ]
```

See? Not too scary! The `// ]` markers aren't strictly needed by the machine, but they help *us* humans see where one section ends and another begins.

---

## 🛠️ The Core Building Blocks

TCSCL operates on a few fundamental concepts:

1.  **Registers:** Think of these as small, super-fast storage slots right inside the virtual machine's "brain". TCSCL gives you a bunch of them (specifically `4096` in this version!). Each register can hold *either* an integer (whole number) or a floating-point number (decimal number). You refer to registers by their number (0 to 4095) or, even better, by the friendly names you define in the `// CONSTANTS` section!
2.  **Opcodes:** These are the action words! They tell the virtual machine *what* to do. Examples: `OP_ADD_INT`, `OP_LOAD_FLOAT`, `OP_JUMP`, `OP_HALT`. Each opcode has a unique number assigned to it (like `OP_HALT` is 0, `OP_LOAD_INT` is 2, etc.).
3.  **Operands/Arguments:** Most opcodes need extra information to work – these are the operands or arguments. They follow the opcode directly. For example, `OP_LOAD_INT, 5, 100` means: use the `OP_LOAD_INT` command (opcode 2), target register `5` (operand 1), and load the value `100` (operand 2).
4.  **Program Counter (PC):** This is like the machine's finger pointing at the current instruction it's about to execute in your `// PROGRAM` section. It usually moves forward one instruction at a time, but opcodes like `OP_JUMP` or `OP_CALL` can make it jump around!
5.  **Lists:** TCSCL lets you work with dynamic lists of floating-point numbers! You can create them, add items, read items, change them, remove them, and even load them from a file or output them. Super handy for storing collections of data.

---

## ✨ The Mighty Instruction Set (Opcodes Explained!) ✨

Here's a cheerful tour of the commands (opcodes) available in TCSCL pre-1.3.5. Each instruction takes up a specific number of "slots" (integers) in the program memory.

**System & Control Flow:**

*   `OP_HALT` (1 slot)
    *   **Action:** Stops the *entire* virtual machine execution immediately. Essential! Put it at the end of your main program sequence.
*   `OP_NOP` (1 slot)
    *   **Action:** No Operation. Does absolutely nothing! Sometimes useful as a placeholder.
*   `OP_JUMP` (2 slots: `OP_JUMP, target_label_id`)
    *   **Action:** Tells the PC to jump to a different part of the code marked by `OP_LABEL, target_label_id`. The host figures out the exact jump distance before sending the code to the GPU.
*   `OP_JUMP_IF_ZERO` (3 slots: `OP_JUMP_IF_ZERO, condition_register, target_label_id`)
    *   **Action:** Checks the integer value in `condition_register`. If it's exactly `0`, it jumps to the `target_label_id`. Otherwise, it continues to the next instruction. Great for conditional logic!
*   `OP_CALL` (2 slots: `OP_CALL, function_id`)
    *   **Action:** Jumps to the start of a function defined with `OP_FUNC_DEF, function_id`. It remembers where it came from so `OP_RET` can return!
*   `OP_RET` (1 slot)
    *   **Action:** Returns from the current function back to the instruction right after the `OP_CALL` that brought it there. Don't use this outside a function!
*   `OP_FUNC_DEF` (2 slots: `OP_FUNC_DEF, function_id`)
    *   **Action (Host):** Marks the beginning of a function definition during pre-processing.
    *   **Action (GPU):** Does nothing (NOP). Execution skips over it unless `OP_CALL`ed. The `function_id` should be unique and between 0 and `MAX_FUNCTIONS - 1` (511).
*   `OP_LABEL` (2 slots: `OP_LABEL, label_id`)
    *   **Action (Host):** Marks a location in the code during pre-processing so `OP_JUMP` and `OP_JUMP_IF_ZERO` can find it.
    *   **Action (GPU):** Does nothing (NOP). Execution skips over it.

**Conditional Execution (IF/ELSE):**

*   `OP_IF` (2 slots: `OP_IF, condition_register`)
    *   **Action:** Checks the integer value in `condition_register`. If it's *not* zero (i.e., true), instructions between this `OP_IF` and the next `OP_ELSE` or `OP_ENDIF` are executed. If it *is* zero (false), they are skipped.
*   `OP_ELSE` (1 slot)
    *   **Action:** If the preceding `OP_IF` condition was false, instructions between this `OP_ELSE` and the `OP_ENDIF` are executed. If the `OP_IF` condition was true, these instructions are skipped.
*   `OP_ENDIF` (1 slot)
    *   **Action:** Marks the end of an `OP_IF` or `OP_IF`/`OP_ELSE` block. Execution continues normally after this point, regardless of the condition.

**Data Movement & Loading:**

*   `OP_LOAD_INT` (3 slots: `OP_LOAD_INT, dest_register, integer_value`)
    *   **Action:** Loads the given `integer_value` directly into the `dest_register`.
*   `OP_LOAD_FLOAT` (3 slots: `OP_LOAD_FLOAT, dest_register, float_value_as_int_bits`)
    *   **Action:** Loads a floating-point number into `dest_register`. **Important:** You need to provide the *integer representation* of the float's bits! Use the magic `float_as_int(your_float_value)` in your `.tcscl` file setup (like in the example) or calculate it beforehand.
    *   *Example:* `OP_LOAD_FLOAT, 0, float_as_int(1.23f)`
*   `OP_MOV` (3 slots: `OP_MOV, dest_register, source_register`)
    *   **Action:** Copies the *exact* value (int or float bits) from `source_register` to `dest_register`.

**Integer Arithmetic:** (Operate on registers as integers)

*   `OP_ADD_INT` (4 slots: `OP_ADD_INT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg + src2_reg`
*   `OP_SUB_INT` (4 slots: `OP_SUB_INT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg - src2_reg`
*   `OP_MUL_INT` (4 slots: `OP_MUL_INT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg * src2_reg`
*   `OP_DIV_INT` (4 slots: `OP_DIV_INT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg / src2_reg`. Integer division (discards remainder). Protect against division by zero! (Result is 0 if `src2_reg` is 0).

**Floating-Point Arithmetic:** (Operate on registers as floats)

*   `OP_ADD_FLOAT` (4 slots: `OP_ADD_FLOAT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg + src2_reg`
*   `OP_SUB_FLOAT` (4 slots: `OP_SUB_FLOAT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg - src2_reg`
*   `OP_MUL_FLOAT` (4 slots: `OP_MUL_FLOAT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg * src2_reg`
*   `OP_DIV_FLOAT` (4 slots: `OP_DIV_FLOAT, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg / src2_reg`. Standard float division.

**Logical & Bitwise Operations:** (Operate on registers as integers)

*   `OP_AND` (4 slots: `OP_AND, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = (src1_reg != 0) && (src2_reg != 0) ? 1 : 0`. Performs logical AND.
*   `OP_OR` (4 slots: `OP_OR, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = (src1_reg != 0) || (src2_reg != 0) ? 1 : 0`. Performs logical OR.
*   `OP_NOT` (3 slots: `OP_NOT, dest_reg, src_reg`)
    *   **Action:** `dest_reg = (src_reg == 0) ? 1 : 0`. Performs logical NOT.
*   `OP_XOR` (4 slots: `OP_XOR, dest_reg, src1_reg, src2_reg`)
    *   **Action:** `dest_reg = src1_reg ^ src2_reg`. Performs bitwise XOR.
*   `OP_POP_REG` (2 slots: `OP_POP_REG, register_to_clear`)
    *   **Action:** Sets the value in `register_to_clear` to integer 0. Useful for resetting flags or values.

**Comparison Operations:** (Result is 1 for true, 0 for false, stored in `dest_reg` as an integer)

*   *Integer Comparisons:*
    *   `OP_EQ_INT` (4 slots): Equal (`==`)
    *   `OP_NEQ_INT` (4 slots): Not Equal (`!=`)
    *   `OP_GT_INT` (4 slots): Greater Than (`>`)
    *   `OP_LT_INT` (4 slots): Less Than (`<`)
    *   `OP_GTE_INT` (4 slots): Greater Than or Equal (`>=`)
    *   `OP_LTE_INT` (4 slots): Less Than or Equal (`<=`)
    *   *Arguments:* `OP_CODE, dest_reg, src1_reg, src2_reg`
*   *Floating-Point Comparisons:*
    *   `OP_EQ_FLOAT` (4 slots): Equal (`==`)
    *   `OP_NEQ_FLOAT` (4 slots): Not Equal (`!=`)
    *   `OP_GT_FLOAT` (4 slots): Greater Than (`>`)
    *   `OP_LT_FLOAT` (4 slots): Less Than (`<`)
    *   `OP_GTE_FLOAT` (4 slots): Greater Than or Equal (`>=`)
    *   `OP_LTE_FLOAT` (4 slots): Less Than or Equal (`<=`)
    *   *Arguments:* `OP_CODE, dest_reg, src1_reg, src2_reg`

**Output:**

*   `OP_OUTPUT_INT` (3 slots: `OP_OUTPUT_INT, source_register, output_id`)
    *   **Action:** Sends the integer value from `source_register` to the output buffer, tagging it with the integer `output_id`. The host program will collect these later.
*   `OP_OUTPUT_FLOAT` (3 slots: `OP_OUTPUT_FLOAT, source_register, output_id`)
    *   **Action:** Sends the float value from `source_register` to the output buffer, tagging it with `output_id`. **Convention:** Often, float output IDs are set high (e.g., >= 10000) to distinguish them from integer outputs, but this isn't strictly enforced by the VM itself.

**List Operations (Dynamic Float Lists!):**

*   **Key Concept:** List operations work with *list IDs*. Critically, you don't specify the list ID directly in the instruction! Instead, you put the desired list ID (0 to `MAX_LISTS - 1`, which is 127) into a register, and then tell the instruction *which register holds the ID*.
*   `OP_FLOAT_LIST` (2 slots: `OP_FLOAT_LIST, list_id_register`)
    *   **Action:** Declares that the list ID currently stored in `list_id_register` is now valid and ready to be used. You *must* call this for a list ID before using other list operations on it (unless it was pre-loaded by `OP_LOAD_LNST`).
*   `OP_ADD_LIST_ITEM` (3 slots: `OP_ADD_LIST_ITEM, list_id_register, value_register`)
    *   **Action:** Appends the floating-point value from `value_register` to the *end* of the list whose ID is in `list_id_register`. Fails silently if the list is full (`MAX_LIST_SIZE`, which is 1024) or the list ID is invalid/undeclared.
*   `OP_READ_LIST_ITEM` (4 slots: `OP_READ_LIST_ITEM, dest_register, list_id_register, index_register`)
    *   **Action:** Reads the float value from the list (whose ID is in `list_id_register`) at the position specified by the integer in `index_register`. The value is stored in `dest_register`. If the index is out of bounds or the list is invalid, `dest_register` gets `0.0f`. Remember, list indices are 0-based!
*   `OP_REMOVE_LIST_ITEM` (3 slots: `OP_REMOVE_LIST_ITEM, list_id_register, index_register`)
    *   **Action:** Removes the item at the position specified by the integer in `index_register` from the list whose ID is in `list_id_register`. It does this efficiently by swapping the item with the *last* item in the list and then shrinking the list size. Fails silently if the index is invalid or the list is empty/invalid.
*   `OP_EDIT_LIST_ITEM` (4 slots: `OP_EDIT_LIST_ITEM, list_id_register, index_register, value_register`)
    *   **Action:** Overwrites the float value in the list (ID in `list_id_register`) at the position specified by the integer in `index_register` with the new float value from `value_register`. Fails silently if the index is out of bounds or the list is invalid.
*   `OP_OUTPUT_LIST` (3 slots: `OP_OUTPUT_LIST, list_id_register, user_output_id`)
    *   **Action:** Tells the host program to read the *entire* contents of the list whose ID is currently in `list_id_register`. The host will output this list associated with the `user_output_id` you provide. This sends special markers to the output buffer for the host to interpret.

**File Input & Utility (v1.3+):**

*   `OP_LOAD_LNST` (3 slots: `OP_LOAD_LNST, load_id, dest_list_id_or_reg`)
    *   **Action (Host Pre-Execution):** THIS IS SPECIAL! The *host* program looks for these *before* running the code on the GPU. It reads the `load_id` (operand 1) and the `dest_list_id` (operand 2, interpreted *directly* as the list ID to write to). It then looks for `load_id:` in the `gpuIn.bin` file, parses the float list `[...]` found there, and *pre-loads* it into the specified `dest_list_id` on the GPU. It also marks that list ID as existing.
    *   **Action (GPU Execution):** Does nothing (NOP). It's already been handled!
*   `OP_RAND_M` (2 slots: `OP_RAND_M, dest_register_float`)
    *   **Action:** Generates a pseudo-random floating-point number between 0.0 and 1.0 (inclusive) and stores it in `dest_register_float`. Each execution context (just the main one now) has its own internal random seed.
*   `OP_ROUND` (3 slots: `OP_ROUND, dest_register_int, source_register_float`)
    *   **Action:** Takes the float value from `source_register_float`, rounds it to the nearest whole number (0.5 rounds up towards positive infinity), and stores the result as an integer in `dest_register_int`.

---

## 💾 Working with Lists: Your Dynamic Data Holders

Lists are powerful! Remember these key points:

*   **Floats Only:** They store sequences of floating-point numbers.
*   **Dynamic Size:** They grow when you add items (`OP_ADD_LIST_ITEM`) and shrink when you remove them (`OP_REMOVE_LIST_ITEM`), up to `MAX_LIST_SIZE` (1024).
*   **Identified by Register:** You tell list opcodes *which register* holds the ID (0-127) of the list you want to work with.
*   **Initialization:** Lists must be initialized either by `OP_FLOAT_LIST` before use on the GPU, or pre-loaded by the host using `OP_LOAD_LNST`.
*   **GPU Magic:** Adding/removing items uses atomic operations on the GPU to avoid race conditions if you ever have multiple threads (though in pre-1.3.5, it's mainly single-threaded execution via `CALL`).
*   **Indices:** List items are accessed by a 0-based index (0 is the first item, 1 is the second, and so on).

---

## 📤📥 Input & Output: Talking to the Outside World

TCSCL interacts with the host system primarily through two files:

*   **`gpuIn.bin` (Input):**
    *   Used *only* by the host during the pre-execution phase for `OP_LOAD_LNST`.
    *   It's a text file you create.
    *   Format: Each line defines a list to be loaded.
        ```
        # Comments start with #
        load_id: [float1, float2, float3, ...]
        # Example:
        5: [10.1, 20.2, 30.3]
        12: [1.0, 2.0]
        7: [] # An empty list
        ```
    *   The `load_id` corresponds to the first argument of `OP_LOAD_LNST`. The list contents will be loaded into the GPU list ID specified by the second argument of `OP_LOAD_LNST`.
*   **`gpuOut.bin` (Output):**
    *   This file is *created* by the host program after the GPU kernel finishes.
    *   It contains the results from `OP_OUTPUT_INT`, `OP_OUTPUT_FLOAT`, and `OP_OUTPUT_LIST`.
    *   Format: Text file, with each line representing an output ID and its value(s).
        ```
        output_id: value1[, value2, ...]
        # Example Output:
        100: 123 # From OP_OUTPUT_INT, reg, 100
        10000: 3.141590 # From OP_OUTPUT_FLOAT, reg, 10000
        50: [10.1, 20.2, 30.3] # From OP_OUTPUT_LIST, list_id_reg, 50
        100: 456 # Another output with ID 100
        ```
    *   Outputs with the same ID might appear grouped or on separate lines depending on host processing, but often they are grouped like the second `100` example. List output (`OP_OUTPUT_LIST`) results in the list content being formatted nicely as `[item1, item2, ...]`.

---

## ⚙️ Behind the Scenes: Host & GPU Teamwork

When you run your TCSCL code using the C++ host program:

1.  **Host Pass 1 (Scanning):** The C++ code first reads your raw `.tcscl` instructions. It finds all `OP_LABEL` and `OP_FUNC_DEF` markers and remembers the Program Counter (PC) location where the code *after* them starts. It also checks if any `OP_LOAD_LNST` instructions exist.
2.  **Host Pass 1.5 (Loading `gpuIn.bin`):** If `OP_LOAD_LNST` was found, the host parses `gpuIn.bin`.
3.  **Host Pass 2 (Resolving Jumps):** The host goes through the instructions again. It replaces the `label_id` in `OP_JUMP` and `OP_JUMP_IF_ZERO` instructions with the actual *offset* (distance) the PC needs to jump.
4.  **Host Pass 2.5 (Pre-Executing `OP_LOAD_LNST`):** The host finds all `OP_LOAD_LNST` instructions. For each one, it takes the `load_id`, finds the corresponding data in the parsed `gpuIn.bin` map, takes the `dest_list_id`, and directly writes that data and size into the GPU's list buffers *before* the main kernel starts.
5.  **GPU Execution:** The host copies the processed program, the function lookup table, and the pre-loaded list data to the GPU's memory. It then launches a *single* GPU kernel thread.
6.  **VM Kernel:** This kernel starts executing your instructions from PC 0 (the beginning of the `// PROGRAM` section). It uses the pre-calculated jump offsets and function PCs. `OP_CALL` pushes the return address onto a stack and jumps; `OP_RET` pops the address and jumps back. List operations manipulate the shared list buffers using safe atomic operations. Outputs are written to shared output buffers.
7.  **Host Reads Output:** Once the GPU kernel hits `OP_HALT` or finishes, the host reads the output buffers (including any list data requested by `OP_OUTPUT_LIST`) and writes everything neatly into `gpuOut.bin`.

---

## 🧩 Example Explained!

Let's look at that `.tcscl` example again:

```tcscl
// CONSTANTS (DECO)
const int RAND_UNIFORM = 1; // Function ID 1
const int RAND_OUTPUT = 1;  // Register ID 1
// ]

// REGISTERS [
OP_LOAD_FLOAT, RAND_OUTPUT, float_as_int(0.0f), // Put 0.0f into Register 1 initially
// ]

// PROGRAM
OP_CALL, RAND_UNIFORM, // Call function 1

// (After returning from the function, Register 1 holds a random float)
OP_OUTPUT_FLOAT, RAND_OUTPUT, 10001, // Output value in Reg 1 with ID 10001

OP_HALT, // Stop!
// ]

// FUNCTION DEFINITIONS
OP_FUNC_DEF, RAND_UNIFORM, // Define function 1
    OP_RAND_M, RAND_OUTPUT, // Put a random float (0-1) into Register 1
OP_RET, // Return from function
// ]
```

**How it runs:**

1.  **Host:** Defines constants, sees `OP_LOAD_FLOAT` for initial register setup. Finds `OP_FUNC_DEF` for function 1 and notes its start PC. Sees no jumps or `OP_LOAD_LNST`.
2.  **Host -> GPU:** Sends program, function table, and initial register value (0.0 in Reg 1) to GPU.
3.  **GPU Kernel Starts (PC=0):**
    *   `OP_CALL, 1`: Pushes return address (PC after this instruction) onto the call stack. Jumps to the PC associated with function ID 1.
    *   *(Inside Function 1)* `OP_RAND_M, 1`: Generates a random float (e.g., 0.789) and stores it in Register 1.
    *   *(Inside Function 1)* `OP_RET`: Pops the return address from the stack. Jumps back to the instruction after `OP_CALL`.
    *   `OP_OUTPUT_FLOAT, 1, 10001`: Reads the value from Register 1 (e.g., 0.789) and sends it to the output buffer with ID 10001.
    *   `OP_HALT`: Kernel stops.
4.  **Host Reads Output:** Host reads the output buffer, finds ID 10001 with value 0.789.
5.  **Host Writes `gpuOut.bin`:** Creates `gpuOut.bin` with content like: `10001: 0.789000`

Pretty neat, huh?

---

## 💡 Tips for Happy TCSCL Coding! 💡

*   **Use Constants!** Define names for registers, function IDs, list IDs, and even magic numbers using `const int NAME = value;`. It makes your code *so* much easier to read and modify.
*   **Comment Generously:** Use `//` to explain *why* you are doing something, not just *what* the instruction does (the opcode name often tells you that). Explain your logic!
*   **Plan Your Registers:** With 4096 registers, you have plenty, but try to group related data or assign specific roles to certain registers (e.g., R0-R9 for temporary calculations, R100+ for persistent state).
*   **Initialize Lists:** Remember to use `OP_FLOAT_LIST` or rely on `OP_LOAD_LNST` before manipulating a list ID.
*   **Handle Float Bits:** Use `float_as_int()` helper (or calculate the bits) when using `OP_LOAD_FLOAT`.
*   **Think Sequentially (Mostly):** Execution flows top-down unless you use jumps or calls. Keep conditional blocks (`IF`/`ELSE`) balanced with `ENDIF`.
*   **End with `OP_HALT`:** Make sure your main program flow eventually reaches an `OP_HALT`.

---

## ⚠️ Important Limits & Quirks (pre-1.3.5) ⚠️

*   `REGISTER_COUNT`: 4096 (0-4095)
*   `STACK_DEPTH`: 256 (limits nested `OP_CALL`s)
*   `MAX_OUTPUTS`: 4096 entries in the output buffer. `OP_OUTPUT_LIST` uses 2 slots per call.
*   `MAX_FUNCTIONS`: 512 (Function IDs 0-511)
*   `MAX_LISTS`: 128 (List IDs 0-127)
*   `MAX_LIST_SIZE`: 1024 float items per list.
*   **Single Main Thread:** Execution starts with one thread. `OP_CALL` happens within that thread. (No automatic parallel function execution like in older versions).
*   **Error Handling:** Many operations fail silently on the GPU (e.g., list index out of bounds, division by zero). Code defensively! Use comparisons (`OP_LT_INT`, etc.) to check list indices or divisors if needed.
*   `OP_LOAD_LNST` is **Host-Only Pre-processing**. It's a NOP on the GPU itself.

---

## 🎉 Conclusion 🎉

You've made it! That's the grand tour of TCSCL pre-1.3.5. It's a unique way to tap into your GPU's power for general-purpose tasks defined by *your* instructions. It requires careful planning, especially with register usage and list management, but gives you direct control.

Go forth, experiment, build amazing things, and have fun commanding your own little GPU virtual machine! Happy coding! ✨
*   **Be Careful with Types:** Use `_INT` opcodes for integers and `_FLOAT` opcodes for floats. Remember `float_as_int()` for loading literal floats.
*   **Plan Register Use:** Think about which registers will hold inputs, intermediate values, and outputs, especially for functions.
*   **Test Incrementally:** Build your program piece by piece and test it.
*   **Understand Parallelism:** Be mindful of shared registers and lists when functions run in parallel. Avoid race conditions.
*   **Use Output for Debugging:** Use `OP_OUTPUT_INT` and `OP_OUTPUT_FLOAT` with distinct IDs to check intermediate values during development.

---
