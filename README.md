# Crashcart Hands On Exercise

## Introduction
[Crashcart](http://github.com/oracle/crashcart) is a tool for debugging running containers.  It sideloads an image with debug tools into an existing container so you can better diagnose runtime issues.

The aims of this exercise are to demonstrate:
1.  Limitations of debugging containers
2.  Using Crashcart to debug containers

## Pre - requisites
- Linux machine (physical or virtual)
- Editor (vim _obviously_)
- Docker installed
- git installed
- tcpdump installed
- lsof installed

## Running some Containers
We're going to run a container and a microcontainer for this exercise.
1.  Open three terminal windows
2.  In the first window run the ("fat") container:

`sudo docker run -i -p 8080:80/tcp --rm --name fat httpd`

3.  In the second window run the ("thin") microcontainer:

`sudo docker run -i -p 80:80/tcp --rm --name thin vishvananda/smith-httpd`

4.  In the third window, check that they're both up and running:
```
$ curl localhost:8080
<html><body><h1>It works!</h1></body></html>
$ curl localhost:80
<html><body><h1>It works!</h1></body></html>
```

## Debugging from the host
### docker exec
The easiest way to start with debugging from the host is to use `docker exec`.

#### running bash
Let's try to run a bash shell in each of our containers.

1.  For the fat container:

```
$ sudo docker exec -it fat bash
root@d5dc0792aa29:/usr/local/apache2# 
```

2.  For the thin container:
```
$ sudo docker exec -it thin bash
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:262: starting container process caused "exec: \"bash\": executable file not found in $PATH"
```

This fails because bash is (typically) not included in a microcontainer.

#### running tcpdump
If we try to run a typical debug tool, such as tcpdump, with `docker exec` we're likely to hit the same problem with both fat and thin containers:

```
$ sudo docker exec fat tcpdump -peni eth0
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:262: starting container process caused "exec: \"tcpdump\": executable file not found in $PATH"

$ sudo docker exec thin tcpdump -peni eth0
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:262: starting container process caused "exec: \"tcpdump\": executable file not found in $PATH"
```

### nsenter
`nsenter` can help in some cases.
#### running tcpdump
We're going to use nsenter to run tcpdump for each container.  To do this we need to get the process id (pid) for each container, so we can supply this as an argument for nsenter.
1.  Open another ssh window
2.  In one window get the pid for the fat container:
```
$ export FAT_PID=`sudo docker inspect --format '{{ .State.Pid }}' fat`
$ echo $FAT_PID
19489
```
3.  Use nsenter to run tcpdump for the fat container:

`sudo nsenter -t $FAT_PID -n -- tcpdump -peni eth0`

4.  From a different ssh window, call the fat container with curl:

`curl localhost:8080`

You should see the output in the window where tcpdump is running.

Repeat steps 1 - 4, substituting `thin` / `THIN_PID` for `fat` / `FAT_PID` and run

`curl localhost:80`

Now you should see output in the window where tcpdump is running for the thin container.

Stop the tcpdump in both windows.

### Limitations of debugging from the host
Debugging from the host can be a useful technique, but it has it's limits.
#### running lsof

`lsof` is a utility that lists open files.  Let's assume we want to know which files one of our httpd processes has open.

You can try to use `lsof` to see the files that the fat container has open, type (in the window where $FAT_PID is set):

`sudo lsof | grep httpd | grep $FAT_PID`

However, you'll get lots of output like this:

`httpd     19489                    root  mem       REG              249,4             33555855 /lib/x86_64-linux-gnu/libexpat.so.1.6.0 (stat: No such file or directory)`

The reason we can't see the information about the files because they are in a different mount namespace.

So let's try `docker exec`:

```
$ sudo docker exec fat lsof
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:262: starting container process caused "exec: \"lsof\": executable file not found in $PATH"
```

So that didn't work.  How about `nsenter`?
```
$ sudo nsenter -t $FAT_PID -m -- lsof | grep httpd
nsenter: failed to execute lsof: No such file or directory
```

No.  So this is the type of situation where we need Crashcart.

## Crashcart

### Installation
1.  Install rust (if you don't have it already):
```
curl https://sh.rustup.rs -sSf | sh
rustup toolchain install stable-x86_64-unknown-linux-gnu
rustup default stable-x86_64-unknown-linux-gnu
rustup target install x86_64-unknown-linux-musl
```

2.  Clone `crashcart` from [GitHub](https://github.com/oracle/crashcart)

3.  cd into the crashcart directory and run `build.sh`:
```
$ cd crashcart
$ ./build.sh
```
This will take a few minutes

4.  While we're waiting for the build to complete, we're going to save a considerable amount of time by downloading the [crashcart image (crashcart.img)](https://drive.google.com/file/d/19FJkANn8PlQb7BIatRIiSfVA_WFgJA7o/view?usp=sharing).

Building the crashcart image locally typically takes several hours.
5.  Move the downloaded crashcart.img file to the crashcart directory.
6.  Run crashcart to get a shell inside the container.  This time do it in the 'thin' container window:

```
$ sudo ./crashcart $THIN_PID
INFO - crashcart.img is backed to /dev/loop2
INFO - crashcart.img is loaded into namespace of pid 21396
bash-4.4#
```
We couldn't even get a bash shell here before.
7.  At the bash prompt run lsof:

`$ lsof`

Now it should run

8.  We can also mount the crashcart image into the container (using the `m` flag):

```
$ sudo ./crashcart -m $THIN_PID
INFO - crashcart.img is backed to /dev/loop2
INFO - crashcart.img is loaded into namespace of pid 21396
$
```

9.  Now we can use `docker exec` or `nsenter` to run binaries from the crashcart image inside the container:
```
sudo docker exec -it thin /dev/crashcart/bin/tcpdump
```
then if you run `curl localhost` in another window, you'll see the tcpdump output.

So `crashcart` gives us a way to debug inside the container, using the tools on the crashcart image (or your own image), when host based debugging fails.
