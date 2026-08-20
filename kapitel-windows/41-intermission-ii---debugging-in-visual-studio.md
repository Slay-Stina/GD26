# 41 Intermission II - Debugging in Visual Studio

We want to be able to understand the flow of our code, and peek at variables to look at their values. We do this by using a debugger. We will be downloading the IDE Visual Studio and installing its Community version.
Download link: [https://visualstudio.microsoft.com/](https://visualstudio.microsoft.com/)
It's important that we download **Visual Studio** and not the smaller light-weight **Visual Studio Code** .
Visual Studio as a complete IDE (Integrated Development Environment) meaning that is a one-stop-shop for everything someone "would need" when working with programming. But it's bloated, slow and cumbersome. We will be using its debugging tools though.
Meaning that our day-to-day text editor is Helix and our debugger is Visual Studio .
With visual studio installed we can chose to open our root folder inside Visual Studio , then on the right-side pane we can right click our games .exe from our build folder and select `set as startup item` , this will tell Visual Studio to run that .exe when we press the green button.
Now we can add breakpoints to our code. A breakpoint is a notice to pause code execution once we hit a specific line of code. This means that we can pause our program at a critical moment to explore the state of our variables as well as stepping through our code as it runs, line by line.
We add breakpoints by pressing the leftmost side of a line of code. When done properly a little red circle will indicate the breakpoint . We can press that red circle again to remove it.
With a breakpoint set we can press "play" or F5 and once our code hits the breakpoint it will pause. At this point we can hover our cursor above variables to evaluate their content. We can stop our debugging by pressing shift+F5 or by pressing the red stop button .
Once our program has paused on a breakpoint we can use F11 and F10 to move the program forward
F10 steps to the next visible line below the current one F11 goes to the next piece of code being executed, jumping into a function if necessary. F10 does not enter functions instead it runs the entire function as if it was a single line of code.
We also have Shift+F11 that goes back out of a function that F11 dove into. Meaning that we can get back to the place where the function was called.

By using F10 and F11 we can learn how our code flows and find bugs that would otherwise be very hard to reason about.
We can also hover our cursor just to the left of a line that is below our current one, a small green play button will appear. Pressing this will run code to that point before stopping again. This is very useful if we want to jump past a for-loop that is going to run the same code 100+ times. Sparing us clicking F10 A LOT.
We might also want to pause execution on a line of code, but only if a certain variable has a specific value. For example a `TakeDamage()` function might only be something we want to evaluate if the damage taken would kill the player. For this we have conditional breakpoints . After we have created a breakpoint we can right-click it and select `condition` inside this new window we can add a little boolean logic like `health <= damage` . Then the code will only pause if the incoming damage would reduce the `health` variable to 0 or below.
There is more we can do with breakpoints but this covers the fundamentals!