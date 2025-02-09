# Basic Operations
- Quick start
  https://web.eecs.umich.edu/~sugih/pointers/gdbQS.html
- Examine memory
  https://sourceware.org/gdb/current/onlinedocs/gdb.html/Memory.html
- Examine CPU registers
  https://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_60.html
- Continuing and stepping
  https://sourceware.org/gdb/current/onlinedocs/gdb.html/Continuing-and-Stepping.html#index-si-_0028stepi_0029
# Encounterred issues
- GDB run to somewhere else after moving value to SP
  > The step and next commands try to be "smarter" about how they operate, they make use of the debug information line table to ensure that GDB step by complete source lines. These commands also track which frame the command starts in, and use this to try and avoid stopping in a callee frame.
  > However, this frame detection doesn't always handle unexpected changes to the stack pointer, and it would appear that in this case, GDB is getting very confused about what's going on.
  [Reference](https://stackoverflow.com/questions/74988578/debugging-issue-moving-value-into-sp-register)