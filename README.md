# debug.rb

This library provides debugging functionality to Ruby.

This debug.rb is replacement of traditional lib/debug.rb standard library which is implemented by `set_trace_func`.
New debug.rb has several advantages:

* Fast: No performance penalty on non-stepping mode and non-breakpoints.
* Remote debugging: Support remote debugging natively.
  * UNIX domain socket
  * TCP/IP
  * VSCode/DAP integration (TODO)
* Extensible: application can introduce debugging support with several methods
  * By `rdbg` command
  * By loading libraries with `-r` command line option
  * By calling Ruby's method explicitly
* Misc
  * Support threads (almost done) and ractors (TODO).
  * Support suspending and entering to the console debugging with `Ctrl-C` at most of timing.
  * Show parameters on backtrace command.

# Installation

```
$ gem install debug --pre
```

or specify `-Ipath/to/debug/lib` in `RUBYOPT` or each ruby command-line option, especially for debug this gem development.

# How to use

## Invoke with debugger

You can run ruby program on debugger with the local debug console or the remote debug console.

* (a) Run a ruby program with the local debug console
* (b) Run a ruby program with the remote debug console by opening a network port
  * (b-1) Open with UNIX domain socket
  * (b-2) Open with TCP/IP port

(b-1) is useful when you want to use debugging features after running the program.
(b-2) is also useful when you don't have a ssh access for the Ruby process.

To use debugging feature, you can have 3 ways.

* (1) Use `rdbg` command
* (2) Use `ruby -r debug...` command line option
* (3) Write `require 'debug...'` in .rb files

### Local debug console

```
# (1) Use `rdbg` command
$ rdbg target.rb
$ rdbg -- -r foo -e expr # -- is required to make clear rdbg options and ruby's options

# (2) Use `-r debug/run` command line option

$ ruby -r debug/run target.rb

# (3) Write `require 'debug...' in .rb files

$ cat target.rb
require 'debug/run' # start the debug console
...

# or

$ cat target.rb
require 'debug/session'  # introduce the functionality
DEBUGGER__.console       # and start the debug console

$ ruby target.rb
```

When you run the program with the debug console, you will see the debug console prompt `(rdbg)`.
The debuggee program (`target.rb`) is suspended at the beginning of `target.rb`.

You can type any debugger's command described bellow. "c" or "continue" resume the debuggee program.
You can suspend the debuggee program and show the debug console with `Ctrl-C`.

The following example shows simple usage of the debug console. You can show the all variables

```
$ rdbg  ~/src/rb/target.rb

[1, 5] in /home/ko1/src/rb/target.rb
=>    1| a = 1
      2| b = 2
      3| c = 3
      4| p [a + b + c]
      5|
--> #0  /home/ko1/src/rb/target.rb:1:in `<main>'

(rdbg) info                                             # Show all local variables
 %self => main
 a => nil
 b => nil
 c => nil

(rdbg) p a                                              # Same as p(a)
=> nil

(rdbg) s                                                # Step in ("s" is a short name of "step")

[1, 5] in /home/ko1/src/rb/target.rb
      1| a = 1
=>    2| b = 2
      3| c = 3
      4| p [a + b + c]
      5|
--> #0  /home/ko1/src/rb/target.rb:2:in `<main>'

(rdbg) <Enter>                                          # Repeat the last command ("step")

[1, 5] in /home/ko1/src/rb/target.rb
      1| a = 1
      2| b = 2
=>    3| c = 3
      4| p [a + b + c]
      5|
--> #0  /home/ko1/src/rb/target.rb:3:in `<main>'

(rdbg)                                                  # Repeat the last command ("step")

[1, 5] in /home/ko1/src/rb/target.rb
      1| a = 1
      2| b = 2
      3| c = 3
=>    4| p [a + b + c]
      5|
--> #0  /home/ko1/src/rb/target.rb:4:in `<main>'

(rdbg) info                                             # Show all local variables
 %self => main
 a => 1
 b => 2
 c => 3

(rdbg) c                                                # Continue the program ("c" is a short name of "continue")
[6]
```

### Remote debug (1) UNIX domain socket

```
# (1) Use `rdbg` command
$ rdbg --open target.rb # or rdbg -O target.rb for shorthand
Debugger can attach via UNIX domain socket (/home/ko1/.ruby-debug-sock/ruby-debug-ko1-5042)
...

# (2) Use `-r debug/open` command line option

$ ruby -r debug/open target.rb
Debugger can attach via UNIX domain socket (/home/ko1/.ruby-debug-sock/ruby-debug-ko1-5042)
...

# (3) Write `require 'debug/open' in .rb files
$ cat target.rb
require 'debug/open' # open the debugger entry point by UNIX domain socket.
...

# or

$ cat target.rb
require 'debug/server' # introduce remote debugging feature
DEBUGGER__.open        # open the debugger entry point by UNIX domain socket.
# or DEBUGGER__.open_unix to specify UNIX domain socket.

$ ruby target.rb
Debugger can attach via UNIX domain socket (/home/ko1/.ruby-debug-sock/ruby-debug-ko1-5042)
...
```

It runs target.rb and accept debugger connection within UNIX domain socket.
The debuggee process waits for debugger connection at the beginning of `target.rb` like that:

```
$ rdbg -O ~/src/rb/target.rb
DEBUGGER: Debugger can attach via UNIX domain socket (/home/ko1/.ruby-debug-sock/ruby-debug-ko1-29828)
DEBUGGER: wait for debugger connection...
```

You can attach the program with the following command:

```
$ rdbg --attach # or rdbg -A for shorthand

[1, 4] in /home/ko1/src/rb/target.rb
      1| (1..).each do |i|
=>    2|   sleep 0.5
      3|   p i
      4| end
--> #0  [C] /home/ko1/src/rb/target.rb:2:in `sleep'
    #1  /home/ko1/src/rb/target.rb:2:in `block in <main>' {|i=17|}
    #2  [C] /home/ko1/src/rb/target.rb:1:in `each'
    # and 1 frames (use `bt' command for all frames)

(rdb)
```

and you can input any debug commands. `c` (or `continue`) continues the debuggee process.

You can detach the debugger from the debugger process with `quit` command.
You can re-connect to the debuggee process by `rdbg -A` command again, and the debuggee process suspends the execution (and debugger can input any debug commands).

If you don't want to stop the debuggee process at the beginning of debuggee process (`target.rb`), you can use the following to specify "non-stop" option.

* Use `rdbg -n` option
* Set the environment variable `RUBY_DEBUG_NONSTOP=1`

If you are running multiple debuggee processes, the attach command (`rdbg -A`) shows the options like that:

```
$ rdbg --attach
Please select a debug session:
  ruby-debug-ko1-19638
  ruby-debug-ko1-19603
```

and you need to specify one (copy and paste the name):

```
$ rdbg --attach ruby-debug-ko1-19638
```

The socket file is located at
* `RUBY_DEBUG_SOCK_DIR` environment variable if available.
* `XDG_RUNTIME_DIR` environment variable if available.
* `$HOME/.ruby-debug-sock` if `$HOME` is available.

### Remote debug (2) TCP/IP

You can open the TCP/IP port instead of using UNIX domain socket.

```
# (1) Use `rdbg` command
$ rdbg -O --port=12345 target.rb
# or
$ rdbg --open --port=12345 target.rb
Debugger can attach via TCP/IP (localhost:12345)
...

# (2) Use `-r debug/open` command line option

$ RUBY_DEBUG_PORT=12345 ruby -r debug/open target.rb
Debugger can attach via TCP/IP (localhost:12345)
...

# (3) Write `require 'debug/open' in .rb files
$ cat target.rb
require 'debug/open' # open the debugger entry point.
...

# and run with environment variable RUBY_DEBUG_PORT
$ RUBY_DEBUG_PORT=12345 ruby target.rb
Debugger can attach via TCP/IP (localhost:12345)
...

# or

$ cat target.rb
require 'debug/server' # introduce remote debugging feature
DEBUGGER__.open(port: 12345)
# or DEBUGGER__.open_tcp(port: 12345)

$ ruby target.rb
Debugger can attach via TCP/IP (localhost:12345)
...
```

You can also specify the host with the `RUBY_DEBUG_HOST` environment variable. And also `DEBUGGER__.open` method accepts a `host:` keyword parameter. If the host is not given, `localhost` will be used.

To attach the debuggee process, specify the port number (and hostname if needed) for the `rdbg --attach` (or `rdbg -A`) command.

```
$ rdbg --attach 12345
$ rdbg --attach hostname 12345
```

### Initial scripts

If there are `.rdbgrc` files are there at the current directory and the home directory, files are loaded as initial scripts which contains debugger commands. `RUBY_DEBUG_INIT_SCRIPT` environment variable can specify the initial script file.

Initial scripts are evaluated at the first suspend timing (generally, it is the beginning of the target script). For example, you can set break points with `break file:123`.

If there are `.rdbgrc.rb` files at the current directory and the home directory, files are loaded as a ruby script at the initializing timing.

### Environment variables

You can control debuggee's behavior with environment variables:

* `RUBY_DEBUG_NONSTOP`: 1 for nonstop at the beginning of program.
* `RUBY_DEBUG_INIT_SCRIPT`: Initial script path loaded at the first stop.
* `RUBY_DEBUG_COMMANDS`: Debug commands invoked at the first stop. Commands should be separated by ';;'.
* `RUBY_DEBUG_SHOW_SRC_LINES`: Show n lines source code on breakpoint (default: 10 lines).
* `RUBY_DEBUG_SHOW_FRAMES`: Show n frames on breakpoint (default: 2 frames).

* Remote debugging
  * `RUBY_DEBUG_PORT`: TCP/IP remote debugging: port to open.
  * `RUBY_DEBUG_HOST`: TCP/IP remote debugging: host (localhost if not given) to open.
  * `RUBY_DEBUG_SOCK_DIR`: UNIX Domain Socket remote debugging: socket directory to open.

## Debug command on the debug console

* `Enter` repeats the last command (useful when repeating `step`s).
* `Ctrl-D` is equal to `quit` command.
* [debug command compare sheet - Google Sheets](https://docs.google.com/spreadsheets/d/1TlmmUDsvwK4sSIyoMv-io52BUUz__R5wpu-ComXlsw0/edit?usp=sharing)

You can use the following debug commands. Each command should be written in 1 line.
The `[...]` notation means this part can be eliminate. For example, `s[tep]` means `s` or `step` are valid command. `ste` is not valid.
The `<...>` notation means the argument.

### Control flow

* `s[tep]`
  * Step in. Resume the program until next breakable point.
* `n[ext]`
  * Step over. Resume the program until next line.
* `fin[ish]`
  * Finish this frame. Resume the program until the current frame is finished.
* `c[ontinue]`
  * Resume the program.
* `q[uit]` or `Ctrl-D`
  * Finish debugger (with the debuggee process on non-remote debugging).
* `q[uit]!`
  * Same as q[uit] but without the confirmation prompt.
* `kill`
  * Stop the debuggee process with `Kernal#exit!`.
* `kill!`
  * Same as kill but without the confirmation prompt.

### Breakpoint

* `b[reak]`
  * Show all breakpoints.
* `b[reak] <line>`
  * Set breakpoint on `<line>` at the current frame's file.
* `b[reak] <file>:<line>` or `<file> <line>`
  * Set breakpoint on `<file>:<line>`.
* `b[reak] <class>#<name>`
   * Set breakpoint on the method `<class>#<name>`.
* `b[reak] <expr>.<name>`
   * Set breakpoint on the method `<expr>.<name>`.
* `b[reak] ... if <expr>`
  * break if `<expr>` is true at specified location.
* `b[reak] if <expr>`
  * break if `<expr>` is true at any lines.
  * Note that this feature is super slow.
* `catch <Error>`
  * Set breakpoint on raising `<Error>`.
* `watch <expr>`
  * Stop the execution when the result of <expr> is changed.
  * Note that this feature is super slow.
* `del[ete]`
  * delete all breakpoints.
* `del[ete] <bpnum>`
  * delete specified breakpoint.

### Information

* `bt` or `backtrace`
  * Show backtrace (frame) information.
* `l[ist]`
  * Show current frame's source code.
  * Next `list` command shows the successor lines.
* `l[ist] -`
  * Show predecessor lines as opposed to the `list` command.
* `l[ist] <start>` or `l[ist] <start>-<end>`
  * Show current frame's source code from the line <start> to <end> if given.
* `edit`
  * Open the current file on the editor (use `EDITOR` environment variable).
  * Note that edited file will not be reloaded.
* `edit <file>`
  * Open <file> on the editor.
* `i[nfo]`
  * Show information about the current frame (local variables)
  * It includes `self` as `%self` and a return value as `%return`.
* `i[nfo] <expr>`
  * Show information about the result of <expr>.
* `display`
  * Show display setting.
* `display <expr>`
  * Show the result of `<expr>` at every suspended timing.
* `undisplay`
  * Remove all display settings.
* `undisplay <displaynum>`
  * Remove a specified display setting.
* `trace [on|off]`
  * enable or disable line tracer.

### Frame control

* `f[rame]`
  * Show current frame.
* `f[rame] <framenum>`
  * Specify frame. Evaluation are run on this frame environment.
* `up`
  * Specify upper frame.
* `down`
  * Specify down frame.

### Evaluate

* `p <expr>`
  * Evaluate like `p <expr>` on the current frame.
* `pp <expr>`
  * Evaluate like `pp <expr>` on the current frame.
* `e[val] <expr>`
  * Evaluate `<expr>` on the current frame.
* `irb`
  * Invoke `irb` on the current frame.

### Thread control

* `th[read]`
  * Show all threads.
* `th[read] <thnum>`
  * Switch thread specified by `<thnum>`.

### Help

* `h[elp]`
  * Show help for all commands.
* `h[elp] <command>`
  * Show help for the given command.


## rdbg command help

```
exe/rdbg [options] -- [debuggee options]

Debug console mode:
    -n, --nonstop                    Do not stop at the beginning of the script.
    -e [COMMAND]                     execute debug command at the beginning of the script.
    -O, --open                       Start debuggee with opening the debugger port.
                                     If TCP/IP options are not given,
                                     a UNIX domain socket will be used.
        --port=[PORT]                Listening TCP/IP port
        --host=[HOST]                Listening TCP/IP host

  Debug console mode runs Ruby program with the debug console.

  rdbg target.rb foo bar                 starts like 'ruby target.rb foo bar'.
  rdbg -- -r foo -e bar                  starts like 'ruby -r foo -e bar'.
  rdbg -O target.rb foo bar              starts and accepts attaching with UNIX domain socket.
  rdbg -O --port 1234 target.rb foo bar  starts accepts attaching with TCP/IP localhost:1234.
  rdbg -O --port 1234 -- -r foo -e bar   starts accepts attaching with TCP/IP localhost:1234.

Attach mode:
    -A, --attach                     Attach to debuggee process.

  Attach mode attaches the remote debug console to the debuggee process.

  'rdbg -A' tries to connect via UNIX domain socket.
            If there are multiple processes are waiting for the
            debugger connection, list possible debuggee names.
  'rdbg -A path' tries to connect via UNIX domain socket with given path name.
  'rdbg -A port' tries to connect to localhost:port via TCP/IP.
  'rdbg -A host port' tries to connect to host:port via TCP/IP.

```

# Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/ruby/debug.