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

Line 1 declares what image base the image will start from. I've selected Debian, but there are thousands and thousands of bases to choose from. Check [here](https://hub.docker.com/explore/) for a list of officially supported images.

Next we have a RUN command. To start, it's generally safe to assume your RUN commands will be executed by a shell like /bin/bash or /bin/sh. This particular run command will update the images package list, and install the latest versions of gcc and make (The Gnu C compiler, and Gnu Make for building).

To build the image:

```
$ docker build .
```

You'll see docker getting the latest copy of the source image, and running the specified RUN command. Now you should see output like the following from listing your images:

```
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
<none>                   <none>              a94f1f09ec00        6 minutes ago       235MB
debian                   latest              2b98c9851a37        2 weeks ago         100MB
```

You can see that you've pulled down the latest docker image, and created a new (untitled and untagged one). The image is untitled and untagged, because those items need to be specified at build time. Lets go ahead and do that now.

```
$ docker build -t 000_create
```

And check your image list again:

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
000_create          latest              a94f1f09ec00        12 minutes ago      235MB
debian              latest              2b98c9851a37        2 weeks ago         100MB
```

Now we've named the image, and docker gives it the default "latest" tag. You can specify a different tag like this:

```
$ docker build -t 000_create:my_tag
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
000_create          latest              a94f1f09ec00        12 minutes ago      235MB
000_create          my_tag              a94f1f09ec00        12 minutes ago      235MB
debian              latest              2b98c9851a37        2 weeks ago         100MB
```

And this is how Docker permits users to specify versions of images.
