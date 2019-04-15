# Terminal curses

RubyKaigi 2019




Shugo Maeda

Network Applied Communication Laboratory Ltd.

## Self introduction

* Shugo Maeda
* Ruby committer
* Director at NaCl Ltd.
* Secretary General at Ruby Association

## Products

* Textbringer: Emacs-like text editor
* Mournmail: Message User Agent on Textbringer

## Terminal curses?

* terminal
    * predicted to lead to death, especially slowly;
      incurable
* curse
    * A solemn utterance intended to invoke a
      supernatural power to inflict harm or punishment
      on someone or something

(Quoted from Oxford Dictionaries)

## Topics

* Basics of terminals
* Terminal programming with Ruby
* curses and curses.gem

## Target platforms

* GNU/Linux, unless otherwise explictly stated
* Most topics apply to other Unix-like platforms
* A few topics on Windows

## What is a terminal?

![VT100](DEC_VT100_terminal.jpg)

## Real text terminals

* Consist of a screen and keyboard
* Communicate with a remote computer called "the host"

## Terminal emulators

* Emulate a text terminal on a PC, etc.
* e.g., Linux console, xterm, mlterm, Windows console,
  screen, tmux, etc.

## Why use terminals?

* Highly abstract interface, which brings
    * Good operability
    * Compatibility
    * Low cost
    * Remote computing
* You look busy at work

## Interface of terminals

* User Interface
* Programming Interface

## User Interface

* Screen and keyboard

## CUI and TUI

* CUI: Character User Interface
    * CLI: Command Line Interface
    * line oriented
    * e.g., shells, line editors
* TUI: Text User Interface
    * Use the entire screen
    * e.g., screen editors

## Programming Interface

* File I/O

## STDIN, STDOUT, STDERR

```ruby
p File.readlink("/proc/self/fd/#{STDIN.fileno}")
# For FreeBSD
require 'fiddle/import'
module M
  extend Fiddle::Importer; dlload "libc.so.7"
  extern('char *fdevname(int)')
end
p "/dev/#{M.fdevname(STDIN.fileno)}"
# tty(1) is specified by POSIX
```

## Terminal devices

* /dev/tty{1..63}: virtual consoles
   * tty stands for teletypewriter
* /dev/ttyS{0..}: serial ports (serial consoles)
* /dev/pts/{0..}: pty slave devices

## Special terminal devices

* /dev/tty0: current virtual console
* /dev/tty: controlling terminal

## The controlling terminal

* Each process belongs to a process group
* Each process group belongs to a session
* Each session can have a controlling terminal

## /dev/tty

```ruby
open("/dev/tty")
# Windows has "con", which stands for console
```

## Detach the controlling terminal

```ruby
fork do
  Process.setsid
  open("/dev/tty") #=> Errno::ENXIO
end
# Or use ioctl(TIOCNOTTY)
open("/dev/tty) { |f| f.ioctl(0x5422) }
```

## pty

* Pseudo terminal
* A pair of virtual devices called master & slave
    * /dev/ptmx: master clone device
    * /dev/pts/*: slave devices
* Used to implement ssh, telnet, terminal emulators, etc.
* ConPTY is available on recent Windows 10

## pty master & slave

```
+----------------+  fork/exec    +---------------+
| parent process |-------------->| child process |
+----------------+               +---------------+
        |                                |
        |                                |
+----------------+               +---------------+
|  pty master    |---------------|   pty slave   |
+----------------+               +---------------+
```

## Another use case of pty

* Fool programs
    * which change behavior when it's connected to
      a terminal
        * C's stdout is line buffered
        * Ruby prints backtrace in reverse order
    * which open /dev/tty
        * get a password from the controlling termianal

## PTY

```ruby
require "pty"
master, slave = PTY.open
read, write = IO.pipe
pid = spawn("factor", in: read, out: slave)
read.close
slave.close
write.puts "42"
p master.gets # "42: 2 3 7\r\n" expected
              # but deadlock on Debian 9.3
```

## Ruby's bug?

* It works on FreeBSD 11.2 as expected
* PTY.spawn works fine on Debian 9.3

## factor.c in GNU coreutils 8.26

```c
if (line_buffered == -1)
  line_buffered = isatty (STDIN_FILENO);
```

## Reported at https://bugs.gnu.org/35046

```c
--- a/src/factor.c
+++ b/src/factor.c
@@ -2403,7 +2403,7 @@ lbuf_putc (char c)
       /* Provide immediate output for interactive input.  */
       static int line_buffered = -1;
       if (line_buffered == -1)
-        line_buffered = isatty (STDIN_FILENO);
+        line_buffered = isatty (STDOUT_FILENO);
       if (line_buffered)
         lbuf_flush ();
       else if (buffered >= FACTOR_PIPE_BUF)
```

## Merged with modification

```c
if (line_buffered == -1)
  line_buffered = isatty (STDIN_FILENO) || isatty (STDOUT_FILENO);
```

## Use cases to check standard input

```
$ factor | sed -u 's/.*: *//'
```

## Control characters

* Characters to control terminals
* Caret notification
    *  H 0b01001000 0x48
    * ^H 0b00001000 0x08
* Control character literals in Ruby
    * "\C-h" represents ^H
    * Emacs compatible

## Example

```ruby
puts "I love Perl\C-h\C-h\C-h\C-hRuby"
```

## ANSI/VT100 escape sequences

```ruby
# Move cursor
print "\e[1;1H"
# Print colored output
puts "\e[31mRed Ruby\e[0m"
```

## Sixel

* Six pixels
* Bitmap graphics format supported by DEC terminals

## Example of Sixel

```ruby
print <<~EOF
\ePq
#0;2;0;0;0#1;2;100;100;0#2;2;0;100;0
#1~~@@vv@@~~@@~~$
#2??}}GG}}??}}??-
#1!14@
\e\\
EOF
```

## termcap/terminfo

* Database of terminal capabilities

## Input processing

* canonical mode
    * line oriented
    * data are accumulated in a line editing buffer
* non-canonical mode
    * byte oriented
    * data are accumulated in a buffer with timers

## Terminal line discipline

```
+------------+    +-----------------+    +------------+
| read/write |<-->| line discipline |<-->| tty driver |
+------------+    +-----------------|    +------------+
                     line editing
                       echoing                   
                   CR/LF conversion
                         etc.
```

## Special input characters

* ^D: end of file
* ^C: interrupt (SIGINT)
* ^\: quit (SIGQUIT)
* ^Z: suspend (SIGTSTP)
* ^S: stop output
* ^Q: restart output
* ^V: literal next character

## io/console

* A standard library of Ruby
* Provides additional methods for IO

## IO#noecho

```ruby
require "io/console"
password = STDIN.noecho {
  # no echo back
  STDIN.gets.chomp
}
```

## IO#raw

```ruby
STDIN.raw do
  # non-canonical mode & no echo back
  # special characters are not interpreted
  p STDIN.getc
end
```

## IO.console

```ruby
f = IO.console #=> #<File:/dev/tty> or #<File:con$>
```

## curses

* Terminal control library
* Pun on "cursor optimization"
* ncurses: new curses
* PDCurses: public domain curses
    * Windows support

## curses.gem

* Formerly part of the Ruby standard library
* Removed in Ruby 2.1
* I'm the original author
* Maintained by me and Eric Hodel

## Hello world

```ruby
require "curses"
Curses.init_screen
Curses.setpos(4, 10)
Curses.addstr("Hello, world")
Curses.setpos(5, 10)
Curses.addstr("Press any key: ")
Curses.get_char
Curses.close_screen
```

## Terminal modes

```ruby
Curses.raw      # similar to STDIN.raw, but echoing and
                # CR/LF conversion are not disabled
Curses.noraw    # back to cooked mode
Curses.cbreak   # similar to Curses.raw, but
                # special characters are interpreted
Curses.nocbreak # back to cooked mode
```

## Other configurations

```ruby
Curses.noecho              # no echo
Curses.nonl                # no CR/LF conversion
Curses.stdscr.keypad(true) # enable keypad
```

## Read characters

```ruby
p Curses.get_char #=> "a"
p Curses.get_char #=> "あ"
p Curses.get_char #=> 265 (Curses::KEY_F1)
p Curses.get_char #=> nil (EOF)
```

## Non blocking read

```ruby
Curses.stdscr.nodelay = true
p Curses.get_char #=> "a"
p Curses.get_char #=> nil (no input is ready or EOF)
Curses.stdscr.timeout = 100 # ms
```

## Curses::Window

## Refresh screen

## Character width

## Windows console

* Different from others
* Dedicated API

## Console API

* High-level console I/O
    * ReadConsole()
    * WriteConsole()
* Low-level console I/O
    * ReadConsoleInput() / WriteConsoleInput()
    * ReadConsoleOutput() / WriteConsoleOutput()

## New Windows console

* WriteConsoleOutput() cannot handle full-width characters

## Menus and forms

* libmenu: menus
* libform: forms
* Not implemented in PDCurses

## Curses::Menu

## Curses::Form

## TUI Framework

* Textbringer

## References

* W. Richard Stevens, Stephen A. Rago
  "Advanced Programming in the UNIX Environment"
  「詳解UNIXプログラミング」

* 田中哲 (Akira Tanaka)
  「APIデザインケーススタディ」 (API Design Case Study)

## Credit

* DEC VT100 terminal at the Living Computer Museum
  https://en.wikipedia.org/wiki/Computer_terminal#/media/File:DEC_VT100_terminal.jpg
  Jason Scott
  CC BY 2.0
