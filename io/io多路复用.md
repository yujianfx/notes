# io多路复用

## select

```go
ln, err := net.Listen("tcp", ":8080")
	if err != nil {
		fmt.Println(err)
		return
	}
	defer ln.Close()
```



```go
	sockfds := make([]int, 1)
	sockfds[0] = int(ln.File().Fd())

	for {
		n, _, err := syscall.Select(len(sockfds), &sockfds, nil, nil, nil)
		if err != nil {
			fmt.Println(err)
			break
		}

		if n > 0 {
			for i := 0; i < n; i++ {
				conn, err := ln.Accept()
				if err != nil {
					fmt.Println(err)
					continue
				}
				go handleConnection(conn)
			}
		}
	}
}
```



```go
func handleConnection(conn net.Conn) {
	defer conn.Close()

	fmt.Println("Accepted connection from", conn.RemoteAddr())
	// Your code here
}
```



默认最大1024个**fd**

与平台无关

## poll

```go
ln, err := net.Listen("tcp", ":8080")
if err != nil {
    fmt.Println(err)
		return
}
defer ln.Close()
fds := []syscall.PollFd{
    {
        Fd: int32(ln.File().Fd()),
        Events: syscall.POLLIN,
    },
}
```

```go
for {
	n, err := syscall.Poll(fds, -1)
	if err != nil {
		fmt.Println(err)
		break
	}

	if n > 0 {
		for i := 0; i < n; i++ {
			conn, err := ln.Accept()
			if err != nil {
				fmt.Println(err)
				continue
			}
			go handleConnection(conn)
		}
	}
}
```

```go
func handleConnection(conn net.Conn) {
	// 处理连接
	defer conn.Close()
}
```



## epoll

```go
// 创建监听套接字
ln, err := net.Listen("tcp", ":8080")
if err != nil {
	fmt.Println(err)
    return
}
defer ln.Close()

```



```go
// 创建epoll
fd, err := syscall.EpollCreate1(0)
if err != nil {
    fmt.Println(err)
	return
}
defer syscall.Close(fd)
```

```go
// 将监听套接字加入epoll
var event syscall.EpollEvent
event.Events = syscall.EPOLLIN
event.Fd = int32(ln.File().Fd())
if err := syscall.EpollCtl(fd, syscall.EPOLL_CTL_ADD, int(ln.File().Fd()), &event); err!= nil {
	fmt.Println(err)
    return
}
```



```go
// 循环处理事件
events := make([]syscall.EpollEvent, 100)
for {
	// 等待事件
	n, err := syscall.EpollWait(fd, events, 100)
	if err != nil {
        fmt.Println(err)
        return
    }
    // 遍历事件
	for i := 0; i < n; i++ {
		if int(events[i].Fd) == int(ln.File().Fd()) {
			// 有新连接
			conn, err := ln.Accept()
			if err != nil {
				fmt.Println(err)
				continue
			}
			// 将新连接加入epoll
			event.Events = syscall.EPOLLIN
			event.Fd = int32(conn.File().Fd())
			if err := syscall.EpollCtl(fd, syscall.EPOLL_CTL_ADD, int(conn.File().Fd()), &event); err != nil {
                fmt.Println(err)
                continue
				}
			} else {
				// 有数据可读
				fmt.Println("data ready")
			}
		}
	}
}
```

没有最大**fd**限制

linux独有