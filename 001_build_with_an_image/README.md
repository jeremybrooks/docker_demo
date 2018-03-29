# Building with an image

In this demo, I'll use the previously created image to build a binary from sources that live outside of the image. You've still got the image from 000, right?

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
000_create          latest              a94f1f09ec00        32 minutes ago      235MB
000_create          my_tag              a94f1f09ec00        32 minutes ago      235MB
debian              latest              2b98c9851a37        2 weeks ago         100MB
```

Have a quick look at the source here:
```
$ cat main.c 

#include <stdio.h>

int main(){
	printf("hello!\n");
	return 0;
}

$ cat Makefile 

all:
	gcc -o hello main.c

```

A simple hello world example in C, and a makefile to tell the compiler how to build it. Now we'll build with:

```
$ docker run -it --rm -v $(pwd):/build -w /build 000_create make
gcc -o hello main.c

$ ls
hello  main.c  Makefile  README.md
```

Here you'll notice two new flags. `-v` (volume) is essentially a mount. You must specify the directory to be used by the container, and the directory to mount to inside the container. This will result in a new top level directory called `/build` which will hold all the files in the current working directory.

Next is the `-w` flag, which specifies the directory in which your container session will start. Here, it's used so that we don't have to tell docker `cd /build && make`.

You can see that make calls the compiler to build the hello binary. The container immediately exits when the specified process returns. `ls` and you'll see the `hello` binary in your current directory! If you're on a Linux machine, feel free to run it. If you're not on Linux, we'll be running it in the next demo.
