---
title: 使用Golang自定义基于Tcp的应用层协议 2:解决黏包与分包
date: 2019-10-24
updated: 2019-10-26
tags:
 - Golang
 - 网络协议
 - Tcp
categories:
 - 技术沉淀
---

&emsp;&emsp;没人能想到这一年来老夫经历了什么……天可怜见曾经茂盛的头发日渐稀疏……<br>&emsp;&emsp;废话不多说，上回书说到由于TCP协议是一种流式传输的网络通信协议，因此一个完整的报文可能会切割为多次报文被接收方接到（分包），或者多个包合在一起被接收方收到（黏包），最终接收方收到的信息与发送方实际发送的有很大差异，由此可知TCP协议本身很难作为网络通信协议被我们开发的程序直接使用。<br>&emsp;&emsp;因此接收方和发送方需要相互之间约定好一个固定的数据格式，大家都以这个格式进行通信，这个约定好的数据格式便被成为“应用层协议”。<br><!-- more -->

----------------

# 发现问题

&emsp;&emsp;从“应用层协议”的需要解决的问题出发，我可以知道通过协议约定好的这个数据结构需要解决以下两个问题。<br>&emsp;&emsp;1. 如何将被切成多个零碎包的数据包恢复为原始的数据包。<br>&emsp;&emsp;2. 如何将多个原始数据包黏成的一个大数据包，切分成多个原始数据包。<br>&emsp;&emsp;透过现象看本质则会发现，这两个问题的本质其实是：<font color = #bb505d>接收方不知道待接收数据包的开始与结束标志</font>



# 解决问题

&emsp;&emsp;既然知道了本质问题那就好办了，现在只需要给每个数据包加上起始和截止的标记就可以了。<br>&emsp;&emsp;首先给整个数据包加一个特殊的字符串作为起始标识，比如：XDZG。然后再看截止的标识。<br>&emsp;&emsp;截止的标识虽然也可以用特殊字符串标记，但是如果数据包实际想要传输的内容里刚好有的值与截止字符串相等就尴尬了，整个报文会提前被认为结束。因此比较稳妥的办法看起来应该是加一个整数标记整个数据包的长度，读完这个整数长度的字节流便可以认为一个数据包被读完了。<br>&emsp;&emsp;说到截止的标识不能用特殊字符串，是因为如果在真正想要传输的数据中有这个字符串时，会导致提前终止；反过来考虑起始的标识位同样可能存在于实际想要传输的数据中，或者存在于网络中的数据乱流。这时就可能会导致读取到错误的数据包。咋整呢……<br>&emsp;&emsp;起始的标识除了用特殊字符以外，暂时似乎没有更好的办法了，那没办法，只能退而求其次，当读到这个错误数据包时，有办法让接收方确认这个包是一个错误的包即可。同时需要注意一下为了防止消息内容被篡改或者网络传输中导致的脏数据（理论上TCP的机制保证不会有脏数据才对），有时需要给body也增加增加一个校验字段，生成逻辑与报头校验和类似。<br>&emsp;&emsp;因此除了起始标识、数据包长度以外，可以考虑再加两个数据校验的字段，先将整个body进行一次MD5，得到一个body的校验和作为报头的一个字段，然后将起始标识、数据包长度、body校验和这三个字段累加成一个长字符串，然后对这个长字符串做一次MD5，再将MD5得到的字符串作为报头的第四个字段。<br>&emsp;&emsp;这样接收方读到信息后起始标识、数据长度以及body校验和对应的字段累加成长字符串，然后MD5得到校验和以后，再与实际接收到的校验和做一下比较，如果相等，再对收到的body做MD5，与收到的body校验和最比对，如果这里也比对成功，则可以证明这个数据包九成九是正确的数据包了。为啥不是十成，因为非洲酋长真的运气不好啊orz……<br>&emsp;&emsp;至此，一个简易的应用层协议报头就有了，结构体定义大概可以如下代码所示：<br>&emsp;&emsp;博主善意的PS：实际业务场景中，用MD5做应用层协议的校验和不太合适，太长了而且算起来也比较费劲，更多的可能会用类似CRC等算法，这里只是为了偷懒就先用MD5了。

```go
type Head struct {
	Mark        string
	BodyLength  uint64
	BodyMd5     [16]byte
	headMD5     [16]byte
}
```

&emsp;&emsp;具体代码及释义如下：<br>&emsp;&emsp;代码地址：https://github.com/xyb-blog-example/tcp-server/tree/master/v2

# 通用协议定义

&emsp;&emsp;RecDataPack方法定义了一次收包流程，每调用一次，便会阻塞，直到收到一个完整的数据包才会返回。返回值是实际需要接收的数据，不包含报头。整个流程为一个死循环，直到收到合法的body才会返回。第一步先使用recMsgHead方法，获取合法的数据头，然后根据数据头中显示的body长度，获取对应长度的body。

```go
func RecDataPack(conn net.Conn) ([]byte, error) {
	recDone         := false      //整个数据包是否已经读完了
	curReadHeadSize := uint64(0)  //目前已经读到报头长度。由于一次读取网络流可能读不完，所以需要用for循环读，记录之前已经读到的长度
	curReadBodySize := uint64(0)  //目前已经读到数据体长度。由于一次读取网络流可能读不完，所以需要用for循环读，记录之前已经读到的长度
	//buffer作为缓冲区，为了接受较长的body，可以随便设置长一点，这里也可以不限制长度，比较随意，因为后面实际读取的时候会动态扩容
	var buffer []byte 
	for ; !recDone ; {
		//1 获取请求报头，recMsgHead在后续定义
		readSize, err := recMsgHead(conn, curReadHeadSize, &buffer)
		if err != nil {
			curReadHeadSize += readSize
			//1.1 HEAD_ERROR代表数据获取成功，但是报头不合法，需要重新获取报头，TODO:可以考虑是不是打个日志啥的
			if err == HeadError {
				fmt.Printf("读取头部数据错误，内容为:%+v", buffer[0:curReadHeadSize])
				continue
			}
			return nil, err
		}
		curReadHeadSize += readSize

		//2 获取请求Body，recMsgBody在后续定义
		head := convertBytesToHeader(buffer)
		//这里把报头传进去的原因是内部流程需要从head中获取body长度，以及body校验和
		readSize, err = recMsgBody(conn, head, &buffer, curReadBodySize)
		if err != nil {
			curReadBodySize += readSize
			//2.1 BODY_ERROR代表数据获取成功，但是Body不合法，需要重新获取报头
			if err == BodyError {
				fmt.Printf("读取Body数据错误，内容为:%+v", buffer[curReadHeadSize:curReadBodySize])
				continue
			}
			return nil, err
		}
		curReadBodySize += readSize
		recDone = true
	}
	return buffer[HeadSize:], nil
}
```

&emsp;&emsp;recMsgHead的作用收取一个完整的报头，内部逻辑是读取约定好的报头长度的字节流到内存，因为有可能一次读不完，所以需要for循环持续读，直到读到等于报头长度的字节流。读完对应长度的字节流以后，需要对这段字节流进行报头以及校验和的检查，判断是不是一个合法的报头；如果不是一个合法的报头则抛弃字节流的第一个字节，然后读入一个新的字节，重新检查报头，直到找到一个合法的报头。

```go
/**
* 函数名：recMsgHead
* 功能描述：获取请求报头，这个函数可重入，如果body获取失败，可以重新获取报头
* 参数1：buffer byte切片，存储从网络流中读取的数据
* 参数2：curReadSize uint64，代表当前包总共读取的字节数
* 返回值1：size uint64，此次读取的字节数，读取失败时为0
* 返回值2：err，返回读取错误
*/
func recMsgHead(conn net.Conn, curReadSize uint64, buffer *[]byte) (size uint64, err error) {
	//1 获取字节流，直到获取到headSize大小的数据。
	headSize         := HeadSize
	thisTimeReadSize := uint64(0)
	for ; curReadSize < headSize ; {
		waitReadSize  := headSize - curReadSize
		headBuffer    := make([]byte, waitReadSize)
		readSize, err := conn.Read(headBuffer)
		if err != nil {
			return thisTimeReadSize, err
		}
		if readSize <= 0 {
			continue
		}
		//buffer长度不足以接收后续长度的字节流时，扩容buffer
		if uint64(len(*buffer)) <= curReadSize + uint64(readSize) {
			newBuffer := make([]byte, curReadSize + uint64(readSize))
			*buffer   = append(*buffer, newBuffer...)
		}
		copy((*buffer)[curReadSize:], headBuffer[:readSize])
		curReadSize       += uint64(readSize)
		thisTimeReadSize  += uint64(readSize)
	}

	//2 校验头部是否完整
	checkErr := checkHead(*buffer)
	if !checkErr {
		//2.1 去掉头部第一个字节
		index := uint64(0)
		for ; index < curReadSize - 1 ; index++ {
			(*buffer)[index] = (*buffer)[index + 1]
		}
		//2.2 再读入一个字节
		oneByteBuffer := make([]byte, 1)
		for {
			readSize, err := conn.Read(oneByteBuffer)
			if err != nil {
				return thisTimeReadSize, err
			}
			if readSize < 1 {
				continue
			}
			break
		}
		//2.3 把读入的字节接到之前的buffer后面
		(*buffer)[index] = oneByteBuffer[0]
		return thisTimeReadSize, HeadError
	}
	return thisTimeReadSize, nil
}

func checkHead(buffer []byte) bool {
  //1. 将字节流转换为报头结构体
  head := convertBytesToHeader(buffer)
  //2. 检查起始标识位是否与约定的一致
	if head.Mark != HeadMark {
		return false
	}
  //3. 检查头部校验和是否合法
	prefix := buffer[0:HeadSize-16]
	md5Byte := md5.Sum(prefix)
	if !reflect.DeepEqual(md5Byte, head.HeadMd5) {
		return false
	}
	return true
}

func convertBytesToHeader(headBuffer []byte) *Head {
	head 		:= new(Head)
	//由于约定好了报头格式，因此HeadSize也就确定了，定义成常量即可，此处报头长度为44个字节
	headSize 	:= HeadSize
	if uint64(len(headBuffer)) < headSize {
		return nil
	}

	head.Mark = string(headBuffer[0:4])              //0-3个字节是起始标记
	BytesToInt(headBuffer[4:12], &(head.BodyLength)) //4-11个字节是body长度
	copy(head.BodyMd5[0:16], headBuffer[12:28])      //12-28个字节是body校验和
	copy(head.HeadMd5[0:16], headBuffer[28:44])      //28-44个字节是head校验和
	return head
}

func BytesToInt(b []byte, i interface{}) interface{} {
	bytesBuffer := bytes.NewBuffer(b)
	//经过网络传输的字节流均为小端模式，具体大小端的相关知识不在此赘述
	binary.Read(bytesBuffer, binary.LittleEndian, i)
	return i
}
```

&emsp;&emsp;recMsgBody方法的作用是收取一个合法的消息体，内部逻辑与recMsgHead类似，根据报头获取到消息体的长度，然后将对应消息体长度的字节流读到内存。如果根据校验和检查body不合法，则将报头的第一个字符丢掉，向后继续读入一个字节，重新走校验报头的逻辑。

```go
/**
* 函数名：recMsgBody
* 功能描述：获取请求Body，这个函数可重入
* 参数1：head byte切片，存储报头的内容，以便获取
* 参数2：buffer byte切片，存储从网络流中读取的数据
* 参数3：curReadSize uint64，代表当前包总共读取的字节数
* 返回值1：size uint64，此次读取的字节数，读取失败时为0
* 返回值2：err，返回读取错误
*/
func recMsgBody(conn net.Conn, head *Head, buffer *[]byte, curReadSize uint64) (size uint64, err error) {
	//1 从Head中获取Body的长度，并初始化buffer大小
	bodySize 	:= head.BodyLength

	//2 获取字节流，直到获取到bodySize大小的数据。
	thisTimeReadSize := uint64(0)
	for ; curReadSize < bodySize ; {
		waitReadSize  := bodySize - curReadSize
		bodyBuffer    := make([]byte, waitReadSize)
		readSize, err := conn.Read(bodyBuffer)
		if err != nil {
			return thisTimeReadSize, err
		}
		if readSize <= 0 {
			continue
		}
		//buffer长度不足以接收后续长度的字节流时，扩容buffer
		if uint64(len(*buffer)) <= HeadSize + curReadSize + uint64(readSize) {
			newBuffer := make([]byte, HeadSize + curReadSize + uint64(readSize))
			*buffer   = append(*buffer, newBuffer...)
		}
		copy((*buffer)[HeadSize + curReadSize:], bodyBuffer[:readSize])
		curReadSize       += uint64(readSize)
		thisTimeReadSize  += uint64(readSize)
	}

	//3 调用实现类的checkBody方法，校验Body是否完整
	checkErr	:= checkBody(head, (*buffer)[HeadSize:HeadSize+curReadSize])
	if !checkErr {
		//3.1 去掉头部第一个字节
		index := uint64(0)
		for ; index < HeadSize + curReadSize - 1 ; index++ {
			(*buffer)[index] = (*buffer)[index + 1]
		}
		//3.2 再读入一个字节
		oneByteBuffer := make([]byte, 1)
		for {
			readSize, err := conn.Read(oneByteBuffer)
			if err != nil {
				return thisTimeReadSize, err
			}
			if readSize < 1 {
				continue
			}
			break
		}
		//3.3 把读入的字节接到之前的buffer后面
		(*buffer)[index] = oneByteBuffer[0]
		return thisTimeReadSize, BodyError
	}
	return thisTimeReadSize, nil
}
```

&emsp;&emsp;接收报文的逻辑基本就是上述代码了，接下来便是发送报文。发送报文的逻辑会简单很多，只需要将数据按照约定好的格式发送到网络流中，同时保证所有的内容都发送完即可。具体代码如下：

```go
func SendTcpMsg(conn net.Conn, bodyBuffer []byte) (err error) {
	//1. 按照约定的格式封装报头结构体
	head := Head{
		Mark: HeadMark,
		BodyLength: uint64(len(bodyBuffer)),
		BodyMd5: md5.Sum(bodyBuffer),
	}

	//2 将报头结构体转换为字节流
	var headBuffer []byte
	headBuffer = append(headBuffer, []byte(head.Mark)...)
	headBuffer = append(headBuffer, IntToBytes(head.BodyLength)...)
	headBuffer = append(headBuffer, head.BodyMd5[0:16]...)
	md5Str := md5.Sum(headBuffer)
	headBuffer = append(headBuffer, md5Str[0:16]...)
	buffer := append(headBuffer, bodyBuffer...)
	//真实业务在使用时，这里可以考虑加锁，防止并发写，因为sendMsg本身不是原子操作。
	err		= sendMsg(conn, buffer)
	return err
}

func sendMsg (conn net.Conn, buffer []byte) (err error){
	allSize		:= 0 //已经发送的字节数
	nowIndex	:= 0 //当前发送到的字节标记位
	bufferSize 	:= len(buffer)
	for {
		//写入网络流
		currSize, err := conn.Write(buffer[nowIndex:])
		if err != nil {
			return err
		}
		allSize += currSize
		if allSize >= bufferSize {
			break
		}
		nowIndex = allSize
	}
	return nil
}
```

&emsp;&emsp;至此一个简单的应用层协议就完成了，整个流程简化来看很简单，主要分为以下六个步骤：<br>&emsp;&emsp;1. 接收报头数据。<br>&emsp;&emsp;2. 校验报头数据，如果报头不合法删除已读入字节流的第一个字节，从网络流多读一个字节追加到已读入字节流的尾部。<br>&emsp;&emsp;3. 持续第2步，直到获取到合法的报头数据。<br>&emsp;&emsp;4. 根据报头中的body长度字段，从网络流中获取对应长度的字节流<br>&emsp;&emsp;5. 校验body数据，如果不合法，则删除已读入字节流的第一个字节，从网络流多读一个字节追加到已读入字节流的尾部。<br>&emsp;&emsp;6. 回到第2步开始向下执行，直到获取到合法的body数据。

&emsp;&emsp;可以看出整个流程其实是具有通用性的，不同的应用协议只有检查报头、检查Body、字节流转换Head、字节流转换Body、报头转字节流、Body转字节流这几个流程有区别。因此我们可以定义一套TCP通信框架，供大部分私有的TCP协议使用。<br>&emsp;&emsp;框架代码地址：https://github.com/xyb-blog-example/tcp-framework

&emsp;&emsp;博主善意的PS：之所以说是大部分私有TCP协议使用而不是全部的TCP协议使用，是因为很多基于TCP的应用层协议为了通用性使得报头的长度也是变长，所以无法提前约定好HeadSize，比如最知名的HTTP协议。具体HTTP协议在golang中是如何实现，以及HTTP具体的协议格式会在后续的博文中讲解。