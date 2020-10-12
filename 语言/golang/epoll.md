# epoll 在golang中的应用

```
一、之前一直使用java做网络编程的时候经常遇到NIO、netty等东西，知道其底层依赖的是操作系统实现的epoll,在使用golang做网络编程的时候发现都不需要关注这些东西，用好携程即可，是不是golang不需要epoll，以为携程搞定了一切，但是想到网络IO其实都需要内核线程处理，一时摸不着头脑。
二、查询资料得知golang网络编程也是使用epoll,只不过因为golang的重要特性携程，把epoll跟携程结合了，这里我简单的理解为：
  1、golang在网络编程通过epollcreate1、epollcreate调用系统方法创建epoll句柄，得到epoll的句柄epfd，以后所有的fd在epoll的事件注册都通过该epfd注册到epoll里, 
  2、epoll事件封装成了一个epollevent结构，将当前运行goroutine的pollDesc结构的地址放到epoll事件的data结构里，待从epoll获取ready事件时找到该goroutine并唤醒。
  3、代码编写的goroutine阻塞在了Read的时候，当有数据来了之后又是如何感知到的？我们知道epoll有个epoll_wait方法用来监听事件，该方法有单独一个常驻的goroutine在高效快速的轮询检测执行，当有网络包收到后，每次都会取最多128个事件，然后在ev.data里就是当初注册事件时的pollDesc，也就找到了要唤醒的G，唤醒之。
```

服务大致过程：

- 1.用户层开始监听一个地址:`net.Listen("tcp", ":8080")`
- 2.调用系统方法sys_socket取得一个fd
- 3.将第2步拿到的fd通过系统方法sys_bind绑定，即绑定本地监听地址、端口
- 4.在系统中初始化epoll
- 5.将2步拿到的fd注册到系统的epoll里，等待有建立连接的事件

客户端大致过程：

- 1.用户发起一个连接请求:`net.Dial("tcp", "127.0.0.1:8080")`
- 2.调用系统方法sys_socket取得一个fd
- 3.将第2步拿到的fd通过系统方法sys_bind绑定，即绑定本地监听地址、端口
- 4.调用系统方法sys_connect发起远程建立连接
- 5.在系统中初始化epoll(同一个进程下如果开启多个连接，只会初始化一次)
- 6.将第2步拿到的fd注册到系统epoll里，用于监听读写事件
- 

# epoll

在上文中对TCP预备的工作有了一些了解。下面继续对下层epoll的封装做一些分析，看下运行时是如何工作的。epoll我们更多的是关注服务端如何实现的，所以这里就只从server端的角度分析（下面假设你对epoll有一定的基础知识）。

## 如何实现异步非阻塞

我们知道epoll属于异步非阻塞IO模型，下面看下如何实现的异步及非阻塞。通过上文的代码结构的介绍，我们就不从头列出代码了，直接跳到关键的代码处看一下：



```go
// internal/poll/fd_unix.go

func (fd *FD) Read(p []byte) (int, error) {
    if err := fd.readLock(); err != nil {
        return 0, err
    }
    defer fd.readUnlock()
    if len(p) == 0 {
        // If the caller wanted a zero byte read, return immediately
        // without trying (but after acquiring the readLock).
        // Otherwise syscall.Read returns 0, nil which looks like
        // io.EOF.
        // TODO(bradfitz): make it wait for readability? (Issue 15735)
        return 0, nil
    }
    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return 0, err
    }
    if fd.IsStream && len(p) > maxRW {
        p = p[:maxRW]
    }
    for {
        n, err := syscall.Read(fd.Sysfd, p)
        if err != nil {
            n = 0
            if err == syscall.EAGAIN && fd.pd.pollable() {
                if err = fd.pd.waitRead(fd.isFile); err == nil {
                    continue
                }
            }

            // On MacOS we can see EINTR here if the user
            // pressed ^Z.  See issue #22838.
            if runtime.GOOS == "darwin" && err == syscall.EINTR {
                continue
            }
        }
        err = fd.eofError(n, err)
        return n, err
    }
}

// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
    if err := fd.writeLock(); err != nil {
        return 0, err
    }
    defer fd.writeUnlock()
    if err := fd.pd.prepareWrite(fd.isFile); err != nil {
        return 0, err
    }
    var nn int
    for {
        max := len(p)
        if fd.IsStream && max-nn > maxRW {
            max = nn + maxRW
        }
        n, err := syscall.Write(fd.Sysfd, p[nn:max])
        if n > 0 {
            nn += n
        }
        if nn == len(p) {
            return nn, err
        }
        if err == syscall.EAGAIN && fd.pd.pollable() {
            if err = fd.pd.waitWrite(fd.isFile); err == nil {
                continue
            }
        }
        if err != nil {
            return nn, err
        }
        if n == 0 {
            return nn, io.ErrUnexpectedEOF
        }
    }
}
```

上面是IO操作的读写方法，其中调用到`syscall.Read`、`syscall.Write`两个系统调用，但我们知道这两个系统调用并不会block住用户线程(系统方法可自行google)，但是我们上层应用代码确实会阻塞住，而真正实现block操作是在`fd.pd.waitRead(fd.isFile)`、`fd.pd.waitWrite(fd.isFile)`两个方法里。下面直接跳到关键实现的代码。



```go
// runtime/netpoll.go

func poll_runtime_pollWait(pd *pollDesc, mode int) int {
    err := netpollcheckerr(pd, int32(mode))
    if err != 0 {
        return err
    }
    // As for now only Solaris and AIX use level-triggered IO.
    if GOOS == "solaris" || GOOS == "aix" {
        netpollarm(pd, mode)
    }
    for !netpollblock(pd, int32(mode), false) {
        err = netpollcheckerr(pd, int32(mode))
        if err != 0 {
            return err
        }
        // Can happen if timeout has fired and unblocked us,
        // but before we had a chance to run, timeout has been reset.
        // Pretend it has not happened and retry.
    }
    return 0
}

// 如果IO已经ready则返回true，IO超时或者关闭返回false
// waitio设置为true的话，则忽略错误，让G进入休眠
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
    gpp := &pd.rg
    if mode == 'w' {
        gpp = &pd.wg
    }

    // set the gpp semaphore to WAIT
    for {
        old := *gpp
        if old == pdReady {
            *gpp = 0
            return true
        }
        if old != 0 {
            throw("runtime: double wait")
        }
        if atomic.Casuintptr(gpp, 0, pdWait) {
            break
        }
    }

    // need to recheck error states after setting gpp to WAIT
    // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
    // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
    if waitio || netpollcheckerr(pd, mode) == 0 { // netpollcheckerr(pd, mode) == 0 表明socket是正常的没有被关闭或者超时
        gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5) // 进入休眠,且在方法中把gpp指向了当前执行的G，即pd.rg指向当前G
    }
    // be careful to not lose concurrent READY notification
    old := atomic.Xchguintptr(gpp, 0)
    if old > pdWait {
        throw("runtime: corrupted polldesc")
    }
    return old == pdReady
}
```

实际通过`gopark`方法把当前的G休眠进入休眠，因此用户层感受到的Read阻塞住是因为通过休眠G实现的，而并没有阻塞到某个系统调用里，因此对于系统来说实现了`非阻塞IO`。

> gopark休眠相关实现参考我之前写的一片文章:[从源码角度看Golang的调度](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fthinkboy%2Fgo-notes%2Fblob%2Fmaster%2F%E4%BB%8E%E6%BA%90%E7%A0%81%E8%A7%92%E5%BA%A6%E7%9C%8BGolang%E7%9A%84%E8%B0%83%E5%BA%A6.md)

## epoll的创建与事件注册

先来看两个epoll api的封装:

**创建epoll句柄**



```swift
// runtime/netpoll_epoll.go

// 内核epoll_create方法创建epoll句柄
func netpollinit() {
    epfd = epollcreate1(_EPOLL_CLOEXEC) // epfd保存epoll句柄
    if epfd >= 0 {
        return
    }
    epfd = epollcreate(1024)
    if epfd >= 0 {
        closeonexec(epfd)
        return
    }
    println("runtime: epollcreate failed with", -epfd)
    throw("runtime: netpollinit failed")
}
```



```php
// system/sys_linux_amd64.s

// int32 runtime·epollcreate(int32 size);
TEXT runtime·epollcreate(SB),NOSPLIT,$0
    MOVL    size+0(FP), DI
    MOVL    $SYS_epoll_create, AX
    SYSCALL
    MOVL    AX, ret+8(FP)
    RET

// int32 runtime·epollcreate1(int32 flags);
TEXT runtime·epollcreate1(SB),NOSPLIT,$0
    MOVL    flags+0(FP), DI
    MOVL    $SYS_epoll_create1, AX
    SYSCALL
    MOVL    AX, ret+8(FP)
    RET
```

时间比较简单,通过`epollcreate1`、`epollcreate`调用系统方法创建epoll句柄，得到epoll的句柄epfd，以后所有的fd在epoll的事件注册都通过该epfd注册到epoll里。

**epoll事件注册**



```go
// runtime/netpoll_epoll.go

// 通过内核epoll_ctl方法在内核epoll注册一个新的fd到epfd中
func netpollopen(fd uintptr, pd *pollDesc) int32 {
    var ev epollevent
    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd 
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

epoll事件封装成了一个`epollevent`结构，将当前运行goroutine的的pollDesc结构的地址放到epoll事件的data结构里，待从epoll获取ready事件时找到该goroutine并唤醒。

## epoll的事件监听

goroutine阻塞在了Read的时候，当有数据来了之后又是如何感知到的？我们知道epoll有个`epoll_wait`方法用来监听事件，我们看下对该方法的封装。



```go
// runtime/netpoll_epoll.go

// 从epoll里获取已经ready事件，并找到因accept、read、write方法无数据进入休眠的G里面的pollDesc结构，通过该结构可以反向找到该G
func netpoll(block bool) gList {
    if epfd == -1 {
        return gList{}
    }
    waitms := int32(-1)
    if !block {
        waitms = 0
    }
    var events [128]epollevent
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms) // 通过epoll_pwait获取被内核ready的一批epoll event，最大128个
    if n < 0 {
        if n != -_EINTR {
            println("runtime: epollwait on fd", epfd, "failed with", -n)
            throw("runtime: netpoll failed")
        }
        goto retry
    }
    var toRun gList
    for i := int32(0); i < n; i++ {
        ev := &events[i]
        if ev.events == 0 {
            continue
        }
        var mode int32
        if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'r'
        }
        if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'w'
        }
        if mode != 0 {
            pd := *(**pollDesc)(unsafe.Pointer(&ev.data)) // ev.data的地址就是pollDesc结构对象地址,该结构对象初始化的时候设置的。

            netpollready(&toRun, pd, mode) // 唤醒G并加入到toRun列表里返回给上层
        }
    }
    if block && toRun.empty() {
        goto retry
    }
    return toRun
}
```

> 在我之前的文章<<从源码角度看Golang的调度>>里已经提到该方法是如何执行的，在这里只简单复述几点

该方法有单独一个常驻的goroutine在高效快速的轮询检测执行，当有网络包收到后，每次都会取最多128个事件，然后在ev.data里就是当初注册事件时的pollDesc，也就找到了要唤醒的G，唤醒之。

## 小结

go在epoll的过程中遵循了epoll的逻辑，借助go自身的goroutine调度机制实现**异步非阻塞IO**。



#### 参考资料

```
https://www.jianshu.com/p/3ff0751dfa04
```

