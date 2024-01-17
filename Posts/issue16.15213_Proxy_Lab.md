# CMU 15213 Proxy Lab

CSAPP 系列课程的最后一个 Lab 啦！采访了一下 Greg 教授，他表示这是最难的一个 Lab，因为往年这个 Lab 的均分是最低的（但是真实原因可能是大家算了一下分，发现已经差不多可以拿 A 了，所以就随便做做了 lol）

Proxy Lab 需要我们手动完成一个 Proxy，这个 Lab 总共有 4 个 Phase：

1. 实现一个基本的 Proxy，可以转发请求
2. 使该 Proxy 可以处理并发的请求
3. 为 Proxy 新增一个 Cache 功能
4. 并发竞争条件控制

首先我们还是了解一些基本的概念

一个最基础且最常见的网络架构就是 C/S 模型 (Client - Server)，客户端发送请求，服务端收到请求后进行解析和处理，在获取了相关的资源之后向客户端发送响应。

![截屏2023-11-25 18.37.20.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/77443678-7255-41b0-a5f0-15286d8366ba/bef8880e-1b30-4226-b21e-325b2930cec6/%E6%88%AA%E5%B1%8F2023-11-25_18.37.20.png)

而 Proxy Server 则类似于一个中间商

![截屏2023-11-25 18.40.50.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/77443678-7255-41b0-a5f0-15286d8366ba/b29e11ad-adc1-486f-bc9a-c075bd3e6906/%E6%88%AA%E5%B1%8F2023-11-25_18.40.50.png)

Proxy 可以帮助转发请求和对请求/回应进行缓存。在实际应用中，Proxy 也有很多的作用，例如提供额外的安全性、负载均衡、减轻 Server 压力、流量控制等。

在完成这个 Lab 的时候，Phase 1 和 Phase 2 因为区别不大，因此我是两个一起完成的。

## 创建线程

为了实现并发性，这里选择使用比较简单的 thread，当有用户发起请求的时候，就会调用 `pthread_create(&tid, NULL, thread, client->conn_fd);` 函数创建一个新的子线程，然后我们通过 `thread` 定义的方法，开始处理用户发起的请求。

## 解析 HTTP 请求

Client 会向 Proxy 发送一个 HTTP 请求，而一个 HTTP 请求的构成主要分为三个部分

1. 一个请求行  request line
    1. `GET http://cmu.edu:8888/test.txt HTTP/1.0`
2. 零个或者多个请求头 request header
    1. 例如 `Connection: close`, `User-Agent: Mozilla/5.0`, …
    2. 每行请求头都以 `"\r\n"` 作为结尾
3. 一个空行来终止请求

request line 中包含了一些必要的信息，例如我们请求的方法，目标域名，文件地址，端口号等。而 request header 我们需要原封不动的转发给 server，因此这二者都需要解析并且记录。

在这里提供一个简单的实现

```c
/**
 * Parse the request line, record the method, uri, version, path, host, and port to the
 * request line object.
 * 
 * @param request 
 * @param buf 
 */
void parse_request_line(request_line *request, char* buf) {

    sscanf(buf, "%s %s %s", request->method, request->uri, request->version);
    // GET http://.../abc.txt HTTP/1.0

    char* host_pps = strstr(request->uri, HTTP_PREFIX) + strlen(HTTP_PREFIX);
    char* slash_pos = strstr(host_pps, "/");

    char host_with_port[MAXLINE];

    if (!slash_pos) {
        strcpy(host_with_port, host_pps);
        strcpy(request->path, "/");
    } else {
        strncpy(host_with_port, host_pps, slash_pos - host_pps);
        strcpy(request->path, slash_pos);
    }

    char* colon_pos = strstr(host_with_port, ":");
    if (!colon_pos) {
        strcpy(request->port, "80");
        strcpy(request->host, host_with_port);
    } else {
        request->host[0] = '\0';
        strncat(request->host, host_with_port, colon_pos - host_with_port);
        strcpy(request->port, colon_pos + 1);
    }
}

/**
 * Parse the request header, for a request, it will have several request headers.
 * We will parse this header and attach to the client_request line by line.
 * @param rio 
 * @param request 
 * @param buf 
 * @param client_request 
 */
void parse_request_header(rio_t rio, request_line* request, char* buf, char* client_request) {
    /** Attach the request line at the header **/
    sprintf(client_request, "%s %s HTTP/1.0\r\n", request->method, request->path);

    rio_readlineb(&rio, buf, MAXLINE);
    int UserAgent = 0, Connection = 0, ProxyConnection = 0, Host = 0;
    char* find;

    /** Parse the request header line by line **/
    while (strcmp(buf, "\r\n")) {

        strcat(client_request, buf);
        /** For each line, we will check have we ever read some special headers **/
        if ((find = strstr(buf, "User-Agent:")) != NULL) {
            UserAgent = 1;
        } else if ((find = strstr(buf, "Proxy-Connection:")) != NULL) {
            ProxyConnection = 1;
        } else if ((find = strstr(buf, "Connection:")) != NULL) {
            Connection = 1;
        } else if((find = strstr(buf, "Host:")) != NULL){
            Host = 1;
        }

        /** read the next line **/
        rio_readlineb(&rio, buf, MAXLINE);
    }

    /** If we haven't read some special headers, we will attach it by ourselves **/
    if (UserAgent == 0) {
        strcat(client_request, header_user_agent);
    }

    if (ProxyConnection == 0) {
        sprintf(buf, "Proxy-Connection: close\r\n");
        strcat(client_request, buf);
    }

    if (Connection == 0) {
        sprintf(buf, "Connection: close\r\n");
        strcat(client_request, buf);
    }

    if (Host == 0) {
        sprintf(buf, "Host: %s\r\n", request->host);
        strcat(client_request, buf);
    }

    /** Attach the \r\n at the end of the request buf **/
    strcat(client_request, "\r\n");
}
```

在解析完请求之后，我们就需要将请求发送出去了，此时我们需要实现一个 `send_request` 函数。因此，整体的调用流程类似于

```c
parse_request_line(request, buf);
memset(&buf[0], 0, sizeof(buf));

parse_request_header(rio, request, buf, client_request);
memset(&buf[0], 0, sizeof(buf));

/** Send the request **/
send_request(conn_fd, request, client_request);
```

## 发送请求

在 `send_request` 函数中，我们首先通过 `open_clientfd` 函数创建了一个文件描述符，然后使用 CMU 提供的 `rio_writen` 将用户的请求发送给真正的 server

接下来我们会使用 `rio_readinitb` 和 `rio_readnb` 来读取来自用户的响应并且转发会用户

```c
  	request_fd = open_clientfd(request->host, request->port);
    rio_writen(request_fd, client_request, strlen(client_request));
    rio_readinitb(&rio, request_fd);
    /** Read the response from the server **/
    while((response_length = rio_readnb(&rio, server_response, MAXLINE)) != 0) {
        rio_writen(conn_fd, server_response, (size_t) response_length);
    }
```

至此，整个基本的请求接受和转发就完成了。

## 增加 Cache

在 Proxy 中，如果多个客户端或者同一个客户端多次访问同一个服务器的同一个对象的时候，如果每次 Proxy 都要从服务端重新请求，那么显然是非常耗时的，而且还会消耗服务端的计算资源。

因此一个非常常见的做法就是在 Proxy 层加入一个 Cache 能力，当 Proxy 从 Server 获得到一个请求对象的时候，就将其缓存在自己这一层，当下次客户端发送一样的请求的时候，Proxy 就可以从缓存中直接将数据返回给客户端。

在 Proxy Lab 中，我们对 Cache 提出了几个要求：

1. 每一个单独的 Cache Object 有大小限制 (100 * 1024)
2. 总共系统能够存储的 Cache 有大小限制 (1024 * 1024)
3. 如果需要驱逐缓存，需要使用 LRU Policy

那么思考一下，在整个 `parse_request_line` → `parse_request_header` → `send_request` 流程中，哪一个地方我们可以增加 cache 相关的逻辑呢？

答案显然是在 `send_request` 函数中，具体而言，在我们调用 `rio_writen` 发送请求之前，我们需要检查一下是否可以直接从缓存中获取响应。如果发现请求已经被缓存了，由于我们使用的是 LRU 机制，因此我们需要额外的更新一下缓存中的计数。

另一方面，当我们接受到来自服务端的响应后，我们要决定是否以及该如何将数据进行缓存。

```c
void send_request(int conn_fd, request_line *request, char* client_request) {

    /** Fetch Data from Cache **/
    cache_item* response = find_cache(request->uri);
    if (response) {
        update_cache(response);
        rio_writen(conn_fd, response->data, response->size);
        close(conn_fd);
        return;
    }

    /** Can not find the data from cache, we should run the request to the server **/
    char server_response[MAXLINE];
    char cache_candidate[MAX_OBJECT_SIZE];
    char *cache_ptr = cache_candidate;

    rio_t rio;
    int cacheable = 1;
    int request_fd;
    int response_length;

    request_fd = open_clientfd(request->host, request->port);

    rio_writen(request_fd, client_request, strlen(client_request));
    rio_readinitb(&rio, request_fd);
    /** Read the response from the server **/
    while((response_length = rio_readnb(&rio, server_response, MAXLINE)) != 0) {
        rio_writen(conn_fd, server_response, (size_t) response_length);
        if (cacheable) {
            if ((response_length + (cache_ptr - cache_candidate)) <= MAX_OBJECT_SIZE) {
                memcpy(cache_ptr, server_response, response_length);
                cache_ptr += response_length;
            } else {
                /** Mark cacheable as 0 if the response size is too large **/
                cacheable=0;
            }
        }
    }

    if (cacheable) {
        int size = cache_ptr - cache_candidate;

        /** Create a new cache node and insert it into the linked list **/
        cache_item *new_line = malloc(sizeof(cache_item));
        new_line->next = NULL;
        new_line->lru_counter = 0;
        new_line->size = size;
        new_line->data = malloc(size);

        strcpy(new_line->uri, request->uri);
        memcpy(new_line->data, cache_candidate, size);

        insert_cache(new_line, size);
        update_cache(new_line);
    }

    close(request_fd);
    close(conn_fd);
}
```

接下来就是实现缓存的相关函数了，我主要是创建了以下的结构，并且完成了以下几个和缓存相关的函数。简而言之，我使用了一个单链表来组织所有的缓存节点，在 main 函数中，我调用 `initialize_cache()` 函数，创建了整个系统的缓存根节点 cache_root，所以在后续的执行中，可以直接从 cache_root 开始遍历缓存节点，来寻找我们需要的缓存。