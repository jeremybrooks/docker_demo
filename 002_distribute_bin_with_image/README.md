# Distributing binaries in images

001 shows how Docker can be used to create tools or development environments. But now we'll look at the distribution/deployment side.

Rather than mounting in the build source, we'll be adding the source and leaving the binary in the image. In the new Dockerfile:

```
FROM debian

RUN apt-get update -qq && apt-get install -y gcc make

RUN mkdir /src
ADD main.c /src
ADD Makefile /src

RUN cd /src && make && cp hello /bin/hello
```

In this format, the source and binary will be added to the image, making the image not only the build tool, but also the artifact and it's "runtime". (If you're thinking this is inefficient, you're correct, and a more efficient implementation will come in the following demos.)

Let's build it and check it out:

```
$ docker build -t 002_distribute .
$ docker run -it --rm 002_distribute /bin/hello
hello!
```

So now the image is ready to deploy to production! ...but it's a little large:

```
$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
002_distribute             latest              3fbdb171ff8f        6 minutes ago       235MB
```

235MB is a bit big for shipping a `hello world`. But there's a few options for slimming down containers.

1. Remove build tools and source.
..* `apt-get purge -y gcc`
..* `apt-get purge -y make`
..* `rm -rf /src`

Let's add these to the `Dockerfile` and rebuild.

```
FROM debian

RUN apt-get update -qq && apt-get install -y gcc make

RUN mkdir /src
ADD main.c /src
ADD Makefile /src

RUN cd /src && make && cp hello /bin/hello

RUN apt-get purge -y gcc
RUN apt-get purge -y make
RUN rm -rf /src
```

```
$ docker build -t 002_distribute .
[...]
Step 9/9 : RUN rm -rf /src
[...]
$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
002_distribute             latest              2fadf8604ac7        2 seconds ago       237MB
```

237MB. It's larger!? This is because of the way Docker utilizes its snapshots. A Docker image is built of layers (note the `Step 9/9 : RUN rm -rf /src`, this image is made of at least 9 layers), each one essentially a `diff` of a filesystem. So even if later versions of the filesystem are smaller, Docker still maintains the history of every layer utilized. To reduce the size here, we can limit the number of layers, by creating less layers.

```
FROM debian

ADD main.c /
ADD Makefile /

RUN apt-get update -qq && \
 apt-get install -y gcc make && \
 make && \
 mv hello /bin/hello && \
 apt-get purge -y gcc make && \
 apt-get clean && \
 rm -rf /var/lib/apt/lists/*
```

There, now we're limiting our number of layers to 3, and now also removing the cache created by the use of the package manager. Perhaps it's smaller.

```
$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
002_distribute             latest              8898ca507027        2 seconds ago        218MB
debian                     latest              2b98c9851a37        2 weeks ago          100MB
```

The image is down ~20MB, but it's still quite a ways off from the image's original 100MB. Here's some more options for slimming a container:

2. Base your image on a smaller image
..* Debian for example also has a tag for this `debian:stretch-slim`
..* Stretch is the codename of the latest distro
..* Slim indicates fewer packages to start from, and a more sparse dependency policy.

3. Base your image on Alpine
..* Alpine is a Linux distro focused on small size and security.
..* It uses musl instead of glibc to keep binaries small.
..* Most of the coreutils are provided by busybox

I won't go through these options here, but I encourage you to try, and investigate the difference in size.

I will, however, skip to my favorite option.

4. Only include what you need
..* There's only one binary that gets called
..* That binary has few dependencies that are easy to look up
..* There's no need for a package manager, man pages, config files, etc

To implement this, we'll be using a newer feature, called multi-stage builds. The Dockerfile usage looks like this:

```
FROM debian

ADD main.c /
ADD Makefile /

RUN apt-get update -qq && \
 apt-get install -y gcc make && \
 make

FROM scratch
COPY --from=0 /hello /hello
```

There are two `FROM` instructions in this file. What this means is that Docker will build two images using the same `context`(the current directory, recursively included), but it will only save the last image created. You can have as many stages as you like. 

Note the `COPY --from=0 /hello /hello`, stages are zero-indexed, and you can copy files from one into the next. Here, the obvious desire is that we'll be copying the binary out onto a special base called `scratch`, meaning the image starts completely empty. Rebuild and check the results:

```
$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
002_distribute             latest              eb6a5f205aca        6 seconds ago       8.64kB
```

Now we're talking! But..

```
$ docker run -it --rm 002_distribute /hello
standard_init_linux.go:187: exec user process caused "no such file or directory"
```

It doesn't run! This is because we just copied in a binary which is dynamically linked, without bringing it's dependencies. So there's two options moving forward, the binary could be statically compiled, or the shared libraries could be copied in. The first is a simple solution, but let's do the latter. At times, you may need to create an image for a binary that you aren't building or don't have access to a static binary for.

Rebuild the image, commenting out the second stage. Run it in interactive mode, and check the dependencies of the binary.

```
FROM debian

ADD main.c /
ADD Makefile /

RUN apt-get update -qq && \
 apt-get install -y gcc make && \
 make

#FROM scratch
#COPY --from=0 /hello /hello
```

```
$ docker build -t 002_distribute .
$ docker run -it --rm 002_distribute /bin/bash
root@cb9161d65006:/# ldd hello 
	linux-vdso.so.1 (0x00007fff19721000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f35c235c000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f35c28fd000)
```

So in addition to the binary itself, we need to copy in `libc`(the standard library) and `ld-linux`(for dynamic linking resolution). (Don't worry about `linux-vdso.so.1`, it's a virtual library of sorts, provided by the kernel)

Add these to the Dockerfile and uncomment the second stage.

```
FROM debian

ADD main.c /
ADD Makefile /

RUN apt-get update -qq && \
 apt-get install -y gcc make && \
 make

FROM scratch
COPY --from=0 /hello /hello
COPY --from=0 /lib/x86_64-linux-gnu/libc.so.6 /lib/x86_64-linux-gnu/libc.so.6
COPY --from=0 /lib64/ld-linux-x86-64.so.2 /lib64/ld-linux-x86-64.so.2
```

Build, check the size, and run the image.

```
$ docker build -t 002_distribute .

$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
002_distribute             latest              9dcf67b068c6        33 seconds ago       1.85MB

$ docker run -it --rm 002_distribute /hello
hello!
```

Boom. Now a 230MB image has become a 1.85MB artifact, ready to be deployed. This process of slimming down ensures smaller storage requirements, faster deployments, and faster initialization once deployed. This process translates extremely well to new programming languages like Rust and Go, which enforce much greater levels of static linking than C/C++ do, and far outperform scripting languages in terms of size and speed. But similarly in terms of simplicity, any scripting language runtime could be copied into a container from scratch, it's dependencies resolved, and source code added alongside, to provide a slim package for a particular application.
