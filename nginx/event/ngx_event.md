# event/ngx_event

## 一、数据结构

### NGX\_EVENT\_MODULE的上下文结构体
```
typedef struct {
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*notify)(ngx_event_handler_pt handler);

    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags);

    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;

typedef struct {
    ngx_str_t              *name;
    
    void                 *(*create_conf)(ngx_cycle_t *cycle);
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);
    
    ngx_event_actions_t     actions;
} ngx_event_module_t;
```

## 二、全局变量/常量

## 三、ngx\_event\_core\_module模块
### 上下文
```
typedef struct {
    ngx_uint_t    connections;
    ngx_uint_t    use;               // 记录所用事件模型的ctx_index

    ngx_flag_t    multi_accept;
    ngx_flag_t    accept_mutex;

    ngx_msec_t    accept_mutex_delay;

    u_char       *name;              // 记录所用事件模型的name

#if (NGX_DEBUG)
    ngx_array_t   debug_connection;  // debug_connection指令支持
#endif
} ngx_event_conf_t;
```