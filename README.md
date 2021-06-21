一、运行方法: 
ubuntu 20.04
MYSQL server

1 build.sh生成 server  
2 ./server运行服务器  


二、分模块介绍  
1 http_conn类:   
主要功能：  
1)read_once()函数从客户端读取数据到m_read_buf   
2)prcess_read()函数解析m_read_buf中数据  
3)process_write()函数生成响应报文存储到m_write_buf  
4)do_request()函数将请求的文件映射到内存  
5)write()函数将m_write_buf和文件写入客户端socket  
    
    class http_conn  
    {  
    public:  
        static const int FILENAME_LEN = 200; //服务器上真实文件地址缓冲区长度  
        static const int READ_BUFFER_SIZE = 2048; //m_read_buf的长度  
        static const int WRITE_BUFFER_SIZE = 1024; //m_write_buf的长度  
        enum METHOD //http请求方法  
        {  
            GET = 0,  
            POST,  
            HEAD,  
            PUT,  
            DELETE,  
            TRACE,  
            OPTIONS,  
            CONNECT,  
            PATH  
        };  
        enum CHECK_STATE //主状态机三种状态  
        {  
            CHECK_STATE_REQUESTLINE = 0,  
            CHECK_STATE_HEADER,  
            CHECK_STATE_CONTENT  
        };  
        enum HTTP_CODE  
        {  
            NO_REQUEST, //请求不完整，需要继续读取请求报文数据  
            GET_REQUEST, //获得了完整的HTTP请求  
            BAD_REQUEST, //HTTP请求报文有语法错误  
            NO_RESOURCE, //请求的文件不存在  
            FORBIDDEN_REQUEST, //请求的文件禁止普通用户访问  
            FILE_REQUEST, //文件请求  
            INTERNAL_ERROR, //服务器内部错误，该结果在主状态机逻辑switch的default下，一般不会触发  
            CLOSED_CONNECTION  
        };  
        enum LINE_STATUS  
        {  
            LINE_OK = 0, //成功解析一行  
            LINE_BAD, //失败解析一行  
            LINE_OPEN //一行解析未完成，需要继续接受数据  
        };  

    public:  
        http_conn() {}  
        ~http_conn() {}  

    public:  
        void init(int sockfd, const sockaddr_in &addr, char *, int, int, string user, string passwd, string sqlname);//初始化http_conn对象，将sockfd添加到监听队列  
        void close_conn(bool real_close = true);//将m_sockfd移除队列  
        void process();//调用process_read和process_write,监听写事件  
        bool read_once();//读取客户端数据到m_read_buf LT ET  
        bool write(); //调用writev将响应报文和文件写入m_sockfd  
        sockaddr_in *get_address()//返回客户端地址  
        {  
            return &m_address;  
        }  
        void initmysql_result(connection_pool *connPool);//取出所有用户名及密码  
        int timer_flag;  
        int improv;  


    private:  
        void init();  
        HTTP_CODE process_read(); //主状态机，处理请求报文  
        bool process_write(HTTP_CODE ret); //生成响应报文,写入m_write_buf  
        HTTP_CODE parse_request_line(char *text); //处理请求行，从请求行中获取method，m_url，m_version  
        HTTP_CODE parse_headers(char *text);//处理请求头，获取m_linger，m_content_length，m_host  
        HTTP_CODE parse_content(char *text);//处理请求体，获取m_string  
        HTTP_CODE do_request();//处理请求，包括登录校验，注册用户，将请求的文件映射到内存中，获得m_file_address  
        char *get_line() { return m_read_buf + m_start_line; };//返回每一行的开头  
        LINE_STATUS parse_line();//将m_read_buf的数据按行划分，保证读text读出来的是一行  
        void unmap();  
        bool add_response(const char *format, ...);//将指定内容写入到m_write_buf  
        bool add_content(const char *content);//调用add_response写请求体  
        bool add_status_line(int status, const char *title);//调用add_response写状态行  
        bool add_headers(int content_length);//调用add_content_length(content_len),add_linger(),add_blank_line();  
        bool add_content_type();//调用add_content写Content-Type  
        bool add_content_length(int content_length); //调用add_response写Content-Length  
        bool add_linger();//调用add_response写Connection  
        bool add_blank_line();//调用add_response写空行  

    public:  
        static int m_epollfd; //所有http_conn对象共用的epoll_fd  
        static int m_user_count;//所有http_conn对象共用的user_count,表示正在监听的客户端  
        MYSQL *mysql;  
        int m_state;  //读为0, 写为1  

    private:  
        int m_sockfd;//一个http_conn对象操作一个sockfd  
        sockaddr_in m_address;//存放客户端地址  
        char m_read_buf[READ_BUFFER_SIZE]; //存储客户端数据的缓冲区  
        int m_read_idx;//指向m_read_buf中有数据的最后位置  
        int m_checked_idx;//遍历，用于获取每一行的开头  
        int m_start_line;//指向每一行的开头  
        char m_write_buf[WRITE_BUFFER_SIZE];//响应报文存储缓冲区  
        int m_write_idx;//指向m_write_buf中有数据的最后位置  
        CHECK_STATE m_check_state;//主状态机的状态  
        METHOD m_method;//请求的方法  
        char m_real_file[FILENAME_LEN]; //服务器上真实文件地址  
        char *m_url; //请求的url  
        char *m_version; //请求的http协议版本  
        char *m_host;//请求报文请求的主机名  
        int m_content_length;//请求报文的请求体长度  
        bool m_linger;//true表示长连接，false表示短连接  
        char *m_file_address;//请求的文件在内存中映射的位置  
        struct stat m_file_stat;//存储映射文件的属性  
        struct iovec m_iv[2];//用于writev函数  
        int m_iv_count;//表示m_iv的长度  
        int cgi;        //是否启用的POST  
        char *m_string; //存储用户名和密码  
        int bytes_to_send;//待发送的字节数，包括m_write_idx和映射文件长度  
        int bytes_have_send;//已发送的字节数  
        char *doc_root;//当前工作目录  
    
    map<string, string> m_m_TRIGModeusers;//存储用户名和密码的映射  
    int m_TRIGMode;//连接套接字的触发模式 0 LT 1 ET  
    int m_close_log;  
    
    char sql_user[100];//用户名  
    char sql_passwd[100];//数据库名  
    char sql_name[100];//密码名  
    };  

2 threadpool类:
1)创建指定数量线程
2)所有线程再工作为队列中无http_conn对象时阻塞
3)从工作队列中取出http_conn对象处理

    template <typename T>
    class threadpool
    {
    public:
        /*thread_number是线程池中线程的数量，max_requests是请求队列中最多允许的、等待处理的请求的数量*/
        threadpool(int actor_model, connection_pool *connPool, int thread_number = 8, int max_request = 10000);
        ~threadpool();
        bool append(T *request, int state);
        bool append_p(T *request);

    private:
        /*工作线程运行的函数，它不断从工作队列中取出任务并执行之*/
        static void *worker(void *arg);
        void run();

    private:
        int m_thread_number;        //线程池中的线程数
        int m_max_requests;         //请求队列中允许的最大请求数
        pthread_t *m_threads;       //线程池数组，其大小为m_thread_number
        std::list<T *> m_workqueue; //请求队列
        locker m_queuelocker;       //保护请求队列的互斥锁
        sem m_queuestat;            //是否有任务需要处理
        connection_pool *m_connPool;  //数据库
        int m_actor_model;          //模型切换
    };

3 lst_timer类:
1)将待监听事件挂到红黑树上
2)sig_handler()函数，将信号通过管道发送到主线程
3)主线程每五秒接收到SIG_ALARM信号，回调timer_handler()函数，遍历定时器链表，删除超时节点并将对应socketfd移除监听

    class util_timer;

    struct client_data//客户数据。客户端地址，文件描述符，定时器节点
    {
        sockaddr_in address;
        int sockfd;
        util_timer *timer;
    };

    class util_timer//定时器节点。超时时间，客户数据
    {
    public:
        util_timer() : prev(NULL), next(NULL) {}

    public:
        time_t expire;//超时时间

        void (* cb_func)(client_data *);//删除监听节点，关闭文件描述符
        client_data *user_data;
        util_timer *prev;
        util_timer *next;
    };

    class sort_timer_lst//容器链表
    {
    public:
        sort_timer_lst();
        ~sort_timer_lst();

        void add_timer(util_timer *timer);//添加节点
        void adjust_timer(util_timer *timer);//调整节点
        void del_timer(util_timer *timer);//删除节点
        void tick();//遍历，删除超时的节点

    private:
        void add_timer(util_timer *timer, util_timer *lst_head);

        util_timer *head;
        util_timer *tail;
    };

    class Utils//定时器类
    {
    public:
        Utils() {}
        ~Utils() {}

        void init(int timeslot);//初始化定时器类,主线程中调用

        //对文件描述符设置非阻塞
        int setnonblocking(int fd);

        //将内核事件表注册读事件，ET模式，ne_shot = true 开启EPOLLONESHOT， TRIGMode == 1, ET模式
        void addfd(int epollfd, int fd, bool one_shot, int TRIGMode);

        //信号处理函数
        static void sig_handler(int sig);

        //设置信号函数
        void addsig(int sig, void(handler)(int), bool restart = true);

        //定时处理任务，重新定时以不断触发SIGALRM信号
        void timer_handler();//调用tick并重新定时

        void show_error(int connfd, const char *info);

    public:
        static int *u_pipefd;//管道
        sort_timer_lst m_timer_lst;//容器链表
        static int u_epollfd;//监听红黑树
        int m_TIMESLOT;//定时任务的定时时间
    };

    void cb_func(client_data *user_data);//删除定时器节点时要配合使用的函数，关闭文件描述符，取下监听节点

4 webserver类:
1)初始化
