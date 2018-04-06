# core/	ngx_module 

## 一、数据结构

### 通用模块

```
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;      // 本模块在ngx_modules中的下标
    // 0 -> ngx_core_module,
    // 1 -> ngx_errlog_module,
    // 2 -> ngx_conf_module,
    // 3 -> ngx_regex_module,
    // 4 -> ngx_events_module,
    // 5 -> ngx_event_core_module,
    // 6 -> ngx_epoll_module,
    // 7 -> ngx_http_module,
    // 8 -> ngx_http_core_module,
    // 9 -> ngx_http_log_module,
    // ... 参考ngx_modules中的顺序

    char                 *name;       // 本模块的名字

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;    // nginx版本号
    const char           *signature;  // nginx签名，描述当前nginx版本所支持的各项能力（支持ipv6? FastOpen? Gzip?等）

    void                 *ctx;
    ngx_command_t        *commands;   // 定义本模块关心的配置关键字
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};

```

### 核心模块
```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

## 二、全局变量/常量
```
// 声明由./configure生成的objs/ngx_modules.c的同名变量，见下面三
extern ngx_module_t  *ngx_modules[]; 
extern char          *ngx_module_names[]; 

extern ngx_uint_t     ngx_max_module;  // ngx_modules数组中的模块数量上限
static ngx_uint_t     ngx_modules_n;   // ngx_modules数组中的当前模块数量
```

## 三、objs/ngx_modules.c

```
ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_regex_module,
    &ngx_events_module,
    &ngx_event_core_module,
    &ngx_epoll_module,
    &ngx_http_module,
    &ngx_http_core_module,
    &ngx_http_log_module,
    &ngx_http_upstream_module,
    &ngx_http_static_module,
    &ngx_http_autoindex_module,
    &ngx_http_index_module,
    ....
    &ngx_http_write_filter_module,
    &ngx_http_header_filter_module,
    &ngx_http_chunked_filter_module,
    &ngx_http_range_header_filter_module,
    &ngx_http_gzip_filter_module,
    &ngx_http_postpone_filter_module,
    &ngx_http_ssi_filter_module,
    &ngx_http_charset_filter_module,
    &ngx_http_userid_filter_module,
    &ngx_http_headers_filter_module,
    &ngx_http_copy_filter_module,
    &ngx_http_range_body_filter_module,
    &ngx_http_not_modified_filter_module,
    NULL
};

char *ngx_module_names[] = {
    "ngx_core_module",
    "ngx_errlog_module",
    "ngx_conf_module",
    "ngx_regex_module",
    "ngx_events_module",
    "ngx_event_core_module",
    "ngx_epoll_module",
    "ngx_http_module",
    "ngx_http_core_module",
    "ngx_http_log_module",
    "ngx_http_upstream_module",
    "ngx_http_static_module",
    "ngx_http_autoindex_module",
    "ngx_http_index_module",
    ....
    "ngx_http_write_filter_module",
    "ngx_http_header_filter_module",
    "ngx_http_chunked_filter_module",
    "ngx_http_range_header_filter_module",
    "ngx_http_gzip_filter_module",
    "ngx_http_postpone_filter_module",
    "ngx_http_ssi_filter_module",
    "ngx_http_charset_filter_module",
    "ngx_http_userid_filter_module",
    "ngx_http_headers_filter_module",
    "ngx_http_copy_filter_module",
    "ngx_http_range_body_filter_module",
    "ngx_http_not_modified_filter_module",
    NULL
};
```