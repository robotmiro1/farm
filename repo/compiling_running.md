Compiling and Running
===================

Compiling
----------------------

A makefile is given due to the many scripts composing the entirety of the 2nd assignment project,
in order to ease the compiling operation.
Three slightly different commands can be issued to compile different subsets
of the entirety of the scripts. This is in order to allow for both distribution of the sole
executables (in which case the first command is the faster approach) or of the source
code themselves (in which case the two parts, server and client, might need to be compiled on two
different machines).

To compile every script presented (already linking libraries and cleaning object files) use
```bash
$ make
```

To compile only the scripts referring to the server side, that is the server itself
and the hoist "simulator" (already linking libraries and cleaning object files) use
```bash
$ make hoist_server
```

To compile only the scripts referring to the client side, that is the "input"
and the "output" user interface (already linking libraries and cleaning object files) use
```bash
$ make hoist_interface
```

Running
--------------------

The server can be runned by providing the port it should listen to for the communication
```bash
$ ./server <portnumber>
```

While the client will need the public address of the machine the server is running on together
with the port to direct data to
```bash
$ ./client <IP address> <portnumber>
```


## Requirements
In order to make the status information print clearer, the "output interface" process is launched in a new shell window using the third party program
"__konsole__", which is thus required, together with a C compiler.