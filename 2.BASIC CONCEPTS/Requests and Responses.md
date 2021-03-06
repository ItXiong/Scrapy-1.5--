# Requests and Responses
Scrapy使用请求和响应对象来抓取网站。

通常情况下，请求对象在蜘蛛中生成并传递到系统，直到它们到达下载器，下载器执行请求并返回一个Response对象，该对象返回发出请求的蜘蛛。

Request和Response类都有子类，它们添加了基类中不需要的功能。 这些在请求子类和响应子类中进行了描述。
## Request objects
>class scrapy.http.Request(url[, callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority=0, dont_filter=False, errback, flags])

一个Request对象表示一个HTTP请求，它通常在Spider中生成并由Downloader执行，从而生成一个Response。
参数:
* url (string) - 此请求的URL
* callback (callable)  - 将这个请求的响应（一旦下载完成）作为第一个参数调用的函数。有关更多信息，请参阅下面将附加数据传递给回调函数。如果请求没有指定回调，则将使用spider的parse（）方法。请注意，如果在处理期间引发异常，则会调用errback。
* method (string) - 此请求的HTTP方法。默认为'GET'。
* meta (dict)  - Request.meta属性的初始值。如果给出，在此参数中传递的字典将被浅拷贝。
* body (str or unicode) - 请求正文。如果传递一个unicode，那么它使用传递的编码（默认为utf-8）编码为str。如果没有给出主体，则存储空字符串。无论此参数的类型如何，存储的最终值都将是一个str（从不unicode或None）。
* headers (dict)  - 这个请求的标题。字典值可以是字符串（对于单值标题）或列表（对于多值标题）。如果None作为值传递，HTTP标头将不会被发送。
* cookies (dict or list) - 请求cookies。 这些可以以两种形式发送。
1.使用字典：
```
request_with_cookies = Request(url="http://www.example.com",
                               cookies={'currency': 'USD', 'country': 'UY'})
```

2.使用列表的字典：
```
request_with_cookies = Request(url="http://www.example.com",
                               cookies=[{'name': 'currency',
                                        'value': 'USD',
                                        'domain': 'example.com',
                                        'path': '/currency'}])```

后一种形式允许自定义domain和path Cookie的属性。这仅在cookie被保存用于以后的请求时才有用。

当某个站点返回cookie（作为响应）时，这些cookie将存储在该域的cookie中，并将在未来的请求中再次发送。 这是任何常规Web浏览器的典型行为。 但是，如果出于某种原因想要避免与现有Cookie合并，可以通过在Request.meta中将dont_merge_cookies项设置为True来指示Scrapy执行此操作。

不合并Cookie的请求示例：
```
request_with_cookies = Request(url="http://www.example.com",
                               cookies={'currency': 'USD', 'country': 'UY'},
                               meta={'dont_merge_cookies': True})
```

欲了解更多信息，请参阅[CookiesMiddleware](https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#cookies-mw)。

* encoding (string) - 此请求的编码（默认为'utf-8'）。此编码将用于百分比编码URL并将主体转换为str（如果作为unicode提供）。
* priority（int） - 此请求的优先级（默认为0）。调度程序使用优先级来定义用于处理请求的顺序。具有较高优先级值的请求将在较早时间执行。为了表示相对低的优先级，允许负值。
* dont_filter（boolean） - 表示这个请求不应该被调度器过滤。当您想多次执行相同的请求时使用此选项，以忽略重复过滤器。小心使用它，否则你将进入爬行循环。默认为False。
* errback（callable） - 如果在处理请求时引发任何异常，将会调用该函数。这包括404 HTTP错误等失败的页面。它收到Twisted Failure实例作为第一个参数。有关更多信息，请参阅下面的请求处理中的使用errbacks来捕获异常。
* flags (list)  - 发送到请求的标志可用于日志记录或类似目的。

>url
包含此请求的URL的字符串。请记住，此属性包含转义的URL，因此它可能与构造函数中传递的URL不同。

该属性是只读的。要更改请求的URL，请使用replace（）。

>method
表示请求中的HTTP方法的字符串。这保证是大写的。例如：“GET”，“POST”，“PUT”等

>headers
一个包含请求头文件的类似字典的对象。

>body
包含请求主体的str。

该属性是只读的。要更改请求的主体，请使用replace（）。

>meta
包含此请求的任意元数据的字典。这个词典对于新的请求是空的，并且通常由不同的Scrapy组件（扩展，中间件等）填充。因此，此字典中包含的数据取决于您启用的扩展。

有关由Scrapy识别的特殊元键列表，请参阅Request.meta特殊键。

当使用copy（）或replace（）方法克隆请求时，该词典被浅拷贝，并且还可以在你的蜘蛛中从response.meta属性访问。

>copy()
返回一个新请求，它是此请求的副本。另请参阅：将附加数据传递给回调函数。

>replace（[url，method，headers，body，cookies，meta，encoding，dont_filter，callback，errback]）
使用相同的成员返回一个Request对象，除了那些由指定的关键字参数赋予新值的成员。 Request.meta属性默认被复制（除非在meta参数中给出一个新值）。另请参见将附加数据传递给回调函数。

### Passing additional data to callback functions
请求的回调函数将在下载该请求的响应时调用。 回调函数将以下载的Response对象作为第一个参数来调用。
例如：
```
def parse_page1(self, response):
    return scrapy.Request("http://www.example.com/some_page.html",
                          callback=self.parse_page2)

def parse_page2(self, response):
    # this would log http://www.example.com/some_page.html
    self.logger.info("Visited %s", response.url)
    ```
在某些情况下，您可能有兴趣将参数传递给这些回调函数，以便稍后在第二个回调函数中接收参数。 您可以使用Request.meta属性。

以下是如何使用此机制传递项目以填充不同页面的不同字段的示例：
```
def parse_page1(self, response):
    item = MyItem()
    item['main_url'] = response.url
    request = scrapy.Request("http://www.example.com/some_page.html",
                             callback=self.parse_page2)
    request.meta['item'] = item
    yield request

def parse_page2(self, response):
    item = response.meta['item']
    item['other_url'] = response.url
    yield item
```

### Using errbacks to catch exceptions in request processing
请求的错误是一个函数，当处理异常时会调用它。

它接收Twisted Failure实例作为第一个参数，可用于跟踪连接建立超时，DNS错误等。

以下是一个蜘蛛日志记录所有错误并在需要时捕获一些特定错误的示例：
```
import scrapy

from scrapy.spidermiddlewares.httperror import HttpError
from twisted.internet.error import DNSLookupError
from twisted.internet.error import TimeoutError, TCPTimedOutError

class ErrbackSpider(scrapy.Spider):
    name = "errback_example"
    start_urls = [
        "http://www.httpbin.org/",              # HTTP 200 expected
        "http://www.httpbin.org/status/404",    # Not found error
        "http://www.httpbin.org/status/500",    # server issue
        "http://www.httpbin.org:12345/",        # non-responding host, timeout expected
        "http://www.httphttpbinbin.org/",       # DNS error expected
    ]

    def start_requests(self):
        for u in self.start_urls:
            yield scrapy.Request(u, callback=self.parse_httpbin,
                                    errback=self.errback_httpbin,
                                    dont_filter=True)

    def parse_httpbin(self, response):
        self.logger.info('Got successful response from {}'.format(response.url))
        # do something useful here...

    def errback_httpbin(self, failure):
        # log all failures
        self.logger.error(repr(failure))

        # in case you want to do something special for some errors,
        # you may need the failure's type:

        if failure.check(HttpError):
            # these exceptions come from HttpError spider middleware
            # you can get the non-200 response
            response = failure.value.response
            self.logger.error('HttpError on %s', response.url)

        elif failure.check(DNSLookupError):
            # this is the original request
            request = failure.request
            self.logger.error('DNSLookupError on %s', request.url)

        elif failure.check(TimeoutError, TCPTimedOutError):
            request = failure.request
            self.logger.error('TimeoutError on %s', request.url)
```

## Request.meta special keys
Request.meta属性可以包含任何任意数据，但Scrapy和它的内置扩展可以识别一些特殊的键。

那些是：
* dont_redirect
* dont_retry
* handle_httpstatus_list
* handle_httpstatus_all
* dont_merge_cookies (see cookies parameter of Request constructor)
* cookiejar
* dont_cache
* redirect_urls
* bindaddress
* dont_obey_robotstxt
* download_timeout
* download_maxsize
* download_latency
* download_fail_on_dataloss
* proxy
* ftp_user (See FTP_USER for more info)
* ftp_password (See FTP_PASSWORD for more info)
* referrer_policy
* max_retry_times

### bindaddress
用于执行请求的传出IP地址的IP。

### download_timeout
下载器在超时之前等待的时间（以秒为单位）。 另请参阅：DOWNLOAD_TIMEOUT。

### download_latency
从请求开始以来（即通过网络发送的HTTP消息）获取响应所花费的时间量。 这个元键只有在响应被下载后才可用。 虽然大多数其他元键用于控制Scrapy行为，但它应该是只读的。

### download_fail_on_dataloss
是否对失败的反应失败。 请参阅：DOWNLOAD_FAIL_ON_DATALOSS。

### max_retry_times
元密钥用于设置每个请求的重试次数。 初始化时，max_retry_times元键优先于RETRY_TIMES设置。

## Request subclasses
这是内置的请求子类的列表。 您也可以将其子类化以实现您自己的自定义功能。

### FormRequest objects
FormRequest类使用用于处理HTML表单的功能来扩展基本Request。 它使用lxml.html表单预先填充来自Response对象的表单数据的表单字段。
>class scrapy.http.FormRequest(url[, formdata, ...])

FormRequest类为构造函数添加一个新参数。 其余参数与Request类相同，这里没有记录。

参数：formdata（字典或元组的迭代） - 是一个包含HTML表单数据的字典（或（键，值）元组的迭代），这些数据将被url编码并分配给请求的主体。
除了标准的Request方法外，FormRequest对象还支持以下类方法：

>classmethod from_response(response[, formname=None, formid=None, formnumber=0, formdata=None, formxpath=None, formcss=None, clickdata=None, dont_click=False, ...])

返回一个新的FormRequest对象，其表单字段值预先填充到给定响应中包含的HTML <form>元素中。 有关示例，请参阅使用FormRequest.from_response（）模拟用户登录。

该策略默认情况下会自动模拟任何可点击的窗体控件上的点击，如<input type =“submit”>。 尽管这很方便，而且通常是所需的行为，但有时它可能会导致难以调试的问题。 例如，在处理使用javascript填充和/或提交的表单时，默认的from_response（）行为可能不是最合适的。 要禁用此行为，可以将dont_click参数设置为True。 另外，如果您想更改点击的控件（而不是禁用它），您还可以使用clickdata参数。

>Caution
由于lxml中存在一个错误，在选项值中使用具有前导空白或尾随空白的select元素使用此方法将不起作用，这应该在lxml 3.8及更高版本中修复。

参数：
* response (Response object) - 包含将用于预填充表单域的HTML表单的响应
* formname（string） - 如果给定，将使用名称属性设置为此值的表单。
* formid（string） - 如果给定，将使用id属性设置为该值的表单。
* formxpath（string） - 如果给定，将使用匹配xpath的第一个表单。
* formcss（string） - 如果给出，将使用匹配CSS选择器的第一个表单。
* formnumber（整数） - 响应包含多个表单时要使用的表单数。第一个（也是默认值）是0。
* formdata（dict） - 要在表单数据中覆盖的字段。如果某个字段已经存在于响应<form>元素中，则其值会被此参数中传递的值覆盖。如果在此参数中传递的值为None，则该字段将不会包含在请求中，即使它存在于响应<form>元素中。
* clickdata（dict） - 用于查找点击的控件的属性。如果没有给出，表单数据将被提交模拟点击第一个可点击的元素。除了html属性之外，控件还可以通过其相对于表单内其他可提交输入的从零开始的索引，通过nr属性进行标识。
* dont_click（boolean） - 如果为True，表单数据将被提交而不需要单击任何元素。

这个类方法的其他参数直接传递给FormRequest构造函数。

0.10.3版新增：formname参数。

版本0.17中的新功能：formxpath参数。

1.1.0版中的新功能：formcss参数。

1.1.0版中的新功能：formid参数。

### Request usage examples
#### Using FormRequest to send data via HTTP POST
如果你想在你的蜘蛛模拟一个HTML表单POST并发送一些键值字段，你可以像这样返回一个FormRequest对象（来自你的蜘蛛）：
```
return [FormRequest(url="http://www.example.com/post/action",
                    formdata={'name': 'John Doe', 'age': '27'},
                    callback=self.after_post)]
                    ```

#### Using FormRequest.from_response() to simulate a user login
网站通常通过<input type =“hidden”>元素（如会话相关数据或身份验证令牌（用于登录页面））提供预填表单字段。 在抓取时，您需要自动预填这些字段，并仅覆盖其中的几个字段，例如用户名和密码。 您可以使用FormRequest.from_response（）方法执行此作业。 这是一个使用它的蜘蛛示例：
```
import scrapy

class LoginSpider(scrapy.Spider):
    name = 'example.com'
    start_urls = ['http://www.example.com/users/login.php']

    def parse(self, response):
        return scrapy.FormRequest.from_response(
            response,
            formdata={'username': 'john', 'password': 'secret'},
            callback=self.after_login
        )

    def after_login(self, response):
        # check login succeed before going on
        if "authentication failed" in response.body:
            self.logger.error("Login failed")
            return

        # continue scraping with authenticated session...
```

### Response objects
>class scrapy.http.Response(url[, status=200, headers=None, body=b'', flags=None, request=None])
一个Response对象表示一个HTTP响应，它通常被下载（通过Downloader）并且被提供给蜘蛛进行处理。

参数：
* url (string)  - 此响应的URL
* status (integer) - 响应的HTTP状态。 默认为200。
* headers (dict) - 这个响应的标题。 字典值可以是字符串（对于单值标题）或列表（对于多值标题）。
* body (bytes) - 响应主体。 要以str（Python 2中的unicode）的形式访问解码文本，可以使用TextResponse等具有编码意识的Response子类中的response.text。
* flags（list） - 是一个包含Response.flags属性初始值的列表。 如果给出，列表将被浅拷贝。
* request (Request object)  - Response.request属性的初始值。 这表示生成此响应的请求。        

>url
一个包含响应URL的字符串。

该属性是只读的。要更改Response的URL，请使用replace（）。

>status
表示响应的HTTP状态的整数。例如：200,404。

>headers
一个包含响应头文件的类似字典的对象。可以使用get（）访问值，以返回具有指定名称的第一个标头值或getlist（）返回具有指定名称的所有标头值。例如，这个调用会为您提供标题中的所有Cookie：

>response.headers.getlist（ '设置Cookie'）
>body
这个响应的主体。请记住，Response.body始终是一个字节对象。如果您希望unicode版本使用TextResponse.text（仅在TextResponse和子类中可用）。

该属性是只读的。要更改Response的主体，请使用replace（）。

>requests
生成此响应的Request对象。在响应和请求已经通过所有Downloader中间件之后，在Scrapy引擎中分配此属性。特别是，这意味着：

* HTTP重定向会将原始请求（重定向前的URL）分配给重定向的响应（重定向后使用最终的URL）。
* Response.request.url并不总是等于Response.url
* 该属性仅在spider代码中，在Spider Middleware中可用，但在Downloader Middleware中不可用（尽管您的请求可通过其他方式在此处获得）以及response_downloaded信号的处理程序。

>meta
Response.request对象的Request.meta属性（即self.request.meta）的快捷方式。

与Response.request属性不同，Response.meta属性沿着重定向和重试传播，因此您将获得从蜘蛛中发送的原始Request.meta。

>see also

Request.meta属性

>flags
包含此响应标志的列表。标志是用于标记响应的标签。例如：“缓存”，“重定向”等。它们显示在引擎用于记录的Response（__str__方法）的字符串表示中。

>copy()
返回一个新的Response，它是此Response的副本。

>replace（[url，status，headers，body，request，flags，cls]）
使用相同的成员返回一个Response对象，除了那些由指定的关键字参数赋予新值的成员。 Response.meta属性默认被复制。

>urljoin（URL）
通过将响应的URL与可能的相对URL结合来构造绝对URL。

这是一个通过urlparse.urljoin的封装，它仅仅是进行这个调用的别名：

>urlparse.urljoin（response.url，url）
>follow(url，callback = None，method ='GET'，headers = None，body = None，cookies = None，meta = None，encoding ='utf-8'，priority = 0，dont_filter = False，errback = None）
返回请求实例以跟随链接url。它接受与Request .__ init__方法相同的参数，但url可以是相对URL或scrapy.link.Link对象，不仅是绝对URL。

除了绝对/相对URL和链接对象之外，TextResponse提供了一个支持选择器的follow（）方法。

### Response subclasses
以下是可用的内置Response子类的列表。 您也可以继承Response类来实现您自己的功能。
#### TextResponse objects
>class scrapy.http.TextResponse(url[, encoding[, ...]])

TextResponse对象将编码功能添加到基本Response类，该类仅用于二进制数据，例如图像，声音或任何媒体文件。

除了基本的Response对象之外，TextResponse对象还支持一个新的构造函数参数。 其余功能与Response类相同，在此不作介绍。

参数：
* encoding（string） - 是一个包含用于此响应的编码的字符串。 如果使用unicode主体创建一个TextResponse对象，它将使用此编码进行编码（请记住，body属性始终是一个字符串）。 如果编码为无（默认值），则将在响应标题和正文中查找编码。

TextResponse对象除了标准的Response对象外，还支持以下属性：

>text
响应主体，如unicode。

与response.body.decode（response.encoding）相同，但结果在第一次调用后被缓存，因此您可以多次访问response.text而无需额外开销。

>>注意
unicode（response.body）不是将响应主体转换为unicode的正确方法：您将使用系统默认编码（通常为ascii）而不是响应编码。

>encoding

一个包含此响应编码的字符串。 通过按以下顺序尝试以下机制来解决编码问题：

1.在构造函数编码参数中传递的编码
2.在Content-Type HTTP头中声明的编码。 如果这种编码无效（即未知），它将被忽略，并尝试下一个解析机制。
3.在响应正文中声明的编码。 TextResponse类没有为此提供任何特殊功能。 但是，HtmlResponse和XmlResponse类可以。
4.通过查看响应主体来推断编码。 这是更脆弱的方法，但也是最后一个尝试。

>selector

使用响应作为目标的选择器实例。 第一次访问时，选择器是懒洋洋地实例化的。

TextResponse对象除了标准的Response对象外，还支持以下方法：

>xpath(query)

TextResponse.selector.xpath（查询）的快捷方式：

>>response.xpath（ '// P'）

>css(query)
TextResponse.selector.css（查询）的快捷方式:

>>response.css('p')

>follow(url, callback=None, method='GET', headers=None, body=None, cookies=None, meta=None, encoding=None, priority=0, dont_filter=False, errback=None)

返回请求实例以跟随链接url。 它接受与Request .__ init__方法相同的参数，但是url不仅可以是绝对URL，还可以是

* 相对URL;
* scrapy.link.Link对象（例如链接提取器结果）;
* 属性选择器（不是选择器列表） - 例如 response.css（'a :: attr（href）'）[0]或response.xpath（'// img / @ src'）[0]。
<a>或<link>元素的选择器，例如response.css（ 'a.my_link'）[0]。
请参阅创建使用示例请求的[快捷方式](https://doc.scrapy.org/en/latest/intro/tutorial.html#response-follow-example)。

>body_as_unicode()

与文本相同，但作为方法可用。 这种方法保持向后兼容; 请优先选择response.text。

#### HtmlResponse objects\
>class scrapy.http.HtmlResponse（url [，...]）

HtmlResponse类是TextResponse的一个子类，它通过查看HTML meta http-equiv属性来添加编码自动发现支持。 请参阅TextResponse.encoding。

#### XmlResponse objects
>类scrapy.http.XmlResponse（url [，...]）

XmlResponse类是TextResponse的一个子类，它通过查看XML声明行来添加编码自动发现支持。 请参阅TextResponse.encoding。
