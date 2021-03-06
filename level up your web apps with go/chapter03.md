# HTTP

发送和接受`HTTP`请求是每一个`web`应用程序的核心工作。Go的标准库提供了很多包让我们轻松的驾驭http请求和响应。本章中，将会涉及响应请求，使用中间件（middleware）路由请求，创建`HTML`模板和响应`JSON`字串，所有这些功能都只依赖Go的标准库完成。

## 响应请求

Go创建web服务器相当简单。在多数其他语言，创建一个web服务器通常还需要一个额外的服务器软件，例如`PHP`需要`Apache`。Go的标准库提供了`http`包，我们无效依赖第三方包，仅使用`http`就能创建web服务。

有多种方式创建GOhttp服务，最简单的方式就是指定一个`url`模式（一下简称模式或URL模式），并注册一个**处理函数**，这个函数的签名是`func(w http.ResponeWriter, r *http.Request)`。一旦url匹配了模式，就会调用注册的函数处理请求。接下来可以调用`http.ListenAndServer`启动服务监听：

```
package main
import (    "fmt"	  "net/http"
)
func main() {    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {			fmt.Fprintf(w, "Hello Gopher")	  })    http.ListenAndServe(":3000", nil)}
```

这个示例中，当请求访问端口3000的任何路径时，注册的函数就会执行。函数中使用了`fmt.Fprintf`往`response writer`对象写入字串`“Hello Gpher”`来返回给客户端的响应。

> 路径模式匹配
> 
> 模式`/`将匹配任何路径的请求。关于url路径的模式匹配需要一点解释，后面的章节再详细阐述。


运行上述代码，使用浏览器访问`http://127.0.0.1:3000`，将会看到如下的显示：

![http hello world](./images/3.1.png)

### 一探究竟

处理函数（handler function）的两个参数提供了给任何请求响应所需要的全部功能。

第一个参数是`http.ResponseWriter`的实例，我们将返回给浏览器的信息写入这个对象中。响应主要由页面的主体（body）组成，不过也可以提供对其他信息其他例如响应头（headers）和状态码（status code）的访问。

> 所谓浏览器
> 
> 本书中发送任何方法的请求所用的“浏览器”不仅仅是浏览器软件，还可以是命令行工具例如`cURL`
> 如果安装了`cURL`， 可以使用命令`curl -i 127.0.0.1:3000`替代浏览器软件。

第二个参数是`http.Request`的实例，它涵盖我们期望看到的所有请求的信息，包括主体数据（data body），查询字串（query string）和请求头（headers）。

### 更多细节

现在的例子响应的信息并不多。实际上，真正的HTTP（raw HTTP） 响应也很小：

```
HTTP/1.1 200 OKDate: Mon, 01 Jun 2015 08:03:30 GMTContent-Length: 12Content-Type: text/plain; charset=utf-8
Hello Gopher
```

第一行的含义是告知接受一个处理成功的HTTP响应（OK HTTP）。然后是少许`header`信息，响应的日期`date`和`content`内容。注意`Content-Type`的值是`text/plain`。Go会读取响应主体的前512个字节并嗅探出响应的类型。例子中并没有明细的类型，因此返回了基本的text类型。除了`header`之外，剩下的内容为响应的主体。

要提供更多信息，我们需要操作处理函数中的`ResponseWriter`对象。通过一个命名贴切的`Header`方法返回一个`Header`结构的图来修改响应头`header`。可以通过其`Add`方法给响应头追加内容：

```
w.Header().Add("Server", "Go Server")
```

写入一些`HTML`标签，将会触发go嗅探出返回的`content-type`为`text/html`类型：

```
fmt.Fprintf(w, `<html>    <body>        Hello Gopher    </body></html>`)
```
再次运行服务器代码，将会收到新的`Server`响应头和`Content-type`：

```
HTTP/1.1 200 OKServer: Go ServerDate: Mon, 01 Jun 2015 09:12:32 GMTContent-Length: 57Content-Type: text/html; charset=utf-8<html>    <body>        Hello Gopher    </body></html
```

### 深入URL模式匹配

因为所有的请求是 `http.ListenAndServe`所监听的端口接受处理。因此我们需要一种将不同资源的请求路由到我们代码的不同部分的方法。幸运的是Go提供了`http.ServeMux`结构原生支持这样的方式。`ServeMux`是一个请求多路复用器，它将请求的`URL`与模式中中`url`进行匹配，并执行最匹配模式的处理函数。

`http`包提供了一个默认的`DefaultServeMux`实例，并且`http.HandleFunc`也是`DefaultServeMux`方法`(*ServeMux) HandleFunc`的包装。

当你创建`ServeMux`的时候，你可以为不同的URL模式注册不同处理函数。模式（Pattern）不必完全匹配路径（Path）。 有两种类型的模式：路径（path）和子树（subtree）。路径的结尾没有斜杠`/`，匹配严格的路径。子树的结尾包含斜杠`/`，并且匹配所有以子树开头的url。因此，在下一个例子中，请求`/articles/latest`将会返回`“Hello from /articles/”`，因为url`/articles/latest`与模式中的`/articles/`子树匹配。可是访问`/users/latest`将会返回一个404错误，因为模式`/users`缺少尾部的`/`，需要完全匹配（译者注：下面注释部分为源代码，可是是有问题的用法，参考上下文和go源代码，正确的用法为非注释的地方）：

```
/*
func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}
*/

func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```

模式的长度也很重要。模式越长的优先级越高。例如，模式`/articles/latest/`要比`/articles/`优先级高。因为有了顺序优先级，所以与添加路由的顺序没有区别。添加在`/articles/latest/`的路由规则在`/articles/`的处理程序之前，实际访问的时候也和预期绝对一致：

```
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/articles/latest/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/latest/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```
> 主机名匹配
> 
> 路由匹配的时候也可以从主机名开始，并且只有匹配的主机名才算真正的匹配。主机名匹配的模式拥有最高的优先级。

### 返回错误

有时候事与愿违，程序并不会正常工作，此时就需要返回一个错误，至少需要返回与`200 ok`不一样的响应。Web开发中状态码至关重要；如果没有状态码，访问不存在的资源的时候，或者网页的移动了，又或者遇了问题的时候希望能够回退。

常见的几个状态码：

■ 301 Moved Permanently― 重定向页面■ 404 Not Found― 访问的资源不存在■ 500 Internal Server Error―服务器内部错误

`http`包提供另一个`Error`函数用于返回错误，它需要两个参数，一个是`ResponseWriter`，另一个是整型的状态码。例如，500的错误返回如下：

```
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {    http.Error(w, "Something has gone wrong", 500)})
```

每个状态代码都有语义，所以你应该努力使用最合适的。如果有疑问，请记住，你并不是第一个吃螃蟹的人，在你试图找到合适的状态码之前，可能有无数的人已经尝试做了。Google是你的好朋友。完整的状态代码的请查看维基百科文章：[List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

> 状态码常量
> 
> Go的http包提供了很多HTTP状态码的常量。你可以直接使用这些常量而不是自己声明整数的code。`http.StatusBadRequest`就比 400 更容易理解。
> 
> 几个常见的的`Helper函数`：
> `http.NotFound` 可能是最普遍错误码是 `404 Not Found`。`http.NotFound`函数的参数为`ResponseWriter` 和`Request`类型的实例。（`http.NotFound(w, r)`）
> 
> `http.Redirect` 它的参数除了是`ResponseWriter`和`Request`的实例之外还有两个参数，一个是字串类型的参数，表示需要重定向的`url`；另外一个是整型的数字状态码。重定向的状态码有好几个。有多种类型的重定向状态代码。 最常见的是301用于永久移动的资源，302用于临时移动的资源。（` http.Redirect(w, r, "http://google.com", http.StatusMovedPermanently)`）

### Handler接口

还有另一种方式注册函数来处理`HTTP`请求。任何实现接口`http.Handler`的类型都可以和模式字串一起传递到`http.Handle`函数。 正如我们之前使用`HandleFunc`函数看到的，这是使用默认ServeMux实例的快捷方式。

`http.Handler`接口只有一个函数，这个函数的签名和`http.HandleFunc`函数一致。

```
type Handler interface {    ServeHTTP(ResponseWriter, *Request)}
```
我们可以给任何类型实现`ServeHTTP`方法。如果我们想创建一个handler函数用来处理返回服务器当前的时间的响应。例如定义一个类型`UptimeHandler`并实现`Handler`接口的`ServeHTTP`方法：

```
package mainimport (    "fmt""net/http""time" )type UptimeHandler struct {    Started time.Time}func NewUptimeHandler() UptimeHandler {
   return UptimeHandler{ Started: time.Now() }}func (h UptimeHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {    fmt.Fprintf(        w,        fmt.Sprintf("Current Uptime: %s", time.Since(h.Started)),    )}func main() {    http.Handle("/", NewUptimeHandler())    http.ListenAndServe(":3000", nil)}
```

使用浏览器访问`http://127.0.0.1:3000`，就能看见服务器返回的当前时间。`time.Since`函数返回一个`time.Duration`结构对象，表示自服务器启动以来所经过的时间。然后`time.Duration`类型被格式化为人类可读的字串; 例如，`"Current Uptime: 4m20.843867792s"`。

### 中间件

在Web开发中，**中间件**（Middleware）是指一系列处理程序，它们围绕Web应用程序，添加额外的功能。这是一个常见的概念，虽然经常被忽略，可是一旦正确的使用就非常强大。中间件可用于验证用户，压缩响应内容，限制请求速率，捕获应用程序异常等等。

使用`http.Handlers`创建中间件很普通，也很容易。例如，我们可以创建一个中间件，他的功能就是检查请求中查询字串的`token`是否合法，否则则返回一个`404 Not Found`的响应错误。

```
// SecretTokenHandler 检查 secret tokentype SecretTokenHandler struct {    next http.Handler    secret string}

// SecretTokenHandler 实现了 http.Handler 接口的 ServeHTTP方法.func (h SecretTokenHandler) ServeHTTP(w ResponseWriter, req *Request) {
    // 检查 查询字串的 token    if req.URL.Query().Get("secret_token") == h.secret {
        // 检查token合法，调用下一个handler函数        h.next.ServeHTTP(w, req)    }else{        // No match, return a 404 Not Found response        http.NotFound(w, req)    }}func main() {    http.Handle("/", SecretTokenHandler{        next: NewUptimeHandler(),        secret: "MySecret",    })    http.ListenAndServe(":3000", nil)}

```
不添加任何`token`的查询字符访问，将会返回 404 错误：```
curl -i 127.0.0.1:3000
 
HTTP/1.1 404 Not FoundContent-Type: text/plain; charset=utf-8Date: Mon, 01 Jun 2015 02:07:59 GMTContent-Length: 19404 page not found
```

如果添加了token，将会返回下一个处理函数的处理响应结果：

```
curl -i 127.0.0.1:3000/?secret_token=MySecretHTTP/1.1 200 OKDate: Mon, 01 Jun 2015 02:09:49 GMTContent-Length: 31Content-Type: text/plain; charset=utf-8Current Uptime: 2m16.570254662s
```

## HTML 模板

到目前为止，我们看过的例子是微不足道的，旨在检查一些特定的用例。如果我们想要开始返回更复杂的响应，该怎么办呢？如今仅仅使用`fmt`包来来生成`HTML`将会很棘手。

幸运的是Go提供了原生的`HTML`模板处理包`html/template`。该包不仅能让我们轻而易举格式化来自Go数据的`HTML`页面，还能正确处理`escaping`转义字符的`HTML`输出。大多数情况下，都无需担心传入模板数据的escaping转义问题，Go将会帮你处理。

> Escaping转义
> 
> 在HTML页面上escaping转义输入对于布局和安全原因都非常重要。 转义是将在HTML中具有特殊含义的字符转换为`HTML`实体的过程。 例如，一个`＆`符号用于表示一个`HTML`实体，因此，如果你想正确显示一个`＆`符号，你必须写一个`HTML`实体代替：`＆amp`; 如果你不关心试图显示在模板中的任何数据，那么最好的情况下默许了用户意外地破坏页面布局，最坏的情况下被打开的页面遭到恶意攻击。
> 
> 例如，如果您有一篇包含技术文章的博客，您不会希望有一个内容`</body>`的帖子来创建实际 `body`内容结束标记。相反，您希望将其显示为文本，这意味着将其转换为`HTML``＆lt;/ body＆gt;`。 如果您在自己的博客上发表了评论，希望避免用户使用`<script>`标记创建评论，因为这些标记会在用户访问您的网页时，在用户的计算机上运行JavaScript。 这种安全漏洞通常被称为**跨站点脚本**或**XSS**。 您可以在开放Web应用程序安全项目（OWASP）网站上阅读有关该[主题](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS))的更多内容


`html/template`包将分两步工作。首先需要需要将`HTML`字符模板**解析**成为`Template`类型。然后**执行**注入模板的数据结构来生成`HTML`字串。

无论是从文件中载入还是直接在Go代码中定义，模板都是从纯文件字符串中创建。变量替换还是被称之为**action**控制结构，都是通过花括号`{{` 和 `}}`对包裹。任何它们之外的字符都不会被修改。

一个简单的模板输出的例子看起来是这样的：

```
package mainimport (    "html/template""os" )func main() {    tmpl, err := template.New("Foo").Parse("<h1>Hello {{.}}</h1>\n")    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, "World")    if err != nil { panic(err) }}
```
每一个模板都需要命名。原因稍后解释，目前都可以命名为“Foo”。

如你所见，例子中的`{{.}}`称之为**点action**(译者：action可以理解为一种操作)，它指的是传递到模板中的数据。因为我们只是传递一个字符串，所有我们需要做的是直接引用数据，但`.`action可以赋值多次不同的数据; 例如，当循环数据时，`.`将分配当前迭代的值。我们将会看到在模板包中点将会被重复使用。

### 访问模板数据

模板中访问更复杂的数据类型也相当简单。虽然在Go访问结构的字段，图的key或者简单的函数不一样，但是在模板中访问方式却很相似。如果我们在渲染模板的时候注入一个结构体，可以通过它们的字段名访问其可以导出的字段。下面例子将会打印输出：`“The Go html/template package” by Mal Curtis`：

```
type Article struct{    Name string    AuthorName string}func main(){    goArticle := Article{        Name: "The Go html/template package",        AuthorName: "Mal Curtis",    }tmpl, err := template.New("Foo").Parse("'{{.Name}}' by {{.AuthorName}}")    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, goArticle)    if err != nil { panic(err) }}
```
上述规则同样适用与图，即通过图的key访问其值。与结构体不一样，key无需以大写字母开头：

```
func main(){    article := map[string]string{        "Name": "The Go html/template package",        "AuthorName": "Mal Curtis",    }	 tmpl, err := template.New("Foo").Parse("'{{.Name}}' by {{.AuthorName}}")    if err != nil { panic(err) }
    err = tmpl.Execute(os.Stdout, article)    if err != nil { panic(err) }}
```


不带参数的函数或方法也可以以相同的方式调用（在Go文档中称为`niladic`函数）。 这个例子将输出`"Written by Mal Curtis"`：

```
type Article struct{    Name string    AuthorName string}func(a Article) Byline() string{    return fmt.Sprintf("Written by %s", a.AuthorName)}func main() {    goArticle := Article{        Name: "The Go html/template package",        AuthorName: "Mal Curtis",    }    tmpl, err := template.New("Foo").Parse("{{.Byline}}")    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, goArticle)    if err != nil { panic(err) }}
```

### 模板中的if else 条件判断

现在我们介绍了如何访问模板中不同类型数据结构，接下来看看是如何通过条件和循环结构控制模板中的执行流程。

就像Go代码一样，模板通过`if`和`else`语句控制模板的条件流程。这没有什么难度，它将为模板提供大量的限制逻辑的部分。

就像在Go代码中一样，我们可以通过使用If和Else语句来访问条件流。 这里没有太多的困难 - 它将弥补模板有限的逻辑的大部分。 因为我们不能使用括号来指定属于`if`语句的内容，所以我们使用`end`语句来表示结束边界。 如果文章尚未发布，此示例会将`"Draft"`一词附加到标题中：

```
type Article struct{    Name string    AuthorName string
    Draft bool 
}func main(){    goArticle := Article{        Name: "The Go html/template package",        AuthorName: "Mal Curtis",    }    tmpl, err := template.New("Foo").Parse(        "{{.Name}}{{if .Draft}} (Draft){{end}}”,    )    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, goArticle)    if err != nil { panic(err) }}

```
你也可以在模板中使用可选的`{{ else }}`语句来执行条件不满足的情况，`{{if .Draft}} (Draft){{else}} (Published){{end}}`。

### 循环

 可以使用`range`来迭代切片或者图。range也可以提供一个可选的`{{ else }}`语句，当循环结束的时候，没有可以循环的项的时候使用else语句对列出一些消息佷有用。。

在每个循环中，`.`被设置为循环变量的值。如果你在一个循环中调用点动作`{{.}}`，它将在循环的每次迭代中设置为不同的值。在这个例子中，我们将获得`Article`的`Name`和`AuthorName`的列表，或者一条消息`"No published articles yet"`：

```
func main(){    tmpl, err := template.New("Foo").Parse(`    {{range .}}        <p>{{.Name}} by {{.AuthorName}}</p>    {{else}}        <p>No published articles yet</p>    {{end}}`)

    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, []Article{})    if err != nil { panic(err) }}

```

### Multiple 模板

当解析模板的时候，实际上可以同时定义多个模板。这样就在运行代码的时候，也可以选择使用哪一个模板，或者在一个模板中调用另外一个模板。这给模板提供了强大的功能，鼓励模板代码的重用---普遍的做法是创建多个模板块。

在模板中使用`{{ define "FOO" }}`action定义命名模板，这些定义是顶级的action，换言之，没有其他的action。

> 添加模板
> 
> 你不必在一次`Parse`调用的时候添加多个模板，而是可以多次调用`Parse`，或者使用`ParseFile`和`ParseGlob`方面从文件中载入模板。后续的章节将会介绍`Glob`模块。

执行模板注入的时候，可以调用`ExecuteTemplate(wr io.Writer, name string, data interface{})`方法取代我们目前所用的`Execute`方法，当上下文不是`.`或者当前上下文的其他值的时候，也可以使用action `{{template "FOO" context}}`调用。模板action了可以被另外一个模板所调用，这样就能通过key实现模板的重用。

例如，在我们前面的`article`循环中，随着每篇article的模板开始增长将会变得更难理解。你失去了很容易找出模板流的能力，因为它不是立即显而易见的。我们可以从循环它的模板中分离文章模板：

```
func main(){    tmpl, err := template.New("Foo").Parse(`    {{define "ArticleResource"}}        <p>{{.Name}} by {{.AuthorName}}</p>    {{end}}
        {{define "ArticleLoop"}}        {{range .}}            {{template "ArticleResource" .}}        {{else}}            <p>No published articles yet</p>        {{end}}{{end}}    {{template "ArticleLoop" .}}    `)    if err != nil { panic(err) }    err = tmpl.Execute(os.Stdout, []Article{})    if err != nil { panic(err) }}

```

现在我们可以增加`ArticleResource`模板的大小，而不必担心`ArticleLoop`模板的可读性。 我们还可以使用来自另一个完全独立的模板的`ArticleResource`模板。

### 管道过滤器（Piplines）



目前为止，我们已经见识了模板中简单的数据访问，但是在实际开发中，模板会收到一些并不合适展示的数据。例如，如果想要在页面输出价格，我们希望使用某种货币符号（例如`$`）格式化数据，又或者限制输出数据的为两位小数。此时管道应运而生。

管道器是一种通过命令链方式计算数据。实际上我们已经见过管道器，我们已经使用过简单的命令，`.`是一个非常简单的管道器，它只有一个命令。管道器通常是一些用`|`分隔的一些命令。命令可以是一个值，例如`.`，也可以是一个函数/方法，它们的参数使用空格分隔，而不是使用圆括号和逗号，例如 `函数/方法 格式化字串 输入值`（`printf format_string input_value`）。
每一个命令执行的结果都通过管道符以下一个函数的参数传入，使得许多命令会连在一起工作。最后一个命令的输出作为管道器的结果输出。每一个管道器都必须输出一个值，和另外一个可选的错误标识值。如果发生了错误，模板将会停止继续执行，并且会返回一个错误，作为执行函数`Execute`的返回值。`Go`内建了一些可以被管道器函数。你也可以自定义自己的管道器。内建的函数有比较（comparsion），转义（escaping），打印（printing）等帮助函数。目前我们只看其中一个，但是你可以访问[Go网站](https://golang.org/pkg/text%2Ftemplate/#hdr-Functions)查看完整的函数。想要格式化一个`float64`的值作为货币表示，可以将值传递给`printf`函数，它是`fmt.Sprintf`函数的别名。下面的例子，将值`12.3`格式化成一个带两个小数位数的浮点数，输出为：`Price: $12.30`：

```
func main(){    tmpl, _ := template.New("Foo").Parse(        "Price: ${{printf \"%.2f\" .}}\n”,    )    tmpl.Execute(os.Stdout, 12.3)}
```

也可以使用`.`命令用管道风格冲洗这个`printf`例子（记住，一个命令的结果值将作为下一个命令的参数）：

```
func main(){    tmpl, _ := template.New("Foo").Parse(        "Price: ${{. | printf \"%.2f\”}}\n”,    )    tmpl.Execute(os.Stdout, 12.3)}
```

如果我们扩展我们的例子将更有意义，使第一个值更复杂，比如，有一个价格（price）和数量（quantity）的产品。如果我们要显示产品的总价格，我们需要先将价格乘以数量。为此，我们需要一个创建一个新的格式化管道器函数。

在模板中注册一个函数很容易。任何向模板传入函数的方式都能遵循我们之前学习的规则（注入数据）：返回一个值和一个可选的错误。写一个函数`Multiply`，它有两个浮点类型的参数，然后两个数的乘积：

```
// 把两个参数相乘并返回乘积func Multiply(a, b float64) float64 {	return a * b 
}
```

定义好函数之后，将函数注册到模板中。使用`Funcs`方法，它有一个`FuncMap`类型的值。`FuncMap`是一个图，模板中的函数名作为图的key，用来执行的函数作为图的值：

```
map := template.FuncMap{    "multiply": Multiply,}
```
创建模板的时候需要分几步。首先创建一个空模板，实例化`FuncMap`然后调用`Funcs`注入过滤器函数。接下来解析模板。在注入新函数之前解析模板将会抛出一个错误，因为此时`go不`知道`multiply`函数。下面的例子，我们看到模板中的`multiply`函数执行结果传入到`printf`方法并输出：`"Price: $24.60"`：

```
type Product struct {        Price    float64        Quantity float64}func main() {    tmpl := template.New("Foo")    tmpl.Funcs(template.FuncMap{ "multiply": Multiply })    tmpl, err := tmpl.Parse(        "Price: ${{ multiply .Price .Quantity | printf \"%.2f\”}}\n”, 
    )    if err != nil { panic(err)  }    err = tmpl.Execute(os.Stdout, Product{        Price:    12.3,
        Quantity: 2,    })    if err != nil { panic(err)  }}
```

上面就是我们添加了自定义的过滤函数并在模板中运行。通过定义命令链，我们可以轻松在模板中操作数据以正确的格式显示。下面是上述例子的合起来的完整代码：

```
package mainimport ( "os"    "text/template")type Product struct {        Price    float64        Quantity float64}func Multiply(a, b float64) float64 {    return a * b}func main() {    tmpl := template.New("Foo")    tmpl.Funcs(template.FuncMap{ "multiply": Multiply })    tmpl, err := tmpl.Parse(        "Price: ${{ multiply .Price .Quantity | printf \"%.2f\”}}➥\n”, )    if err != nil { panic(err)  }    err = tmpl.Execute(os.Stdout, Product{        Price:    12.3,		   Quantity: 2,
	  })    if err != nil { panic(err)  }}
```

### 模板变量

模板变量并不想之前介绍的概念那么重要，你应该知道你不必每次都访问传入到模板的数据值。你可以在模板中定义变量。之所以这样做的原因是当你想在模板多个地方以相同的方式格式化一个值的时候。它更容易和更快地把格式化的值赋给一个变量，并在多个地方输出。

变量作为管道的结果，看起来与常规`Go`变量略有不同，因为它们必须以美元符号`$`开头。 我们可以用这种方式重写前面的示例的模板：

```
{{$total := multiply .Price .Quantity}}Price: ${{ printf "%.2f" $total}}
```

> 变量作用域（Scope）
> 
> 变量的作用域只在其定义的代码块中。`if`或者`range`语句里定义的变量，在其外面的代码块将无效。

## 渲染 JSON

`JSON`，全称`Javascript 对象标记`的缩写，是一种常见的数据格式。虽然`XML`以前是用于在`Web`系统之间通信的事实上的格式，但是过去五年来，JSON已经取代了它。 如果你现在将在互联网上传输数据，`JSON`是最好的数据格式。如果你不熟悉`JSON`，我重申一下应该看看`JSON`维基百科的[文章](https://en.wikipedia.org/wiki/JSON#Data_types.2C_syntax_and_example)。

GO中`encoding/json`读（Unmarshaled）写（Marshaled）`JSON`对象。

### 序列化（Marshling）

将数据类型转换成`JSON`的过程称之为`Marshaling`序列化。json包是用于转换Go数据类型为JSON对象而无需编程很多样本代码。当你序列化一个数据类型的时候，Go会推断出它最适合的JSON形式。

> Marshling
> 
> **Marshling**指计算机中用于存储或者发送数据到另外一个系统的时候，表示一个对象转换格式化。当你指代JSON的时候也可以称之为`序列化`（serialising）。可是Go的作者命名还是选择了`marshaling`。它的反义词则为`unmarshaling`或者`反序列化`（serialising）

`json.Marshal(interface{}) ([]byte, error)`的主要功能是序列化一个对象。方法的参数空接口适配所有数据类型，因此它可以接受任何类型的参数。它将返回一个byte类型的切片和错误标识。

> Byte数组
> 
> 使用string函数可以轻松的将byte数组和切片转换成字串，因为字串本质上也是byte切片。这样说过于简单抽象，如果你想了解跟多关于GO中的字串的工作形式，请阅读Go的blog[文章](https://blog.golang.org/strings)。

Go会尝试检查类型的值然后序列化值。字串会被序列化为JSON的字串，布尔值转换为JSON 布尔值，所有数字类型都编程为JSON的数字。对于复杂组合类型，Go能遍历该值的字段（对于结构体）或循环该值（例如使用地图或切片，尝试编组每个类型的每一项。

### 序列化结构体

继续之前的Article例子：

```
package maintype Article struct {    Name string    AuthorName string    draft bool}func main(){    article := Article{        Name:       "JSON in Go",        AuthorName: "Mal Curtis",        draft:      true,    }    data, err := json.Marshal(article)    if err != nil {        fmt.Println("Couldn’t marshal article:", err)    }else{        fmt.Println(string(data))    }}

```

运行将会输出JSON字串：

```
 {"Name":"JSON in Go","AuthorName":"Mal Curtis"}
```

你肯定主要到了仅有`Name`和`AuthorName`两个字段，而`draft`字段却没有序列化。原因是Go只会转换结构体中的可导出字段。这很容易理解，非导出字段通常是私有信息，是否转换字段只与类型实现字段属性有关，而不是类型暴露的字段。

你可能也看见上述格式化显示的结果有点粗爆。实际上可以使用`MarshalIndent`函数输出美观的`JSON`格式。使用`MarshalIndent`的时候可以提供前缀和缩进两个参数。通常前缀都不使用，但是缩进都会在每一行输出相应的缩进。普遍的做法是缩进两个空格，也可以缩进四个空格或者一个tab键，又或者是`emoji`表情例如一只喵，当然，其他语言未必就能正确读写emoji的缩进：

```
data, _ := json.MarshalIndent(article, "", "  ")fmt.Println(string(data))
```

输出：

```
{
  "Name": "JSON in Go",
  "AuthorName": "Mal Curtis"
}
```

### 自定义JSON字段名

如果你之前使用过JSON，那么肯定会看见很多key使用下划线命名，例如`author_name`代替`AuthorName`。当然这并不是强制性规定，不过遵守这些约定对数据的使用有利。幸运的，我们可以使用Go的语言特性**标签**（tag）轻松的修改`JSON`的字段字面量，同时不用修改Go的数据结构。

tag写在结构体字段定义的后面。它们的语法格式 `逗号，分隔符，元数据`（"comma,separated,metadata"）。对于JSON而言，指定tag的一些数据信息，可以修改key的名，选择需要输出的字段，或者当字段是为空的时候是否输出：

```
type Product struct{
    // 修改json显示的字段为 "name".
    Name string `json:"name"`
    // 修改json显示的字段为 "author_name"，但是指定`omitempty`的时候，字段值为空是不显示key
    AuthorName string `json:"author_name,omitempty"`
    // 字段将隐藏，不会在json中输出
    CommissionPrice float64 `json:"-"`
}
```

### 内嵌类型
因为Go在编码JSON的时候会遍历结构体的所有项，因此很容易传教复杂的JSON结构。如果我们想返回一个文章元数据的集合对象，我们可以定义一个新的类型：

```
package main

import (
    "fmt"
    "encoding/json"
)


type Article struct {
Name string }
type ArticleCollection struct {

    Articles []Article `json:"articles"`
    Total    int       `json:"total"`
}

func main(){
    p1 := Article{ Name: "JSON in Go" }
    p2 := Article{ Name: "Marshaling is easy" }
    articles := []Article{p1, p2}
    collection := ArticleCollection{
        Articles: articles,
        Total: len(articles),
    }
    data, err := json.MarshalIndent(collection, "", "  ")
    if err != nil { panic(err) }
    fmt.Println(string(data))
}

```
输出：

```
{  "articles": [    {      "name": "JSON in Go"}, {      "name": "Marshaling is easy"    }],"total": 2 }
```

这就是Go中编码复杂JSON的介绍。

### 反序列化（Unmarshling）

**反序列化**Unmarshling）是序列化的想法操作，即将JSON结构转换成GO类型的数据结构。Go中反序列化是先创建一个空的类型对象，然后尝试将JSON的字串映射到这个对象上。

例如，我们将从一个文件中载入一些配置数据映射到一个特定的Cconfig结构体。Go将尽最大努力将数据转换为适当的格式。。我们可以将下面的代码保存在`config.json`文件中：

```
{    "SiteName": "My Cat Blog",    "SiteUrl": "www.mycatblog.com",    "Database": {        "Name": "cats",        "Host": "127.0.0.1",        "Port": 3306,        "Username": "user1",        "Password": "Password1"} }
```

我们可以创建一个用来匹配的数据结构，当调用`json.Unmashal`函数的时候反序列化数据。该函数有两个参数，一个JSON数据的字串byte数组，第二次则是用来匹配的数据结构的指针：

```
type Config struct {    Name     string `json:"SiteName"`    URL      string `json:"SiteUrl"`    Database struct {        Name     string        Host     string        Port     int        Username string        Password string} }conf := Config{}data, err := iouitil.ReadData("config.json")if err != nil { panic(err) }err = json.Unmarshal(data, &conf)if err != nil { panic(err) }fmt.Printf("Site: %s (%s)", conf.Name, conf.URL)db := conf.Database// Print out a database connection string.fmt.Printf(    "DB: mysql://%s:%s@%s:%d/%s",    db.Username,    db.Password,
    db.Host,    db.Port,    db.Name,)
```
这个例子将会输出：

```
Site: My Cat Blog (www.mycatblog.com)DB: mysql://user1:Password1@127.0.0.1:3306/cats
```

注意数据库的端口是如何被赋值为int数据类型的。Go有几个int数据类型（int， int32和int64），它不会挑剔你尝试反序列化的JSON数字类型，只要是数字类型就好，例如整型或者浮点型。

如果配巧JSON中还有额外的键并且不被Go结构所匹配，那么Go会静默忽略。

### 未知的JSON结构处理

有时候你会遇到需要反序列化一些额外格式的JSON。此时，将会有点麻烦。
首先建议尝试确保你的数格输入是已知格式。如果是未知的输入，使用空接口`interface{}`，这样任何输入都会被匹配。然后我而提编程了数据访问，如果你想使用该数据做有效的操作的时候，将不得不的强制转换该数据为更严格的类型。

载入未知JSON数据的例子将会打印`"foo"`:

```
package mainfunc FooJSON(input string) {    data := map[string]interface{}{}    err := json.Unmarshal([]byte(input), &data)    if err != nil { panic(err) }    foo, _ := data["foo"]    switch foo.(type) {
		 case float64:        fmt.Printf("Float %f\n", foo)    	 case string:        fmt.Printf("String %s\n", foo)       default:        fmt.Printf("Something else\n")	  } }func main() {    FooJSON(`{        "foo": 123    }`)    FooJSON(`{        "foo": "bar"    }`)    FooJSON(`{"foo": [] }`)}
```
将会输出： `Float 123.000000 String bar Someting selse`。


注意输入带有整数的`JSON`的时候会以类型`float64`结束。 这是因为`JSON`不区分整数和浮点数，它只有类型`"number"`。 由于Go无法推断预期的数据类型，因此默认为`float64`。

## 总结

本章我们学习了如何使用Go创建web服务器，以及如何创建中间件函数。我们全面见识了`html/template`包如何组织我们的html模板以及该如何序列化和反序列化JSON数据。

这就是关于`HTTP`章节的介绍。接下来我们将很快把学到的知识组合在一起，学以致用。



