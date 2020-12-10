> [TinyHttpd](https://github.com/EZLippi/Tinyhttpd) 是一个用 C 语言写的及其简洁的 HTTP 服务程序，一共只有 500 行左右代码，非常适合拿来学习 HTTP。

这篇博客也是自己一边学习一遍记录的学习笔记。

2020-4-18

----

## 效果演示

1. 先 clone 下源码到本地：

    ```
    git clone https://github.com/EZLippi/Tinyhttpd.git
    ```

2. 进入 Tinyhttpd/htdocs 目录，将 cgi 后缀结尾的文件赋予执行权限（我在这里困扰了好久，一直没能执行成功的原因就是没有给 cgi 文件加执行权限）。

    ```
    cd Tinyhttpd/htdocs
    chmod a+x *.cgi
    cd ../
    ```
3. 编译并运行 Tinyhttpd：

    ```
    make
    ./httpd
    ```

    执行结果：

    ```
    🐠 jony@deepin # ./httpd 
    httpd running on port 4000
    ```

4. 在浏览器输入 `localhost:4000`，进入测试页面：

    ![index](https://gitee.com/jiaoqiyuan/pics/raw/master/blog/tinyhttpd/index.png)

    输入 `blue`，点击提交。

    ![color](https://gitee.com/jiaoqiyuan/pics/raw/master/blog/tinyhttpd/color-cgi.png)


## 源码解析

TinyHttpd 源码在 github上有，这里也贴出来吧，反正也只有500行左右。

源码实现的功能也比较简单，就是通过浏览器向 httpd 服务发送数据请求，然后接收 httpd 服务返回的数据。

每个函数的作用：

- accept_request:  处理从套接字上监听到的一个 HTTP 请求，在这里可以很大一部分地体现服务器处理请求流程。

- bad_request: 返回给客户端这是个错误请求，HTTP 状态吗 400 BAD REQUEST.

- cat: 读取服务器上某个文件写到 socket 套接字。

- cannot_execute: 主要处理发生在执行 cgi 程序时出现的错误。

- error_die: 把错误信息写到 perror 并退出。

- execute_cgi: 运行 cgi 程序的处理，也是个主要函数。

- get_line: 读取套接字的一行，把回车换行等情况都统一为换行符结束。

- headers: 把 HTTP 响应的头部写到套接字。

- not_found: 主要处理找不到请求的文件时的情况。

- sever_file: 调用 cat 把服务器文件返回给浏览器。

- startup: 初始化 httpd 服务，包括建立套接字，绑定端口，进行监听等。

- unimplemented: 返回给浏览器表明收到的 HTTP 请求所用的 method 不被支持。

### main 函数入手

源码阅读从 `main` 函数切入即可：

```C
int main(void)
{
    //创建服务端 socket 文件描述符
    int server_sock = -1;
    //指定使用 4000 端口
    u_short port = 4000;
    //创建客户端 socket 文件描述符
    int client_sock = -1;
    //创建 sockaddr_in 存储 IP 和端口信息
    struct sockaddr_in client_name;
    //sockaddr_in 长度
    socklen_t  client_name_len = sizeof(client_name);
    //多线程 ID
    pthread_t newthread;

    //startup 函数用于初始化 httpd 服务，包括建立套接字，绑定端口，进行监听等。
    server_sock = startup(&port);
    printf("httpd running on port %d\n", port);

    while (1)
    {
        //阻塞监听客户端请求
        client_sock = accept(server_sock,
                (struct sockaddr *)&client_name,
                &client_name_len);
        if (client_sock == -1)
            error_die("accept");
        /* accept_request(&client_sock); */
        //每监听到一个请求就创建一个线程使用 `accept_request` 处理请求
        if (pthread_create(&newthread , NULL, (void *)accept_request, (void *)(intptr_t)client_sock) != 0)
            perror("pthread_create");
    }

    //关闭连接
    close(server_sock);

    return(0);
}
```

### `startup` 初始化 httpd 服务

```C
/**********************************************************************/
/* This function starts the process of listening for web connections
 * on a specified port.  If the port is 0, then dynamically allocate a
 * port and modify the original port variable to reflect the actual
 * port.
 * Parameters: pointer to variable containing the port to connect on
 * Returns: the socket */
/**********************************************************************/
int startup(u_short *port)
{
    int httpd = 0;
    int on = 1;
    //创建存储 IP 和端口的 sockaddr_in 结构体
    struct sockaddr_in name;
    //创建 socket 连接
    httpd = socket(PF_INET, SOCK_STREAM, 0);
    if (httpd == -1)
        error_die("socket");
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;
    name.sin_port = htons(*port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);
    if ((setsockopt(httpd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))) < 0)  
    {  
        error_die("setsockopt failed");
    }
    if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0)
        error_die("bind");
    if (*port == 0)  /* if dynamically allocating a port 如果port为0就随机分配一个可用的值给port*/
    {
        socklen_t namelen = sizeof(name);
        if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
            error_die("getsockname");
        *port = ntohs(name.sin_port);
    }
    //监听 httpd 这个 socket
    if (listen(httpd, 5) < 0)
        error_die("listen");
    return(httpd);
}
```

`startup` 函数用于初始化 httpd 服务，包括建立套接字，绑定端口，进行监听等。

```C
httpd = socket(PF_INET, SOCK_STREAM, 0);
```

PF_INET 指定使用 ipv4，SOCK_STREAM 指定使用 TCP 通信，第三个参数 0 表示根据前面两个参数使用默认协议。

`setsockopt` 函数用于设置套接字的关联项，为了操作套接字层的选项，应该将层的值指定为SOL_SOCKET。SO_REUSEADDR 表示允许重用本地地址和端口，一般来说，一个端口释放后会等待两分钟之后才能再被使用，SO_REUSEADDR是让端口释放后立即就可以被再次使用。有关 SO_REUSEADDR 的说明可以参考 [这篇文章](https://www.cnblogs.com/qq78292959/archive/2013/01/18/2865926.html) 。

设置完 socket 后使用 bind 绑定 socket 到指定地址。

`listen` 使得一个进程可以接受其它进程的请求，从而成为一个服务器进程。其中 `listen` 的第二个参数为 backlog。

    这个参数涉及到一些网络的细节。在进程正理一个一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态，有时由于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。如果这个情况出现了，服务器进程希望内核如何处理呢？内核会在自己的进程空间里维护一个队列以跟踪这些完成的连接但服务器进程还没有接手处理或正在进行的连接，这样的一个队列内核不可能让其任意大，所以必须有一个大小的上限。这个backlog告诉内核使用这个数值作为上限。

    毫无疑问，服务器进程不能随便指定一个数值，内核有一个许可的范围。这个范围是实现相关的。很难有某种统一，一般这个值会小30以内。

    TCP的服务器端socket基本流程socket->bind->listen->accept->send/recv->closesocket，客户端基本流程socket->[bind->]->connect->send/recv->closesocket，其中客户端connect函数应该是和服务器端的listen函数相互作用，而不是accept函数。在listen函数中的第二个参数backlog代表着等待处理的连接队列（以下简称队列）的长度，神马意思？我也不太懂，但是通过代码实践，我可以简单的说，每当有一个客户端connect了，listen的队列中就加入一个连接，每当服务器端accept了，就从listen的队列中取出一个连接，转成一个专门用来传输数据的socket（accept函数的返回值），所以在服务器端程序中有两个socket，前者是用来接收客户端连接的socket.

最后返回这个 socket，至此 httpd 就初始化完成了，创建并初始化了一个 socket，绑定并监听指定 IP 和端口后返回这个 socket。

### `accept_request` 接受请求

```C
/**********************************************************************/
/* A request has caused a call to accept() on the server port to
 * return.  Process the request appropriately.
 * Parameters: the socket connected to the client */
/**********************************************************************/
void accept_request(void *arg)
{
    int client = (intptr_t)arg;
    char buf[1024];
    size_t numchars;
    char method[255];
    char url[255];
    char path[512];
    size_t i, j;
    struct stat st;
    int cgi = 0;      /* becomes true if server decides this is a CGI
                       * program */
    char *query_string = NULL;
    //从 socket 中读取一行数据，这里就是获取首行数据
    numchars = get_line(client, buf, sizeof(buf));
    i = 0; j = 0;
    //将buf中去除空白字符后的数据存放到 method 中
    while (!ISspace(buf[i]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[i];
        i++;
    }
    //j 用于存放 method 中最后一个字符的位置
    j=i;
    method[i] = '\0';
    //如果请求方法不是 GET 或 POST，就返回一个 501 页面
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        unimplemented(client);
        return;
    }

    //如果是 POST 请求，就执行了 cgi
    if (strcasecmp(method, "POST") == 0)
        cgi = 1;

    i = 0;
    //过滤空白字符
    while (ISspace(buf[j]) && (j < numchars))
        j++;
    //获取 url
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < numchars))
    {
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

    //如果是 GET 请求
    if (strcasecmp(method, "GET") == 0)
    {
        query_string = url;
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?')
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }
    //设置请求页面地址
    sprintf(path, "htdocs%s", url);
    //如果页面地址以 / 结尾，就添加 index.html 作为默认请求页面
    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    //如果 path 在本地不存在，就忽略 client 剩余的内容并显示 404 页面
    if (stat(path, &st) == -1) {
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    }
    else
    {
        //再次判断 path 如果是目录，就添加默认的 index.html 作为请求页面
        if ((st.st_mode & S_IFMT) == S_IFDIR)
            strcat(path, "/index.html");
        //如果 path 文件可执行，就将 cgi 变量赋值为 1
        if ((st.st_mode & S_IXUSR) ||
                (st.st_mode & S_IXGRP) ||
                (st.st_mode & S_IXOTH)    )
            cgi = 1;
        //判断 是否执行 cgi文件如果 cgi 为 0 就执行服务端文件，否则执行客户端本地 cgi。
        if (!cgi)
            serve_file(client, path);
        else
            execute_cgi(client, path, method, query_string);
    }
    //关闭 socket
    close(client);
}
```

`accept_request` 是处理请求的主体，在服务端初始化 httpd 后并接受客户端的连接请求后，服务端与客户端建立了 TCP 连接，接收客户端发来的请求信息，通过 `get_line` 函数获取一行数据内容，然后进行相应的字符串解析，获取出请求类型、URL等信息。

根据请求是 `POST` 还是 `GET` 分别进行不同的处理。

默认情况下是使用 `htdocs` 目录下的 index.html 页面作为默认请求页面，如果自己手动指定请求页面就以用户指定页面为准，这里我们使用的是 `color.cgi` 文件。

最后根据请求类型以及 path 文件的可执行权限决定采用不同的执行方式， `serv_file` 是将 path 文件读取展示在浏览器上，`execute_cgi` 是执行本地的 cgi 文件。

上面程序中的 `ISspace` 函数是用于判断空白字符的：

```C
#define ISspace(x) isspace((int)(x))
```

`isspace` 是系统函数，用于检查参数c是否为空格字符，也就是判断是否为空格(' ')、水平定位字符
('\t')、归位键('\r')、换行('\n')、垂直定位字符('\v')或翻页('\f')的情况。如果是空白字符就返回 TRUE，不是空白字符就返回 NULL；

### `get_line` 函数

`get_line` 函数就是读取一行以 `\r\n`结尾的文本数据，这是基本的字符串处理程序，了解好指针和基本库函数的使用就比较简单了。

```C
/**********************************************************************/
/* Get a line from a socket, whether the line ends in a newline,
 * carriage return, or a CRLF combination.  Terminates the string read
 * with a null character.  If no newline indicator is found before the
 * end of the buffer, the string is terminated with a null.  If any of
 * the above three line terminators is read, the last character of the
 * string will be a linefeed and the string will be terminated with a
 * null character.
 * Parameters: the socket descriptor
 *             the buffer to save the data in
 *             the size of the buffer
 * Returns: the number of bytes stored (excluding null) */
/**********************************************************************/
int get_line(int sock, char *buf, int size)
{
    int i = 0;
    char c = '\0';
    int n;

    while ((i < size - 1) && (c != '\n'))
    {
        //recv 接收一个字节的数据
        n = recv(sock, &c, 1, 0);
        /* DEBUG printf("%02X\n", c); */
        if (n > 0)
        {
            //如果以 \r 结尾，就继续判断下一个字符是否是 \n
            if (c == '\r')
            {
                //预读取下一个字符，这里使用 MSG_PEEK 的作用就是预读取，不影响下次recv接收到的数据
                n = recv(sock, &c, 1, MSG_PEEK);
                /* DEBUG printf("%02X\n", c); */
                //如果读取到的是 \n，就读取 \n 到变量 c，否则赋值变量c 为 \n
                //这里不太明白，两种情况下变量 c 都被赋值为 \n 了，有什么区别吗？
                if ((n > 0) && (c == '\n'))
                    recv(sock, &c, 1, 0);
                else
                    c = '\n';
            }
            //赋值buf[i]
            buf[i] = c;
            i++;
        }
        else
            c = '\n';
    }
    buf[i] = '\0';

    return(i);
}
```

### `unimplemented` 和 `not_found` 函数分别用于返回 501 页面和 404 页面

`unimplemented` 用于向客户端发送方法未实现的页面信息，具体内容就是 `send` 函数发送的那些字符串。

这里只有一个 `send` 函数需要留意，`send` 

```C
/**********************************************************************/
/* Inform the client that the requested web method has not been
 * implemented.
 * Parameter: the client socket */
/**********************************************************************/
void unimplemented(int client)
{
    char buf[1024];

    sprintf(buf, "HTTP/1.0 501 Method Not Implemented\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, SERVER_STRING);
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-Type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<HTML><HEAD><TITLE>Method Not Implemented\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</TITLE></HEAD>\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<BODY><P>HTTP request method not supported.\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</BODY></HTML>\r\n");
    send(client, buf, strlen(buf), 0);
}
```

同理 `not_found` 返回 404 页面。

```C
/**********************************************************************/
/* Give a client a 404 not found status message. */
/**********************************************************************/
void not_found(int client)
{
    char buf[1024];

    sprintf(buf, "HTTP/1.0 404 NOT FOUND\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, SERVER_STRING);
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-Type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<HTML><TITLE>Not Found</TITLE>\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<BODY><P>The server could not fulfill\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "your request because the resource specified\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "is unavailable or nonexistent.\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</BODY></HTML>\r\n");
    send(client, buf, strlen(buf), 0);
}
```

### `serv_file` 返回文件内容给客户端

```C
/**********************************************************************/
/* Send a regular file to the client.  Use headers, and report
 * errors to client if they occur.
 * Parameters: a pointer to a file structure produced from the socket
 *              file descriptor
 *             the name of the file to serve */
/**********************************************************************/
void serve_file(int client, const char *filename)
{
    FILE *resource = NULL;
    int numchars = 1;
    char buf[1024];

    buf[0] = 'A'; buf[1] = '\0';
    //将 client 中的数据都读出来，不做任何处理，相当于丢弃
    while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
        numchars = get_line(client, buf, sizeof(buf));
    //读取文件内容
    resource = fopen(filename, "r");
    if (resource == NULL)
        not_found(client);
    else
    {   
        //发送头数据信息
        headers(client, filename);
        //发送文件内容
        cat(client, resource);
    }
    fclose(resource);
}
```

这里的逻辑也比较简单，读取文件内容，发送给客户端。给客户端发送数据时需要先使用 `headers` 组织数据头信息，然后发送文件内容。

- headers 函数

    很简单，没什么说的，就是发送一些字符串信息，用于标示 HTTP 头部信息。

    ```C
    /**********************************************************************/
    /* Return the informational HTTP headers about a file. */
    /* Parameters: the socket to print the headers on
    *             the name of the file */
    /**********************************************************************/
    void headers(int client, const char *filename)
    {
        char buf[1024];
        //可以根据文件名确定文件类型
        (void)filename;  /* could use filename to determine file type */

        strcpy(buf, "HTTP/1.0 200 OK\r\n");
        send(client, buf, strlen(buf), 0);
        strcpy(buf, SERVER_STRING);
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "Content-Type: text/html\r\n");
        send(client, buf, strlen(buf), 0);
        strcpy(buf, "\r\n");
        send(client, buf, strlen(buf), 0);
    }
    ```

- `cat` 发送文件内容

    `cat` 函数用于发送文件内容到客户端，在客户端浏览器上显示出来。代码也比较简单。

    ```C
    /**********************************************************************/
    /* Put the entire contents of a file out on a socket.  This function
    * is named after the UNIX "cat" command, because it might have been
    * easier just to do something like pipe, fork, and exec("cat").
    * Parameters: the client socket descriptor
    *             FILE pointer for the file to cat */
    /**********************************************************************/
    void cat(int client, FILE *resource)
    {
        char buf[1024];

        fgets(buf, sizeof(buf), resource);
        while (!feof(resource))
        {
            send(client, buf, strlen(buf), 0);
            fgets(buf, sizeof(buf), resource);
        }
    }
    ```

### `execute_cgi` 执行 cgi 文件

```C
/**********************************************************************/
/* Execute a CGI script.  Will need to set environment variables as
 * appropriate.
 * Parameters: client socket descriptor
 *             path to the CGI script */
/**********************************************************************/
void execute_cgi(int client, const char *path,
        const char *method, const char *query_string)
{
    char buf[1024];
    int cgi_output[2];
    int cgi_input[2];
    pid_t pid;
    int status;
    int i;
    char c;
    int numchars = 1;
    int content_length = -1;

    //设置默认 buf 值
    buf[0] = 'M'; buf[1] = '\0';
    //如果是 GET 方法
    if (strcasecmp(method, "GET") == 0)
        //过滤头部信息
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
    //如果是 POST 方法
    else if (strcasecmp(method, "POST") == 0) /*POST*/
    {
        //获取一行数据
        numchars = get_line(client, buf, sizeof(buf));
        while ((numchars > 0) && strcmp("\n", buf))
        {
            buf[15] = '\0';
            //获取 Content-Length 大小
            if (strcasecmp(buf, "Content-Length:") == 0)
                content_length = atoi(&(buf[16]));
            numchars = get_line(client, buf, sizeof(buf));
        }
        //如果获取不到 Content-Length 就表示请求出错
        if (content_length == -1) {
            bad_request(client);
            return;
        }
    }
    else/*HEAD or other*/
    {
    }

    //创建管道
    if (pipe(cgi_output) < 0) {
        cannot_execute(client);
        return;
    }
    if (pipe(cgi_input) < 0) {
        cannot_execute(client);
        return;
    }

    //创建子进程
    if ( (pid = fork()) < 0 ) {
        cannot_execute(client);
        return;
    }
    //发送 200 OK 信息给客户端表示请求被成功处理
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    send(client, buf, strlen(buf), 0);
    //子进程执行 cgi 脚本
    if (pid == 0)  /* child: CGI script */
    {
        char meth_env[255];
        char query_env[255];
        char length_env[255];
        //复制文件描述符，将 cgi_output[1]重定向到标准输出，cgi_input[0]重定向到标准输入
        dup2(cgi_output[1], STDOUT);
        dup2(cgi_input[0], STDIN);
        close(cgi_output[0]);
        close(cgi_input[1]);
        sprintf(meth_env, "REQUEST_METHOD=%s", method);
        putenv(meth_env);
        if (strcasecmp(method, "GET") == 0) {
            sprintf(query_env, "QUERY_STRING=%s", query_string);
            putenv(query_env);
        }
        else {   /* POST */
            sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
            putenv(length_env);
        }
        execl(path, NULL);
        exit(0);
    } else {    /* parent */
        close(cgi_output[1]);
        close(cgi_input[0]);
        if (strcasecmp(method, "POST") == 0)
            for (i = 0; i < content_length; i++) {
                recv(client, &c, 1, 0);
                write(cgi_input[1], &c, 1);
            }
        while (read(cgi_output[0], &c, 1) > 0)
            send(client, &c, 1, 0);

        close(cgi_output[0]);
        close(cgi_input[1]);
        waitpid(pid, &status, 0);
    }
}
```

`execute_cgi` 先从 client socket 中继续读取数据，判断 `Content-Length` 的大小是正确，然后创建管道。

管道是进程间通信的一种方式，这里创建了两个管道 `cgi_input` 和 `cgi_output`，并 fork 出一个子进程。

在子进程中，把 STDOUT 重定向到 cgi_outputt 的写入端，把 STDIN 重定向到 cgi_input 的读取端，关闭 cgi_input 的写入端 和 cgi_output 的读取端，设置 request_method 的环境变量，GET 的话设置 query_string 的环境变量，POST 的话设置 content_length 的环境变量，这些环境变量都是为了给 cgi 脚本调用，接着用 execl 运行 cgi 程序。

在父进程中，关闭 cgi_input 的读取端 和 cgi_output 的写入端，如果 POST 的话，把 POST 数据写入 cgi_input，已被重定向到 STDIN，读取 cgi_output 的管道输出到客户端，该管道输入是 STDOUT。接着关闭所有管道，等待子进程结束。

用图表示出来就是：

图1 管道初始状态：

![pipe1](https://camo.githubusercontent.com/cd2c864c743f7135829e728efc15d4e16af1bc71/687474703a2f2f696d672e626c6f672e6373646e2e6e65742f32303134313232363137333232323735303f77617465726d61726b2f322f746578742f6148523063446f764c324a736232637559334e6b626935755a585176616d4e71597a6b784f413d3d2f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f37302f677261766974792f43656e746572)

图2 管道最终状态：

![pipe2](https://camo.githubusercontent.com/ddc3e769843c534bcecf8b9c191f4d923c5190a3/687474703a2f2f696d672e626c6f672e6373646e2e6e65742f32303134313232363136313131393938313f77617465726d61726b2f322f746578742f6148523063446f764c324a736232637559334e6b626935755a585176616d4e71597a6b784f413d3d2f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f37302f677261766974792f43656e746572)

图片直观地显示了 GET 和 POST 请求通过管道将数据发送到客户端浏览器的过程。

还有一张图片描述了 execute_cgi 的整个流程：

![execute_cgi](https://gitee.com/jiaoqiyuan/pics/raw/master/blog/tinyhttpd/execute_cgi.png)

这里重点关注一下管道的概念和使用方式。

## 网络抓包

在运行 httpd 服务端后打开 wireshark 抓包看一下具体的数据信息；在输入颜色提交后，可以抓取到如下数据：

![wireshark](https://gitee.com/jiaoqiyuan/pics/raw/master/blog/tinyhttpd/httpwireshark.png)

执行提交后客户端向浏览器发送了一个 POST 请求，附带有颜色信息。

其实向前倒还能看到建立连接的三次握手信息，以及服务端返回的 200 OK 的成功信息。

## 总结

到这里源码就分析完了，整个过程涉及到网络的连接建立，网络数据的收发，cgi 程序的运行，浏览器页面的展示等内容；有一些字符串的处理，管道的使用，非常短小精悍，值得好好学习一些。
