mod_negotiation 模块
====================

总结：

内容协商，或者更准确的说，内容选择，为的是从多个文档中选择最符合客户端性能的文档。
有两种实现方法：

* 类型映射（带 ``type-map`` 解析器的文件），将含有变体的文件显式列举
* ``MultiViews`` 搜索（通过指令 ``Options MultiViews`` 开启）。
  服务器通过文件模式匹配，选择最符合的结果。

类型映射
--------

类型映射的格式类似于标准协议 ``RFC822`` 中对邮件头的定义格式。
它包含文档描述和注释；文档描述由空行隔开，注释行开头带有 ``#`` 字符。
文档描述由多个头记录构成；每个头记录可以有多行，后续行必须以空格开头。
多行进行合并的时候，行前面的空格会被删除。
每个头记录包含关键字名称，以冒号结尾，之后是值。
头名称与值之间，值标号中可以存在空白符。

头名称包括：

* Content-Encoding
  
  文件的编码方式。
  阿帕奇只能识别由指令 ``AddEncoding`` 定义的编码方式。
  该指令通常包含 ``x-compress`` 和 ``x-gzip`` 两种压缩方式。
  匹配压缩方法时，``x-`` 前缀会被忽略。

* Content-Language
  
  变体采用的语言必须是协议 ``RFC 1766`` 定义的国际标准语言标签。
  比如 ``en`` 表示英语。
  如果变体不止一种语言，通过逗号隔开。

* Content-Length
  
  文件的字节长度。
  如果没有指定这个头，则使用文件的实际长度值。

* Content-Type
  
  文档的 ``mime`` 媒体类型，后面可以跟其它参数。
  参数和媒体类型之间，参数彼此之间都通过分号隔开。
  参数语法格式是 ``name=value``。
  常用的参数为：

  * level
    
    表示媒体类型的版本，是一个整数。如 ``text/html``，这个值默认为 2，否则为 0

  * qs
    
    浮点型数，范围从 0[.000] 到 1[.000]，表示变体相对于其他变体而言的 **起始质量**。
    它和客户端的性能无关。
    比如，对于一张照片，jpeg 文件格式通常比 ascii 码文件格式的起始质量高。
    而对于 ascii 码艺术品，则 ascii 文件格式要比 jpeg 文件格式的起始质量高。
    qs 值对于给定资源来说是很重要的参数。

    例如：

    ``Content-Type: image/jpeg; qs=0.8``

* URI
  
  表示含有不同媒体类型，不同编码方式等变体的文件。
  通过类型映射文件，映射为相应的 url；
  不同变体对应的文件必须处在同一台服务器上，且必须是客户端可以直接请求访问的文件。

* Body
  
  资源内容可以通过 ``Body`` 头包含在类型映射文件中。
  这个头字段含有一个字符串分隔符，用于划定主体内容范围。
  因此，如下内容中 ``Body`` 头以下，``----xyz----`` 分隔符之上属于资源内容。

  .. code-block:: text
  
      Body:----xyz----
      <html>
      <body>
      <p>Content of the page.</p>
      </body>
      </html>
      ----xyz----

  打个比方，如资源文件名为 ``document.html``，它有英文，法文和德文三种版本。
  每个版本对应的文件名分别为 ``document.html.en``, ``document.html.fr`` 以及 ``document.html.de``。
  类型映射文件名是 ``document.html.var``，其内容如下：

  .. code-block:: text
  
      URI: document.html

      Content-Language: en
      Content-Type: text.html
      URI: document.html.en

      Content-Language: fr
      Content-Type: text.html
      URI: document.html.fr

      Content-Language: de
      Content-Type: text.html
      URI: document.html.de

四个文件必须处在同一个目录中。
``.var`` 文件应该用指令 ``AddHandler`` 指定了 ``type-map`` 解析器：

``AddHandler type-map .var``

服务器收到对 ``document.html.var`` 文件的请求之后，会根据用户请求头 ``Accept-Language`` 设定的值，选择一个最匹配的变体。

如果开启了 ``MultiViews`` 选项，且指令 ``MultiViewsMatch`` 设置了选项 ``handlers`` 或者 ``any``，
则针对文档 ``document.html`` 发起的请求，服务器会先找到 ``document.html.var`` 文档，然后根据类型映射进行内容协商。

其他指令，比如 ``Alias``，可以建立从 ``document.html`` 到 ``document.html.var`` 的映射。

MultiViews
----------

``MultiViews`` 搜索通过指令 ``Options MultiViews`` 开启。
如果服务器收到请求 ``/some/dir/foo``，但是 ``/some/dir/foo`` 文件不存在。
则服务器会搜索目录 ``/some/dir/`` 下所有符合模式 ``foo.*`` 的文件。
同时高效地伪造一个类型映射，放入所有匹配的文件名，并给它们赋相同的媒体类型和内容编码。
如果客户端通过文件名来查询这些值，则服务器返回伪造的映射值。
服务器返回给客户端的是最匹配度的结果。

``MultiViewsMatch`` 对阿帕奇服务器进行设置，在匹配文件时，是否考虑那些没有分配协商信息的文件。

模块指令
--------

CacheNegotiatedDocs
^^^^^^^^^^^^^^^^^^^

描述：允许代理服务器存储协商的文档
语法：``CacheNegotiatedDocs On|Off``
默认：``CacheNegotiatedDocs Off``
应用场合：server config. virtual host
模块：mod_negotiation

一旦设置以后，允许代理服务器缓存协商的文档。
这就意味着客户端可以找回那些不能完全满足自身性能的文档版本，
但可以让代理服务器缓存更加高效。

指令只对 HTTP/1.0 浏览器的请求有用。
HTTP/1.1 对于协商文档的缓存提供了更完善的控制。
所以该指令对 	HTTP/1.1 请求的响应没有任何作用。

ForceLanguagePriority
^^^^^^^^^^^^^^^^^^^^^

描述：一个匹配的文档都没有时，执行该指令
语法：``ForceLanguagePriority None|Prefer|Fallback [Prefer|Fallback]``
默认：``ForceLanguagePriority Prefer``
应用场合：server config, virtual host, directory, .htaccess
覆盖：FileInfo
模块：mod_negotiation

如果服务器通过协商，没有找到一个匹配的文档，则 ``ForceLanguagePriority`` 指令会使用 ``LanguagePriority`` 完成协商。
当存在多个同等有效的请求时，``ForceLanguagePriority Prefer`` 使用 ``LanguagePriority`` 提供一个有效的结果；
而不是返回 HTTP 响应状态码 300（``MULTIPLE CHOICES``)。
如果设置了如下指令，且用户的 ``Accept-Language`` 头给 ``en`` 和 ``de`` 赋相同的质量值 0.500，则服务器会返回第一个匹配的变体 ``en``：

.. code-block:: text

    LanguagePriority en fr de
    ForceLanguagePriority Prefer

``ForceLanguagePriority Fallback`` 使用 ``LanguagePriority`` 提供一个有效结果，而不是返回HTTP 状态码 406（``NO ACCEPTABLE``)。
如果设置了如下指令，且用户的 ``Accept-Language`` 只允许响应使用 ``en`` 语言标签，但这样的变体不存在，则 ``LanguagePriority`` 列表中第一个变体会返回给客户端：

.. code-block:: text

    LanguagePriority en fr de
    ForceLanguagePriority Fallback

如果同时指定 ``Prefer`` 和 ``Fallback``，则有多个匹配变体时，选择第一个匹配的变体；
如果没有匹配的，则选择列表中的第一个。

LanguagePriority
^^^^^^^^^^^^^^^^

描述：在客户端没有设定的情况下，指定变体语言的优先级
语法：``LanguagePriority MIME-lang [MIME-lang] ...``
应用场合：server config, directory, .htaccess
覆盖：FileInfo
模块：mod_negotiation

在对一个请求执行 ``MultiViews`` 解析时，如果客户端没有设定语言偏好，则 LanguagePriority`` 指定变体语言的优先级。
``MIME-lang`` 列表按照偏好程度进行降序排列。

``LanguagePriority en fr de``

对于文件请求 ``foo.html``，当 ``foo.html.fr`` 和 ``foo.html.de`` 同时存在，但是浏览器并没有给出偏好值，那么 ``foo.html.fr`` 会作为结果返回。

**注意**，这个指令只在其它方法无法确定最好的语言变体或者 ``ForceLanguagePriority`` 指令设置的不是 ``None`` 的情况下，才会生效。
一般而言，客户端决定了语言的偏好，不是服务器。