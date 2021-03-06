# Redis

Redis组件提供了redis客户端的功能.支持连接池，支持数据的自动序列化，支持故障自动转移，除Do方法以外如果key未明确指定过期时间，则默认过期时间为1天。
    
    1. cluster,数据在多个server间按一致性hash分布。
    2. masterSlave,写命令使用master，读命令在有slave的情况下，选用slave机器。

## 配置文件

```yaml
app.yaml
components:
    # 组件ID，默认为"redis"
    redis:
        # KEY前缀，默认"pgo2_"
         prefix: "pgo2_"
        # 连接密码，默认为空
         password: ""
        # 使用DB的序号，默认为0
         db: 0
        # 每个地址最大空闲连接数，默认为10
         maxIdleConn: 10
        # 连接空闲时间(在这个时间内再次进行操作不需要进行探活)，默认1分钟
         maxIdleTime: "60s"
        # 网络超时(连接、读写等)，默认1秒
         netTimeout: "1s"
        # 服务器探活间隔(自动剔除和添加server)，默认关闭
         probInterval: "0s"
        # 模式,是cluster还是masterSlave,默认cluster 可选值：cluster/masterSlave
         mod:cluster
        # 服务器地址，如果server有权重，请自行按比例添加,如果是master 最好配置在第一个
        servers:
            - "127.0.0.1:6379"
            - "127.0.0.1:6380"
```

## 功能列表

```go
// 从对象池获取对象
this.GetObjBox(adapter.RedisClass).(*adapter.Redis/adapter.IRedis) // (v0.1.131+)
// 初始化方法
NewRedis(componentId ...string) // 对象 this.GetObj(adapter.NewRedis()).(*adapter.Redis/adapter.IRedis)

// redis 方法
SetPanicRecover(v bool)
Get(key string) *value.Value // 获取key的值
MGet(keys []string) map[string]*value.Value  // 并行获取多个key的值
Set(key string, value interface{}, expire ...time.Duration) bool  // 设置key的值
MSet(items map[string]interface{}, expire ...time.Duration) bool // 并行设置多个key的值
Add(key string, value interface{}, expire ...time.Duration) bool // 添加key的值(key存在时添加失败)
MAdd(items map[string]interface{}, expire ...time.Duration) bool // 并行添加多个key的值
Del(key string) bool // 删除key的值
MDel(keys []string) bool // 并行删除多个key的值
Exists(key string) bool // 判断key是否存在
IncrBy(key string, delta int64) (int64, error)  // 累加key的值
Do(cmd string, args ...interface{}) interface{} // 可以使用更多的读写命令
ExpireAt(key string, timestamp int64) bool // 设置过期时间时间点，unix数字，单位秒
Expire(key string, expire time.Duration) bool // 设置有效期，单位秒
RPush(key string, values ...interface{}) bool
LPush(key string, values ...interface{}) bool
RPop(key string) *value.Value
LPop(key string) *value.Value
LLen(key string) int64
HDel(key string, fields ...interface{}) int64
HExists(key, field string) bool
HSet(key string, fv ...interface{}) bool
HGet(key, field string) *value.Value
HGetAll(key string) map[string]*value.Value
HMSet(key string, fv ...interface{}) bool
HMGet(key string, fields ...interface{}) map[string]*value.Value
HIncrBy(key, field string, delta int64) (int64, error)
ZRange(key string, start, end int) []*value.Value
ZRevRange(key string, start, end int) []*value.Value
ZRangeWithScores(key string, start, end int) []*redis.ZV
ZRevRangeWithScores(key string, start, end int) []*redis.ZV
ZAdd(key string, members ...*redis.Z) int64
ZAddOpt(key string, opts []string, members ...*redis.Z) (int64, error) // 增加数据，并可以传入选项 NX XX CH等
ZCard(key string) int64
ZRem(key string, members ...interface{}) int64
// TODO 其它非cache功能
```

## 使用示例

```go

type RedisController struct {
    pgo2.Controller
}

// curl -v http://127.0.0.1:8000/redis/set
func (r *RedisController) ActionSet() {
    // 获取redis的上下文适配对象
    redis := r.GetObj(adapter.NewRedis()).(*adapter.Redis)

    // 设置用户输入值
    key := r.Context().ValidateParam("key", "test_key1").Do()
    val := r.Context().ValidateParam("val", "test_val1").Do()
    ok := redis.Set(key, val)
    fmt.Println("redis set test_key1=test_val1:", ok)

    // 设置自定义过期时间
    ok = redis.Set("test_key2", 100, 2*time.Minute)
    fmt.Println("redis set test_key2=100:", ok)

    // 设置map值，会自动进行json序列化
    data := map[string]interface{}{"f1": 100, "f2": true, "f3": "hello"}
    ok = redis.Set("test_key3", data)
    fmt.Println("redis set test_key3=<map>:", ok)

    // 并行设置多个key
    items := map[string]interface{}{
        "test_key4": []int{1, 2, 3, 4},
        "test_key5": "test_val5",
        "test_key6": map[string]interface{}{"f61": 11, "f62": "hello"},
    }
    ok = redis.MSet(items)
    fmt.Println("redis mset test_key[4-6]:", ok)
}

// curl -v http://127.0.0.1:8000/redis/get
func (r *RedisController) ActionGet() {
    // 对象池获取redis的上下文适配对象
    redis := r.GetObjBox(adapter.RedisClass).(*adapter.Redis)

    // 获取string
    if val := redis.Get("test_key1"); val != nil {
        fmt.Println("value of test_key1:", val.String())
    }

    // 获取int
    if val := redis.Get("test_key2"); val != nil {
        fmt.Println("value of test_key2:", val.Int())
    }

    // 获取序列化的数据
    if val := redis.Get("test_key3"); val != nil {
        var data pgo2.Map
        val.Decode(&data)
        fmt.Println("value of test_key3:", data)
    }

    // 获取多个key
    if res := redis.MGet([]string{"test_key4", "test_key5", "test_key6"}); res != nil {
        for key, val := range res {
            fmt.Printf("value of %s: %v\n", key, val.String())
        }
    }
}
```

