# 调试内存溢出

在Scrapy中，类似Requests, Response及Items的对象具有有限的生命周期: 他们被创建，使用，最后被销毁。

这些对象中，Request的生命周期应该是最长的，其会在调度队列(Scheduler queue)中一直等待，直到被处理。 更多内容请参考 [架构概览](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/architecture.html#topics-architecture) 。

由于这些Scrapy对象拥有很长的生命，因此将这些对象存储在内存而没有正确释放的危险总是存在。 而这导致了所谓的”内存泄露”。

为了帮助调试内存泄露，Scrapy提供了跟踪对象引用的机制，叫做 [trackref](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#topics-leaks-trackrefs)， 或者您也可以使用第三方提供的更先进内存调试库 [Guppy](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#topics-leaks-guppy) (更多内容请查看下面)。而这都必须在 [Telnet终端](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/telnetconsole.html#topics-telnetconsole) 中使用。

## 内存泄露的常见原因

内存泄露经常是由于Scrapy开发者在Requests中(有意或无意)传递对象的引用(例如，使用 [`meta`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/request-response.html#scrapy.http.Request.meta) 属性或request回调函数)，使得该对象的生命周期与 Request的生命周期所绑定。这是目前为止最常见的内存泄露的原因， 同时对新手来说也是一个比较难调试的问题。

在大项目中，spider是由不同的人所编写的。而这其中有的spider可能是有”泄露的”， 当所有的爬虫同时运行时，这些影响了其他(写好)的爬虫，最终，影响了整个爬取进程。

如果您没有正确释放（以前分配的）资源，泄漏也可能来自您编写的自定义中间件，管道或扩展。例如，如果您[为每个进程](https://doc.scrapy.org/en/1.5/topics/practices.html#run-multiple-spiders)运行[多个蜘蛛](https://doc.scrapy.org/en/1.5/topics/practices.html#run-multiple-spiders)，分配资源[`spider_opened`](https://doc.scrapy.org/en/1.5/topics/signals.html#std:signal-spider_opened) 但不释放它们[`spider_closed`](https://doc.scrapy.org/en/1.5/topics/signals.html#std:signal-spider_closed)可能会导致问题。

### 请求过多？

默认情况下，Scrapy将请求队列保存在内存中; 它包含 [`Request`](https://doc.scrapy.org/en/1.5/topics/request-response.html#scrapy.http.Request)对象和Request属性中引用的所有对象（例如in [`meta`](https://doc.scrapy.org/en/1.5/topics/request-response.html#scrapy.http.Request.meta)）。虽然不一定是泄漏，但这可能需要很多内存。启用 [持久作业队列](https://doc.scrapy.org/en/1.5/topics/jobs.html#topics-jobs)可以帮助控制内存使用情况。

## 使用 `trackref` 调试内存泄露

`trackref` 是Scrapy提供用于调试大部分内存泄露情况的模块。 简单来说，其追踪了所有活动(live)的Request, Request, Item及Selector对象的引用。

您可以进入telnet终端并通过 `prefs()` 功能来检查多少(上面所提到的)活跃(alive)对象。 `pref()` 是 [`print_live_refs()`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#scrapy.utils.trackref.print_live_refs) 功能的引用:

```
telnet localhost 6023

>>> prefs()
Live References

ExampleSpider                       1   oldest: 15s ago
HtmlResponse                       10   oldest: 1s ago
Selector                            2   oldest: 0s ago
FormRequest                       878   oldest: 7s ago
```

正如所见，报告也展现了每个类中最老的对象的时间(age)。

如果您有内存泄露，那您能找到哪个spider正在泄露的机会是查看最老的request或response。 您可以使用 [`get_oldest()`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#scrapy.utils.trackref.get_oldest) 方法来获取每个类中最老的对象， 正如此所示(在终端中)(原文档没有样例)。

### 哪些对象被追踪了?

`trackref` 追踪的对象包括以下类(及其子类)的对象:

- `scrapy.http.Request`
- `scrapy.http.Response`
- `scrapy.item.Item`
- `scrapy.selector.Selector`
- `scrapy.spider.Spider`

### 真实例子

让我们来看一个假设的具有内存泄露的准确例子。

假如我们有些spider的代码中有一行类似于这样的代码:

```
return Request("http://www.somenastyspider.com/product.php?pid=%d" % product_id,
    callback=self.parse, meta={referer: response})
```

代码中在request中传递了一个response的引用，使得reponse的生命周期与request所绑定， 进而造成了内存泄露。

让我们来看看如何使用 `trackref` 工具来发现哪一个是有问题的spider(当然是在不知道任何的前提的情况下)。

当crawler运行了一小阵子后，我们发现内存占用增长了很多。 这时候我们进入telnet终端，查看活跃(live)的引用:

```
>>> prefs()
Live References

SomenastySpider                     1   oldest: 15s ago
HtmlResponse                     3890   oldest: 265s ago
Selector                            2   oldest: 0s ago
Request                          3878   oldest: 250s ago
```

事实上有很多现场responses（并且它们太旧了）肯定是可疑的，因为与responses相比，Requests应该具有相对较短的生命周期。responses的数量与Requests的数量相似，因此看起来它们以某种方式相关联。我们现在可以去检查蜘蛛的代码来发现产生泄漏的令人讨厌的线（在请求中传递响应引用）。

有时候关于活动对象的额外信息会有所帮助。让我们来检查最老的回应：

```
>>> from scrapy.utils.trackref import get_oldest
>>> r = get_oldest('HtmlResponse')
>>> r.url
'http://www.somenastyspider.com/product.php?pid=123'
```

如果你想遍历所有的对象，而不是获取最老的对象，你可以使用这个[`scrapy.utils.trackref.iter_all()`](https://doc.scrapy.org/en/1.5/topics/leaks.html#scrapy.utils.trackref.iter_all)函数：

```
>>> from scrapy.utils.trackref import iter_all
>>> [r.url for r in iter_all('HtmlResponse')]
['http://www.somenastyspider.com/product.php?pid=123',
 'http://www.somenastyspider.com/product.php?pid=584',
...
```

### 很多spider?

如果您的项目有很多的spider，`prefs()` 的输出会变得很难阅读。针对于此， 该方法具有 `ignore` 参数，用于忽略特定的类(及其子类)。例如，这不会显示任何对蜘蛛的实时引用：

```
>>> from scrapy.spiders import Spider
>>> prefs(ignore=Spider)
```

### scrapy.utils.trackref模块

以下是 [`trackref`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#module-scrapy.utils.trackref) 模块中可用的方法。

- *class*`scrapy.utils.trackref.object_ref`

  如果您想通过 `trackref` 模块追踪活跃的实例，继承该类(而不是对象)。


- `scrapy.utils.trackref.print_live_refs`(*class_name*, *ignore=NoneType*)

  打印活跃引用的报告，以类名分类。参数:**ignore** (*类或者类的元组*) – 如果给定，所有指定类(或者类的元组)的对象将会被忽略。


- `scrapy.utils.trackref.get_oldest`(*class_name*)

  返回给定类名的最老活跃(alive)对象，如果没有则返回 `None` 。首先使用[`print_live_refs()`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#scrapy.utils.trackref.print_live_refs) 来获取每个类所跟踪的所有活跃(live)对象的列表。


- `scrapy.utils.trackref.iter_all`(*class_name*)

  返回一个能给定类名的所有活跃对象的迭代器，如果没有则返回 `None`。首先使用 [`print_live_refs()`](http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/leaks.html#scrapy.utils.trackref.print_live_refs) 来获取每个类所跟踪的所有活跃(live)对象的列表。

## 使用Guppy调试内存泄露

`trackref` 提供了追踪内存泄露非常方便的机制，其仅仅追踪了比较可能导致内存泄露的对象 (Requests, Response, Items及Selectors)。然而，内存泄露也有可能来自其他(更为隐蔽的)对象。 如果是因为这个原因，通过 `trackref` 则无法找到泄露点，您仍然有其他工具: [Guppy library](http://pypi.python.org/pypi/guppy) 。

如果使用 `setuptools` , 您可以通过下列命令安装Guppy:

```
easy_install guppy
```

telnet终端也提供了快捷方式(`hpy`)来访问Guppy堆对象(heap objects)。 下面给出了查看堆中所有可用的Python对象的例子:

```
>>> x = hpy.heap()
>>> x.bytype
Partition of a set of 297033 objects. Total size = 52587824 bytes.
 Index  Count   %     Size   % Cumulative  % Type
     0  22307   8 16423880  31  16423880  31 dict
     1 122285  41 12441544  24  28865424  55 str
     2  68346  23  5966696  11  34832120  66 tuple
     3    227   0  5836528  11  40668648  77 unicode
     4   2461   1  2222272   4  42890920  82 type
     5  16870   6  2024400   4  44915320  85 function
     6  13949   5  1673880   3  46589200  89 types.CodeType
     7  13422   5  1653104   3  48242304  92 list
     8   3735   1  1173680   2  49415984  94 _sre.SRE_Pattern
     9   1209   0   456936   1  49872920  95 scrapy.http.headers.Headers
<1676 more rows. Type e.g. '_.more' to view.>
```

您可以看到大部分的空间被字典所使用。接着，如果您想要查看哪些属性引用了这些字典， 您可以:

```
>>> x.bytype[0].byvia
Partition of a set of 22307 objects. Total size = 16423880 bytes.
 Index  Count   %     Size   % Cumulative  % Referred Via:
     0  10982  49  9416336  57   9416336  57 '.__dict__'
     1   1820   8  2681504  16  12097840  74 '.__dict__', '.func_globals'
     2   3097  14  1122904   7  13220744  80
     3    990   4   277200   2  13497944  82 "['cookies']"
     4    987   4   276360   2  13774304  84 "['cache']"
     5    985   4   275800   2  14050104  86 "['meta']"
     6    897   4   251160   2  14301264  87 '[2]'
     7      1   0   196888   1  14498152  88 "['moduleDict']", "['modules']"
     8    672   3   188160   1  14686312  89 "['cb_kwargs']"
     9     27   0   155016   1  14841328  90 '[1]'
<333 more rows. Type e.g. '_.more' to view.>
```

如上所示，Guppy模块十分强大，不过也需要一些关于Python内部的知识。关于Guppy的更多内容请参考 [Guppy documentation](http://guppy-pe.sourceforge.net/).

## Leaks without leaks

有时候，您可能会注意到Scrapy进程的内存占用只在增长，从不下降。不幸的是， 有时候这并不是Scrapy或者您的项目在泄露内存。这是由于一个已知(但不有名)的Python问题。 Python在某些情况下可能不会返回已经释放的内存到操作系统。关于这个问题的更多内容请看:

- [Python Memory Management](http://evanjones.ca/python-memory.html)
- [Python Memory Management Part 2](http://evanjones.ca/python-memory-part2.html)
- [Python Memory Management Part 3](http://evanjones.ca/python-memory-part3.html)

改进方案由Evan Jones提出，在 [这篇文章](http://evanjones.ca/memoryallocator/) 中详细介绍，在Python 2.5中合并。 不过这仅仅减小了这个问题，并没有完全修复。引用这片文章:

> *不幸的是，这个patch仅仅会释放没有在其内部分配对象的区域(arena)。这意味着 碎片化是一个大问题。某个应用可以拥有很多空闲内存，分布在所有的区域(arena)中， 但是没法释放任何一个。这个问题存在于所有内存分配器中。解决这个问题的唯一办法是 转化到一个更为紧凑(compact)的垃圾回收器，其能在内存中移动对象。 这需要对Python解析器做一个显著的修改。*

这个问题将会在未来Scrapy发布版本中得到解决。我们打算转化到一个新的进程模型， 并在可回收的子进程池中运行spider。