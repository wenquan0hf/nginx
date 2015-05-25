# Nginx 的请求处理阶段

## 接收请求流程

### http请求格式简介

首先介绍一下rfc2616中定义的http请求基本格式：

```
 Request = Request-Line 
        * (( general-header         
              | request-header          
              | entity-header ) CRLF)  
           CRLF
          [ message-body ]
```

第一行是请求行（request line），用来说明请求方法，要访问的资源以及所使用的HTTP版本：

```
Request-Line   = Method SP Request-URI SP HTTP-Version CRLF
```

请求方法（Method）的定义如下，其中最常用的是GET，POST方法：

```
 Method = "OPTIONS" 
    | "GET" 
    | "HEAD" 
    | "POST" 
    | "PUT" 
    | "DELETE" 
    | "TRACE" 
    | "CONNECT" 
    | extension-method 
    extension-method = token
```

要访问的资源由统一资源地位符URI(Uniform Resource Identifier)确定，它的一个比较通用的组成格式（rfc2396）如下：

```
 <scheme>://<authority><path>?<query> 
```

一般来说根据请求方法（Method）的不同，请求URI的格式会有所不同，通常只需写出path和query部分。

http版本(version)定义如下，现在用的一般为1.0和1.1版本：

```
HTTP/<major>.<minor>
```
请求行的下一行则是请求头，rfc2616中定义了3种不同类型的请求头，分别为general-header，request-header和entity-header，每种类型rfc中都定义了一些通用的头，其中entity-header类型可以包含自定义的头。

### 请求头读取

这一节介绍nginx中请求头的解析，nginx的请求处理流程中，会涉及到2个非常重要的数据结构，ngx_connection_t和ngx_http_request_t，分别用来表示连接和请求，这2个数据结构在本书的前篇中已经做了比较详细的介绍，没有印象的读者可以翻回去复习一下，整个请求处理流程从头到尾，对应着这2个数据结构的分配，初始化，使用，重用和销毁。

nginx在初始化阶段，具体是在init process阶段的ngx_event_process_init函数中会为每一个监听套接字分配一个连接结构（ngx_connection_t），并将该连接结构的读事件成员（read）的事件处理函数设置为ngx_event_accept，并且如果没有使用accept互斥锁的话，在这个函数中会将该读事件挂载到nginx的事件处理模型上（poll或者epoll等），反之则会等到init process阶段结束，在工作进程的事件处理循环中，某个进程抢到了accept锁才能挂载该读事件。

```
static ngx_int_t
    ngx_event_process_init(ngx_cycle_t *cycle)
    {
        ...

        /* 初始化用来管理所有定时器的红黑树 */
        if (ngx_event_timer_init(cycle->log) == NGX_ERROR) {
            return NGX_ERROR;
        }
        /* 初始化事件模型 */
        for (m = 0; ngx_modules[m]; m++) {
            if (ngx_modules[m]->type != NGX_EVENT_MODULE) {
                continue;
            }

            if (ngx_modules[m]->ctx_index != ecf->use) {
                continue;
            }

            module = ngx_modules[m]->ctx;

            if (module->actions.init(cycle, ngx_timer_resolution) != NGX_OK) {
                /* fatal */
                exit(2);
            }

            break;
        }

        ...

        /* for each listening socket */
        /* 为每个监听套接字分配一个连接结构 */
        ls = cycle->listening.elts;
        for (i = 0; i < cycle->listening.nelts; i++) {

            c = ngx_get_connection(ls[i].fd, cycle->log);

            if (c == NULL) {
                return NGX_ERROR;
            }

            c->log = &ls[i].log;

            c->listening = &ls[i];
            ls[i].connection = c;

            rev = c->read;

            rev->log = c->log;
            /* 标识此读事件为新请求连接事件 */
            rev->accept = 1;

            ...

    #if (NGX_WIN32)

            /* windows环境下不做分析，但原理类似 */

    #else
            /* 将读事件结构的处理函数设置为ngx_event_accept */
            rev->handler = ngx_event_accept;
            /* 如果使用accept锁的话，要在后面抢到锁才能将监听句柄挂载上事件处理模型上 */
            if (ngx_use_accept_mutex) {
                continue;
            }
            /* 否则，将该监听句柄直接挂载上事件处理模型 */
            if (ngx_event_flags & NGX_USE_RTSIG_EVENT) {
                if (ngx_add_conn(c) == NGX_ERROR) {
                    return NGX_ERROR;
                }

            } else {
                if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                    return NGX_ERROR;
                }
            }

    #endif

        }

        return NGX_OK;
    }
    ```
    当一个工作进程在某个时刻将监听事件挂载上事件处理模型之后，nginx就可以正式的接收并处理客户端过来的请求了。这时如果有一个用户在浏览器的地址栏内输入一个域名，并且域名解析服务器将该域名解析到一台由nginx监听的服务器上，nginx的事件处理模型接收到这个读事件之后，会交给之前注册好的事件处理函数ngx_event_accept来处理。

在ngx_event_accept函数中，nginx调用accept函数，从已连接队列得到一个连接以及对应的套接字，接着分配一个连接结构（ngx_connection_t），并将新得到的套接字保存在该连接结构中，这里还会做一些基本的连接初始化工作：

1, 首先给该连接分配一个内存池，初始大小默认为256字节，可通过connection_pool_size指令设置；

2, 分配日志结构，并保存在其中，以便后续的日志系统使用；

3, 初始化连接相应的io收发函数，具体的io收发函数和使用的事件模型及操作系统相关；

4, 分配一个套接口地址（sockaddr），并将accept得到的对端地址拷贝在其中，保存在sockaddr字段；

5, 将本地套接口地址保存在local_sockaddr字段，因为这个值是从监听结构ngx_listening_t中可得，而监听结构中保存的只是配置文件中设置的监听地址，但是配置的监听地址可能是通配符*，即监听在所有的地址上，所以连接中保存的这个值最终可能还会变动，会被确定为真正的接收地址；

6, 将连接的写事件设置为已就绪，即设置ready为1，nginx默认连接第一次为可写；

7, 如果监听套接字设置了TCP_DEFER_ACCEPT属性，则表示该连接上已经有数据包过来，于是设置读事件为就绪；

8, 将sockaddr字段保存的对端地址格式化为可读字符串，并保存在addr_text字段；

最后调用ngx_http_init_connection函数初始化该连接结构的其他部分。

ngx_http_init_connection函数最重要的工作是初始化读写事件的处理函数：将该连接结构的写事件的处理函数设置为ngx_http_empty_handler，这个事件处理函数不会做任何操作，实际上nginx默认连接第一次可写，不会挂载写事件，如果有数据需要发送，nginx会直接写到这个连接，只有在发生一次写不完的情况下，才会挂载写事件到事件模型上，并设置真正的写事件处理函数，这里后面的章节还会做详细介绍；读事件的处理函数设置为ngx_http_init_request，此时如果该连接上已经有数据过来（设置了deferred accept)，则会直接调用ngx_http_init_request函数来处理该请求，反之则设置一个定时器并在事件处理模型上挂载一个读事件，等待数据到来或者超时。当然这里不管是已经有数据到来，或者需要等待数据到来，又或者等待超时，最终都会进入读事件的处理函数-ngx_http_init_request。

ngx_http_init_request函数主要工作即是初始化请求，由于它是一个事件处理函数，它只有唯一一个ngx_event_t \*类型的参数，ngx_event_t 结构在nginx中表示一个事件，事件处理的上下文类似于一个中断处理的上下文，为了在这个上下文得到相关的信息，nginx中一般会将连接结构的引用保存在事件结构的data字段，请求结构的引用则保存在连接结构的data字段，这样在事件处理函数中可以方便的得到对应的连接结构和请求结构。进入函数内部看一下，首先判断该事件是否是超时事件，如果是的话直接关闭连接并返回；反之则是指之前accept的连接上有请求过来需要处理。

ngx_http_init_request函数首先在连接的内存池中为该请求分配一个ngx_http_request_t结构，这个结构将用来保存该请求所有的信息。分配完之后，这个结构的引用会被包存在连接的hc成员的request字段，以便于在长连接或pipelined请求中复用该请求结构。在这个函数中，nginx根据该请求的接收端口和地址找到一个默认虚拟服务器配置（listen指令的default_server属性用来标识一个默认虚拟服务器，否则监听在相同端口和地址的多个虚拟服务器，其中第一个定义的则为默认）。

nginx配置文件中可以设置多个监听在不同端口和地址的虚拟服务器（每个server块对应一个虚拟服务器），另外还根据域名（server_name指令可以配置该虚拟服务器对应的域名）来区分监听在相同端口和地址的虚拟服务器，每个虚拟服务器可以拥有不同的配置内容，而这些配置内容决定了nginx在接收到一个请求之后如何处理该请求。找到之后，相应的配置被保存在该请求对应的ngx_http_request_t结构中。注意这里根据端口和地址找到的默认配置只是临时使用一下，最终nginx会根据域名找到真正的虚拟服务器配置，随后的初始化工作还包括：

1, 将连接的读事件的处理函数设置为ngx_http_process_request_line函数，这个函数用来解析请求行，将请求的read_event_handler设置为ngx_http_block_reading函数，这个函数实际上什么都不做（当然在事件模型设置为水平触发时，唯一做的事情就是将事件从事件模型监听列表中删除，防止该事件一直被触发），后面会说到这里为什么会将read_event_handler设置为此函数；

2, 为这个请求分配一个缓冲区用来保存它的请求头，地址保存在header_in字段，默认大小为1024个字节，可以使用client_header_buffer_size指令修改，这里需要注意一下，nginx用来保存请求头的缓冲区是在该请求所在连接的内存池中分配，而且会将地址保存一份在连接的buffer字段中，这样做的目的也是为了给该连接的下一次请求重用这个缓冲区，另外如果客户端发过来的请求头大于1024个字节，nginx会重新分配更大的缓存区，默认用于大请求的头的缓冲区最大为8K，最多4个，这2个值可以用large_client_header_buffers指令设置，后面还会说到请求行和一个请求头都不能超过一个最大缓冲区的大小；

3, 为这个请求分配一个内存池，后续所有与该请求相关的内存分配一般都会使用该内存池，默认大小为4096个字节，可以使用request_pool_size指令修改；

4, 为这个请求分配响应头链表，初始大小为20；

5, 创建所有模块的上下文ctx指针数组，变量数据；

6, 将该请求的main字段设置为它本身，表示这是一个主请求，nginx中对应的还有子请求概念，后面的章节会做详细的介绍；

7, 将该请求的count字段设置为1，count字段表示请求的引用计数；

8, 将当前时间保存在start_sec和start_msec字段，这个时间是该请求的起始时刻，将被用来计算一个请求的处理时间（request time），nginx使用的这个起始点和apache略有差别，nginx中请求的起始点是接收到客户端的第一个数据包的事件开始，而apache则是接收到客户端的整个request line后开始算起；

9, 初始化请求的其他字段，比如将uri_changes设置为11，表示最多可以将该请求的uri改写10次，subrequests被设置为201，表示一个请求最多可以发起200个子请求；

做完所有这些初始化工作之后，ngx_http_init_request函数会调用读事件的处理函数来真正的解析客户端发过来的数据，也就是会进入ngx_http_process_request_line函数中处理。

### 解析请求行

ngx_http_process_request_line函数的主要作用即是解析请求行，同样由于涉及到网络IO操作，即使是很短的一行请求行可能也不能被一次读完，所以在之前的ngx_http_init_request函数中，ngx_http_process_request_line函数被设置为读事件的处理函数，它也只拥有一个唯一的ngx_event_t \*类型参数，并且在函数的开头，同样需要判断是否是超时事件，如果是的话，则关闭这个请求和连接；否则开始正常的解析流程。先调用ngx_http_read_request_header函数读取数据。

由于可能多次进入ngx_http_process_request_line函数，ngx_http_read_request_header函数首先检查请求的header_in指向的缓冲区内是否有数据，有的话直接返回；否则从连接读取数据并保存在请求的header_in指向的缓存区，而且只要缓冲区有空间的话，会一次尽可能多的读数据，读到多少返回多少；如果客户端暂时没有发任何数据过来，并返回NGX_AGAIN，返回之前会做2件事情：

1，设置一个定时器，时长默认为60s，可以通过指令client_header_timeout设置，如果定时事件到达之前没有任何可读事件，nginx将会关闭此请求；

2，调用ngx_handle_read_event函数处理一下读事件-如果该连接尚未在事件处理模型上挂载读事件，则将其挂载上；

如果客户端提前关闭了连接或者读取数据发生了其他错误，则给客户端返回一个400错误（当然这里并不保证客户端能够接收到响应数据，因为客户端可能都已经关闭了连接），最后函数返回NGX_ERROR；

如果ngx_http_read_request_header函数正常的读取到了数据，ngx_http_process_request_line函数将调用ngx_http_parse_request_line函数来解析，这个函数根据http协议规范中对请求行的定义实现了一个有限状态机，经过这个状态机，nginx会记录请求行中的请求方法（Method），请求uri以及http协议版本在缓冲区中的起始位置，在解析过程中还会记录一些其他有用的信息，以便后面的处理过程中使用。如果解析请求行的过程中没有产生任何问题，该函数会返回NGX_OK；如果请求行不满足协议规范，该函数会立即终止解析过程，并返回相应错误号；如果缓冲区数据不够，该函数返回NGX_AGAIN。

在整个解析http请求的状态机中始终遵循着两条重要的原则：减少内存拷贝和回溯。

内存拷贝是一个相对比较昂贵的操作，大量的内存拷贝会带来较低的运行时效率。nginx在需要做内存拷贝的地方尽量只拷贝内存的起始和结束地址而不是内存本身，这样做的话仅仅只需要两个赋值操作而已，大大降低了开销，当然这样带来的影响是后续的操作不能修改内存本身，如果修改的话，会影响到所有引用到该内存区间的地方，所以必须很小心的管理，必要的时候需要拷贝一份。

这里不得不提到nginx中最能体现这一思想的数据结构，ngx_buf_t，它用来表示nginx中的缓存，在很多情况下，只需要将一块内存的起始地址和结束地址分别保存在它的pos和last成员中，再将它的memory标志置1，即可表示一块不能修改的内存区间，在另外的需要一块能够修改的缓存的情形中，则必须分配一块所需大小的内存并保存其起始地址，再将ngx_buf_t的temporary标志置1，表示这是一块能够被修改的内存区域。

再回到ngx_http_process_request_line函数中，如果ngx_http_parse_request_line函数返回了错误，则直接给客户端返回400错误；
如果返回NGX_AGAIN，则需要判断一下是否是由于缓冲区空间不够，还是已读数据不够。如果是缓冲区大小不够了，nginx会调用ngx_http_alloc_large_header_buffer函数来分配另一块大缓冲区，如果大缓冲区还不够装下整个请求行，nginx则会返回414错误给客户端，否则分配了更大的缓冲区并拷贝之前的数据之后，继续调用ngx_http_read_request_header函数读取数据来进入请求行自动机处理，直到请求行解析结束；

如果返回了NGX_OK，则表示请求行被正确的解析出来了，这时先记录好请求行的起始地址以及长度，并将请求uri的path和参数部分保存在请求结构的uri字段，请求方法起始位置和长度保存在method_name字段，http版本起始位置和长度记录在http_protocol字段。还要从uri中解析出参数以及请求资源的拓展名，分别保存在args和exten字段。接下来将要解析请求头，将在下一小节中接着介绍。

