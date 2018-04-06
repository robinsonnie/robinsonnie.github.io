# core/ngx_conf\_file

## 一、数据结构

### 关键字结构体：描述与处理配置文件中的关键字
```


struct ngx_command_s {
    ngx_str_t             name;    // 配置关键字的名称
    ngx_uint_t            type;    // 配置关键字的类型，64位划分TT - FF - AAAA，每位表示1个含义，可以叠加。
                                   //     AAAA  number of arguments     
                                   //   FF      command flags
                                   // TT        command type, i.e. HTTP "location" or "server" command
// ngx_command_s:type中的位宏（TT）
// #define NGX_MAIN_CONF        0x01000000  
// #define NGX_ANY_CONF         0x1F000000 
                                   
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
                                   // 解析配置文件时，遇到关键字时的回调函数
                                   // 参数1：指向配置结构体（正在解析配置文件）
                                   // 参数2：指向定义此directive的module的cmd定义，即ngx_module_t::commands数组中的成员
                                   // 参数3：
                                   
    ngx_uint_t            conf;    // 配置结构体的起始偏移
    ngx_uint_t            offset;  // 配置结构体的对应于本directive的成员的偏移
                                   // 对应directive成员的地址偏移 = conf + offset；
                                   // 设置有conf + offset的ngx_command_s，可以直接使用如下函数将指令配置设置进对应成员：
// char *ngx_conf_set_flag_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_str_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_str_array_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_keyval_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_num_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_size_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_off_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_msec_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_sec_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_bufs_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_enum_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
// char *ngx_conf_set_bitmask_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);

    void                 *post;     // 在ngx_conf_set_xx_slot完成设置后，回根据不同类型的指令，进行回调如下函数。
// typedef char *(*ngx_conf_post_handler_pt) (ngx_conf_t *cf, void *data, void *conf);
};
```

### 配置结构体：描述与处理nginx.conf配置文件
```
typedef char *(*ngx_conf_handler_pt)(ngx_conf_t *cf,
                                     ngx_command_t *dummy,
                                     void *conf);
typedef struct {
    ngx_file_t            file;
    ngx_buf_t            *buffer;
    ngx_buf_t            *dump;        // 支持-d选项，dump所有配置
    ngx_uint_t            line;        // 解析配置文件过程中，记录行数
} ngx_conf_file_t;

struct ngx_conf_s {
    char                 *name;
    ngx_array_t          *args;        // 数组，用于解析配置时的token
                                       // set $var 1;  // 3个token

    ngx_cycle_t          *cycle;       // reference to cycle
    ngx_pool_t           *pool;        // reference to cycle->pool
    ngx_pool_t           *temp_pool;   // 创建单独的内存池，用于解析配置
    ngx_conf_file_t      *conf_file;   // 用于打开、读取配置文件
    ngx_log_t            *log;

    void                 *ctx;         // reference to cycle->conf_ctx
    ngx_uint_t            module_type; // 解析配置文件的过程中，用于指明正在解析哪级模块支持的指令： NGX_CORE_MODULE 
    ngx_uint_t            cmd_type;    // 解析配置文件的过程中，用于指明正在解析何位置的指令：NGX_MAIN_CONF

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
};
```

## 二、全局变量/常量

## 三、conf模块

本文件定义了 ngx\_conf\_module 模块

* 该模块是nginx中所有模块唯一一个 NGX\_CONF\_MODULE 类型的模块。
* 支持include指令
 



