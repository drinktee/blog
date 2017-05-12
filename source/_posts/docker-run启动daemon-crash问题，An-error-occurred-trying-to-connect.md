---
title: docker启动容器后，daemon crash问题，An error occurred trying to connect
date: 2017-04-16 10:44:54
tags:
---
最近运行docker run 命令后出现了，容器无法启动的情况，出现了以下信息。

```
$ docker run -d  registry.baidu.com/public/nginx_1-8-0:1.0.1
33d0ea46c4015f5ec6dee16475b0e66b3077cca6214c6d797c5967bbe2279307

An error occurred trying to connect: Post http:///var/run/docker.sock/v1.21/containers/33d0ea46c4015f5ec6dee16475b0e66b3077cca6214c6d797c5967bbe2279307/start: EOF
```

查看日志后发现panic,docker进程挂了。
```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow

runtime stack:
runtime.throw(0x1d967b9)
        /usr/local/go/src/runtime/panic.go:491 +0xad
runtime.newstack()
        /usr/local/go/src/runtime/stack.c:784 +0x555
runtime.morestack()
        /usr/local/go/src/runtime/asm_amd64.s:324 +0x7e
goroutine 67 [stack growth]:
path/filepath.(*lazybuf).append(0xc228c102a8, 0x2f)
        /usr/local/go/src/path/filepath/path.go:35 fp=0xc228c101f0 sp=0xc228c101e8
path/filepath.Clean(0xc208869e60, 0x1, 0x0, 0x0)
        /usr/local/go/src/path/filepath/path.go:103 +0x1e4 fp=0xc228c102f8 sp=0xc228c101f0
path/filepath.Dir(0xc208869e60, 0x1, 0x0, 0x0)
        /usr/local/go/src/path/filepath/path.go:454 +0xdd fp=0xc228c10360 sp=0xc228c102f8
github.com/opencontainers/runc/libcontainer/cgroups/fs.(*CpusetGroup).ensureParent(0x1dbd8c0, 0xc208869e60, 0x1, 0xc2088b128d, 0x8, 0x0, 0x0)
        /go/src/github.com/docker/docker/vendor/src/github.com/opencontainers/runc/libcontainer/cgroups/fs/cpuset.go:91 +0x50 fp=0xc228c103e0 sp=0xc228c10360
goroutine 10 [chan receive, 1 minutes]:
github.com/docker/docker/api/server.(*Server).ServeAPI(0xc208037980, 0x0, 0x0)
        /go/src/github.com/docker/docker/api/server/server.go:94 +0x1b6
main.func·007()
        /go/src/github.com/docker/docker/docker/daemon.go:255 +0x3b
created by main.(*DaemonCli).CmdDaemon
        /go/src/github.com/docker/docker/docker/daemon.go:261 +0x1571        

```

google之后，发现这个[ISSUE](https://github.com/docker/docker/issues/18048) 有点类似，bschiffthaler将内核升级到`3.13.0-70`之后，修复了此问题。


> @tiborvass Seems it was an issue with the kernel. The error persisted through a reboot, but upgrading the kernel to 3.13.0-70 fixed the issue. We have other machines with kernels older than 3.13.0-34 and docker runs just fine, I would chuck it up to a systems "hiccup".
I'll be closing the issue with this.

此问题也可能与机器有关，相同内核的不同机器，也会出现此问题。
按照ISSUE中所说，在1.10以上版本已经修复此问题。

>@sanwan the error discussed here was in docker 1.10 and older, and was resolved; if you're running the current version of docker, please open a new issue with more details (as requested in the bug report template), or search if there's an existing open issue.



