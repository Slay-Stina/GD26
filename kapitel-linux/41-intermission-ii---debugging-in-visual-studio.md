# 41 Intermission II - Debugging in Visual Studio

We want to be able to understand the flow of our code, and peek at variables to look at their values. We do this by using a debugger.

> [!NOTE]
> On Linux, we don't use Visual Studio. Instead, we can use:
> - **GDB** (GNU Debugger) — a command-line debugger, usable with your editor (e.g. nvim with `vim-dap` or `gdb` integration)
> - **LLDB** — the LLVM debugger, works great with Clang
> - **VS Code** with the C++ extension as a lightweight debugger GUI
> - **Qt Creator** or **CLion** as full IDEs with debugging support
>
> We'll use LLDB as it pairs well with the Clang compiler we have been using.

We want to be able to understand the flow of our code, and peek at variables to look at their values. We do this by using a debugger.
Install LLDB via your package manager: `sudo pacman -S lldb` (Arch) or `sudo apt install lldb` (Debian/Ubuntu).

With LLDB installed we can launch our game with `lldb ./program` from the build directory. Then we can set breakpoints before running the program.

A breakpoint is a notice to pause code execution once we hit a specific line of code. This means that we can pause our program at a critical moment to explore the state of our variables as well as stepping through our code as it runs, line by line.
We add breakpoints in LLDB by typing `breakpoint set --file game.cpp --line 10` or using the shorthand `b game.cpp:10`. When done properly LLDB will confirm the breakpoint. We can remove it with `breakpoint delete <id>`.

With a breakpoint set we can type `run` and once our code hits the breakpoint it will pause. At this point we can type `frame variable` to see all local variables, or `p variable_name` to evaluate a specific variable. We can stop debugging by typing `quit` or pressing Ctrl+C then `quit`.

Once our program has paused on a breakpoint we can use `next` (or `n`) and `step` (or `s`) to move the program forward:
- `next` steps to the next visible line below the current one (does not enter functions)
- `step` goes to the next piece of code being executed, jumping into a function if necessary
- `finish` goes back out of a function that `step` dove into

By using `next` and `step` we can learn how our code flows and find bugs that would otherwise be very hard to reason about.

We can also set a breakpoint and then use `continue` (or `c`) to run to the next breakpoint. This is very useful if we want to jump past a for-loop that is going to run the same code 100+ times, sparing us pressing `next` a lot.

We might also want to pause execution on a line of code, but only if a certain variable has a specific value. For example a `TakeDamage()` function might only be something we want to evaluate if the damage taken would kill the player. For this we have conditional breakpoints. After we have created a breakpoint we can use `breakpoint modify <id> -c 'health <= damage'`. Then the code will only pause if the incoming damage would reduce the `health` variable to 0 or below.

There is more we can do with breakpoints but this covers the fundamentals!