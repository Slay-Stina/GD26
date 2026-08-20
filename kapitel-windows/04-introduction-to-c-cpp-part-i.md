# Introduction to C/C++ - Part I

Remember SDL3, CMake, and Ninja — we're going to ignore those for now as we will need to learn more about C++ before we can honestly start working with SDL3. Soon 95%+ percent of all code we'll write will be C++, only writing PowerShell script or CMake syntax from time to time.

The next step is to create a new empty .cpp file, open it, write some code, then use clang to compile it. Once that is done we run it. The reason we can do this is that we're just compiling a single C++ script, no need for a build system generator (CMake), instructions (CMakeLists.txt) or a build system (Ninja) because we require no linking or multiple files that need to be compiled together.

We need to update our directory to somewhere we can create and access a small .cpp file. This can be done by using the basic building blocks found in terminal programming.

> [!NOTE]
> A full exhaustive list can be found here: [Windows Commands](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/windows-commands)

We will instead create a little function in `$profile` (PowerShell) to help us get things set up.

> [!NOTE]
> I like the convenience of the following setup as I will show you a bunch of coding examples throughout the course.

```powershell
function practice {
    Set-Location -Path "D:\PROJECTS\PRACTICE"
    New-Item example.cpp -Force
    subl example.cpp
}
```

This function sets our working directory to a folder I've already created. The path will not be the same on your computer, it depends on where you decided to create your folder. I create a new file called `example.cpp`. I then run Sublime Text opening up that newly created file. The `-Force` parameter will force our computer to re-create the file, therefore making it empty.

Now we need a function to help us compile and run this small program:

```powershell
function practice_run {
    clang++ example.cpp -std=c++23 -o example.exe
    if ($?) {
        ./example.exe
    }
}
```

Ok, so this function first runs a line of code then it asks a question in form of an **if statement**. An if statement, used in all programming languages you will come across, is one of the most fundamental ways of controlling code flow. Let's explore it a bit.

```cpp
if (question) {
    // here we can do things only if the answer to the question was true
}
```

We can put any question we want into the parenthesis following our `if`. Let's for example imagine this being a nightclub:

```cpp
if (age > 18) {
    // welcome in
}
```

We can also explore code that only runs in case the answer is false using `else`:

```cpp
if (age > 18) {
    // welcome in
}
else {
    // get outta here!
}
```

With this in mind, let's break down the function: using our system environment variable `Path` to the folder containing `clang++` we get to call into it without specifying ITS/FULL/PATH. What follows are a set of instructions to clang:

- `example.cpp` — the name of the file we want to compile
- `-std=c++23` — tells clang++ to use C++ version 23, this will help us make some simpler code later
- `-o example.exe` — specifies the name of the output file, namely `example.exe`. Without `-o`, the program is given the default name `a.exe`. So whilst not a strictly necessary parameter, it helps keep things tidy.
- `if ($?)` — is true if the code we ran above it succeeded (returned 0). Not encountering an error means that the program exited with the exit code 0 — just like in our first SDL3 `main.cpp` example. 0 means all clear. Had the program returned any other int (integer) then the code inside the if statement would not be executed.
- `./example.exe` — look for a file in this directory called `example.exe` and run it

After saving our `$profile` file we need to reload it in PowerShell before these two newly created functions can be found. Running this simple line of code does the trick:

```powershell
. $profile
```

Let's add some code to our `example.cpp`:

```cpp
#include <print>
int main() {
    std::println("hello, sailor!");
    return 0;
}
```

All C++ programs need an entry point. This is the function called `main`. It has the `int` (integer) return type and at the last line of the `main()` function we return a `0` meaning the code ran without issue.

We want to be able to write text from our program into our terminal so we want to use the `println` function (this stands for **print on new line**). The `println` function is made available by the inclusion of `<print>` at the top of the program.

`std::` that comes before `println` is a safeguard put in place by the ISO C++ Standards Committee way back in the 90's. All functions added to the standard library are put into a namespace called `std` (stands for **standard**). This has two purposes:

1. Clearly show if a function was from the standard library by the addition of `std::`
2. Through the use of a namespace, it allows us, the programmer, to write our own `println` function if we would like, without causing a compile error when the compiler finds 2 functions with the same exact name.

Writing the `std::` namespace indicator everywhere is pretty tedious. We can tell our program to look in the `std` namespace for functions by telling it to `use` that namespace. This is done by adding the following code:

```cpp
using namespace std;
```

Making the new program:

```cpp
#include <print>
using namespace std;
int main() {
    println("hello, sailor!");
    return 0;
}
```

We could then remove `std::` and just write `println()`.

> [!NOTE]
> Namespaces will be part of all games you will work on, including those using C# and game engines like Unity.

You might have noticed that we add a semicolon to the end of all lines in C++. This is a required step dating back to the C programming language and the inception of C++. It was added for clarity, to show when we are at the end of a line. It's something we just have to accept, for both C++ and C# (unfortunately).

> [!NOTE]
> Thankfully our intellisense will catch us if we miss a `;`. And if it doesn't then our compiler will spit out a pretty clear error message when it fails to compile our program.

`main()` is a function, `println()` is a function. All functions have a pair of parenthesis after its name, this parenthesis hold the **parameters** we can send to the function.

## Let's learn about scope

Let's create a small function that adds two numbers together. To do this we leave the `main()` function's scope, marked by the curly braces `{}`:

```cpp
void AddNumbers(int a, int b){
    int result = a + b;
    println("{}", result);
}
```

A return type of `void` means that the function doesn't return anything. And therefore a `return X` line is not added (as that would create an error). By introducing two integers (`int a` and `int b`) to the parenthesis of the function we have declared that anyone using this function must provide two integers separated by a `,`.

There are many ways of printing something to the console. The `println` function can't work with an integer directly. Other methods like `cout` can. By adding `"{}"` as the first parameter to `println` it can take the second parameter (in our case the number 15) and replace the placeholder `"{}"` with it later. But don't worry, there was really no way for you to know this, and no way from reading the code for us to understand this syntax — some parts of how code is written we kinda just gotta learn.

If we use the following program instead, including `<iostream>` instead of `<print>` and working with the older `cout` syntax, we can get the same result with a program that looks like this instead:

```cpp
#include <iostream>
using namespace std;
void AddNumbers(int a, int b){
    int result = a + b;
    cout << result << endl;
}
int main(){
    AddNumbers(5, 10);
}
```

But notice those strange `<<` — we will not be seeing them in any other setting than these practice functions. Thankfully SDL3 has a helpful function that works like `println` but can accept more types of data without needing the strange placeholder `"{}"`.

Let's learn about scope. The newly created integer variable `result` is only available inside the curly braces of the function. Once the code reaches the end of the last line, all locally scoped variables are cleaned up.

> [!NOTE]
> A variable is a named piece of memory that stores something for us (a number, a word, a sentence, etc.)

Updating our `main()` function we can take a look at our program:

```cpp
#include <print>
using namespace std;
int main() {
    AddNumbers(5, 10);
    return 0;
}
void AddNumbers(int a, int b){
    int result = a + b;
    println("{}", result);
}
```

Now we can clearly see the difference between the function itself and calling that function. Our `main` function uses the `AddNumbers` function inside its scope, then outside of the `main()` function scope we write the actual function.

**BUT!** There is one problem: in C++ (not in C#) we can't use a function by another function before it has been seen by the compiler, and that happens in a top-down fashion. This feels silly and like something the computer should be able to handle, and here we touch on the idea of how programming languages are written by people and have different philosophies and trade-offs. By forcing the declaration of functions to be done in sequence the compiler can work faster.

> [!NOTE]
> With even simple Unity projects taking AGES to compile, I would like to stress the importance of a design decision as this one.

Swapping the position of the `AddNumbers()` and `main()` functions, then compiling and running our program, it spits out 15 — the combined total of the two values we passed to the function (5 and 10) that are then printed to the console via the `println()` function. After that we hit the `return 0` and the program closes.

What we've created is sorta the world's slowest and worst calculator. But it has taught us a few things:

1. What the difference between an `int` and a `void` function is
2. What scope is
3. How to wrangle the `println` function and other ways we can print text to the console
4. What a namespace is
5. That we can compile a simple program with CMake or Ninja
6. Function declaration order matters in C++
7. The difference between calling a function and defining a function
8. How parameters are passed to functions
9. How if statements control code flow
10. That we need semicolons at the end of lines

## Introduction to C/C++ - Part II

This lecture will focus on how we use and write our own .h files. .h files, called **headers**, are smaller files containing function declarations but not their bodies. Meaning that we don't write the logic inside their scopes, just their names, return types and the parameters they accept:

```cpp
void Add(int a, int b);
```

We will use `New-Item` to create a new .h file called `example.h` and the code written above will be the only content of this .h file (for now).

Back in our `example.cpp` we can now include this .h file at the top:

```cpp
#include <print>
#include "example.h"
using namespace std;
int main() {
    Add(5, 10);
    return 0;
}
void Add(int a, int b){
    int result = a + b;
    println("{}", result);
}
```

> [!NOTE]
> I also simplified the function name in the .h file and our .cpp file to just `Add`.

Because our .cpp includes this .h file, we get it added to the top of the file during compilation.

If you've noticed that when we include a .h file we do it by declaring its path (based on the root directory) using quotation marks `""`, but when we added `<print>` we used angle brackets `<>`. Angle brackets are used to include system files whose locations are already known to our program, like those that are part of the standard library.

## Let's learn about return types

Our `Add` function is poorly named because nowhere in the name `Add` can we infer that it prints the result. Let's change the return type on our `Add()` function from `void` to `int`. This will require the function to use a `return`:

```cpp
int Add(int a, int b){
    int result = a + b;
    return result;
}
```

Our function no longer prints anything, instead it returns the integer we've named `result`. We have to do 2 things to get our program to compile and get the number to print to the console:

1. Update our .h file so the function declaration matches the changes we made in the .cpp file (changed `void` to `int`)
2. Update our `main()` function to print the result of the function itself, but we will use some fancy footwork to put the `println()` and `Add()` functions on the same line:

```cpp
int main(){
    println("{}", Add(5, 10));
    return 0;
}
```

Look, we've added the function `Add()` as the parameter we pass to `println()`. When our code executes it will call the `Add` function and whatever value we return at the end will be substituted in place of the function call. So it looks something like this in the end:

```cpp
println("{}", 15);
```

In case we find the function-as-parameter thing difficult at this time, here's the same logic over a few more lines:

```cpp
int main(){
    int toPrint = Add(5, 10);
    println("{}", toPrint);
    return 0;
}
```

Here we store the result (the thing we return) of our `Add` function into the integer variable we've named `toPrint`, we then use that as the parameter in our `println`.

Time to come clean, the `Add` function is still not a good function. Doing basic arithmetic does not require a function. It can be done in just a normal line of code:

```cpp
println("{}", 5 + 10);
```

This works just as well. (But of course we are just using these basic functions to learn the basics of code flow.)

## Variable types

Let's look at a series of basic building blocks:

- `int`
- `float`
- `double`
- `bool`

All four of these are types of variables. We've already familiarized ourselves with `int`. The `float` type is similar to an `int` but holds a number with decimal precision. An `int` can't store 1.5 but a `float` can.

A `double` is also a variable type that stores a decimal number, but it's given more memory to work with than a `float` and can therefore be more precise (more decimal values stored).

A `bool` (short for **boolean**) holds one of two states: `true` or `false`. That's it.

When creating any variable we start with its **type**, followed by a **name**, then an assignment operator `=`, then its **value**, and lastly a `;`:

```cpp
int wholeNumber = 5;
float percentageValue = 0.75;
double precisionValue = 0.75443341234114;
bool isThisCool = false;
```

> [!NOTE]
> We will need to use the assignment operator `=` whenever we want to store the right side value in a left side variable.

There is a bit of syntax we need to learn, and it's in regards to `float` type variables. The following way of writing decimal numbers `0.34` is interpreted as a `double` by the computer and then converted to a `float` when assigned to it. This means that our computer does a little conversion each time we assign a float like this. To tell the computer that the decimal number we've written is really a `float` we append an `f` to the end of the decimal chain:

```cpp
float aFloatFromTheStart = 0.34f;
```

Let's look at a small program featuring a few of our variable types:

```cpp
#include <print>
using namespace std;
int main(){
    int playerHealth = 10;
    int enemyDamage = 6;
    playerHealth -= enemyDamage;
    bool isPlayerDead = playerHealth <= 0;

    if(isPlayerDead == true){
        println("Ugh! I'm dead!");
    }
    else{
        println("is this all you got?");
    }
}
```

First we create two variables, both `int`s. Then we use a minus `-` and an assignment operator `=` to subtract the value of `enemyDamage` from `playerHealth` and store the result back in `playerHealth`. If we just typed `playerHealth - enemyDamage` without the assignment operator then the resulting value `4` would not be stored anywhere and no change would be assigned to `playerHealth`.

We then create a `bool` variable. The statement after the assignment operator can only be either `true` or `false` — either `playerHealth` is above or equal to 0 or it isn't.

What follows is a common part of programming: an `if` statement followed by an `else` statement. The question being answered in the parenthesis of the `if`-statement is responsible for deciding if the code flow enters the scope of the `if` or the `else`.

The **assignment operator** `=` is different from the **equality operator** `==`. The equality operator doesn't assign a new value to the left hand side, instead it checks that the value on the left side and the right side are the same. So this if-statement asks if the value stored in `isPlayerDead` is the same as `true`. And with the `playerHealth` above 0 the value of `isPlayerDead` is `false`. The if-statement then looks like this: `if(false == true)` and because these are not equal to each other we skip the if-scope and jump directly to the else-scope.

> [!NOTE]
> If we increased `enemyDamage` above or equal to the value of `playerHealth`, then the if-statement would evaluate to `true == true` and the code inside the if-scope would execute instead of the else-scope.

## Nesting

**Nesting** refers to putting a scope within another scope. We have already done this by putting an if-statement inside a function. Though that is the shallowest nesting possible. A lot of nesting should be scrutinized as there is often a more readable solution. Too much nesting will cause code flow that is hard to manage.

We can for example nest if-statements:

```cpp
void DealDamage(int damageAmount, bool isHardcore){
    playerHealth -= damageAmount;
    if(playerHealth <= 0){
        if(isHardcore){
            RetireHero();
            ReturnToTitleScreen();
        }
        else{
            ReturnToTitleScreen();
        }
    }
}
```

Here we have an if statement within another if statement. Still no problem to read and parse. Though you can imagine that with 1-2 more if-statements our code would begin to drift right at an alarming rate.

> [!NOTE]
> The sideways drift towards right is sometimes referred to as a "pyramid of death".

We can use a `return` to do an **early return** as well as ask the opposite question:

```cpp
void DealDamage(int damageAmount, bool isHardcore){
    playerHealth -= damageAmount;
    if(playerHealth > 0){
        return;
    }
    if(isHardcore){
        RetireHero();
    }
    ReturnToTitleScreen();
}
```

Look, we removed the nested if's. We also moved the `ReturnToTitleScreen()` call to outside of the if-statement as this was present in both the `if` and the `else` scope. By adding a `return` in the case of the player still being alive we can guarantee that the code afterwards is only executed if the if-statement was false (aka the player's health was in fact less than or equal to 0).

In this lecture we've looked at:

- Writing and using our own .h files
- How to `#include` standard library headers vs our own .h files
- Return types
- Variable types
- Assignment vs equality operators
- If/else statements
- Nesting and early returns