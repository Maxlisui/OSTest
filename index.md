
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

See Exercise 3, A2 or A3, for concrete implementation.  
[List of Signals](http://man7.org/linux/man-pages/man7/signal.7.html)  

**[Execv](https://linux.die.net/man/3/execv)**  
Library: unistd.h  
Is passed a NULL-terminated array where the first entry is the name of the program to be executed, and any following array entries before NULL are the parameters.  

    char *args[2];
    args[0] = "/bin/grep";
    args[1] = argv[1];
    args[2] = NULL;
    
    //call grep
    execv(args[0], args);

See Exercise 4, A2 for implementation.  

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

## Shared Memory Segment
### System V
Libraries: sys/types.h, sys/ipc.h, sys/shm.h  
See Ex. 5 - A1 for implementation.  

Create Shared Memory Segment:  

    int shmid = shmget(ftok("keyfile", SHMKEY), SHMSZ, IPC_CREAT | 0666);

Attach Shared Memory Segment to variable:

    //Shared Memory Int
    int *shm;
    //attach shared memory segment to variable shm
    shm = shmat(shmid, NULL, 0);

Get existing Shared Memory Segment ID (Attach the same way):  

    int shmid = shmget(ftok("keyfile", SHMKEY), SHMSZ, 0666);

### POSIX
Libraries: sys/mman.h, sys/stat.h, fnctl.h **and** linked with -lrt  
Create Shared Memory Segment  

    int shmid = shm_open("posixshared", O_CREAT | O_RDWR, 0666);
    ftruncate(shmid, SHMSZ);
    
Open existing Shared Memory Segment  

    int shmid = shm_open("posixshared", O_RDWR, 0666);
    ftruncate(shmid, SHMSZ);

Attach Shared Memory Segment (Here to int * shm)

     int * shm = mmap(0, SHMSZ, PROT_WRITE, MAP_SHARED, shmid, 0);
    
## Semaphores
### System V
Libraries: sys/types.h, sys/ipc.h, sys/sem.h  
See Ex. 5 - A2 for implementation.  

This union **needs to be defined by the calling program!**

    union semun
    {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
        struct seminfo *__buf;
    };

Create Semaphore: 

    int semid = semget(ftok("keyfile", SEMKEY), 1, IPC_CREAT | 0666);

Initialize Semaphore:  

    
        //Set value to 1
        union semun arg;
        arg.val = 1;
        semctl(semid, 0, SETVAL, arg);

Increasing/Decreasing Semaphores:   
See A2-common Ex. 5

Close Semaphore (Only on one process)

    //remove semaphore
    semctl(semid, 0, IPC_RMID, arg);
    
Getting an existing Semaphore (Increasing / Decreasing the same way):  

    semid = semget(ftok("keyfile", SEMKEY), 1, 0666);

### POSIX 
Libraries: semaphore.h **and** linked with -pthread  

Creating semaphore:  
    sem_t sema;

Initializing semaphore:

    sem_init(&sema, 1, 1);

Or when using IPC:

    sem_open(<Semname>, O_CREAT | O_RDWR, 0666, 1);

Increasing and decreasing semaphore:  

     sem_wait(&sema); //Decrease Semaphore
     sem_post(&sema); //Increase Semaphore

## POSIX Threads
Libraries: pthread.h **and** linked with pthread  

[**Creating a thread**](https://linux.die.net/man/3/pthread_create)  

    pthread_create(pthread_t * thread, NULL, *thread_func, void * arg)
    
**Killing a thread**  

    pthread_cancel(pthread_t * thread);
    
**Joining a thread**  

    pthread_join(pthread_t * thread, NULL);
    
**Cleanup Methods**
Adding a cleanup method to the stack

    pthread_cleanup_push(*cleanup_method, void * arg);
    
Manually popping the stack (end of method)

    pthread_cleanup_pop(1);
    
If thread is cancelled, all methods on the stack are executed automatically

## Mutexes
Libraries: pthread.h **and** link with pthread  

Initialize Mutex statically

    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    
or dynamically

    pthread_mutex_init(&mutex)
    
Lock Mutex

    pthread_mutex_lock(&mutex);
    
Unlock Mutex

    pthread_mutex_unlock(&mutex);

## Spinlock
Libraries: pthread.h **and** link with pthread  

[Initialize Spinlock](https://linux.die.net/man/3/pthread_spin_init) - only dynamically:

    pthread_spinlock_t spinlock;
    pthread_spin_init(&spinlock, PTHREAD_PROCESS_PRIVATE);
    
Lock Spinlock

    pthread_spin_lock(&spinlock);

Unlock Spinlock

    pthread_spin_unlock(&spinlock);

## Condition Variables
Libraries: pthread.h **and** link with pthread  
Initialize statically

    pthread_cond_t cv = PTHREAD_COND_INITIALIZER;

Initialize dynamically

    pthread_cond_t cv;
    pthread_cond_init(&cv, NULL);
    
Signal Variable

    pthread_cond_signal(&cv);
    
Broadcasting Variable

    pthread_cond_broadcast(&cv);

Wait for Variable

    pthread_cond_wait(&cv, &mutex);

Rules for using condition variables:  

 - Should always be used together with a mutex
 - A thread must have the mutex before calling cond_wait
 - Calling cond_wait unlocks the mutex for other threads to use
 - Upon receiving cond_signal the thread will try to get the mutex back
 - (Ideally) the thread calling cond_wait should have the mutex before calling it so that unlocking the mutex will allow the signaled thread to pick it up
 - cond_signal sends a signal to a random blocked thread
 - cond_broadcast sends a signal to all blocked threads
 - signal and broadcast **are NOT persistent**, if no thread picks up the signal at the time, the signal is **lost**.


 hallo i
 bims

## Atomic Variables
Defining atomic Variables in C11
```
_Atomic int global = 0;
global++;
```
Atomic add procedures

    atomic_fetch_add_explicit(&x, 1, memory_order_relaxed);

## Example Makefile

    CFLAGS = -std=gnu11 -Wall -Werror -Wextra
    
    .PHONY: all clean
    
    all: program1 program2 program3
    
    clean:
    	$(RM) program1 program2 program3
    
    program1: program1.c -lpthread
    program2: program2.c -lpthread
    program3: program3.c -lpthread
