# Introduction to Neovim - Part II (Linux)

> **Linux:** This chapter is adapted for Linux.

So far we've mostly been working in our `main.cpp` file. Though this is the most important file, holding our program's entry point, we will begin doing 2 things differently:

1. Work with .h files for function declarations
2. Split our program up into multiple .cpp files

Your editor (e.g. nvim) can help us create files and give us the ability to get an overview of our code by splitting our editor into multiple smaller **windows** (also called panes).

To split our view into two panes that sit side by side we press the following (when in **Normal Mode**):

- `:split` — split horizontally (top/bottom)
- `:vsplit` — split vertically (left/right)

Once we do this, the same file will be opened in both panes. If we modify one, the other will update instantly. Though useful when working with very long files, the far more common case is to have multiple different files open at once.

We can check what folder newly created files will get created in by pressing:

- `:pwd` — prints the current working directory

If we have set things up correctly (via our `dev` function) then this will print the path to our `src` folder at the bottom of your editor.

We can switch between our active panes by pressing:

- `Ctrl+w h` — move to the left pane
- `Ctrl+w l` — move to the right pane
- `Ctrl+w j` — move to the pane below
- `Ctrl+w k` — move to the pane above
- `Ctrl+w w` — cycle through panes

Once we are on a pane we can open a file:

- `:e filename` — open a file (relative to current working directory)
- `:tabnew filename` — open a file in a new tab

Your editor (e.g. nvim)'s tab-completion will help you find files. Type part of the filename and press Tab to cycle through matches.

If you attempt to open a file that doesn't exist, your editor (e.g. nvim) will create a new buffer with that name. It is not yet saved to disk.

> **NOTE:** When your editor (e.g. nvim) stores text before it has been written, it is being stored in something known as a **buffer**.

Once you press:

- `:w` — write (save)

You will have executed a write command. This will save the file to disk, creating it if necessary. This allows us to open and create files as needed, without leaving your editor (e.g. nvim) to use bash's `touch` command.

If we are done with a pane, we need to decide if we want to write its content to disk or if we want to discard the changes we've written.

If we try and close our pane using:

- `:q` — quit current window

Your editor (e.g. nvim) will warn us and nothing will happen (if we have unsaved changes).

We can combine our write and quit:

- `:wq` — write and quit

Once we have multiple windows open at once, with multiple files, we can write all of them to disk at once using:

- `:wa` — write-all

To delete a buffer (close a file without closing the window):

- `:bd` — buffer delete

To list all open buffers:

- `:ls` — list buffers, then `:b N` (where N is the buffer number) to switch to it

So let's say that we're just starting our workday, we want to begin devvin' in your editor (e.g. nvim) and start working on a new script called "bomb":

```
Super key
    press the Super (Windows) key to open the application menu
`alacritty` (or your terminal)
    type the name of your terminal
`enter`
    press enter to start the terminal
`dev project-name`
    type dev and then the name of our project to open it in your editor
`enter`
    press enter to execute the dev command opening your editor
`:e bomb.cpp`
    in your editor, our dev function opened our main.cpp. `:e` to open a new buffer for bomb.cpp
`enter`
    executes the open file command
`:vsplit`
    split the editor into two vertical panes
`Ctrl+w l`
    move to the right pane
`:e bomb.h`
    will open a new buffer for a new file
`enter`
    executes the open file command
    write some code in the .h file - an empty buffer won't save to disk
`Ctrl+w h`
    swap to the pane on the left (holding the buffer for bomb.cpp)
    write some code in the .cpp file - an empty buffer won't save to disk
`:wa`
    this is the write-all command
`enter`
    executes the write-all command, saving bomb.cpp and bomb.h to disk in the src folder
```

To list all open buffers:

- `:ls` — list buffers with their numbers
- `:b 2` — switch to buffer number 2
- `:bd 3` — delete buffer number 3

To browse files in the project directory:

- `:e .` — open netrw (built-in file browser) in the current directory
- Or if you have LazyVim or a similar distribution: `<leader>ff` to open telescope/fzf fuzzy finder

> **NOTE:** For heavier projects, you can also use an IDE with its own project-wide file navigation and refactoring tools.