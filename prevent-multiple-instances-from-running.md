# Prevent Multiple Instances From Running

## Intro

This note is to show examples to use [flock](https://man7.org/linux/man-pages/man2/flock.2.html) call to prevent multiple instances from running.

## What is flock()?

```
       flock - apply or remove an advisory lock on an open file

       4.4BSD (the flock() call first appeared in 4.2BSD).  A version of
       flock(), possibly implemented in terms of fcntl(2), appears on
       most UNIX systems.
       
       Since kernel 2.0, flock() is implemented as a system call in its
       own right rather than being emulated in the GNU C library as a
       call to fcntl(2).  With this implementation, there is no
       interaction between the types of lock placed by flock() and
       fcntl(2), and flock() does not detect deadlock.  (Note, however,
       that on some systems, such as the modern BSDs, flock() and
       fcntl(2) locks do interact with one another.)

       flock() places advisory locks only; given suitable permissions on
       a file, a process is free to ignore the use of flock() and
       perform I/O on the file.

       flock() and fcntl(2) locks have different semantics with respect
       to forked processes and dup(2).  On systems that implement
       flock() using fcntl(2), the semantics of flock() will be
       different from those described in this manual page.
```

Above words are from the flock syscall manual.

## flock() usage

```c
#include <sys/file.h>

int flock(int fd, int operation);
```

**fd** is the file descriptor.

**operation** has 3 different values:

```
           LOCK_SH  Place a shared lock.  More than one process may hold
                    a shared lock for a given file at a given time.

           LOCK_EX  Place an exclusive lock.  Only one process may hold
                    an exclusive lock for a given file at a given time.

           LOCK_UN  Remove an existing lock held by this process.1
```

And there's a catch: A call to flock() may block if an incompatible lock is held by another process.  To make a nonblocking request, include **LOCK_NB** (by ORing) with any of the above operations.

## Python Examples

Python uses [fcntl](https://docs.python.org/3/library/fcntl.html) module to implement flock() call.

### flock() without LOCK_NB

Example Code:

```python
#!/usr/bin/env python3

import fcntl
import os
import sys
import time

lock_file = 'service.lock'
pid = os.getpid()

f = open(lock_file, 'w')

try:
    fcntl.flock(f, fcntl.LOCK_EX)
except BlockingIOError:
    f.close()
    print('Failed to acquire the exclusive lock, a running process exists already.')
    sys.exit(2)
except OSError as e:
    f.close()
    print('Failed to initial flock() call: %s' %(e))
else:
    print('PID: %d' %(pid))
    time.sleep(10)
    f.close()
```

On Terminal 1, running:

```
$ time python3 lock.py
PID: 3023361

real	0m10.057s
user	0m0.039s
sys	0m0.008s
```

Then on Terminal 2, running the same command immediately:

```
$ time python3 lock.py 
PID: 3023367

real	0m18.199s
user	0m0.039s
sys	0m0.008s
```

There is ~2 seconds difference because I switched the terminal. So without **LOCK_NB**, another process will be blocked and resumed once the first program closed the lock file and exited.

### flock() with LOCK_NB

Example Code:

```python
#!/usr/bin/env python3

import fcntl
import os
import sys
import time

lock_file = 'service.lock'
pid = os.getpid()

f = open(lock_file, 'w')

try:
    fcntl.flock(f, fcntl.LOCK_EX | fcntl.LOCK_NB)
except BlockingIOError:
    f.close()
    print('Failed to acquire the exclusive lock, a running process exists already.')
    sys.exit(2)
except OSError as e:
    f.close()
    print('Failed to initial flock() call: %s' %(e))
else:
    print('PID: %d' %(pid))
    time.sleep(10)
    f.close()
```

On Terminal 1, running:

```
$ time python3 lock.py 
PID: 3023699

real	0m10.055s
user	0m0.038s
sys	0m0.007s
```

Then on Terminal 2, running the same command immediately:

```
$ time python3 lock.py 
Failed to acquire the exclusive lock, a running process exists already.

real	0m0.065s
user	0m0.050s
sys	0m0.015s
```

In this case, the second process exited immediately when it's failed to acquire the exclusive lock from the lock file. We must use the exception **BlockingIOError** to capture the lock acquisition failure.
