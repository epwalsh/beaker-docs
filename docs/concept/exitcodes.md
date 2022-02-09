# Exit codes

Beaker reports a numeric exit code for completed tasks. Usually this code is 0,
which means there was no error.

If the code is above 0, its meaning varies, but largely follows [chroot
semantics](https://tldp.org/LDP/abs/html/exitcodes.html). For example, if you
see an exit code above 128, you can decode it to a Linux signal that was likely
received by the process from Beaker's runtime environment.

The meaning of a Linux signal is only enforced by convention. This means that their meaning can change
from one program and/or distro to another. Any process can be sent any signal (`kill -9 1234` sends signal 9
to process 1234) and the process can handle the signal any way it wants. Most
processes are well behaved and follow the common conventions.

To help you decode the Beaker exit code to a Linux signal, here is a table of
exit codes (and corresponding signals) we've encountered running workloads on
Beaker.

| Beaker exit code | Linux signal    | Explanation |
|:----------------:|:---------------:|-------------|
|  129             |  1 (SIGHUP)     |             |
|  130             |  2 (SIGINT)     | Your program was killed outside your control. For example, someone pressed ^C. |
|  131             |  3 (SIGQUIT)    |             |
|  132             |  4 (SIGILL)     |             |
|  133             |  5 (SIGTRAP)    |             |
|  134             |  6 (SIGABRT)    |             |
|  134             |  6 (SIGIOT)     |             |
|  135             |  7 (SIGBUS)     | [SIGBUS (bus error) is a signal that happens when you try to access memory that has not been physically mapped. This is different to a SIGSEGV (segmentation fault) in that a segfault happens when an address is invalid, while a bus error means the address is valid but we failed to read/write.](https://www.sublimetext.com/blog/articles/use-mmap-with-care) |
|  136             |  8 (SIGFPE)     |             |
|  137             |  9 (SIGKILL)    | Your program was killed outside your control. For example, it was preempted or using too many resources.  This is seen often when your program runs out of memory, because it tries to allocate more than is free on the system.
|  138             | 10 (SIGUSR1)    |             |
|  139             | 11 (SIGSEGV)    |             |
|  140             | 12 (SIGUSR2)    |             |
|  141             | 13 (SIGPIPE)    |             |
|  142             | 14 (SIGALRM)    |             |
|  143             | 15 (SIGTERM)    |             |
|  144             | 16 (SIGSTKFLT)  |             |
|  145             | 17 (SIGCHLD)    |             |
|  146             | 18 (SIGCONT)    |             |
|  147             | 19 (SIGSTOP)    |             |
|  148             | 20 (SIGTSTP)    |             |
|  149             | 21 (SIGTTIN)    |             |
|  150             | 22 (SIGTTOU)    |             |
|  151             | 23 (SIGURG)     |             |
|  152             | 24 (SIGXCPU)    |             |
|  153             | 25 (SIGXFSZ)    |             |
|  154             | 26 (SIGVTALRM)  |             |
|  155             | 27 (SIGPROF)    |             |
|  156             | 28 (SIGWINCH)   |             |
|  157             | 29 (SIGIO)      |             |
|  158             | 30 (SIGPWR)     |             |
|  159             | 31 (SIGSYS)     |             |

And here are some documents which provide explanations for these exit code and
signal numbers.

* [Linux signal(7) man page with a list of signals](https://man7.org/linux/man-pages/man7/signal.7.html)
* https://stackoverflow.com/questions/31297616/what-is-the-authoritative-list-of-docker-run-exit-codes
* https://unix.stackexchange.com/questions/99112/default-exit-code-when-process-is-terminated
* https://intl.cloud.tencent.com/document/product/457/35758
