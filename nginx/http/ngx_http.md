# http/ngx_http

## 一、数据结构

### NGX\_HTTP\_MODULE的上下文结构体
```
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```

## 二、全局变量/常量
```
ngx_uint_t   ngx_http_max_module;    // http module数量
ngx_http_output_header_filter_pt  ngx_http_top_header_filter;
ngx_http_output_body_filter_pt    ngx_http_top_body_filter;
ngx_http_request_body_filter_pt   ngx_http_top_request_body_filter;

```

## 三、ngx\_http\_module模块
### 上下文
```
typedef struct {
    void        **main_conf;         // http{}块；指针数组，ngx_http_max_module个元素
    void        **srv_conf;          // server{}块；同上
    void        **loc_conf;          // location{}块；同上
} ngx_http_conf_ctx_t;
```