
# Processes
**[Fork](https://linux.die.net/man/3/fork)**
Library: unistd.h
Syntax: fork();

Returns 0 if the process is the child process
Returns the PID of the child if the process is the parent


**[Wait](https://linux.die.net/man/3/wait)**
Library: sys/wait.h
Syntax: wait(NULL);

With argument NULL, waits for all child processes to exit.

**[Exit](https://linux.die.net/man/3/exit)**
Library: stdlib.h
Syntax: exit(0);
causes normal process termination. (Parent will receive SIGCHLD Signal)

**[Defining Handlers](http://man7.org/linux/man-pages/man2/sigaction.2.html)**
Library: signal.h

    struct sigaction act;
    act.sa_handler=handler;
    sigaction(SIGCHLD, &act, NULL);

`SIGCHLD` is, in this example, the signal to look out for
`handler` is a method of signature `void handler(int signal)` which is called when the signal is received.

See exercise 3, A2 or A3, for concrete implementation.
[List of Signals](http://man7.org/linux/man-pages/man7/signal.7.html)
