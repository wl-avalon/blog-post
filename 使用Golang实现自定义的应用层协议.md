title: 使用Golang自定义基于Tcp的应用层协议服务器
date: 2018年10月09日
tags:
 - Golang
 - 网络协议
 - Tcp
categories:
 - 技术沉淀
---
&emsp;&emsp;最近工作中遇到这样一个场景：服务端控制机器人执行动作，机器人和服务端都是C++写的，应用层协议是自定义的。然后要写一个模拟器，模拟机器人的逻辑，执行服务端下发的控制指令。以便服务端升级算法时，不需要机器人实际运行，只需要在模拟器运行一遍就知道算法的优劣。
&emsp;&emsp;由于公司的技术栈收敛成了Golang，所以模拟器也需要用Golang写。这样就出现了一个问题，需要用Golang实现服务端与机器人的通信协议，期间踩了不少坑，在这里复盘一下，从零搭建一个自定义应用层协议的服务器。
<!-- more -->
----------------
# 1. Hello World版服务器
## 1.1 服务端代码
```go
package server

import (
	"fmt"
	"net"
	"io"
)

func CreateServer() {
	//1 创建listener，内部创建了socket，开始监听端口
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		fmt.Println("Error listening", err.Error())
		return //终止程序
	}

	//2 持续等待来自客户端的连接并对连接进行处理
	for {
		//2.1 listener.Accept是一个阻塞方法，只有客户端连接时才会有返回
		fmt.Println("正在等待客户端的连接...")
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("接受连接失败,error:", err.Error())
			continue
		}
		fmt.Println("接收到了一个客户端的连接")

		//2.2 开goroutine对连接上来的客户端进行处理
		go handleClient(conn)
	}
}

func handleClient(conn net.Conn) {
	i := 0
	for {
		headBuffer		:= make([]byte, 20)
		//headBuffer		:= make([]byte, 3)
		_, err	:= conn.Read(headBuffer)
		if err != nil {
			if err == io.EOF {
				return
			}
			fmt.Println("从网络流中读取数据失败,error:", err.Error())
		}
		fmt.Printf("第%d次接收到客户端发送的数据：%s\n", i, string(headBuffer))
		i++

		conn.Write([]byte("0123456789"))
	}
	return
}
```