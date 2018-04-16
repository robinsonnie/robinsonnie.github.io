# core/ngx_cycle

## 一、数据结构
### 进程

Nginx的所有进程，包括Master、Worker、CacheManager，每个进程均拥有唯一一个ngx_cycle_t结构体，每个进程的启动、初始化、招行都是围绕着此结构体展开。

```
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;            // 内存池，cycle本身就存储在此

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;       // 模块数组，指向各个模块的结构体；由ngx_modules变量拷贝而来
    ngx_uint_t                modules_n;     // 模块数组中的数量
    ngx_uint_t                modules_used;  /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;

    ngx_array_t               listening;
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;

    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n; // worker_connections指令配置值
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;    // 指向init_cycle，该cycle存储命令行参数解析结果、临时的log、等

    ngx_str_t                 conf_file;    // nginx.conf的完整路径，由-c指定
    ngx_str_t                 conf_param;   // 独立于nginx.conf外的全局配置指令，由-g指定
    ngx_str_t                 conf_prefix;  // nginx.conf的前缀，例如：/xx/xx/
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```

## 二、全局变量/常量

```
volatile ngx_cycle_t  *ngx_cycle;
```

## 三、执行流
* init_cycle

```
* 解析配置前：所有NGX_CORE_MODULE类型的模块，调用其m->ctx成员（ngx_core_module_t）的create_conf()创建各自模块所需的上下文，并由cycle->conf_ctx[m->index]指向各自模块所创建的上下文。

* 调用ngx_conf_parse(&conf, &cycle->conf_file)，开始解析配置文件。
    * 遇到directive时，遍历所有模块以确保哪个模块支持此directive，并回调相应ngx_command_t::set()方法。

    * 遇到event{}时：
        * 解析event{}配置前：所有NGX_EVENT_MODULE类型的模块，调用其m->ctx成员（ngx_event_module_t）的create_conf()创建各自模块所需的上下文，并由cycle->conf_ctx[ngx_events_module.index][m->ctx_index]指向各自模块所创建的上下文。
        * 调用ngx_conf_parse(&conf, NULL)，开始解析event{}配置块。
        * 解析event{}配置后：所有NGX_EVENT_MODULE类型的模块，调用其m->ctx成员（ngx_event_module_t）的init_conf()初始化各自模块的上下文。

    * 遇到http{}时：
        * 解析http{}配置前：所有NGX_HTTP_MODULE类型的模块，调用其m->ctx成员（ngx_http_module_t）的main_conf/srv_conf/loc_conf创建上下文：
           * create_main_conf()创建的上下文，由cycle->conf_ctx[ngx_http_module.index]->main_conf[m->ctx_index]存储。
           * create_srv_conf()创建的上下文，由cycle->conf_ctx[ngx_http_module.index]->srv_conf[m->ctx_index]存储。
           * create_loc_conf()创建的上下文，由cycle->conf_ctx[ngx_http_module.index]->loc_conf[m->ctx_index]存储。
        * 解析http{}配置前：所有NGX_HTTP_MODULE类型的模块，调用其m->ctx成员(ngx_http_module_t)的preconfiguration()，此阶段用于创建nginx变量，例如：limit_rate。
        * 调用ngx_conf_parse(&conf, NULL)，开始解析http{}配置块。
           * 遇到server{}时：
              * 

* 解析配置后：所有NGX_CORE_MODULE类型的模块，调用其m->ctx成员（ngx_core_module_t）的init_conf()初始化各自模块的上下文：特别是针对回调set的directive，在上下文中设置其缺省值。
```