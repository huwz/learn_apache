mod_negotiation 模块
====================

总结：

内容协商，或更准确地说是内容选择，是指从多个文档中选择最符合客户端性能的文档。
有两种实现方法：

* 类型映射（通过 ``type-map`` 解析的文件），可以显式列举包含变体信息的所有文件；
* ``MultiViews`` 搜索（通过指令 ``Options MultiViews`` 开启）。
  服务器通过 ``MultiViews`` 搜索执行隐式的文件名模式匹配，选择一个最符合的结果。

类型映射
--------

类型映射的格式类似于标准协议 ``RFC822`` 中定义的邮件头的格式。
它包含文档描述和注释；文档描述由空行隔开，注释行开头带有 ``#`` 字符。
每个文档描述由几个头记录构成；每个头记录可以有多行，后续行必须以空格开头。
合并的时候，后续行前面的空格会被删除。
每个头记录包含关键字名称，以冒号结尾，之后是记录值。
名称与值之间，值符号中可以有空白符。

允许的头记录包括：

* Content-Encoding
  
  文件编码。
  阿帕奇只能识别由指令 ``AddEncoding`` 定义的编码方式。
  正常情况下，会包含两种压缩方式，``x-compress`` 和 ``x-gzip``；
  前者针对 ``compress`` 格式的文件，后者针对 ``gzip`` 格式的文件。
  在做编码比较时，会忽略 ``x-`` 前缀。

* Content-Language
  
  变体语言，作为因特网标准语言标签（``RFC 1766``）。
  比如 ``en`` 表示英语。
  如果变体语言不止一种，通过逗号隔开。

* Content-Length
  
  文件的字节长度。
  如果没有出现 ``Content-Length`` 头，则使用文件的实际长度。

* Content-Type
  
  文档的 ``mime`` 媒体类型，附带有可选参数。
  参数和媒体类型之间以及参数彼此都通过分号隔开。
  参数语法格式是 ``name=value``。
  常用的参数为：

  * level
    
    表示媒体类型的版本，是一个整数。如 ``text/html``，这个值默认为 2，否则为 0

  * qs
    
    浮点型数，范围从 0[.000] 到 1[.000]，表示变体相对于其他变体而言的 **起始质量**。
    它和客户端的性能无关。
    比如，对于一张照片，jpeg 文件格式通常比 ascii 码文件格式的起始质量高。
    而对于 ascii 码艺术品，ascii 文件格式要比 jpeg 文件格式的起始质量高。
    qs 值对于给定资源来说是很重要的参数。

    例如：

    ``Content-Type: image/jpeg; qs=0.8``

* URI
  
  表示包含指定媒体类型，指定编码方式的变体的文件 uri。
  它们被解析成 url，解析方式与映射文件相关；
  它们必须处在同一台服务器上，而且必须是客户端可以通过直接请求访问的文件。

* Body
  
  资源的实际内容可以通过 ``Body`` 头包含在类型映射文件中。
  ``Body`` 头字段必须为主体内容指定一个字符串分隔符。
  从该分隔符开始，直到找到另一个分隔符为止，中间的所有行被视为资源主体部分。

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
  类型映射文件是 ``document.html.var``，其内容如下：

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
``.var`` 文件必须通过 ``AddHandler`` 设定，与 ``type-map`` 解析器相关联：

``AddHandler type-map .var``

服务器收到用户对目录下的 ``document.html.var`` 的请求后，会根据请求头 ``Accept-Language`` 设定的语言偏好值，选择一个最匹配的变体。

如果开启了 ``MultiViews`` 选项，且 ``MultiViewsMatch`` 设置选项为 ``handlers`` 或者 ``any``，
则对 ``document.html`` 发起的请求，服务器先找到 ``document.html.var`` 文档，然后根据显式类型映射进行内容协商。

其他指令，比如 ``Alias``，可以建立从 ``document.html`` 到 ``document.html.var`` 的映射。

MultiViews
----------

``MultiViews`` 搜索通过指令 ``Options MultiViews`` 开启。
如果服务器收到文件请求 ``/some/dir/foo``，但是 ``/some/dir/foo`` 文件不存在。
则服务器会搜索目录 ``/some/dir/`` 下所有符合模式 ``foo.*`` 的文件。
同时高效地伪造一个类型映射，放入所有匹配的文件名，并给它们赋相同的媒体类型和内容编码。
如果客户端通过文件名来查询这些值，则服务器返回伪造的映射值。
服务器最终返回给客户端的是最匹配度的结果。

``MultiViewsMatch`` 配置阿帕奇服务器，选择文件时是否考虑没有分配协商元信息的文件。

模块指令
--------

CacheNegotiatedDocs
^^^^^^^^^^^^^^^^^^^

描述：允许代理服务器缓存内容协商的文档
语法：``CacheNegotiatedDocs On|Off``
默认：``CacheNegotiatedDocs Off``
应用场合：server config. virtual host
模块：mod_negotiation

一旦设置，指令允许代理服务器缓存内容协商的文档。
这意味着代理服务器背后的客户端可以找回那些不能完全满足自身性能的文档版本。
可以让缓存更加高效。

指令只对 HTTP/1.0 浏览器的请求有用。
HTTP/1.1 对协商文档的缓存提供了更好的控制。
指令对 HTTP/1.1 请求的响应不起作用。

ForceLanguagePriority
^^^^^^^^^^^^^^^^^^^^^

描述：没找到匹配的文档时，执行该指令
语法：``ForceLanguagePriority None|Prefer|Fallback [Prefer|Fallback]``
默认：``ForceLanguagePriority Prefer``
应用场合：server config, virtual host, directory, .htaccess
覆盖：FileInfo
模块：mod_negotiation

如果服务器在协商之后，没有找到匹配的文档，则 ``ForceLanguagePriority`` 通过 ``LanguagePriority`` 满足协商。
如果有多个同等有效的选择，``ForceLanguagePriority Prefer`` 通过 ``LanguagePriority`` 提供一个有效的结果，而不是返回 HTTP 响应状态码 300（``MULTIPLE CHOICES``)。
如果设置如下指令，且用户请求头 ``Accept-Language`` 中给 ``en`` 和 ``de`` 赋相等的质量值 0.500，则第一个匹配的变体 ``en`` 会被返回给客户端：

.. code-block:: text

    LanguagePriority en fr de
    ForceLanguagePriority Prefer

``ForceLanguagePriority Fallback`` 通过 ``LanguagePriority`` 提供一个有效结果，而不是返回HTTP 状态码 406（``NO ACCEPTABLE``)。
如果设置如下指令，且用户请求头 ``Accept-Language`` 只允许 ``en`` 语言标签，但这样的变体不存在，则 ``LanguagePriority`` 列表中第一个变体会返回给客户端：

.. code-block:: text

    LanguagePriority en fr de
    ForceLanguagePriority Fallback

如果同时指定 ``Prefer`` 和 ``Fallback``：

* 存在多个可选变体时，返回 ``LanguagePriority`` 中的第一个匹配结果
* 没有变体满足客户端能接受的语言列表时，则从 ``LanguagePriority`` 中选择第一个服务器存在的变体文档。

LanguagePriority
^^^^^^^^^^^^^^^^

描述：在客户端没有设定语言偏好的情况下，指定变体语言的优先级
语法：``LanguagePriority MIME-lang [MIME-lang] ...``
应用场合：server config, virtual host, directory, .htaccess
覆盖：FileInfo
模块：mod_negotiation

解析 ``MultiViews`` 请求时，如果客户端没有设定语言偏好，则 LanguagePriority`` 指定变体语言的优先级。
``MIME-lang`` 列表按照偏好程度从大到小排序：

``LanguagePriority en fr de``

对于请求 ``foo.html``，如果 ``foo.html.fr`` 和 ``foo.html.de`` 同时存在，但是浏览器并没有给出语言偏好，那么 ``foo.html.fr`` 会作为结果返回。

**注意**，只有其它途径无法确定最好语言或者 ``ForceLanguagePriority`` 不是 None 时，``LanguagePriority`` 才有作用。
大致上说，由客户端来决定语言的偏好，而不是服务器。

学习总结
--------

+--------------------------------+-----------------------------------------------------+
| ``ForceLanguagePriority`` 选项 | LanguagePriority 列表                               |
+================================+=====================================================+
| Prefer                         | 有多个匹配时，返回第一个匹配 Accept-Language 的变体 |
+--------------------------------+-----------------------------------------------------+
| Fallback                       | 没有匹配时，返回第一个变体                          |
+--------------------------------+-----------------------------------------------------+
| Prefer Fallback                | 以上两者结合                                        |
+--------------------------------+-----------------------------------------------------+
