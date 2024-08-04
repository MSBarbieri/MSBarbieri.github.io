---
title: "The current right way to log"
subtitle: ""
date: 2024-05-26T18:00:59-03:00
lastmod: 2024-05-26T18:00:59-03:00
draft: false
description: ""
images: []

tags: ["DX","Logging","Monitoring","Observability","Tracing","DevOps"]
categories: ["DX","Logging","Monitoring","Observability","Tracing","DevOps"]

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
code:
  copy: true
math:
  enable: false
  # ...
mapbox:
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
---
Logging is the first thing we learn in programming, but we are very good at
forgetting how complicated it can be, and the proper way to handle it in
production.
These days our production environments are more complex than ever; we have lots
of systems running in different places and in different modes.

I will talk about a good way to handle it in our code without adding too much
complexity and loss of performance, and a little bit about how it works in modern
environments and how we can take advantage of it to make it more interesting.

I'm gonna talk from the basics to a complex environment, with different types of
systems in a multi-cluster environment.


## The Logging Itself.

When we see a "Hello World" being printed in our system, we don't think about
what this log really is and how it came to be in our console. I'm not going to talk
about how the log works in every modern system, but I need to remind us
about some simple things that we forget.

First of all, the log is not a simple print like in Python or a console.log in
JavaScript. These log functions represent write operations in a file, by a syscall
in the kernel, and our terminal is just a consumer of this file.

If you want to check a little bit about how it works, you can do a funny test,
which will not cover the entire process, but will give you a better understanding
of how it works.

First of all, you need to open two terminals (terminal-emulator to be more
precise) in a Unix system. In one of
them, you will run the command `tty`. The command will return the terminal file
it is looking at.

```bash
$ tty
/dev/pts/5
```

In the other terminal, you will run the command `echo "Hello World" > /dev/pts/5`.
As you probably noticed, the first terminal will print the "Hello World" message.

This will give you a little idea about how everything in the system is a file,
and our logs are not excluded from it.


### stdin, stdout and stderr

When we run a program in the terminal, we have two main outputs: stdout and stderr, as well as one main input: stdin. The stdout is the standard output, where the program prints what we want to see. The stderr is the standard error, where the program prints errors and warnings. Meanwhile, stdin is the entry point for new information that the program uses during execution. It may seem obvious from their names, but the process by which these three are decided and handled is actually the important part.

The stdout is the file descriptor 1 and the stderr is the file descriptor 2 of our program. When we start a program, the kernel opens these two files, while the parent process, which is the process that started this program, defines where these file descriptors will be pointed to. By default, a bash script will point them to the terminal, but we can change it to a file or another process.

Example 1: Pointing the stdout to a file
```bash
$ echo "Hello World" > /tmp/hello.txt
```
With this command, we are pointing the stdout to the file `/tmp/hello.txt`. You probably have noticed that the command didn't print anything in the terminal, but if you execute the command `cat` on the file `/tmp/hello.txt`, you will see "Hello World".

Example 2: Pointing the stderr to a file
```bash
$ echo "Hello Error" 2> /tmp/hello_err.txt
Hello Error
```

you have noticed that "Hello World" showed in your terminal and if you execute
a cat on the file `/tmp/hello_err.txt` you are not going to see the "hello world". This
happens because the "Hello World" was printed to stdout, and we are redirecting
stderr to the file `/tmp/hello_err.txt`. To transform an echo into an stderr message
we need to use `>&2` at the end of the message.
```bash
$ (echo "Hello Error" >&2) 2> /tmp/hello_err.txt
```


well, you can imagine "ok, this is cool, but how can i confirm my code really redirect to my tty".
we can do this in a simple way, first let's create a simple script in bash which is gonna log
in stdout every second without any redirecting.
```bash
#!/bin/bash
for i in {1..100}; do
    echo "Hello World"
    sleep 1
done
```

now let's execute it in background to be able to check the file descriptor.
```bash
$ ./simple_log.sh &
bash ./simple_log.sh &
[1] 126699
```
this number (in my case 126699) is the pid of the process, and we can check the file descriptor
path using readlink in this process, as i have said previously the stdout is the file descriptor 1
so we can check it using the following command.
```bash
$ readlink /proc/126699/fd/1
/dev/pts/2
```
in your case the pid is gonna be different, but the result is gonna be the same pattern,
you will see the file descriptor stdout pointing to your tty.

### Redirecting stdout in our code

Ok, this is a fancy trick with bash, but how is this related to my programming
language? Well, in every programming language we have a way to handle stdout
and stderr, for example, we can change the direction of our stdout in C.

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    // Open the file where you want to redirect stdout
    int fd = open("/tmp/c_hello.txt", O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // Duplicate the file descriptor to stdout (file descriptor 1)
    if (dup2(fd, STDOUT_FILENO) == -1) {
        perror("dup2");
        return 1;
    }

    // Close the original file descriptor
    close(fd);

    // Now stdout is redirected to the new file
    printf("This will be written to file c_hello.txt\n");

    return 0;
}
```

If we compile this code and run it, we will not see the `printf` message in our terminal, but if we check the file `/tmp/c_hello.txt`, we will see the message there.

```bash
$ gcc redirect_stdout.c -o redirect_stdout && ./redirect_stdout
```

we can do this in other languages to:
```javascript
const fs = require('fs');

const newStdoutStream = fs.createWriteStream('/tmp/js_hello.txt');

process.stdout.write = process.stderr.write = newStdoutStream.write.bind(newStdoutStream);

console.log("This will be written to file js_hello.txt");
```

``` bash
$ node redirect_stdout.js
```
but we can go further. When we create a child process in our program, we can redirect the stdout and stderr of this child process to our program. This is very useful when we want to capture the logs of a child process. We can do that easily because the child process will write to the stdout file descriptor which the main process created.

```javascript
const child_process = require("child_process");

const job = child_process.exec(`node -e "console.log('Hello World')"`);
job.stdout.on('data', (data) => {
    console.log(`[CHILD] ${data}`);
});

job.stderr.on('data', (data) => {
    console.error(`[CHILD-ERROR] ${data}`);
});
```

and our result gonna be:
```bash
$ node foo.js
[CHILD] Hello World
```

ok in js is a liitle bit simpler, and we already understand which is the parent process who takes
care of the stdout and stderr of the child process, but how we can do this in
c? well, we can use the `pipe` function to redirect the stdout and stderr of the
```c
#include <stdio.h>
#include <unistd.h>

int main() {
  int pipefd[2];
  pipe(pipefd);
  pid_t pid = fork();
  if (pid == -1) {
    perror("fork");
    return 1;
  }
  if (pid == 0) {
    // Child process
    close(pipefd[0]);
    dup2(pipefd[1], STDOUT_FILENO);
    close(pipefd[1]);
    printf("Hello World from child\n");
    return 0;
  }
  // Parent process
  close(pipefd[1]);
  char buffer[1024];
  ssize_t count = read(pipefd[0], buffer, sizeof(buffer));
  write(STDOUT_FILENO, buffer, count);
  close(pipefd[0]);
  return 0;
}
```

ok we understand about how to handle the stdout but were it is used? well
probably one of the main systems in most of the linux systems is the systemctl
which is the system that manages the services in the system, and we can see a
example of how a service is declared in the systemctl in this case is a default
config of redis.

```bash /etc/systemd/system/redis.service
[Unit]
Description=Advanced key-value store
After=network.target
Documentation=http://redis.io/documentation, man:redis-server(1)

[Service]
Type=notify
ExecStart=/usr/bin/redis-server /etc/redis/redis.conf --supervised systemd --daemonize no
PIDFile=/run/redis/redis-server.pid
TimeoutStopSec=0
Restart=always
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=2755

UMask=007
PrivateTmp=yes
LimitNOFILE=65535
PrivateDevices=yes
ProtectHome=yes
ReadOnlyDirectories=/
ReadWritePaths=-/var/lib/redis
ReadWritePaths=-/var/log/redis
ReadWritePaths=-/var/run/redis


NoNewPrivileges=true
CapabilityBoundingSet=CAP_SETGID CAP_SETUID CAP_SYS_RESOURCE
MemoryDenyWriteExecute=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectControlGroups=true
RestrictRealtime=true
RestrictNamespaces=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# redis-server can write to its own config file when in cluster mode so we
# permit writing there by default. If you are not using this feature, it is
# recommended that you replace the following lines with "ProtectSystem=full".
ProtectSystem=true
ReadWriteDirectories=-/etc/redis

[Install]
WantedBy=multi-user.target
Alias=redis.service
```

at this point you can image where the logs of redis are going, and you are
right, the logs of redis are going to the file
`/var/log/redis/redis-server.log`, the redis-server is because is the name of
the program started.


## The Logging in our code.

ok, we understand how the logs are handled in the system, but how we can handle 
more eficiently now in our code? well, we can use the stdout and stderr to in
dev mode to our default stdout and stderr, and in production we can redirect to
a file where we gonna store our logs.

but this is just the level one of the logging, we can go further, and by that we
gonna look at some logging libraries that we can use in our code, and some
features they have.

### The logging libraries

to give a decent example about some improtant features, i gonna show you logging
libraries in three diferent languages, python, javascript and rust, not all of
them have all the features we gonna need for our final result, but the core
features all have, and if you don't use any of this languages you will fore
sure find a library which does that.

#### Log Leveling

we already have different types of logs, like stdout and stderr, but most of the
time we need a more precise way to identify our logs, and for that we have a
feature called log leveling, which is a way to define the importance of the log.

normally we have 5 levels of logs, which are:
- TRACE
- DEBUG
- INFO
- WARN
- ERROR

and some libraries have more levels like `CRITICAL`, but this 5 are the most
common, and we can work with them just fine.
most of the libs put them in numbers of priority, like 0 for TRACE and 4 for
warning, and if you set the log level to 2 (INFO), you gonna see all the logs with 
level 2 or higher, while other libraries use the names.
all work just fine, but the names are more human readable, you just gonna need
to discover the way of your library.

***Example 1 Rust***

in rust we have the `pretty_env_logger` crate, which is a very simple crate, and
we works with a core fundamental for log library called `log`, which is used by most of
the systems in rust, `pretty_env_Logger` don't have must of the features we gonna
talk later, but if you are using rust you can change easily for a library called
`fern` which gonna have all of them and works if log to.

in rust log is not initialized by default so we use `pretty_env_logger` or
`fern` to initialize it.
```rust
extern crate pretty_env_logger;
use log::{debug, error, info, trace, warn};

fn main() {
    pretty_env_logger::init();

    trace!("a trace example");
    debug!("deboogging");
    info!("such information");
    warn!("o_O");
    error!("boom");
}
```

the `pretty_env_log` setted al the log levels for us, and when we run the code
it will look like this:
```bash  
$ cargo run 
TRACE 2024-05-26T18:00:59 [main] a trace example
debug 2024-05-26T18:00:59 [main] deboogging
INFO 2024-05-26T18:00:59 [main] such information
WARN 2024-05-26T18:00:59 [main] o_O
ERROR 2024-05-26T18:00:59 [main] boom
```


***Example 2 Python***
python has a library called `logging` which is a very complete library, and the
most used in python.

```
import logging
logger = logging.getLogger(__name__)

def main():
    logging.basicConfig(filename='myapp.log', level=logging.INFO)
    logger.info('such information')
    logger.error('boom')

if __name__ == '__main__':
    main()
```

as you can see and test the log levels in python are the same as in rust, but
here already have a default log file, and we can change the minimum log level.

if we execute the code we gonna see the following output:
```bash
$ python3 foo.py && cat myapp.log
INFO:__main__:such information
ERROR:__main__:boom
```

#### Log Formating

you have noticed that the logs in the previous examples are not very pretty, and 
a lot different from each other, but we can change that, we can format the logs
to be in different ways, like json, or a simple string, or even a custom format.

this is very useful and you gonna see later in more details, but imagine that
you gonna search for a log, is better for search it in a json format than a
simple text right? you can write a program for look to it or somothing like
that (i'm not saying that you should do that, but you can, we have better ways
to do that).
