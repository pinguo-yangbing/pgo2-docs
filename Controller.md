# 控制器(Controller)

- 支持HTTP(controller)和命令行(command)控制器
- 支持`URL`路由和`正则`路由(详见Router组件)
- 支持URL动作(Action)和RESTFULL动作(Action)
- 支持参数验证(详见ValidateXxx方法)
- 支持BeforeAction/AfterAction钩子
- 支持HandlePanic钩子，捕获未处理异常
- 提供Json,Jsonp,Data,Xml,ProtoBuf方法，方便输出各种类型数据
- 提供自定义数据类型输出方法Render

由于控制器是请求的入口，需要在main包中手动导入，为方便起见，建议不论controller层有多深统一在controller或command包的init方法中注册。

HTTP控制器位于`pkg/controller`目录下，类后后缀为`Controller`。

命令控制器位于`pkg/command`目录下，类后缀为`Command`，运行命令控制器需要通过`--cmd`选项指定，例如：`bin/pgo-demo --cmd /test/index`。

## 使用示例

```go
package controller

import (
    "net/http"

    "pgo2-demo/pkg/service"

    "github.com/pinguo/pgo2"
)

type WelcomeController struct {
    pgo2.Controller
}

// curl -v http://127.0.0.1:8000/welcome/index
// 默认动作(index)
func (w *WelcomeController) ActionIndex() {
    w.Json("hello world", http.StatusOK)
}

// curl -v http://127.0.0.1:8000/welcome/view
// 模板渲染
func (w *WelcomeController) ActionView() {
    // 获取并验证参数
    name := w.Context().ValidateParam("name", "hitzheng").Do()
    age := w.Context().ValidateParam("age", "100").Int().Do()

    data := map[string]interface{}{
        "name": name,
        "age":  age,
    }

    // 渲染html模板
    w.View("welcome.html", data)
}

// curl -v http://127.0.0.1:8000/welcome/say-hello
// URL路由控制器，根据url自动映射控制器及方法，不需要配置.
// url的最后一段为动作名称，不存在则为index,
// url的其余部分为控制器名称，不存在则为index,
// 例如：/welcome/say-hello，控制器类名为
// controller/WelcomeController 动作方法名为ActionSayHello
func (w *WelcomeController) ActionSayHello() {
    ctx := w.Context() // 获取PGO2请求上下文件

    // 验证参数，提供参数名和默认值，当不提供默认值时，表明该参数为必选参数。
    // 详细验证方法参见Validate.go
    name := ctx.ValidateParam("name").Min(5).Max(50).Do()          // 验证GET/POST参数(string)，为空或验证失败时panic
    age := ctx.ValidateQuery("age", 20).Int().Min(1).Max(100).Do() // 只验证GET参数(int)，为空或失败时返回20
    ip := ctx.ValidatePost("ip", "").IPv4().Do()                   // 只验证POST参数(string), 为空或失败时返回空字符串

    // 打印日志
    ctx.Info("request from welcome, name:%s, age:%d, ip:%s", name, age, ip)
    ctx.PushLog("clientIp", ctx.ClientIp()) // 生成clientIp=xxxxx在pushlog中

    // 调用业务逻辑，一个请求生命周期内的对象都要通过GetObj()获取，
    // 这样可自动查找注册的类，并注入请求上下文(Context)到对象中。
    svc := w.GetObj(service.NewWelcome()).(*service.Welcome)

    // 添加耗时到profile日志中
    ctx.ProfileStart("Welcome.SayHello")
    svc.SayHello(name, age, ip)
    ctx.ProfileStop("Welcome.SayHello")

    // 调用业务逻辑，一个请求生命周期内的对象通过GetObjPool()从对象池获取对象，
    // 这样可自动查找注册的类，并注入请求上下文(Context)到对象中。
    // 简易的从对象池获取对象
    svcPool := w.GetObjPool(service.WelcomeClass, nil).(*service.Welcome)
    svcPool.ShowId()
    // 从对象池获取对象，并初始化某个方法
    svcPool1 := w.GetObjPool(service.WelcomeClass, service.NewWelcomePool, "1123").(*service.Welcome)
    svcPool1.ShowId()

    data := map[string]interface{}{
        "name": name,
        "age":  age,
        "ip":   ip,
    }

    // 输出json数据
    w.Json(data, http.StatusOK)
}

// 正则路由控制器，需要配置Router组件(components.router.rules)
// 规则中捕获的参数通过动作函数参数传递，没有则为空字符串.
// eg. "^/reg/eg/(\\w+)/(\\w+)$ => /welcome/regexp-example"
func (w *WelcomeController) ActionRegexpExample(p1, p2 string) {
    data := map[string]interface{}{"p1": p1, "p2": p2}
    w.Json(data, http.StatusOK)
}
```