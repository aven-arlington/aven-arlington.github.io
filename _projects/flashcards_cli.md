---
layout: page
title: Flashcards Pt 1
date: 2024-01-16 00:00:00-0000
description: Simple CLI utility that quizzes a user on custom clue-answer pairs
img: assets/img/flashcards_cli_thumbnail.png
importance: 1
category: Flashcards
giscus_comments: true
giscus_repo: aven-arlington/flashcards-cli
toc:
  sidebar: left
---
## Topics Covered
- Rust modules
- Visibility and privacy

## TLDR
[Try it out in a sandbox...](#try-it-out)\
[Source code for this project post...](https://github.com/aven-arlington/flashcards-cli/tree/project-modules)\
[Completed Project...](https://github.com/aven-arlington/flashcards-cli)

## Background
For a brief time before I had the opportunity to study computer engineering, I was a machinist. The first thing my trades teacher had me build was a simple brass hammer. I learned a lot while creating this tool. Lathe fundamentals, milling machine operation, and how to start a project over after a mistake. By the end of the project I had grown as a machinist but I also had a nice brass hammer that I still have and continue to use. Whenever possible, I try to take a similar approach to software and hardware development. When I want to experiment with a new language I try to build something useful with it and learn along the way. This approach has served me well over the years and left me with a collection of useful utilities. Well some are more useful than others. 

Along those lines, I recently wanted to switch to Neovim as my code editor. A few of my most admired colleagues swore by Vim/Neovim and watching them absolutely fly around a code base to debug an issue amazed me to say the least. If that wasn't incentive enough, I would also get super frustrated every time I set up a new embedded project or Linux machine and needed to make a simple edit to a config file. What should take seconds would end up taking 10 minutes trying to figure out how to move the cursor, enter edit mode, and make a change. I didn't do it frequently enough to remember the commands but it was often enough to frustrate me to no end. The inspiration for this flashcard utility was born.

My requirements for the utility were pretty simple. I needed something that would quiz me on semi-random Neovim shortcuts. It needs to be something I can do quickly for 5 minutes during downtime so a CLI interface is perfect. I needed the clue-answer pairs to be configurable with a easy to edit file format like YAML. I wanted the difficult to ramp up as I warmed up and improved. 

I had learning requirements for the project as well. I wanted a better understanding of the Rust module system, code visibility, and privacy. I wanted to make a project whose complexity sat somewhere between a trivial example and a full blown commercial application.

## Let's Tackle the Problem
### Project Layout
While this project is relatively simple and could easily be contained in a single main.rs file, I prefer my projects to be broken down into easy to understand build blocks. Doing so helps not only helps me to remember how the code functions months (or even years) after I touched it last, but it can also serve as template to jump start other projects with shared or similar functionality. Since I enjoy jumping straight into the interesting part of a project, minimizing the time spent on project setup is valuable.

The first thing I want to do is think about the separation of concerns and how I can break up my project into building blocks. I know I want to have a simple main function for the root of the binary crate to facilitate stand-alone operation. I also know I want the main functionality of the code to be in a library so I can call it as a dependency should I need to sometime in the future in a different project. Thus to start out I need a minimum of a main.rs and a lib.rs file in the src directory.
```console
src
  ├── lib.rs
  └── main.rs
```

I want the application to be configurable and I want the configuration functionality to be separate from the main logic of the application. Similarly, the application functionality needs to siloed into it's own building block.

Rust utilizes a module system to break code into functional units, control access, and limit visibility. The Rust compiler will start by looking at the root of the crate for a unit of code to compile and expand it's search from there. One important thing to note is that Rust does not consider individual files to be units of code. Instead, units of code are explicitly defined with ```mod``` code blocks and a few simple rules on what files to search for them in.

[Summarized from The Rust Programming Language book](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html)
1. Crate root - Often a Rust package will contain both a library crate root and a binary crate root that defaults to src/lib.rs and src/main.rs respectively. Rust doesn't consider other files by themselves to be code units unless a ```mod``` keyword is found that explicitly tells the compiler to do so. Accordingly, if the crate roots (binary and/or library) do not contain ```mod``` declarations the compiler will ignore any other *.rs files in the directory.
2. Inline modules - Modules declared and defined in the same file with curly brackets. Example ```mod my_module { ...my_code... }```.
3. In the file src/module_name.rs - The compiler will look for the file name that matches the string after the ```mod``` declaration. For example if the compiler finds ```mod engine``` it will look for ```src/engine.rs```.
4. In the file src/module_name/mod.rs - The compiler will look for a ```mod.rs``` file in the directory with a name that matches the string after the ```mod``` declaration. For example the compiler will look for ```src/engine/mod.rs``` when it finds ```mod engine``` in the source.
5. Submodules - Submodules can be declared in any file with the exception of the crate root. The rules are basically the same as before but the tree structure is rooted in the parent module's directory.
    1. Inline
    2. File src/module_name/sub_module_name.rs
    3. File src/module_name/sub_module_name/mod.rs


For example, when ```cargo build``` is called to compile the application, a pre-compiler will first open up lib.rs because that is the crate root for the library by default. From there, if a ```mod``` declaration is found, the file tree will be searched according to the rules and replace the ```mod``` declaration with the code it finds. It does this recursively until all modules and submodules are placed inline and most importantly, it does this before running the compiler over anything. The same steps are then repeated for the binary crate root, main.rs.

Using this information, I can surmise that the application library crate root will need to declare ```mod config``` and ```mod engine``` to contain the respective code blocks. The ```engine.rs``` and ```config.rs``` files can then be added to the tree. 
```console
src
  ├── config.rs
  ├── engine.rs
  ├── lib.rs
  └── main.rs
```
```rust
// src/lib.rs
mod engine;
mod config;
```

The Flashcards utility/game can logically be broken up into the game engine itself, a representation of a deck of flashcards, and the representation of the flashcards themselves. The game logic can naturally go in the ```engine``` module located in the ```engine.rs``` file. The other two will be sub-modules of ```engine```. To create a submodule following the rules outlined above, I need a ```engine``` folder with both ```deck.rs``` and ```flashcard.rs``` contained within.
```console
src
  ├── engine
  │     ├── deck.rs
  │     └── flashcard.rs
  ├── config.rs
  ├── engine.rs
  ├── lib.rs
  └── main.rs
```
```rust
// src/engine.rs
mod deck;
mod flashcard;
```

### Visibility
I want to hide as much functionality from the outside world as possible and only expose the things that the ```main``` function needs. By default most things in Rust are private and can only be made visible to the outside world through explicit declarations.

The application will need a ```Config``` struct that can be used by both main and the engine to determine the applications configuration. To implement this I use the ```pub``` keyword before the struct name so that it will be visible from outside the ```config``` module. The implementation of ```Config``` also needs a single public function so that main can create a ```Config``` object so I add ```pub fn build``` to the mix. The other functions in ```config.rs``` will be hidden by default by simply omitting the ```pub``` keyword.
```rust
// config.rs
pub struct Config {...}
impl Config {
    pub fn build(){...}
    fn other_function(){...}
}
```

The previous step should make the public components visible to the library crate but they won't be visible to the binary crate. I add the ```pub``` keyword to the definition of the ```config``` module to make it part of the library's public API. I also re-export all public elements with ```pub use config::*;```. This enable the ```Config``` struct to be referenced anywhere in the library crate with ```crate::Config``` instead of a more fully qualified name which is much easier to read.
```rust
// src/lib.rs
mod engine;
pub mod config;
pub use config::*;
```

I perform similar modifications to ```engine.rs``` and declare the ```flashcard``` module as public so that ```config``` module can have access to the ```Flashcard``` type when parsing the configuration file. The ```deck``` module is purely internal logic so it should remain the default of private.
```rust
// engine.rs
mod deck;
pub mod flashcard;
```
```rust
// lib.rs
mod engine;
pub use engine::run;
pub use engine::flashcard::*;
pub mod config;
pub use config::*;
```

Now that the basic layout is thought out and implemented I can block out the functionality. I want to keep the main theme of this post on modularity and visibility so I won't spend much time on it here. Just know that I added some functions and had a working utility before the next step but the following instructions can be used at any time.

There is a utility module for cargo that makes visualizing the module tree a breeze. To verify the structure of your library crate and visibility settings:
1. Install the cargo-modules utility with ```cargo install cargo-modules```
2. Generate the module tree with ```cargo modules structure --lib```

Which generates the following output for the finished project:
```console
C:\repos\flashcards_cli> cargo modules structure --lib

crate flashcards_cli
├── mod config: pub
│   └── struct Config: pub
│       ├── fn build: pub
│       └── fn check_default_path: pub(self)
└── mod engine: pub(crate)
    ├── mod deck: pub(self)
    │   └── struct Deck: pub
    │       ├── fn draw_hand: pub
    │       ├── fn hand_count: pub
    │       ├── fn increase_level_pool: pub
    │       ├── fn new: pub
    │       └── fn next_card: pub
    ├── mod flashcard: pub
    │   └── struct FlashCard: pub
    │       └── fn new: pub
    ├── fn get_input: pub(self)
    ├── fn run: pub
    └── fn show_card: pub(self)
```

## Try It Out
I have created a CodeSandbox which follows the content so far. Once the CodeSandbox is opened, you have the option of signing in and making edits such as adding modules or modifying visibility settings. Modifications are automatically saved during context switches from file-to-file or when the Ctrl+s keyboard shortcut is used. Once a change to any of the source files is detected, the module structure tree will be automatically regenerated.

[![Edit flashcards-project-modules](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/p/devbox/flashcards-project-modules-dq8zwx?embed=1&file=%2Fsrc%2Fmain.rs)


## Summary
- The compiler looks for modules starting at the crate root.
- Modules are found in the following order:
  - Inline.
  - In src/module_name.rs.
  - In src/module_name/mod.rs.
- Sub-modules are found in the following order:
  - Inline.
  - In src/module_name/sub_module_name.rs.
  - In src/module_name/sub_module_name/mod.rs.
- Most things in Rust have private visibility by default.
- Items marked with ```pub``` are visible outside of the containing module
- Re-exporting items from a top level module makes them visible to other sub modules and the outside world.

## Additional Resources
- [Defining Modules to Control Scope and Privacy](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html)
- [Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
- [Clear explanation of Rust’s module system](https://www.sheshbabu.com/posts/rust-module-system/)
- [Rust Modules - Explained Like I'm 5](https://www.youtube.com/watch?v=969j0qnJGi8)
