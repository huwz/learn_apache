mod_mime 模块
=============

总结
----

模块功能：将 URI 中的文件名或者文件模式转化为元数据，分配给响应消息。
文件扩展名通常对应媒体类型，表达语言，字符集和内容编码等元数据。
HTTP 响应消息中含有这些元数据信息。
元数据参与内容协商，可以保证服务器在选择响应文件时充分尊重用户的偏好。

以下指令将扩展名映射成对应的元数据

* AddCharset
* AddEncoding
* AddLanguage
* AddType

这些指令对字符集，内容编码，语言以及媒体类型进行设置

以下指令为服务器响应指定解析器和过滤器：

* AddHandler
* AddOutputFilter
* AddInputFilter
  
扩展名类型映射可以单独存储在文件中。
文件名由指令 TypesConfig 指定。

``MultiviewsMatch`` 控制 ``Multiviews`` 搜索。

.. note:: ``mod_mime`` 将扩展名关联到元数据，``core`` 将节点文件关联到元数据。

与元数据相关的 core 指令：

* ``ForceType``
* ``SetHandler``
* ``SetInputFilter``
* ``SetOutputFilter``。

功能重复时，core 指令覆盖 ``mod_mime`` 指令。

修改文件的元信息不影响 ``Last-Modified`` 头字段。
所以在修改文件的元信息之后，最好能手动更新文件最近修改日期。

多扩展名文件
------------

文件可以有多个扩展名，扩展名的顺序一般不重要。
如果 ``welcome.html.fr`` 对应媒体类型为 ``text/html``，对应语言为法语，则 ``welcome.fr.html`` 同样如此。
如果多个扩展名对应同种元类型，则使用最右边的结果。
例如 ``welcome.gif.html`` 对应媒体类型为 ``text/html``。

.. note:: 语言和内容编码除外

语言和编码可以累加。
如 ``welcome.html.en.de`` 在传送时，HTTP 响应头中有 ``Content-Language: en, de`` 和 ``Content-Type: text/html`` 等头字段。

需要注意多扩展名同时关联到某个媒体类型和解析器的情况。
这表示文件由特定模块，特定解析器解析。
假设 ``.imap`` 对应解析器 ``imap-file``，``.html`` 对应媒体类型 ``text/html``，
则 ``world.imap.html`` 是 ``text/html`` 类型，由 ``imap-file`` 解析。
而且文件可以看做 ``mod_imagemap`` 内存映像文件。

如果只映射最后一个扩展名，则用 ``Set*`` 代替 ``Add*`` 指令。

.. code-block:: text

    <FilesMatch \.cgi$>
        SetHandler cgi-script
    </FilesMatch>

则 ``foo.html.cgi`` 是 CGI 脚本，``foo.cgi.html`` 不是。

内容编码
--------

为简化互联网数据传输，对特定媒体类型的文件做内容编码。
编码方法可能是压缩，也可能是加密，比如 ``gzip`` 或 ``pgp``；
发送的文件以 ASCII 码格式二进制存储。

``HTTP/1.1 RFC <http://www.ietf.org/rfc/rfc2616.txt>``_ 第 14.11 节中这样描述::
 
 Content-Encoding 头可以作为媒体类型的标识符。
 头字段值表示实体内容采取的内容编码。
 在解析 ``Content-Type`` 头指定的媒体类型时，必须使用对应的解码机制。
 ``Content-Encoding`` 允许对文档进行无损压缩。

通过多扩展名，可以将文件与特定媒体类型、编码方式绑定。
假设 ``.doc`` 表示 ``Microsoft Word`` 文档，``.zip`` 表示 ``pkzip`` 压缩编码，
则 ``Resume.doc.zip`` 表示 ``pkzip`` 压缩的 ``Microsoft Word`` 文档。

阿帕奇通过发送 ``Content-Encoding`` 头，告诉浏览器资源的编码方式。

``Content-Encoding: pkzip``

字符集和语言
------------

除了媒体类型和内容编码，文档的语言和字符集也很重要。
例如使用越南语或者西里尔语的文档，显示时也必须使用对应的语言。
语言信息会包含在 ``Content-Language`` 头中。

字符集，语言，编码方式以及 ``mime`` 类型在内容协商中都要用到。
以下指令将扩展名映射为元信息，参与协商过程。

* ``AddCharset``
* ``AddEncoding``
* ``AddLanguage``
* ``AddType`` 

.. note:: 扩展名列举在 MimeMagicFile 指定的文件中。

只通过以下指令关联的扩展名，协商时是否考虑由 ``MultiviewsMatch`` 决定：

* ``AddHandler``
* ``AddInoputFilter``
* ``AddOutputFilter``
  
字符集
^^^^^^

为转达字符集信息，阿帕奇选择发送 ``Content-Language`` 头指定文档语言。
再附加一些信息到 ``Content-Type`` 头上，表示正确渲染信息必须采用的字符集。

``Content-Language: en, fr Content-Type: text/plain; charset=IOS-8859-1``

语言标签是语言前两个字母构成的缩写。
charset 是特殊字符集的名称。

``mod_mime`` 模块中的指令：

AddCharset
----------

* 描述：将扩展名映射为具体字符集
* 语法：``AddCharset charset extension [extension] ...``
* 覆盖：FileInfo
* 场合：server config, virtual host, directory, .htacess

`charset` 是媒体类型的字符集参数。
指定含有 `extension` 后缀的资源文件对应的字符集。
这种转换是强制的，可以覆盖之前的设置。

.. code-block:: text

    AddLanguage ja .ja
    AddCharset EUC-JP .euc
    AddCharset ISO-2022-JP .jis
    AddCharset SHIFT_JIS .sjis

则文档 ``xxxx.ja.jis`` 的语言是日语，字符集为 ``ISO-2022-JP`` （``xxxx.jis.ja`` 也是如此）
AddCharset 有两个作用：

* 通知客户端文档的字符编码
* 内容协商

`extension` 参数不区分大小写。
可以带 ``.`` 也可以不带。
对多个扩展名的文件名，`extension`` 会一一比较。

AddEncoding
-----------

* 描述：将扩展名映射为具体编码类型
* 语法：``AddEncoding encoding extension [extension] ...``
* 场合：server config, virtual host, directory, .htacess
* 覆盖：FileInfo

将扩展名映射成具体的 HTTP 内容编码。
对所有后缀为 `extension` 的文件，将指定的内容编码追加到 Content-Encoding 头中。
该映射是强制执行的，覆盖之前的设置。

.. code-block:: text

    AddEncoding x-gzip .gz
    AddEncoding x-compress .Z

则带 ``.gz`` 后缀的文件会被标记为经过 ``x-gzip`` 编码；
而带 ``.Z`` 后缀的文件会被标记为经过 ``x-compress`` 编码。

老版的客户端希望用 ``x-gzip`` 和 ``x-compress``。
但标准协议认定它们和 ``gzip``, ``compress`` 等价。
阿帕奇进行内容编码比较时，会忽略前缀 **x-**。
对应服务器响应的编码采用的是客户端请求的格式。
如果客户端没有要求特定格式，则阿帕奇采用 ``AddEncoding`` 中的编码。
为简化步骤，针对两种压缩编码，应该用 ``x-gzip`` 和 ``x-compress``。
其它的编码，比如 deflate，则不要加 **x-** 前缀。

`extension` 参数不区分大小写。
可以带 ``.`` 也可以不带。
对多个扩展名的文件名，`extension`` 会一一比较。

AddHandler
----------

描述：将扩展名映射为具体解析器
语法：``AddHandler handler-name extension [extension]...``
场合：server config, virtual host, directory, .htacess
覆盖：FileInfo

带 `extension` 后缀的文件由 `handler-name`` 解析器解析。
这种映射是强制执行的，覆盖之前的设置。

``AddHandler cgi-script .cgi``

表示 **.cgi** 后缀的文件都是 CGI 脚本，由 **cgi-script** 解析器解析。
该指令如果放在 ``http.conf`` 中，则含有 ``.cgi`` 的文件都会被当成 CGI 脚本。

`extension` 参数不区分大小写。
可以带 ``.`` 也可以不带。
对多个扩展名的文件名，`extension`` 会一一比较。

AddInputFilter
---------------

描述：将扩展名映射成过滤器，处理客户端请求
语法：``AddInputFilter filter[;filter] extension [extension]...``
场合：server config, virtual host, directory, .htacess
覆盖：FileInfo

带有 `extension` 扩展名的文件用 `filter` 处理。
指令同样会覆盖之前的设置。

如果有多个 `filter`，则必须用分号隔开，处理请求时的顺序依指令中顺序。
`filter` 不区分大小写。
`extension` 参数不区分大小写。
可以带 ``.`` 也可以不带。
对多个扩展名的文件名，`extension`` 会一一比较。

AddLanguage
-----------

描述：将扩展名映射成具体的语言
语法：``AddLanguage language-tag extension [extension] ...``
场合： server config, virtual host, directory, .htacess
覆盖：FileInfo

