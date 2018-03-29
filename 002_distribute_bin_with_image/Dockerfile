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
