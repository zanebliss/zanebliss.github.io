---
layout: post
title: "Rusty Revisions: On Maintaining Git Revision History 🦀"
date: 2024-08-25 12:00:00 +0000
tags: rust
---

Have you ever worked in a repository where the maintainers cared about
revision history? I thought I did, until I started working on a new
codebase where my team put an emphasis on it. In source control, the default
merge strategy is [Rebase and
Merge](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/about-pull-request-merges#rebase-and-merge-your-commits),
meaning every commit in feature branches is recorded individually rather than as
a single merge commit.

This approach was partly inspired by the talk [A Branch in Time (A Story About
Revision
Histories)](https://tekin.co.uk/2019/02/a-talk-about-revision-histories), a
conference talk about the value of maintaining a git project's revision history.
When submitting code for review in the codebase, it is desirable for PRs to
consist of a series of small logical units of change that tell a story. It's
even better if each commit leaves the application in a deployable state with
passing tests.

Before joining this team, I hadn't used git in this way. Adjusting my workflow
to support maintaining revision history was a learning experience. In my early
branches, I found myself repeating some variation of the following steps:
1. `git rebase -i main` // Start an interactive rebase
1. Edit commit 1 of n
1. `git diff HEAD^ --name-only | xargs bundle exec rspec` // Get changed test
   files and pass them to a test runner
1. Address any test failures and amend the commit if necessary
1. Repeat n times until reaching HEAD

Although this made me more confident that my commits were "good," I struggled to
integrate it into my workflow. Anything beyond a few commits felt tedious and
error prone.

## Writing a CLI tool

I decided to write a tool to automate this part of my workflow _and_ because I
wanted to learn more about systems programming, git, and Linux. I considered
using C or Rust because either of these languages seemed like a natural choice
for the problem. I chose Rust because I was intrigued by the language, and
because I wanted to target MacOS and Linux easily.

My goal with this project was to decrease the amount of time it took to validate
my commits by an order of magnitude. I knew there would be a learning curve. At
the time, I didn't have much experience in systems programming. I hadn't had
exposure to concepts such as file descriptors, the I/O model, signals,
processes, threads, and more. Additionally, Rust would be very different from
the dynamically typed, garbage collected languages I was familiar with like
Ruby.

## Design principles

A few key design principles emerged early on.

_Non-invasive and isolated._ Existing uncommited work in the project would never
be at risk of being lost, corrupted, or broken. There wouldn't be any need to
checkout to another branch or create a "wip" commit to run the executable.

_Interruptable._ If there were long running test files or if the user
unintentionally started the program, it should be easy to interrupt the process.
Test programs like RSpec gracefully handle SIGINT and shutdown. The program
should continue this behavior.

_Transient._ When the program ended whether finished or interrupted, any
remaining and related resources (files, directories, processes) would be cleaned
up. Like [Leave No
Trace](https://www.nps.gov/articles/leave-no-trace-seven-principles.htm), but
for a filesystem.

I debated what to call the program for a while, but I eventually landed on
"Teva" (a nod to outdoor sandal brand). I'm an outdoorsy person, so I decided to
use an outdoor metaphor :-). Commits in your feature branch take readers on a
journey. Adventure sandals support and protect your feet each step (commit) of
the way. Cheesy, but I like it.

## Usage

Teva relies on a config file in the project's root directory to determine what
executable to use for tests, what type of file to pass the test runner, and if
there are any setup steps.

Below is a starter config file.

```toml
[test]
pattern = "" # File pattern for determining what file type to run tests for

[test.setup]
steps = [
    { name = "", command = "", args = [] }
    ...
]

[test.run]
command = "" # Executable for running tests
args = [] # Optional args
```

On starting, Teva creates a git worktree in `/tmp`. Git worktrees do not include
files in `.gitignore`, so `test.setup.steps` provides a way to run setup
commands for applications with "pre-test" steps.

For example, in a Rails app it may be necessary to run database migrations,
install frontend dependencies for sprockets, or symlink files required to run
the test suite.

```toml
[test]
pattern = "_spec.rb"
[test.setup]
steps = [
    { name = "symlink_foo", command = "ln", args = ["-s", "/home/zanebliss/foo", "./bar" },
    { name = "echo_hello", command = "echo", args = ["'hello'"],
    { name = "yarn", command = "yarn", args = ["install"] },
    { name = "rails db:migrate", command = "rails", args = ["RAILS_ENV='test'", "db:migrate"] }
]
[test.run]
command = "bundle"
args = ["exec", "rspec"]
```
_Example `.teva.toml` for a Rails app using RSpec and yarn_

_Note: shell expansions are not supported. You can use `./` for relative
directory paths, but will need to type full paths for something in the home
directory._

Invoke `teva` to run the program. Optionally, pass `--commit` or `-c` with a
SHA to start at a base commit that is not `HEAD+1` on `main`.

```bash
test-gem main % bin/teva -c 51ec94d
[teva] ⚙️ Setting up environment... Done ✔️
[teva] 6e01f31 add bar test (1 of 5)
[teva] Changed files: spec/test/bar_spec.rb
[teva] Running tests...

Bar
  works!

Finished in 0.00085 seconds (files took 0.04658 seconds to load)
1 example, 0 failures

[teva] 9046878 add baz test (2 of 5)
...
```

## Learnings

### Git worktrees

In order to be non-invasive and isolated, I had to determine how to run Teva
without affecting the main repository. I learned about [git
worktrees](https://git-scm.com/docs/git-worktree/en). Worktrees allowed me to
"... check out more than one branch at a time" and provided an isolated
environment for the Teva executable to run. I also chose the
[`/tmp`](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/tmp.html)
directory as the location on the filesystem for the Teva worktree since it
seemed an appropriate directory for temporary resources.

### Handling signals

In order to be interruptable, Teva needed to handle signals such as SIGINT
(commonly triggered with ctrl-c in a shell). I learned about different kinds of
signals, signal handling related to what I was trying to accomplish, and how to
implement handling Rust. The conference talk [RustConf 2023 - Beyond Ctrl-C the dark corners of Unix
signal handling](https://youtu.be/zhbkp_Fzqoo?si=IDB0CjLHWcrmO52S) was
particularly helpful in this area.

I chose to use the
[signal_hook](https://docs.rs/signal-hook/latest/signal_hook/) crate to manage
signal handling because of it's approachable API and documentation. I made Teva
interruptable by performing work in the main thread with a loop that terminates
itself when either SIGINT is received, or there are no more commits left to
checkout to.

Below are a few snippets.
```rust
fn main() -> Result<(), Box<dyn Error>> {
    // ...
   let term = Arc::new(AtomicBool::new(false));

   signal_hook::flag::register(signal_hook::consts::SIGINT, term.clone())?;

    while !term.load(Ordering::SeqCst) {
        match core::do_work(&client, config, term) {
            Err(err) => {
                println!("{} Failed with error: {err}", "[teva]".red());
                println!("{} Exiting...", "[teva]".red());
            }
            _ => {}
        }

        // break after first iteration because work is not performed in a continuous loop
        break;
    }
    // ...
}
```
_main.rs_

Elsewhere, before checking out to another commit:
```rust
if term.load(std::sync::atomic::Ordering::SeqCst) {
    break;
}
```
_core.rs_

### Processes, I/O and file descriptors

A core function of Teva has to do with orchestrating git processes and working
with I/O. Working with processes was integral, and I enjoyed using Rust's
`Command` struct with uses a builder pattern to call executables with options.

```rust
use std::process::Command;

let output = if cfg!(target_os = "windows") {
    Command::new("cmd")
        .args(["/C", "echo hello"])
        .output()
        .expect("failed to execute process")
} else {
    Command::new("sh")
        .arg("-c")
        .arg("echo hello")
        .output()
        .expect("failed to execute process")
};

let hello = output.stdout;
```
_Example from std Command [docs](https://doc.rust-lang.org/std/process/struct.Command.html)_

It was fun to think through things like deciding which standard [file
descriptors](https://en.wikipedia.org/wiki/File_descriptor) to read and write
to, what informational output looked like, and the flow of data through the
system.

One specific abstraction I enjoyed writing was a `Client` struct which was
responsible for holding initial state and providing convenience methods for
working with spawning processes and transforming I/O into useful data
structures. Check out a small snippet below.

```rust
pub struct Client {
    pub root_commit: String,
    pub commits: Vec<Commit>,
}

impl Client {
    // ...
    pub fn get_changed_files(&self, sha_1: &String, sha_2: &String) -> Vec<String> {
        let output = self.execute_command(vec!["diff", "--name-only", &sha_1, &sha_2]);

        self.transform_stream(output.stdout)
    }

    fn transform_stream(&self, stdout: Vec<u8>) -> Vec<String> {
        String::from_utf8(stdout)
            .unwrap()
            .split('\n')
            .filter_map(|line| {
                if line.is_empty() {
                    None
                } else {
                    Some(line.to_string())
                }
            })
            .collect()
    }

    fn execute_command(&self, args: Vec<&str>) -> Output {
        match Command::new("git").args(args).output() {
            Ok(output) => output,
            Err(err) => {
                eprint!("Error executing git command: {}", err);
                process::exit(1);
            }
        }
    }
}
```
_[git.rs](https://github.com/zanebliss/teva/blob/zb/feat/bubble-up-git-client-errors/src/git.rs)_

While the process and I/O centric approach was effective in the initial proof of
concept, it eventually felt cumbersome. If I were to do it again I'd rather find
another way to read from the filesystem and git's internals. 

A side effect of this is that integration testing was _interesting_. I ended up
using a combination of unit tests and snapshot tests with the
[insta](https://github.com/mitsuhiko/insta) crate. This project really
emphasized the usefulness of automated testing, as there were several instances
where I broke the program unintentionally before I had good test coverage.

## Closing thoughts

Overall, I'm happy with how everything went. I accomplished my goals and learned
a lot. I use the tool daily for my job and it has made me more productive. I
enjoy working with Rust. Rust's type system, error handling, and compiler are
great, and I have a new appreciation managing program memory.

While I'm planning on taking a break from Teva for a while, it would be
interesting to do the following next:

- Make Teva work in CI
- Support linting and code analysis
- Multiple test runners and files at the same time
- Eliminate reliance on processes and I/O for git

I've made a public repository on GitHub where you can look at the code and
download the binary. Check it out here: [https://github.com/zanebliss/teva](https://github.com/zanebliss/teva).
