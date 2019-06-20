
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

# IPC
## [Pipes](https://linux.die.net/man/3/pipe)
Library: unistd.h  
Create integer array of size [2]  

    int filedes[2];
Create pipe, THEN fork process - both processes have access to the filedes[] pipe object.

    pipe(filedes);
    fork();
One end should be closed in each process(0 is reading end, 1 is writing end) 

    close(filedes[0]); //closes reading end
    close(filedes[1]); //closes writing end
[dup2](https://linux.die.net/man/3/dup2) duplicates the output from the second filedescriptor into the firstparameter. [dup](https://linux.die.net/man/3/dup) duplicates the output from the given filedescriptor to the lowest available file descriptor number.

    //route stdout into pipe
    dup2(filedes[1], STDOUT_FILENO);

With dup

    //Reroute pipe read to stdin
     close(STDIN_FILENO);
     dup(filedes[0]);
Implementation see Exercise 4, A2

## [FIFO Pipes](https://www.geeksforgeeks.org/named-pipe-fifo-example-c-program/)
Library: sys/types.h **and** sys/stat.h, stdio.h for fopen/fclose

[mkfifo](https://linux.die.net/man/3/mkfifo)(char * filestring, 0666) creates the FIFO  
[fopen](https://linux.die.net/man/3/fopen)(char * filestring, "w") opens fifo for writing, with "r" opens fifo for reading  
[fclose](https://linux.die.net/man/3/fclose)(FILE * filepointer) closes the file  

Files take string reading operations ([fprintf](https://linux.die.net/man/3/fprintf), [fscanf](https://linux.die.net/man/3/fscanf), [fgets](https://linux.die.net/man/3/fgets), [fputs](https://linux.die.net/man/3/fputs), ...)  and binary reading operations ([fwrite](https://linux.die.net/man/3/fwrite), [fread](https://linux.die.net/man/3/fread)).  

### [File Monitoring](https://linux.die.net/man/3/fd_set)
See Exercise 4, A1 for implementation example.
Get Filedescriptor of type int by calling `fileno(FILE * fp)`  
Create file set of type `fd_set`, and timeout var of type `struct timeval`  

**All the following variables need to be set in a loop as the select call can change them!**  
`FD_ZERO(fd_set * fileset)` sets the fileset to not contain any descriptors  
`FD_SET(int filedescriptor, fd_set * fileset)` adds the file descriptor to be monitored  
`timeout.tv_sec` and/or `timeout.tv_usec` are the time select() waits for a change. (If both are 0, select returns immediately)  

the [select](https://linux.die.net/man/3/select) call returns the number of file pointers who have pending data, or < 0 if an error occured. After the call `FD_ISSET(int filedescriptor, fd_set * fileset)`can be used on the set, if this returns a non-zero value, this file descriptor has been modified.

 ## Message Queues
### System V
See Ex. 4 A3 for implementation example.  
Libraries: sys/types.h, sys/ipc.h, sys/msg.h  

[ftok](https://linux.die.net/man/3/ftok)(char * filepath, int keyID) should be used to generate a collision-free key.  
The msgbuf struct **needs to be defined by the calling program!**  

    typedef struct msgbuf {
        long mtype;
        char mtext[MessageSize];
    } message_buf;

Creating a Message Queue:  

    int msqid = msgget(ftokkey, 0666 | IPC_CREAT | IPC_EXCL);

Create a message_buf struct and set `sbuf.mtype = 1;` and `sbuf.mtext` to the string value to be written.  
Sending the Message

    msgsnd(msqid, &sbuf, strlen(sbuf.mtext) + 1, IPC_NOWAIT);
To load a message queue:  

    int msqid = msgget(ftokkey, 0666);

To check a message queue:  
Create `struct msqid_ds buf;`  

        msgctl(msqid, IPC_STAT, &buf);
`buf.msg_qnum` holds the number of messages available.  

To read a message from the queue:

            message_buf rbuf;
            msgrcv(msqid, &rbuf, MessageSize, 1, 0);
Text of the message is in `rbuf.mtext`
