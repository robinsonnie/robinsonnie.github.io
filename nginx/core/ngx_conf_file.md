# core/ngx_conf\_file

## 一、数据结构

### 关键字结构体：描述与处理配置文件中的关键字
```
// ngx_command_s:type中的位宏
// AAAA：0-31位
#define NGX_CONF_NOARGS      0x00000001
#define NGX_CONF_TAKE1       0x00000002
#define NGX_CONF_TAKE2       0x00000004
#define NGX_CONF_TAKE3       0x00000008
#define NGX_CONF_TAKE4       0x00000010
#define NGX_CONF_TAKE5       0x00000020
#define NGX_CONF_TAKE6       0x00000040
#define NGX_CONF_TAKE7       0x00000080
#define NGX_CONF_TAKE12      (NGX_CONF_TAKE1|NGX_CONF_TAKE2)
#define NGX_CONF_TAKE13      (NGX_CONF_TAKE1|NGX_CONF_TAKE3)
#define NGX_CONF_TAKE23      (NGX_CONF_TAKE2|NGX_CONF_TAKE3)
#define NGX_CONF_TAKE123     (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3)
#define NGX_CONF_TAKE1234    (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3|NGX_CONF_TAKE4)
#define NGX_CONF_ARGS_NUMBER 0x000000ff
#define NGX_CONF_BLOCK       0x00000100    # 表明本direcitve定义了一个{}
#define NGX_CONF_FLAG        0x00000200
#define NGX_CONF_ANY         0x00000400
#define NGX_CONF_1MORE       0x00000800
#define NGX_CONF_2MORE       0x00001000
// FF：32-47位
#define NGX_DIRECT_CONF      0x00010000
// TT: 48-63位
#define NGX_MAIN_CONF        0x01000000  
#define NGX_ANY_CONF         0x1F000000 

struct ngx_command_s {
    ngx_str_t             name;    // 配置关键字的名称
    ngx_uint_t            type;    // 配置关键字的类型，64位，每位表示1个含义，可以叠加。
                                   //     AAAA  number of arguments     
                                   //   FF      command flags
                                   // TT        command type, i.e. HTTP "location" or "server" command
                                   
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
                                   // 解析配置文件时，遇到关键字时的回调函数
                                   // 参数1：指向配置结构体（正在解析配置文件）
                                   // 参数2：指向定义此directive的模块的cmd定义，即ngx_module_t::commands数组中的成员
                                   // 参数3：指向定义此directive的模块的上下文，由m->ctx所创建。
                                   
    ngx_uint_t            conf;    // 配置结构体的起始偏移
    ngx_uint_t            offset;  // 配置结构体的对应于本directive的成员的偏移
                                   // 对应directive成员的地址偏移 = conf + offset；
                                   // 惯用法：conf = 0, offset = offsetof(配置结构体, 成员)
                                   // 设置有conf + offset的ngx_command_s，可以直接使用如下函数作为set方法，以将directive配置情况同步至上下文的对应成员：
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

    void                 *post;     // 根据指令类型的不同，post有不同含义。
                                    // 对于大部分指令：在ngx_conf_set_xx_slot完成设置后，回根据不同类型的指令，进行回调如下函数。
                                    //     typedef char *(*ngx_conf_post_handler_pt) (ngx_conf_t *cf, void *data, void *conf);
                                    // 对于ngx_conf_set_enum_slot：post存储其枚举定义，参见debug_points指令的定义。
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

    // 解析配置文件的过程中：ctx指明上下文 | module_type指明正在解析哪级模块支持的指令 | cmd_type指明正在解析何位置的指令
    // 解析NGX_CORE_MODULE指令：cycle->conf_ctx | NGX_CORE_MODULE | NGX_MAIN_CONF
    // 解析NGX_EVENT_MODULE指令：cycle->conf_ctx[ngx_events_module.index] | NGX_EVENT_MODULE | NGX_EVENT_CONF
    void                 *ctx; 
    ngx_uint_t            module_type;
    ngx_uint_t            cmd_type;

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
};
```

## 二、全局变量/常量

## 三、ngx\_conf\_module模块

本文件定义了 ngx\_conf\_module 模块

* 该模块是nginx中所有模块唯一一个 NGX\_CONF\_MODULE 类型的模块。
* 支持include指令
 



