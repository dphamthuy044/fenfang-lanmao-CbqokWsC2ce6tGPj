本文分享自天翼云开发者社区《[nginx证书缓存功能](https://github.com)》.作者：云海

# 背景：

ssl证书之前是不支持公用的，不同的域名，如果引用同一本证书，是无法公用的，每个域名都要加载同一个证书，浪费内存

# 新版本：

在1.27.2版本中，nginx官方更新了ssl证书相关的实现，支持了ssl证书缓存共享。

# 实现原理：

在配置初始化时，初始了ssl证书缓存的红黑树：`ngx_openssl_cache_create_conf -> ngx_rbtree_init`

在要获取ssl证书时，通过如下接口获取对应ssl证书：ngx\_ssl\_cache\_fetch该接口的index参数可以传递不同的标记位，因为不同功能ssl证书有不同的创建释放和引用增加处理不同功能ssl证书有如下定义：`typedef struct {``ngx_ssl_cache_create_pt     create;``ngx_ssl_cache_free_pt       free;``ngx_ssl_cache_ref_pt        ref;``} ngx_ssl_cache_type_t;`

核心函数分析：`void *``ngx_ssl_cache_fetch(ngx_conf_t *cf, ngx_uint_t index, char **err,``ngx_str_t *path, void *data)``{``uint32_t               hash;              // 用于存储计算出的哈希值``ngx_ssl_cache_t       *cache;             // 指向 SSL 缓存的指针``ngx_ssl_cache_key_t    id;                // 缓存键，用于标识特定缓存项``ngx_ssl_cache_type_t  *type;              // 缓存类型对象指针``ngx_ssl_cache_node_t  *cn;                // 指向缓存节点的指针`

`// 初始化缓存键（如失败则返回 NULL）``if (ngx_ssl_cache_init_key(cf->pool, index, path, &id) != NGX_OK) {``return NULL;``}`

`// 获取全局的 SSL 缓存对象``cache = (ngx_ssl_cache_t *) ngx_get_conf(cf->cycle->conf_ctx,``ngx_openssl_cache_module);`

`// 根据索引获取对应的缓存类型``type = &ngx_ssl_cache_types[index];`

`// 计算缓存键的哈希值``hash = ngx_murmur_hash2(id.data, id.len);`

`// 在缓存中查找节点``cn = ngx_ssl_cache_lookup(cache, type, &id, hash);``if (cn != NULL) {``// 如果找到节点，则调用类型的 ref 函数返回缓存值``return type->ref(err, cn->value);``}`

`// 如果未找到，则分配一个新的缓存节点内存``cn = ngx_palloc(cf->pool, sizeof(ngx_ssl_cache_node_t) + id.len + 1);``if (cn == NULL) {``return NULL; // 分配失败，返回 NULL``}`

`// 初始化缓存节点的属性``cn->node.key = hash;                // 设置节点的哈希键``cn->id.data = (u_char *)(cn + 1);   // 将 ID 数据存储在分配内存之后``cn->id.len = id.len;                // 设置 ID 长度``cn->id.type = id.type;              // 设置 ID 类型``cn->type = type;                    // 设置节点的类型指针`

`// 复制缓存键数据``ngx_cpystrn(cn->id.data, id.data, id.len + 1);`

`// 调用类型的 create 函数创建缓存值``cn->value = type->create(&id, err, data);``if (cn->value == NULL) {``return NULL; // 创建失败，返回 NULL``}`

`// 将新节点插入到缓存的红黑树中``ngx_rbtree_insert(&cache->rbtree, &cn->node);`

`// 返回节点值的引用``return type->ref(err, cn->value);``}`

nginx官方此功能并没有开关控制，是默认开启的

本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
