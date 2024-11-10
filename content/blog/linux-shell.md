+++
title = "How does `ls /var/log | grep '.log' | sort > log_files.txt` work?"
date = 2024-11-10
description = "Behind the scenes of pipe operator (`|`) and redirects (`<`, `>`) on linux shell."
[taxonomies]
categories = ["linux/unix", "c"]
tags = ["linux", "c"]
[extra]
lang = "en"
toc = true
+++
Behind the scenes of pipe operator (`|`) and redirects (`<`, `>`) on linux shell.

### Intro

Recently, I was tasked to build a linux shell from scratch as part of an assignment at school. One of the tricky parts of it was implementing support for the pipe operator (`|`) and redirects (`<` and `>`) as it required a careful manipulation of file descriptors. So what actually happens when you type in a complex command like:

```shell
ls /var/log | grep ".log" | sort > log_files.txt
```

The command finds all the files that has ".log" in its name, sorts them in alphabetical order and puts in a log_files.txt file. The output of `ls` command is somehow taken as input by `grep`, and the output of `grep` is taken as input by `sort` and the result of sort is sent into an actual file instead of printing it on the screen.

Now, the interesting part is that all these commands are independent. By that, I mean `grep` doesn't actually _know_ that it's reading the output of `ls`. Which means, the input and output layers are manipulated at the operating system level.

### [Everything is a file in Unix](https://en.wikipedia.org/wiki/Everything_is_a_file)

You probably heard of this phrase before. Pipes and redirects leverage this idea to work. That is, things like standard input, standard output, pipes (created using `pipe()` system call -- a unidirectional communication link b/w two file descriptors), actual files, network connections etc. are all the same thing, a FILE.

More clearly, system calls like write(), open(), read() etc work with all types of I/O and use the [file descriptor](https://en.wikipedia.org/wiki/File_descriptor#:~:text=In%20Unix%20and%20Unix%2Dlike,a%20pipe%20or%20network%20socket.) to identify them. Whenever a file is opened, the OS assigns a file descriptor - which is an index in the "file table" that the OS maintains per process. By default, the standard input is always 0, the standard output is 1 by the Unix standard. Calls like write, read take in the file descriptor and the OS knows how to do things for that specific type of input/output source.

### How do file descriptors help in redirecting I/O?

The `ls /var/log` command by default reads input from standard input (file descriptor 0) and writes output to standard output (file descriptor 1). For the output of this `ls` command to be taken as input to another command `grep` -- all we need to do is manipulate output file descriptor of ls and input file descriptor of grep.

For this, we would need a pipe to communicate between the two processes (yes, `ls` and `grep` run on two different processes created by `fork()` system call). A pipe is created using the `pipe()` system call - and it returns two file descriptors: one is a "read end" of the pipe and the other is a "write end" of the pipe. If one writes bytes to the write end of the pipe, they can be read from the read end of the pipe. It is important to note that the size of the buffer used by pipe is very small -- which means `grep` can't wait for `ls` to finish writing to the pipe but should start reading as soon as something is available (`ls` and `grep` run parallel in a way).

Which means, in the process used for executing `ls`, the file descriptor 1 should point to a write end of a `pipe()` instead of standard output and in the process used for executing `grep` the file descriptor 0 should point to the read end of the pipe.

Sounds pretty simple in concept. Coding it in C can be little tricky because the use of `fork()` system call enforces certain structure in the code, and causes duplication of file descriptors that needs closing. If a write end of a pipe is not properly closed after there is nothing more to write, the read end would never stop blocking -- and might result in your shell hanging!

### The `dup2` system call

The manipulation of file descriptors is also very simple, thanks to the `dup2` system call. It takes in two file descriptors -- duplicates the first one and assigns it the value of second fd. For example, `dup2(3, 0)` would create a duplicate of file descriptor `3` and is now referenced by file descriptor `0`. That is, after doing that dup2 call, fd 0 points to whatever fd 3 points to, and not necessarily standard input anymore.

For the `ls | grep` pipeline we'll have:

* parent: `pipe` -> `read_fd`, `write_fd`
* child1: `dup2(write_fd, 1)`, `close(write_fd)`, `close(read_fd)`
* child2: `dup2(read_fd, 0)`, `close(read_fd)`, `close(write_fd)`
* parent: `close(read_fd)`, `close(write_fd)`


Here is some sample C code on how `ls /var/log | grep "log"` would execute:

```c
int pipe_fd[2];
pipe(pipe_fd); // pipe_fd[0] -> read end, pipe_fd[1] -> write end

if (fork() == 0) {
    /* Child */
    dup2(pipe_fd[1], STDOUT_FILENO); // printf() would now write to write end of the pipe for example
    close(pipe_fd[1]); // duplicate
    close(pipe_fd[0]); // duplicate in child proc
    execvp("ls", (char*[]){"ls", "/var/log", NULL}) // output is written to pipe_fd[1], can be read from pipe_fd[0]
} else {
    /* parent */

    if (fork() == 0) {
        /* second child */
        dup2(pipe_fd[0], STDIN_FILENO); // fd 0 is now read end of the pipe
        close(pipe_fd[0]); // duplicate
        close(pipe_fd[1]); // duplicate
        execvp("grep", (char*[]){"grep", "log", NULL}); // reads input from pipe_fd[0]
    } else {
        close(pipe_fd[1]); // close write end from parent
        close(pipe_fd[0]); // close read end from parent
        // waitpid() of all processes to complete
    }
}
```

### What about redirects? (`<` and `>`)

They work very similar to pipes -- but instead of using the file descriptros from `pipe()`, we use the file descriptor returned by `open()` in the `dup2` system calls.

### Conclusion

To wrap it up, the magic behind pipes and redirects in Unix-like systems lies in how everything is treated as a file. Whether it's a standard input/output, a pipe, or a regular file, all these are represented by file descriptors that can be easily manipulated using system calls like dup2. By understanding how file descriptors work, you can see how processes can communicate through pipes or redirect data to/from files, all while keeping the processes independent. This allows for a clean, flexible, and efficient way to manage data flow in the command line.