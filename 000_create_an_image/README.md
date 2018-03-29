# Creating a Docker image

Before we begin, you can check to see what images and containers are already installed on your system with:

```
// images
$ docker images

// containers
$ docker ps -a
```

Anyway, first things first, we have a Dockerfile

```
FROM debian

RUN apt-get update -qq && apt-get install -y gcc make
```

Line 1 declares what image base the image will start from. I've selected `debian`, but there are thousands and thousands of bases to choose from. Check [here](https://hub.docker.com/explore/) for a list of officially supported images.

Next we have a `RUN` command. To start, it's generally safe to assume your `RUN` commands will be executed by a shell like `/bin/bash` or `/bin/sh`. This particular run command will update the images package list, and install the latest versions of `gcc` and `make` (The Gnu C compiler, and Gnu Make for building).

To build the image:

```
$ docker build .
```

You'll see docker getting the latest copy of the source image, and running the specified `RUN` command. Now you should see output like the following from listing your images:

```
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
<none>                   <none>              a94f1f09ec00        6 minutes ago       235MB
debian                   latest              2b98c9851a37        2 weeks ago         100MB
```

You can see that you've pulled down the latest docker image, and created a new (untitled and untagged one). The image is untitled and untagged, because those items need to be specified at build time. Lets go ahead and do that now.

```
$ docker build -t 000_create .
```

And check your image list again:

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
000_create          latest              a94f1f09ec00        12 minutes ago      235MB
debian              latest              2b98c9851a37        2 weeks ago         100MB
```

Now we've named the image, and docker gives it the default `latest` tag. You can specify a different tag like this:

```
$ docker build -t 000_create:my_tag .
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
000_create          latest              a94f1f09ec00        12 minutes ago      235MB
000_create          my_tag              a94f1f09ec00        12 minutes ago      235MB
debian              latest              2b98c9851a37        2 weeks ago         100MB
```

And this is how Docker permits users to specify versions of images. You can see here that `latest` and `my_tag` share the same `IMAGE ID`. This is because Docker is smart enough to know that the image hasn't changed. Under the hood, Docker heavily utilizes a snapshotting mechanism to keep disk usage to a minimum (in fact, the `000_create` image on the filesystem is actually a 135MB layer which is applied on top of the `debian` image at runtime).

So now let's test the image:

```
$ docker run -it --rm 000_create /bin/bash

root@2e5846327dea:/# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

root@2e5846327dea:/# gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/6/lto-wrapper
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Debian 6.3.0-18+deb9u1' --with-bugurl=file:///usr/share/doc/gcc-6/README.Bugs --enable-languages=c,ada,c++,java,go,d,fortran,objc,obj-c++ --prefix=/usr --program-suffix=-6 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-libmpx --enable-plugin --enable-default-pie --with-system-zlib --disable-browser-plugin --enable-java-awt=gtk --enable-gtk-cairo --with-java-home=/usr/lib/jvm/java-1.5.0-gcj-6-amd64/jre --enable-java-home --with-jvm-root-dir=/usr/lib/jvm/java-1.5.0-gcj-6-amd64 --with-jvm-jar-dir=/usr/lib/jvm-exports/java-1.5.0-gcj-6-amd64 --with-arch-directory=amd64 --with-ecj-jar=/usr/share/java/eclipse-ecj.jar --with-target-system-zlib --enable-objc-gc=auto --enable-multiarch --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu
Thread model: posix
gcc version 6.3.0 20170516 (Debian 6.3.0-18+deb9u1) 

root@2e5846327dea:/# 
```

We inform docker to run the binary `/bin/bash`, using our image `000_create`, in interactive mode and allocating a tty `-it`, and to remove the container(essentially an instance of the image) it creates upon exit `--rm`. Once the shell has loaded, you can browse around like you would on any VM or server.
