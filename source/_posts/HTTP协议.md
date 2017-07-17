---
title: HTTP服务 socket的简单实现
tags: 
- socket
- HTTP
- 网络
categories: 网络
---
## HTTP
### HTTP协议
http协议是应用层协议，它建立在传输层之上,默认端口80，客户端向服务器发送请求，服务器收到请求之后，对其内容进行解析并向客户端回送一个响应

#### HTTP请求报文
HTTP请求分三个部份：状态行，请求头，消息主体。类似下在这样：

	<method> <request-URL> <version>
	<headers>
	/r/n(这是一个空行)
	<entity-body>
HTTP定义了与服务器交互的不同方法，最基本四种GET,POST,PUT,DELETE。URL的全称是资源描述符，可以这样认为：一个URL地址，它用于描述一个网络上的资源，而HTTP中的GET,POST,PUT,DELETE,分别对应这个资源的查，增，改，删4个操作，

GET请求报文示例：

	 GET /books/?sex=man&name=Professional HTTP/1.1
	 Host: www.example.com
	 User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
	 Gecko/20050225 Firefox/1.0.1
	 Connection: Keep-Alive
	
POST请求表示可能改变服务器上的资源的请求：

	 POST / HTTP/1.1
	 Host: www.example.com
	 User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
	 Gecko/20050225 Firefox/1.0.1
	 Content-Type: application/x-www-form-urlencoded
	 Content-Length: 40
	 Connection: Keep-Alive
	
	 sex=man&name=Professional  

#### HTTP响应报文

HTTP响应与HTTP请求相似，也由3个部分构成，分别是：

* 状态行
* 响应头(Response Header)
* /r/n (空行)
* 响应正文

状态行由协议版本、数字形式的状态代码、及相应的状态描述，各元素之间以空格分隔。
常见的状态码有如下几种：

* 200 OK 客户端请求成功
* 301 Moved Permanently 请求永久重定向
* 302 Moved Temporarily 请求临时重定向
* 304 Not Modified 文件未修改，可以直接使用缓存的文件。
* 400 Bad Request 由于客户端请求有语法错误，不能被服务器所理解。
* 401 Unauthorized 请求未经授权。这个状态代码必须和WWW-Authenticate报头域一起使用
* 403 Forbidden 服务器收到请求，但是拒绝提供服务。服务器通常会在响应正文中给出不提供服务的原因
* 404 Not Found 请求的资源不存在，例如，输入了错误的URL
* 500 Internal Server Error 服务器发生不可预期的错误，导致无法完成客户端的请求。
* 503 Service Unavailable 服务器当前不能够处理客户端的请求，在一段时间之后，服务器可能会恢复正常。

下面是一个HTTP响应的例子

	HTTP/1.0 200 OK 
	Content-Type: text/plain
	Content-Length: 137582
	Expires: Thu, 05 Dec 1997 16:00:00 GMT
	Last-Modified: Wed, 5 August 1996 15:55:28 GMT
	Server: Apache 0.84
	
	<html>
	  <body>Hello World</body>
	</html>

#### HTTP首部字段

* Content-Type / Accept  
1.0版本规定，头信息是asscii码，后面的信息可以任何格式，因此服务器回应的时候必须告诉客户端，数据是什么格式，这就是Conntent-Type的作用。常见的Content-Type字段值如下
			
		text/plain
		text/html
		text/css
		image/jpeg
		image/png
		image/svg+xml
		audio/mp4
		video/mp4
		application/javascript
		application/pdf
		application/zip
		application/atom+xml  

 这些数据类型总称为**MIME type,**每个值，每个值包括一级类型和二级类型，之间用斜杠分隔。MIME type还可以在尾部使用分号，添加参数。

	Content-Type: text/html; charset=utf-8
	

* Connection  
HTTP/1.0 版的主要缺点是，每个TCP连接只能发送一个请求。发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接。
TCP连接的新建成本很高，因为需要客户端和服务器三次握手，并且开始时发送速率较慢（slow start）。所以，HTTP 1.0版本的性能比较差。随着网页加载的外部资源越来越多，这个问题就愈发突出了。
为了解决这个问题，有些浏览器在请求时，用了一个非标准的Connection字段。

		Connection: keep-alive

 这个字段要求服务器不要关闭TCP连接，以便其他请求可以复用。服务器同样回应这个字段。

		Connection: keep-alive
		
 HTTP/1.1引入了持久连接，TCP连接默认不关闭，不用声明

* Content-Encoding / Accept-Encoding  
由于发送的数据可以是任何格式，因此可以把数据压缩后再发送。Content-Encoding字段说明数据的压缩方法。

#### HTTP服务 socket简单实现
	//
	//  main.c
	//  HTTPSocketServer
	//
	//  Created by xufan on 2017/7/13.
	//  Copyright © 2017年 xufan. All rights reserved.
	//
	
	#include <stdio.h>
	#include <stdlib.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <sys/event.h>
	#include <sys/types.h>
	#include <sys/time.h>
	#include <arpa/inet.h>
	#include <string.h>
	#include <unistd.h>
	#define BACKLOG 5 //完成三次握手但没有accept的队列的长度
	#define CONCURRENT_MAX 8 //应用层同时可以处理的连接
	#define SERVER_PORT 1024
	#define BUFFER_SIZE 1024
	#define QUIT_CMD ".quit"
	int client_fds[CONCURRENT_MAX];
	struct kevent events[10];//CONCURRENT_MAX + 2
	
	void sendHello();
	
	int main (int argc, const char * argv[])
	{
	//    sendHello();
	//    return 0;
	    char input_msg[BUFFER_SIZE];
	    char recv_msg[BUFFER_SIZE];
	    //本地地址
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(SERVER_PORT);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&(server_addr.sin_zero),8);
	    //创建socket
	    int server_sock_fd = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_sock_fd == -1) {
	        perror("socket error");
	        return 1;
	    }else {
	        printf("socket success\n");
	    }
	    
	    int flag =1;
	    /**/if( setsockopt(server_sock_fd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(int)) == -1)
	    {
	        perror("setsockopt");
	        exit(1);
	    }
	    //绑定socket
	    int bind_result = bind(server_sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
	    if (bind_result == -1) {
	        perror("bind error");
	        return 1;
	    }else {
	        printf("bind success\n");
	    }
	    //listen
	    if (listen(server_sock_fd, BACKLOG) == -1) {
	        perror("listen error");
	        return 1;
	    }else {
	        printf("listen success\n");
	    }
	    struct timespec timeout = {10,0};
	    //kqueue
	    int kq = kqueue();
	    if (kq == -1) {
	        perror("创建kqueue出错!\n");
	        exit(1);
	    }
	    struct kevent event_change;
	    EV_SET(&event_change, STDIN_FILENO, EVFILT_READ, EV_ADD, 0, 0, NULL);
	    kevent(kq, &event_change, 1, NULL, 0, NULL);
	    EV_SET(&event_change, server_sock_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
	    kevent(kq, &event_change, 1, NULL, 0, NULL);
	    while (1) {
	        int ret = kevent(kq, NULL, 0, events, 10, &timeout);
	        if (ret < 0) {
	            printf("kevent 出错!\n");
	            continue;
	        }else if(ret == 0){
	            printf("kenvent 超时!\n");
	            continue;
	        }else{
	            //ret > 0 返回事件放在events中
	            for (int i = 0; i < ret; i++) {
	                struct kevent current_event = events[i];
	                //kevent中的ident就是文件描述符
	                if (current_event.ident == STDIN_FILENO) {
	                    //标准输入
	                    bzero(input_msg, BUFFER_SIZE);
	                    fgets(input_msg, BUFFER_SIZE, stdin);
	                    //输入 ".quit" 则退出服务器
	                    if (strcmp(input_msg, QUIT_CMD) == 0) {
	                        exit(0);
	                    }
	                    for (int i=0; i<CONCURRENT_MAX; i++) {
	                        if (client_fds[i]!=0) {
	                            send(client_fds[i], input_msg, BUFFER_SIZE, 0);
	                        }
	                    }
	                }else if(current_event.ident == server_sock_fd){
	                    //有新的连接请求
	                    struct sockaddr_in client_address;
	                    socklen_t address_len;
	                    int client_socket_fd = accept(server_sock_fd, (struct sockaddr *)&client_address, &address_len);
	                    if (client_socket_fd > 0) {
	                        int index = -1;
	                        for (int i = 0; i < CONCURRENT_MAX; i++) {
	                            if (client_fds[i] == 0) {
	                                index = i;
	                                client_fds[i] = client_socket_fd;
	                                break;
	                            }
	                        }
	                        if (index >= 0) {
	                            EV_SET(&event_change, client_socket_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
	                            kevent(kq, &event_change, 1, NULL, 0, NULL);
	                            printf("新客户端(fd = %d)加入成功 %s:%d \n",client_socket_fd,inet_ntoa(client_address.sin_addr),ntohs(client_address.sin_port));
	                        }else{
	                            bzero(input_msg, BUFFER_SIZE);
	                            strcpy(input_msg, "服务器加入的客户端数达到最大值,无法加入!\n");
	                            send(client_socket_fd, input_msg, BUFFER_SIZE, 0);
	                            printf("客户端连接数达到最大值，新客户端加入失败 %s:%d \n",inet_ntoa(client_address.sin_addr),ntohs(client_address.sin_port));
	                        }
	                    }
	                }else{
	                    //处理某个客户端过来的消息
	                    bzero(recv_msg, BUFFER_SIZE);
	                    long byte_num = recv((int)current_event.ident,recv_msg,BUFFER_SIZE,0);
	                    if (byte_num > 0) {
	                        if (byte_num > BUFFER_SIZE) {
	                            byte_num = BUFFER_SIZE;
	                        }
	                        recv_msg[byte_num] = '\0';
	                        printf("客户端(fd = %d):%s",(int)current_event.ident,recv_msg);
	                        // 发送响应报文
	                        sendHello((int)current_event.ident);
	                    }else if(byte_num < 0){
	                        printf("从客户端(fd = %d)接受消息出错.\n",(int)current_event.ident);
	                    }else{
	                        EV_SET(&event_change, current_event.ident, EVFILT_READ, EV_DELETE, 0, 0, NULL);
	                        kevent(kq, &event_change, 1, NULL, 0, NULL);
	                        close((int)current_event.ident);
	                        for (int i = 0; i < CONCURRENT_MAX; i++) {
	                            if (client_fds[i] == (int)current_event.ident) {
	                                client_fds[i] = 0;
	                                break;
	                            }
	                        }
	                        printf("客户端(fd = %d)退出了\n",(int)current_event.ident);
	                    }
	                }
	            }
	        }
	    }
	    return 0;
	}
	
	void sendHello(int fd)
	{
	    time_t rawtime = 0;
	    time(&rawtime);
	    char statusCode[] = "HTTP/1.1 200 OK\r\n";
	    char server[]="Server: localhost\r\n";
	    char header[] = "Content-Type: text/html\r\n";
	    char contentLength[50]="Content-Length: ";
	    char date[100] ={0};
	    sprintf(date, "Date: %s\r\n",ctime(&rawtime));
	    
	    char content[] = "\r\n<html><body>Hello World</body></html>\r\n";
	    sprintf(contentLength, "%s%ld\r\n",contentLength,strlen(content));
	    char res[1024]={0};
	    sprintf(res, "%s%s%s%s%s%s",statusCode,server,header,contentLength,date,content);
	    
	//    printf("%s",res);
	    send(fd, res, strlen(res), 0);
	}



参考：  
http://www.ruanyifeng.com/blog/2016/08/http.html  
https://hit-alibaba.github.io/interview/basic/network/HTTP.html


