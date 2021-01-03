Code Description
===================

The scripts presented together with this file implement a crude way of 
driving a simulated hoist along its _z_ axis via an user interface placed
in a different machine with respect to the device to be driven.

## General Approach

The hoist is driven by a user interface implemented by the **client** process, forwarding commands to the **server** through a socket, 
while the status of the device is displayed to the end user by the **reader** process, launched by the first one in a new shell.
The **server** process is logically placed inbetween a server and a device driver, generating periodic messages to control the hoist from
the aperiodic user requests received via socket.
Finally a process simulates the behaviour of the hoist by simply modifying an height parameter according to received input (and to limits
imposed offline).

----------------

## Files

Several scripts compose the entirety of the project, of which a brief description
is presented here while a more extensive analysis will be later carried out
in their respective paragraph under **Project Components**.

1.	makefile			-	the makefile used for compiling the scripts
2.	hoistlib.h			-	the header file containing definition of parameters, types, libraries and functions
3.	hoistlib.c			-	the file containing the actual implementation of structs and functions
4.	sever.c				-	the server on the machine with public IP, controls the hoist
5.	hoist.c				-	simple simulator of the hoist at the lowest level
6.	client.c			-	the client of the communication, the input user interface
7.	reader.c			-	the output user interface, printing status updates on a new shell
8. 	server_log.txt		-	the log file where server-side processes will update their status (\**generated at runtime if not already present*)
8. 	client_log.txt		-	the log file where client-side processes will update their status (\**generated at runtime if not already present*)

---------------

## Project Components

### Makefile

Used to simplify compilation and linking: extensive notes are reported in the file *compiling_running.md*.

### Hoistlib

Composed of a header file (*hoistlib.h*) and a script (*hoistlib.c*).\
The header contains the definitions of parameters such as maximum and minimum hoist height and hoist height increments. 
An opaque pointer to the struct defining status messages is also defined here, while the actual parameters contained therein are 
defined in the script. The choice of not giving the processes knowledge on the structure fields is made both for security
reasons as well as to improve scalability (allowing for an easy update of the structure inner fields such as adding new parameters
or changing their values). Nonetheless it should be noted that the non-direct access to the struct fields requires to use run-time
memory allocation, which should be generally avoided in these systems: that being said, the usage in the scripts is such that a 
single dinamic instance will be created (in both *server.c* and *reader.c*) in the part of the code deputed to initialization, so
not inside control loops, thus avoiding possible memory related errors (or ensuring their occurrence will be ahead of the actual
control section).\
An enumerator is present in order to facilitate the status check inside the other scripts.\
Many functions are also available for manipulation of the messages structure, which description is presented directly inside the 
script.

### Client

The client script is responsible for the connection via socket to the server process, which address and port need to be passed as command line
arguments (see *compiling_running.md* for more details). After such communication channel is established the client will be the 
input interface where the end user can insert commands to drive the hoist, which will be passed to the server through the socket.
The status report (also identified as the _output interface_) is not carryed out by this script but by the __reader__ further presented, in 
order to improve readability without mixing input and output user data.

### Server

Thi script defines the code to run on a machine with a public IP address (or other ways of being reachable from the outside of local area network),
which will listen through a socket on a port which number should be passed as command line argument.
Before actually opening the socket, the process spawns the hoist, with which it will communicate via a couple of shared unnamed pipes.\
After the communication channel is opened the process waits for a connection request (since it's assumed only one client will at any moment request
control over the hoist) then it starts reading data from the socket.
The approach chosen for the control over the hoist was **not** to forward just a single command once in a while, which the device will had to follow
until a new command was issued, but to constantly send _instantaneous_ commands which the hoist has to follow for each individual time period.
So the read from the socket is performed inside a periodic cycle (1 second period) in a non-deterministic way - using a _select()_ - and
* if a valid command was available in the socket, update the internal status of the server accordingly
* otherwise maintain the previous status

in any case the status implies a command to send to the hoist in order to control it.
Possibile status are
* **UP**:		the hoist should be driven upward
* **DOWN**:		the hoist should be driven downward
* **STOP**:		the hoist should be stopped
* **TOP**:		the hoist reached the topmost position (the hoist is stopped and cannot be driven upward)
* **BOTTOM**:	the hoist reached the downmost position (the hoist is stopped and cannot be driven downward)
* **EXIT**:		the communication should be interrupted

It should be noted that this approach, while being not the most straightforward solution, was chosen for specific security reasons (see **Security Notes**
section for further information).
Commands are sent via unnamed pipe to the hoist process, which in turns provides information on the current height of the device, that the server inserts
into a message structure together with the current status, sending this packet through the socket.

### Hoist

The hoist process is treated as a simulator more than a driver, simply going up or down (updating its stored actual height) when such commands
are received via the unnamed pipe shared with the server, and simply waiting in all other cases. The height value is passed via unnamed pipe back to the server.
>NOTE: the hoist frame is considered to be placed so that the height *z* = 0 occurs when the hoist reaches the *downmost* position, so coinciding with the 
ground reference frame.

### Reader

The reader process is connected to the same socket of the **client** (being it a child of that process), but its sole purpose is to read status reports from it and print the values received
on a new shell (opened using "_konsole_" third party program), in a user friendly way.

### Log files

Both files are created by the server and client respectively if not already present, and states the state of each process involved in one 
of the scenarios during time, allowing for a quicker detection of unexpected behaviours. The messages there are not preceeded
by timestamp, as should generally be for larger project, due to the small nature of this task, for which the order of status
updates itself constitutes a sufficent source of information for debugging purposes.

-----------

## Security Notes

The **server** process built is actually an hybrid between a server and a device driver, both receiving asynchronous commands via socket and
synchronously sending instantaneous commands to the **hoist**, holding the description of the status of such hoist.
This allows for a more robust system, with high tolerance to faults everywhere on the chain:
- if the server crashes the hoist will automatically stop in the next period, since it will not have received any instantaneous commands 
- if the client crashes the server will still try to drive the hoist in the closest extrema position
A further development of the code could modify the way the **client** operates, making it a cyclic process with a non-deterministic read from 
the user command line, sending simply connection-related data each period. The server could check, at each of its periods (which may differ from 
the client ones) the time elapsed since the last packet was received from the client, stopping the hoist after a timeout.

Note that, however, this approach reduces scalability, since to address multiple clients commands we should need to generate a child process
of the server dealing with that particular communication channel, moving the device driver aspect to it: this nevertheless introduces a new 
possible breakage point in the command chain, since in that case both a client and a server crash would let the hoist move toward the topmost or
downmost position.


