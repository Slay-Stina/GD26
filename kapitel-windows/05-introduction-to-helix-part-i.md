# Introduction to the Helix Editor - Part I

Helix is a code editor, and unlike the commonly used Visual Studio it is not also a debugger or has integrated build systems. Helix, compared to Visual Studio is:

- **Light-weight** (memory footprint)
- **Fast** (starts almost instantly)
- **Tailored for use with VIM-style motions and modes** — made to not use the mouse at all
- **Terminal-based**

When we dive into programming applications in SDL3 we will be using Helix to edit our code and Visual Studio to debug it.

> [!NOTE]
> Debugging in Visual Studio is another lecture.

If you have computer experience then you will quickly find that Helix is unlike any other software you've used. Just writing in it, before knowing how it works will feel alien and strange. You may eventually decide to move away from Helix and towards more mainstream and less opinionated editors. But for this course, you will be using the software that I use myself.

Helix uses a way of typing that was first introduced with the vi text editor in 1976. It utilizes sequential keystrokes to change text by leveraging different editor modes:

1. **Normal Mode**
2. **Insert Mode**
3. **Select Mode**

**Normal Mode** is used to move the caret around. The Caret is the point in your file where text will be added as you type. In Normal Mode the user can't add any text by typing. This is the part that is the most confusing to new users as pressing keys will move the caret around or enter other modes. When working in Helix we also get access to Helix-specific menus by using the correct keystrokes in Normal Mode.

**Insert Mode** — the keys are used to type text like in any editor. Pressing Escape will return us out of Insert Mode and back into Normal Mode.

**Select Mode** — we can select multiple pieces of text to be copied, moved and otherwise manipulated.

We enter Insert Mode using `i` or `a` or `o` or `O` (note how upper and lowercase are distinct from each other). We exit Insert Mode and Select Mode going back to Normal Mode by pressing Escape.

To code in both Visual Studio and Helix in a fashion that is at all acceptable we need to ensure that we have an English keyboard layout selected on Windows. This means that ÅÄÖ are no longer available. The reason is that a lot(!) of symbols we will be typing when programming are very easy to access on the English layout and horrible to type on a Swedish keyboard layout.

Inside Windows Settings we can add another keyboard layout by adding another preferred language. I added English (United States) as my secondary language and English (Swedish) as my primary. You will need to do the same. Swapping between the two is done by pressing `WIN+Space`.

We will be pressing Escape a lot, and because the Escape key is so far away from the keyboard's home row we will download **Microsoft Powertoys** to rebind our keyboard.

> [Powertoys](https://learn.microsoft.com/en-us/windows/powertoys/)

Once you've installed Powertoys you will have access to a bunch of (more or less) useful features. We will go to the **Keyboard Manager** and add two remaps:

1. Caps Lock → Escape
2. Escape → Caps Lock

Now we turn Caps ON and OFF using Escape and exit Insert Mode and Select Mode using Caps Lock. This will, like many new things, feel strange at first. But this remapping is very common when using Helix or other VIM-style software. And now we're using our computer as developers not hobbyists, and that should naturally come with changes to how we use our hardware.

> [!NOTE]
> I suggest unplugging your mouse when learning Helix if you can't help but reach for it all the time.

Helix and VIM style systems are so notorious that there are even a slew of memes relating to the fact that people don't know how to exit them. ("how to quit vim" on Google will yield a number of results). So let's learn how to close down Helix. This is done from the **Helix Command Line**, which we access by typing `:`. Once we have done so, we can type a massive number of functions.

Quitting Helix from the command line is done by typing `q` and pressing Enter. You can also spell out `quit` or typing a single `q` then using Tab to cycle between the different quit commands.

You deserve a treat. Reopen Helix (if you closed it previously), write some sample code, just practice the `main()` function syntax. Once happy with a few lines, open the Command Line and type `theme` followed by a space, then cycle through the different color themes. Once you have found one you like, remember its name because we will make sure that each time you open your cool light-weight ultra-fast editor you will be greeted by it.

Open the command line, type `config`, tab to `config-open` and press Enter. Go into Insert Mode and type:

```
theme = "your-chosen-theme"
```

Then we need to save the changes to this file. But out-of-the-box Helix saves using the `write` command rather than our usual `Ctrl+S`. So go ahead and open the command line again, type `w` and press Enter.

> [!NOTE]
> You can also type `write` or even `write-quit-all` to save all your files at once then quit Helix, or use the short form `wqa`.

We will be working with C++ files, and it would be very nice to catch errors before we try and compile. Luckily we can do just that. Once we compile, it's clang that finds and spits out any errors. But using what is known as a **language server** we can run background processes that look at and understand our code. This info is then given to Helix so it can display red errors for us.

In PowerShell we can run:

```powershell
hx --health
```

This will list all programming languages as well as any known language servers. Find the `cpp` row. If `clangd` (yes with a `d`) is not set as the language server then we need to download it and add it to our environment variable paths.

> [!NOTE]
> `clangd` is not the same as `clang`. It actually stands for **clang daemon**. A daemon is a silent background process that just listens to requests that come in then shuts down when not needed anymore. This specific language server daemon is a repackaged part of clang that editors can talk to.

Clangd can be downloaded from: [Clangd](https://clangd.llvm.org/)

Once downloaded, find the binary folder (`bin`). Inside there should be a `clangd.exe`. Add this folder's path to your system environment variable `PATH`.

Once we have clangd up and running, it runs in the background each time we open a `.h` or `.cpp` file. Then it lets us:

- **A)** Get diagnostics inside Helix (red underlines, error messages)
- **B)** Use `SPACE+A` when our caret is over a part of our code with an error and clangd will offer fixes