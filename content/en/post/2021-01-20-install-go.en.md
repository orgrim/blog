---
title: "Install Go"
date: 2021-01-20T21:58:54+01:00
categories:
- Code
tags:
- go
- install
---

The Go language has been around for quite some time and one finds
more and more programs written in Go. The Go compiler and tools can be
installed using the packages of the Linux distribution, but the
easiest way to have the latest version is to use the official compiled
tools.

The version of the `golang-go` package in Debian stable, 1.11, is too
old, so I installed the official binaries this way.

<!--more-->

Go to <https://golang.org/dl/> to find out what is the lastest
version. Here, we'll be using 1.15.7.

```
$ mkdir -p ~/install
$ cd ~/install
$ wget https://golang.org/dl/go1.15.7.linux-amd64.tar.gz
```

Copy the checksum in `go1.15.7.linux-amd64.tar.gz.sha256` as
`sha256sum` excepts it for checking:

```
0d142143794721bb63ce6c8a6180c4062bcf8ef4715e7d6d6609f3a8282629b3  go1.15.7.linux-amd64.tar.gz
```

Check the sum:

```
$ sha256sum -c go1.15.7.linux-amd64.tar.gz.sha256
go1.15.7.linux-amd64.tar.gz: OK
```

Prepare the installation directory:

```
$ cd /usr/local/
$ sudo rm -f go
```

Extract the tarball:

```
$ sudo tar xf ~/install/go1.15.7.linux-amd64.tar.gz
```

Rename the directory to add the version number and create a symbolic
link:

```
$ sudo mv go go1.15.7
$ sudo ln -s go1.15.7 go
```

Update the PATH in `~/.bashrc` :

```
PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
export PATH
```

Source it and check the version:

```
$ . ~/.bashrc
$ go version
go version go1.15.7 linux/amd64
```

The older versions can be purged from `/usr/local` or we can go back to
a previous version by updating the symbolic link.

The defaut environment configuration of Go is used to let `go get` and
others install binaires and modules into `~/go`.
