# isuperagent

[![Build Status](https://travis-ci.org/charleslxh/isuperagent.svg?branch=master)](https://travis-ci.org/charleslxh/isuperagent)

isuperagent 是一个通用、灵活的 HTTP CLIENT 库，包装了请求、响应参数。提供了灵活的属性，支持自定义 parser、支持组件（洋葱模型）、链式调用等特性。 

## 示例

```go
timeMiddleware, err := isuperagent.NewMiddleware("request_time")
basicAuthMiddleware, err := isuperagent.NewMiddleware("basic_auth", "username", "password")
debugMiddleware, err := isuperagent.NewMiddleware("debug", func(ctx isuperagent.Context) {
    log.Println(fmt.Sprintf("req headers: %+v", ctx.GetReq().GetHeaders()))
})

res, err := isuperagent.NewRequest().Get("http://localhost:8080/").Middleware(timeMiddleware, basicAuthMiddleware, debugMiddleware).Do()
```

## 安装

```bash
go get github.com/charleslxh/isuperagent
```

## 特性

1. 支持中间件（洋葱模型）。
2. 支持灵活的请求解析器 body parser，并且支持自定义解析器、覆盖现有解析器功能。
3. 封装了请求、响应属性。
4. 支持链式调用。
6. 支持 HTTPS 调用请求、支持 SSL 单向认证、双向认证。
7. 支持出错重试机制。

### 中间件

支持 `中间件` 是 `ispueragent` 的最大特色，灵活的中间件能给开发者带来极大的便捷性、可操作性。

洋葱模型出自 `NodeJS` 的 WEB 框架 [Koa](https://github.com/koajs/koa)，`isuperagent` 由参考该框架的中间件思想构成。

参考资料：[洋葱模型](https://www.jianshu.com/p/dce0727002a6)。

整体设计思想如下：

用户调用时，发起请求之前（调用 `.Do()` 为发起请求）用户可以通过 `Middleware` 函数注册中间件，中间件的请求调用顺序（在发送请求之前）与注册时的顺序一致，
中间件的响应调用顺序（在返回结果之前）与注册时的顺序相反，具体链路如下：

`发起请求 -> middleware A -> middleware B -> 请求服务器 -> middleware B -> middleware A -> 返回响应`

![洋葱模型](https://github.com/charleslxh/isuperagent/blob/master/middleware.png)

**注意：请求响应和错误需要中间件一层一层的向上返回，如果中间某个中间件没有返回，则响应会被该中间价吞噬，错误也会被隐藏。**

#### 如何自定义中间件

1. 自定义中间件必须实现 `isuperagent.Middleware` 签名。

    ```go
   func(ctx isuperagent.Context, next isuperagent.Next) error {
       return nil
   }
    ```

3. 中间件需要返回响应体和错误，如果你不关心，可以直接吞噬掉。

    ```go
   func(ctx isuperagent.Context, next isuperagent.Next) error {
        mockRes := &Response{StatusCode: 404, StatusText: "Not Found"}
        ctx.SetRes(mockRes)
   
        return nil
    }
    ```

4. 中间件提供 `next` 函数，用于调用下一个中间件，如果你不需要继续调用下一个中间件，则不需要调用该函数。

    ```go
   func(ctx isuperagent.Context, next isuperagent.Next) error {
        return next()
    }
    ```

    > 注意：在 isuperagent 中，所有都是有中间件组成，请求的执行本质上就是中间件的遍历过程，所以最终发送请求的过程也是一个中间件，该中间价是内置在最末端的，所以，如果你中间某个中间件没有调用 `next` 方法，则请求也将不会发生，建议必须调用 next 函数，防止中间件链路完整

**提示：可以参考现有的中间件写法。**

#### 中间件如何创建

`isuperagent` 支持中间件本质上就是一个签名为 `func(ctx isuperagent.Context, next isuperagent.Next) error` 的函数，任何改签名的函数都能注入。

```go
func(ctx isuperagent.Context, next isuperagent.Next) error {
    start := time.Now()
    defer func() {
    	duration := fmt.Sprintf("%s", time.Now().Sub(start))
    	ctx.Set("request_time", duration)
    	ctx.GetRes().GetHeaders().Set("X-SuperAgent-Duration", duration)
    }()
    
    return next()
}
```

**提示：可以参考现有的中间件工厂方法写法。**

#### 如何为中间件绑定不同的参数

`isuperagent` 提供了工厂方法来统一生产中间件，前提是必须向工厂注册你的中间件，`isuperagent` 内置一系列的中间件。

1. 创建工厂方法，函数签名必须是 `func NewTimeMiddlewareFactory(v ...interface{}) (isuperagent.Middleware, error)`。

    ```go
    // Middleware: record the request duration
    func NewTimeMiddlewareFactory(v ...interface{}) (isuperagent.Middleware, error) {
    	headerName := "x-SuperAgent-Duration"
    	if len(v) > 0 {
    		if h, ok := v[0].(string); ok {
    			headerName = h
    		} else {
    			return nil, errors.New(fmt.Sprintf("excepted header_name is string, but got %v(%s)", v[0], reflect.TypeOf(v[0])))
    		}
    	}
    
    	return func(ctx isuperagent.Context, next isuperagent.Next) error {
    		start := time.Now()
    		defer func() {
    			duration := fmt.Sprintf("%s", time.Now().Sub(start))
    			ctx.Set("request_time", duration)
    			ctx.GetRes().GetHeaders().Set("X-SuperAgent-Duration", duration)
    		}()
    
    		return next()
    	}, nil
    }
    ``` 

2. 注册工厂方法，**工厂方法必须注册之后才能使用，配合 `init` 函数**。

    ```go
    func init() {
    	isuperagent.RegisterMiddlewareFactory("request_time", NewTimeMiddlewareFactory)
    }
    ```

3. 创建中间件实例。

    ```go
    middleware, err := isuperagent.NewMiddleware("request_time")
   	if err != nil {
   		return nil, err
   	}
    ```
   
    或者不使用 `isuperagent` 提供的统一工厂函数。
   
    ```go
    middleware, err := NewTimeMiddlewareFactory()
   	if err != nil {
   		return nil, err
   	}
    ```

**提示：可以参考现有的中间件工厂方法写法。**

#### 中间件如何应用

中间件的使用需要在初始化请求的时候（发送请求之前 `调用 Request.Do() 函数`）注册，具体参考以下代码：

```go
res, err := isuperagent.NewRequest().Get("http://localhost:8080/").Middleware(timeMiddleware, basicAuthMiddleware, debugMiddleware).Do()
```

### 解析器 BodyParser

`isuperagent` 提供了多样化的 `bodyParser`，如 `html`、`json`、`xml` 等多种解析器。

解析器 `bodyParser` 其实就是序列化请求对象 `request body`、反序列化 `response body` 的组件。该组件通过请求头的 `content-type` 值会自动调用对应的 `Parser` 来解析请求/响应内容，如果没有定义响应的解析器，会使用默认的文本解析器（`text/plain`）解析。

用户可以自定义解析器，通过自定义解析器可以新增解析逻辑、或者覆盖默认的解析逻辑，灵活且强大。

#### 如何自定义解析器

1. 在 `util/isuperagent/bodyParser` 目录下新建解析器的 go 文件。

    ```bash
    $ touch your/path/xxxx_parser.go
    ```

2. 解析器必须实现 `bodyParser.BodyParserInterface` 接口。

    ```go
    type XxxParser struct{}
    
    func (p *XxxParser) Unmarshal(data []byte, v interface{}) error {
        // TODO: add you Unmarshal logic code   
    	return nil
    }
    
    func (p *XxxParser) Marshal(v interface{}) ([]byte, error) {
    	// TODO: add you Marshal logic code   
    	return bs, nil
    }
    ```
   
   - `Unmarshal` 函数用于反序列化响应体。
   - `Marshal` 用于序列化请求体。

3. 注册解析器，只有注册过的解析器才能生效，第一个参数为别名，第二个参数为所对应的 `content-type` 值，第三个为解析器实例。

    ```go
    func init() {
    	Register("json", []string{
    		"application/json",
    		"application/javascript",
    		"application/ld+json",
    	}, &JsonParser{})
    }
    ```
 
**提示：请参考其他解析器的写法。**

#### 如何应用解析器

1. 请求体的序列化是自动的，你不需要关心它，通过不同的 `content-type` 可以调用不同的解析器。

2. 响应体的反序列化，需要手动调用，具体参考如下方式。

    ```go
    type ResponeData struct {
        Code int                 `json:"code"`
        Msg  string              `json:"msg"`
        Data map[string][]string `json:"data"`
    }
   
    res, err = isuperagent.NewRequest().Get("http://localhost:28080/v1/getHeader").Headers(headers).Do()
    if err != nil {
        return nil, err
    }
       
    var data ResponeData
    err = res.GetBody().Unmarshal(&data)
    if err != nil {
        return nil, err
    }
    log.Println(fmt.Sprintf("%+v", data))
    ```
   
    程序会自动调用对应的解析器解析响应体。
    
### 链式调用

`isuperagent` 支持链式调用，方便简洁，如下示例：

```go
res, err := isuperagent.NewRequest().
    Post("http://localhost:8080").
    SetHeader("a", "1").
    SetHeaders(map[string]string{
        "a": 2,
        "b": 3,
    }).
    SetQuery("token", "3ausdygiausyd1").
    SetQueries(map[string]string{
        "key1": "v1",
        "key2": "v2",
    }).
    SetTimeout(5 * time.Second).
    SetRetry(5).
    Middleware(middlewareA, middlewareB, middlewareC).
    Do()

if err != nil {
    return nil, err
}

return res, err
```

### SSL 请求

`isuperagent` 支持 HTTPS 请求、支持单向认证、支持双向认证。

1. 发起 HTTPS 请求

    ```go
    isuperagent.NewRequest().Get("https://www.baidu.com")
    ```
   
2. 支持自签名服务器请求，忽略未知机构签发的证书（自签名）

    ```go
    isuperagent.NewRequest().SetInsecureSkipVerify(true).Get("https://self-signed-cert-server.com")
    ```

3. 支持单向认证

    ```go
    isuperagent.NewRequest().SetCa("your/server_root_ca/path").Get("https://self-signed-cert-server.com")
    ```

4. 支持双向认证

    ```go
    isuperagent.NewRequest().SetCa("your/server_root_ca/path").SetCert("your/client_cert/path", "your/client_key/path").Get("https://self-signed-cert-server.com")
    ```

### 请求出错重试

`isuperagent` 支持出错重试机制。

```go
isuperagent.NewRequest().Get("http://unknown-server.com").SetRetry(3).Do()
```

**注意：所有中间件只会触发一次，与重试次数无关。**

### 丰富的请求属性

具体属性查看 `request.go` 和 `response.go` 文件。

## 其他

1. 拥有足够的测试用例保证功能完善。
2. 压力测试（待完善）。
