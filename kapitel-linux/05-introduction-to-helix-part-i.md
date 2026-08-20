# Introduction to Helix - Part I (Linux)

> **Linux:** This chapter is adapted for Linux.

Helix is a code editor, and unlike a full IDE it is not also a debugger or has integrated build systems. Compared to an IDE, Helix is:

- **Light-weight** (memory footprint)
- **Fast** (starts almost instantly)
- **Tailored for use with VIM-style motions and modes** -- made to not use the mouse at all
- **Terminal-based**

When we dive into programming applications in SDL3 we will be using Helix to edit our code.

> [!NOTE]
> Debugging is another lecture.

If you have computer experience then you will quickly find that Helix is unlike any other software you've used. Just writing in it, before knowing how it works will feel alien and strange. You may eventually decide to move away from Helix and towards more mainstream and less opinionated editors. But for this course, you will be using the software that I use myself.

Helix uses a way of typing that was first introduced with the vi text editor in 1976. It utilizes sequential keystrokes to change text by leveraging different editor modes:

1. **Normal Mode**
2. **Insert Mode**
3. **Visual Mode**
4. **Command Mode**

**Normal Mode** is used to move the caret around. The Caret is the point in your file where text will be added as you type. In Normal Mode the user can't add any text by typing. This is the part that is the most confusing to new users as pressing keys will move the caret around or enter other modes. Arrow keys work in Helix by default. When working in Helix we also get access to editor-specific commands by using the correct keystrokes in Normal Mode.

**Insert Mode** -- the keys are used to type text like in any editor. Pressing Escape will return us out of Insert Mode and back into Normal Mode.

**Visual Mode** -- we can select multiple pieces of text to be copied, moved and otherwise manipulated.

**Command Mode** -- accessed by typing `:` in Normal Mode. Here we type commands like `:q` to quit, `:w` to save, `:e filename` to open a file.

We enter Insert Mode using `i` or `a` or `o` or `O` (note how upper and lowercase are distinct from each other). We exit Insert Mode, Visual Mode, and Command Mode going back to Normal Mode by pressing Escape.

> [!NOTE]
> On Linux, keyboard layout is handled by your system. For programming, an English (US) layout is recommended for easy access to symbols like `{}`, `[]`, `|`, `~`, etc.

We will be pressing Escape a lot, and because the Escape key is so far away from the keyboard's home row we can remap Caps Lock to Escape. This is done on Linux via your desktop environment or window manager. For example, on systems using X11:

```bash
setxkbmap -option caps:escape
```

On Wayland compositors, this is usually configured in the compositor's config file. Search for "remap caps lock to escape" + your desktop environment to find the right method.

Now we turn Caps ON and OFF using Escape and exit Insert Mode and Visual Mode using Caps Lock. This will, like many new things, feel strange at first. But this remapping is very common when using Helix or other VIM-style software. And now we're using our computer as developers not hobbyists, and that should naturally come with changes to how we use our hardware.

> [!NOTE]
> I suggest unplugging your mouse when learning Helix if you can't help but reach for it all the time.

Helix and VIM style systems are so notorious that there are even a slew of memes relating to the fact that people don't know how to exit them. ("how to quit vim" on Google will yield a number of results). So let's learn how to close down Helix. This is done from the **Command Mode**, which we access by typing `:`. Once we have done so, we can type a massive number of commands.

Quitting Helix from the command line is done by typing `q` and pressing Enter. You can also type `quit` or `wq` (write and quit) or `q!` (quit without saving).

You deserve a treat. Reopen Helix (if you closed it previously), write some sample code, just practice the `main()` function syntax. Once happy with a few lines, open the Command Mode and type `:colorscheme` followed by a space, then Tab to cycle through the different color themes. Once you have found one you like, remember its name because we will make sure that each time you open your cool light-weight ultra-fast editor you will be greeted by it.

Open the command line, type `:e $MYVIMRC` to open your config. Go into Insert Mode and type:

```vim
colorscheme your-chosen-theme
```

Then save with `:w` and quit with `:q`.

> [!NOTE]
> You can also type `:wq` to save and quit in one command, or `:wqa` to save all files and quit.

We will be working with C++ files, and it would be very nice to catch errors before we try and compile. Luckily we can do just that. Once we compile, it's clang that finds and spits out any errors. But using what is known as a **language server** we can run background processes that look at and understand our code. This info is then given to Helix so it can display red errors for us.

In bash we can run:

```bash
hx --headless -c "LspInfo" -c "q"
```

But more directly, we can check if clangd is available:

```bash
clangd --version
```

> [!NOTE]
> `clangd` is not the same as `clang`. It actually stands for **clang daemon**. A daemon is a silent background process that just listens to requests that come in then shuts down when not needed anymore. This specific language server daemon is a repackaged part of clang that editors can talk to.

Install clangd using your package manager:

```bash
# Debian/Ubuntu
sudo apt install clangd

# Fedora
sudo dnf install clang-tools-extra

# Arch Linux
sudo pacman -S clang
```

The `clang` package on Arch includes both `clang` and `clangd` -- no manual path configuration needed.

Once we have clangd up and running, it runs in the background each time we open a `.h` or `.cpp` file. Then it lets us:

- **A)** Get diagnostics inside Helix (red underlines, error messages)
- **B)** Use `K` (in Normal Mode) to hover over a symbol and see documentation, and use the built-in LSP integration to see fixes

> [!NOTE]
> If you use a distribution like LazyVim, you get telescope, autocompletion, and LSP diagnostics out of the box. For heavier projects, you can also use an IDE with built-in clangd integration.