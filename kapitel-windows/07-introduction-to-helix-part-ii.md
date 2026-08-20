# Introduction to the Helix Editor - Part II

So far we've mostly been working in our `main.cpp` file. Though this is the most important file, holding our program's entry point, we will begin doing 2 things differently:

1. Work with .h files for function declarations
2. Split our program up into multiple .cpp files

Helix can help us create files and give us the ability to get an overview of our code by splitting our editor into multiple smaller **panes**.

To split our view into two panes that sit side by side we press the following button sequence (when in **Normal Mode**):

- `space` — open the Helix menu
- `w` — open the window sub-menu
- `v` — hotkey to split the editor with a vertical right split

Once we do this, the same file will be opened in both panes. If we modify one, the other will update instantly. Though useful when working with very long files, the far more common case is to have multiple different files open at once.

We can check what folder newly created files will get created in by pressing the following sequence:

- `:` — open the Helix console
- `pwd` — shortcut for the command "show-directory" (`pwd` stands for "print working directory")

If we have set things up correctly in PowerShell then this will print the path to our `src` folder at the bottom of our Helix editor.

We can switch between our active pane by pressing:

- `space` — open Helix menu
- `w` — open window submenu
- `left/right arrow` — navigate to the left or right pane
- `h/l` key — can also be used to navigate to the left or right pane

Once we are on a pane we can press:

- `:`
- `o` (note the space after the `o`) — `o` is the shortcut for the `open` command
- `a-specific-file-name`
- `enter`

The Helix fuzzy-search will look for and list all the files that match what you put in as the file name. You can flip between them using Tab. With the file-name selected, pressing Enter will open it.

But the beautiful part is that if you attempt to open a file that doesn't exist, it will be added to the pane anyways, but it is not yet in your `src` folder. It lives only within Helix.

> [!NOTE]
> When Helix stores text before it has been written, it is being stored in something known as a **buffer**.

Once you press:

- `:`
- `w` — write

You will have executed a write command. This will save the file to disk, creating it if necessary. This allows us to open and create files as needed, without leaving Helix to use PowerShell's `New-Item` function.

If we are done with a pane, we need to decide if we want to write its content to disk or if we want to discard the changes we've written.

If we try and close our pane using:

- `:`
- `q` — quit

Helix will warn us and nothing will happen (if we have unsaved changes).

We can combine our write and quit (quit actually doesn't quit, it closes the current view/pane):

- `:`
- `wq` — write-quit

Once we have multiple panes open at once, with multiple files, we can write all of them to disk at once using:

- `:`
- `wa` — write-all

So let's say that we're just starting our workday, we want to begin devvin' in Helix and start working on a new script called "bomb":

```
`win`
    press the windows key to open the start menu
`pw`
    type `pw` to search for PowerShell
`enter`
    press enter to start PowerShell
`dev project-name`
    type dev and then the name of our project to open it in Helix
`enter`
    press enter to execute the dev command opening Helix
`:`
    in Helix, our dev function opened our main.cpp. use `:` to open the Helix command
`o bomb.cpp`
    this command will open a new buffer for a file we call bomb.cpp (not yet saved to disk)
`enter`
    executes the open file command
`space`
    open the Helix menu
`w`
    opens the Helix window submenu
`v`
    split the editor into two panes
`:`
    open the Helix command line again
`o bomb.h`
    will open a new buffer for a new file
`enter`
    executes the open file command
    write some code in the .h file - an empty buffer won't save to disk
`space`
    open the Helix menu
`w`
    open the Helix window submenu
`h`
    swap to the pane on the left (holding the buffer for bomb.cpp)
    write some code in the .cpp file - an empty buffer won't save to disk
`:`
    open the Helix command line
`wa`
    this is the write-all command
`enter`
    executes the write-all command, saving bomb.cpp and bomb.h to disk in the src folder
```

To open and look through the currently active buffers we can press:

- `space`
- `b`

Then step through them using Tab and select which one to open with Enter.

To open and look through all files in the project we can press:

- `space`
- `f`

Then step through them using Tab and select which one to open with Enter.