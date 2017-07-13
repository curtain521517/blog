---
title: socket
tags: 
	- socket
	- 网络
categories: 网络
---

## socket
#### socket到底是什么？
简单来说，socket就是进程通讯的一种方式，调用网络库的一些API函数实现分布在不同主机的相关进程之间的数据交换。
从编程语言角度来看，socket是一个无符号整型变量，用来标识一个通信进程

1. **IP地址：**即依照TCP/IP协议分配给本地主机的网络地址，两个进程要通讯，任一进程首先要知道通讯对方的位置，即对方的IP。
2. **端口号：**用来辨别本地通讯进程，一个本地的进程在通讯时均会占用一个端口号，不同的进程端口号不同，因此在通讯前必须要分配一个没有被访问的端口号。
3. **连接：**指两个进程间的通讯链路。
4. **半相关：**网络中用一个三元组可以在全局唯一标志一个进程：（协议，本地地址，本地端口号）这样一个三元组，叫做一个半相关,它指定连接的每半部分。
5. **全相关：**一个完整的网间进程通信需要由两个进程组成，并且只能使用同一种高层协议。也就是说，不可能通信的一端用TCP协议，而另一端用UDP协议。因此一个完整的网间通信需要一个五元组来标识：（协议，本地地址，本地端口号，远地地址，远地端口号）这样一个五元组，叫做一个相关（association），即两个协议相同的半相关才能组合成一个合适的相关，或完全指定组成一连接。

#### socket如何使用?
服务端：建立监听套接字，绑定地址和端口号，开始监听，等待客户机连接，连接后会生成一个新的套接字，然后读写信息  
客户端：建立套接字，然后申请连接服务器，连上服务器后开始读写信息

#### 示例代码
服务端

	#include <stdio.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <string.h>
	
	
	int main(int argc, const char * argv[]) {
	    
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(12345);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&server_addr.sin_zero, 8);
	    
	    // 创建socket
	    int server_socket = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_socket == -1) {
	        perror("socket error");
	        return 1;
	    }else {
	        printf("socket success\n");
	    }
	    
	    int bind_result = bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr));
	    if (bind_result == -1) {
	        perror("bind error");
	        return 1;
	    }else {
	        printf("bind success\n");
	    }
	    
	    if (listen(server_socket, 5) == -1) {
	        perror("listen error");
	        return 1;
	    }else {
	        printf("listen...\n");
	    }
	    
	    struct sockaddr_in client_address;
	    socklen_t address_len;
	    int client_socket = accept(server_socket, (struct sockaddr*)&client_address, &address_len);
	    if (client_socket == -1) {
	        perror("accept error");
	        return -1;
	    }else {
	        printf("accept success...\n");
	    }
	    
	    char recv_msg[1024]={0};
	    char send_msg[1024]={0};
	    
	    while (1) {
	        bzero(recv_msg, 1024);
	        bzero(send_msg, 1024);
	        
	        printf("reply:\n");
	        scanf("%s",send_msg);
	        send(client_socket, send_msg, 1024, 0);
	        
	        long byte_num = recv(client_socket,recv_msg,1024,0);
	        recv_msg[byte_num]='\0';
	        printf("client said:%s\n",recv_msg);
	    }
	    
	    return 0;
	}

客户端

	#include <stdio.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <string.h>
	
	
	int main(int argc, const char * argv[]) {
	    
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(12345);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&server_addr.sin_zero, 8);
	    
	    // 创建socket
	    int server_socket = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_socket == -1) {
	        perror("socket error");
	        return 1;
	    }else {
	        printf("socket success\n");
	    }
	    
	    char recv_msg[1024]={0};
	    char send_msg[1024]={0};
	    
	    if (connect(server_socket, (struct sockaddr*)&server_addr, sizeof(struct sockaddr_in)) != 0) {
	        perror("connect failed");
	        return 1;
	    }else {
	        printf("connect success\n");
	    }
	    
	    while (1) {
	        bzero(recv_msg, 1024);
	        bzero(send_msg, 1024);
	        
	        long byte_num = recv(server_socket,recv_msg,1024,0);
	        recv_msg[byte_num]='\0';
	        printf("server said:%s\n",recv_msg);
	        
	        printf("reply:");
	        scanf("%s",send_msg);
	        send(server_socket, send_msg, 1024, 0);
	        
	    }
	    
	    return 0;
	}

#### select的使用  
select这个系统调用，是一种多路复用IO方案，可以同时对多个文件描述符进行监控，从而知道哪些文件描述符可读，可写或者出错，不过select方法是阻塞的，可以设定超时时间。  
	1. 创建一个fd_set变量（fd_set实为包含了一个整数数组的结构体），用来存放所有的待检查的文件描述符  
	2.清空fd_set变量，并将需要检查的所有文件描述符加入fd_set  
	3.调用select。若返回-1，则说明出错;返回0,则说明超时，返回正数，则为发生状态变化的文件描述符的个数  
	4.若select返回大于0,则依次查看哪些文件描述符变的可读，并对它们进行处理  
	5.返回步骤2，开始新一轮的检测  
	
服务端

	#include <stdio.h>
	#include <stdlib.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <string.h>
	#include <unistd.h>
	#define BACKLOG 5 //完成三次握手但没有accept的队列的长度
	#define CONCURRENT_MAX 8 //应用层同时可以处理的连接
	#define SERVER_PORT 11332
	#define BUFFER_SIZE 1024
	#define QUIT_CMD ".quit"
	int client_fds[CONCURRENT_MAX];
	
	int main()
	{
	    char input_msg[BUFFER_SIZE];
	    char recv_msg[BUFFER_SIZE];
	    
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(SERVER_PORT);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&server_addr.sin_zero, 8);
	    
	    // 创建socket
	    int server_sock_fd = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_sock_fd == -1) {
	        perror("socket error");
	        return 1;
	    }else {
	        printf("socket success\n");
	    }
	    
	    int bind_result = bind(server_sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
	    if (bind_result == -1) {
	        perror("bind error");
	        return 1;
	    }else {
	        printf("bind success\n");
	    }
	    
	    if (listen(server_sock_fd, 5) == -1) {
	        perror("listen error");
	        return 1;
	    }else {
	        printf("listen...\n");
	    }
	    
	    //fd set
	    fd_set server_fd_set;
	    int max_fd = -1;
	    struct timeval tv;
	    tv.tv_sec = 20;
	    tv.tv_usec = 0;
	    while (1) {
	        FD_ZERO(&server_fd_set);
	        //标准输入
	        FD_SET(STDIN_FILENO, &server_fd_set);
	        if (max_fd < STDIN_FILENO) {
	            max_fd = STDIN_FILENO;
	        }
	        
	        //服务器端socket
	        FD_SET(server_sock_fd, &server_fd_set);
	        if (max_fd < server_sock_fd) {
	            max_fd = server_sock_fd;
	        }
	        
	        //客户端连接
	        for (int i = 0; i < CONCURRENT_MAX; i++) {
	            if (client_fds[i]!=0) {
	                FD_SET(client_fds[i], &server_fd_set);
	                
	                if (max_fd < client_fds[i]) {
	                    max_fd = client_fds[i];
	                }
	            }
	        }
	        
	        int ret = select(max_fd+1, &server_fd_set, NULL, NULL, &tv);
	        if (ret < 0) {
	            perror("select 出错\n");
	            continue;
	        }else if(ret == 0){
	            printf("select 超时\n");
	            continue;
	        }else{
	            //ret为未状态发生变化的文件描述符的个数
	            if (FD_ISSET(STDIN_FILENO, &server_fd_set)) {
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
	            }
	            if (FD_ISSET(server_sock_fd, &server_fd_set)) {
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
	                        printf("新客户端(%d)加入成功 %s:%d \n",index,inet_ntoa(client_address.sin_addr),ntohs(client_address.sin_port));
	                    }else{
	                        bzero(input_msg, BUFFER_SIZE);
	                        strcpy(input_msg, "服务器加入的客户端数达到最大值,无法加入!\n");
	                        send(client_socket_fd, input_msg, BUFFER_SIZE, 0);
	                        printf("客户端连接数达到最大值，新客户端加入失败 %s:%d \n",inet_ntoa(client_address.sin_addr),ntohs(client_address.sin_port));
	                    }
	                }
	            }
	            for (int i = 0; i <CONCURRENT_MAX; i++) {
	                if (client_fds[i]!=0) {
	                    if (FD_ISSET(client_fds[i], &server_fd_set)) {
	                        //处理某个客户端过来的消息
	                        bzero(recv_msg, BUFFER_SIZE);
	                        long byte_num = recv(client_fds[i],recv_msg,BUFFER_SIZE,0);
	                        if (byte_num > 0) {
	                            if (byte_num > BUFFER_SIZE) {
	                                byte_num = BUFFER_SIZE;
	                            }
	                            recv_msg[byte_num] = '\0';
	                            printf("客户端(%d):%s\n",i,recv_msg);
	                        }else if(byte_num < 0){
	                            printf("从客户端(%d)接受消息出错.\n",i);
	                        }else{
	                            FD_CLR(client_fds[i], &server_fd_set);
	                            client_fds[i] = 0;
	                            printf("客户端(%d)退出了\n",i);
	                        }
	                    }
	                }
	            }
	        }
	
	    }
	    return 0;
	}
	
客户端
		
	#include <stdio.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <string.h>
	#include <unistd.h>
	#include <stdlib.h>
	
	#define BUFFER_SIZE 1024
	
	int main (int argc, const char * argv[])
	{
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(11332);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&(server_addr.sin_zero),8);
	    
	    int server_sock_fd = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_sock_fd == -1) {
	        perror("socket error");
	        return 1;
	    }
	    char recv_msg[BUFFER_SIZE];
	    char input_msg[BUFFER_SIZE];
	    
	    if (connect(server_sock_fd, (struct sockaddr *)&server_addr, sizeof(struct sockaddr_in))==0) {
	        fd_set client_fd_set;
	        struct timeval tv;
	        tv.tv_sec = 20;
	        tv.tv_usec = 0;
	        
	        
	        while (1) {
	            FD_ZERO(&client_fd_set);
	            FD_SET(STDIN_FILENO, &client_fd_set);
	            FD_SET(server_sock_fd, &client_fd_set);
	            
	            int ret = select(server_sock_fd + 1, &client_fd_set, NULL, NULL, &tv);
	            if (ret < 0 ) {
	                printf("select 出错!\n");
	                continue;
	            }else if(ret ==0){
	                printf("select 超时!\n");
	                continue;
	            }else{
	                if (FD_ISSET(STDIN_FILENO, &client_fd_set)) {
	                    bzero(input_msg, BUFFER_SIZE);
	                    fgets(input_msg, BUFFER_SIZE, stdin);
	                    if (send(server_sock_fd, input_msg, BUFFER_SIZE, 0) == -1) {
	                        perror("发送消息出错!\n");
	                    }
	                }
	                
	                if (FD_ISSET(server_sock_fd, &client_fd_set)) {
	                    bzero(recv_msg, BUFFER_SIZE);
	                    long byte_num = recv(server_sock_fd,recv_msg,BUFFER_SIZE,0);
	                    if (byte_num > 0) {
	                        if (byte_num > BUFFER_SIZE) {
	                            byte_num = BUFFER_SIZE;
	                        }
	                        recv_msg[byte_num] = '\0';
	                        printf("服务器:%s\n",recv_msg);
	                    }else if(byte_num < 0){
	                        printf("接受消息出错!\n");
	                    }else{
	                        printf("服务器端退出!\n");
	                        exit(0);
	                    }
	                    
	                }
	            }
	        }
	        
	    }
	    
	    return 0;
	}

#### kqueue
kqueue使用事件触发的机制
服务端

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
	#define SERVER_PORT 11332
	#define BUFFER_SIZE 1024
	#define QUIT_CMD ".quit"
	int client_fds[CONCURRENT_MAX];
	struct kevent events[10];//CONCURRENT_MAX + 2
	int main (int argc, const char * argv[])
	{
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
	    }
	    //绑定socket
	    int bind_result = bind(server_sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
	    if (bind_result == -1) {
	        perror("bind error");
	        return 1;
	    }
	    //listen
	    if (listen(server_sock_fd, BACKLOG) == -1) {
	        perror("listen error");
	        return 1;
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
	                        printf("客户端(fd = %d):%s\n",(int)current_event.ident,recv_msg);
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


客户端

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
	
	
	#define BUFFER_SIZE 1024
	struct kevent events[2];
	
	int main (int argc, const char * argv[])
	{
	    struct sockaddr_in server_addr;
	    server_addr.sin_len = sizeof(struct sockaddr_in);
	    server_addr.sin_family = AF_INET;
	    server_addr.sin_port = htons(11332);
	    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	    bzero(&(server_addr.sin_zero),8);
	    
	    int server_sock_fd = socket(AF_INET, SOCK_STREAM, 0);
	    if (server_sock_fd == -1) {
	        perror("socket error");
	        return 1;
	    }
	    char recv_msg[BUFFER_SIZE];
	    char input_msg[BUFFER_SIZE];
	    
	    if (connect(server_sock_fd, (struct sockaddr *)&server_addr, sizeof(struct sockaddr_in))==0) {
	        
	        int kq = kqueue();
	        if (kq == -1) {
	            perror("创建kqueue出错!\n");
	            exit(1);
	        }
	        
	        struct timespec timeout = {10,0};
	        struct kevent event_change;
	        EV_SET(&event_change, STDIN_FILENO, EVFILT_READ, EV_ADD, 0, 0, NULL);
	        kevent(kq, &event_change, 1, NULL, 0, NULL);
	        EV_SET(&event_change, server_sock_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
	        kevent(kq, &event_change, 1, NULL, 0, NULL);
	        
	        while (1) {
	            
	            int ret = kevent(kq, NULL, 0, events, 2, &timeout);
	            if (ret < 0 ) {
	                printf("kevent 出错!\n");
	                continue;
	            }else if(ret ==0){
	                printf("kevent 超时!\n");
	                continue;
	            }else{
	                struct kevent current_event = events[0];
	                if (current_event.ident == STDIN_FILENO) {
	                    bzero(input_msg, BUFFER_SIZE);
	                    fgets(input_msg, BUFFER_SIZE, stdin);
	                    if (send(server_sock_fd, input_msg, BUFFER_SIZE, 0) == -1) {
	                        perror("发送消息出错!\n");
	                    }
	                }
	                
	                else if (current_event.ident == server_sock_fd) {
	                    bzero(recv_msg, BUFFER_SIZE);
	                    long byte_num = recv(server_sock_fd,recv_msg,BUFFER_SIZE,0);
	                    if (byte_num > 0) {
	                        if (byte_num > BUFFER_SIZE) {
	                            byte_num = BUFFER_SIZE;
	                        }
	                        recv_msg[byte_num] = '\0';
	                        printf("服务器:%s\n",recv_msg);
	                    }else if(byte_num < 0){
	                        printf("接受消息出错!\n");
	                    }else{
	                        printf("服务器端退出!\n");
	                        exit(0);
	                    }
	                    
	                }
	            }
	        }
	        
	    }
	    
	    return 0;
	}
