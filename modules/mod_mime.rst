mod_mime 模块
=============

总结
----

``mod_mime`` 模块将 URI 中的文件名或者文件匹配模式映射成元数据值。
再将元信息分配给 HTTP 响应消息选定的内容。
比如响应文件的文件扩展名通常定义了响应内容的媒体类型，表达语言，字符集和内容编码。
这些元信息通过含有响应内容的 HTTP 消息发送；
同时在面临多个选择时，作用于内容协商；保证从多个可能的内容文件中选择一个，能尊重用户的偏好。

``AddCharset``, ``AddEncoding``, ``AddLanguage`` 以及 ``AddType``，这些指令都是把文件扩展名映射为文件的元数据。
它们分别用于设置字符集，内容编码，内容语言以及文档媒体类型。
``typesConfig`` 指定一个文件，它的作用同样是将扩展名映射成媒体类型。

另外，``mod_mime`` 可以定义解析器和过滤器，用于生成和处理响应内容。
``AddHandler``, ``AddOutputFilter`` 和 ``AddInputFilter``，这些指令控制那些提供文档的模块或脚本。
``MultiviewsMatch`` 对 ``mod_negotiation`` 模块进行设置，它在做 ``Multiviews`` 搜索时是否考虑即将加入的文件扩展名。

``mod_mime`` 将元数据和文件扩展名关联起来；
而 ``core`` 指令将容器（``<Location>``, ``<Directory>`` 或 ``<Files>``）中所有文件和特定元数据关联起来。
这些指令包括 ``ForceType``, ``SetHandler``, ``SetInputFilter`` 以及 ``SetOutputFilter``。
核心模块指令将所有 ``mod_mime`` 中定义的扩展名映射都覆盖了。

注意改变文件的元数据会修改 ``Last-Modified`` 头的字段值。
因此，之前缓存的副本，头字段没变，依然可以由客户端或者用户代理使用。
如果你改变了文件的元数据（语言，内容类型，字符集或者内容编码），你可能需要 ``touch`` 改动的文件（更新它们的最近修改日期），来保证，所有访问者获取到的都是正确的内容头字段。

多扩展名文件
------------

文件可以有不止一个扩展名；
扩展名的顺序正常情况下是不重要的。
假如 ``welcome.html.fr`` 映射到媒体类型 ``text/html``，其语言映射为法语。
则文件 ``welcome.fr.html`` 也会映射到相同的信息。
如果有多个扩展名都映射到同一种元数据类型，则最右边的类型会被采用。

.. note:: 语言和内容编码除外

假如，``.gif`` 映射成 ``image/gif``, ``.html`` 映射成 ``text/html``，
则文件 ``welcome.gif.html`` 被映射成 ``text/html``。

语言和内容编码的效果是叠加的，因为可以将多个语言标签或者编码方式赋值给某个特定资源。
例如，``welcome.html.en.de`` 在传送时，HTTP 响应头中有 ``Content-Language: en, de`` 和 ``Content-Type: text/html`` 等头字段。

如果多扩展名文件和某个媒体类型以及某个解析器关联起来，需要特别注意。
这通常表示相应请求由特定模块解析，该模块和解析器有关。
假如， ``.imap`` 后缀映射成 ``imap-file`` 解析器（``mod_imagemap``）， ``.html`` 被映射成 ``text/html``，则文件 ``world.imap.html`` 同时与 ``imap-file`` 解析器以及 ``text/html`` 媒体类型关联。
处理 ``world.imap.html`` 时， 用 ``imap-file`` 进行解析。
所以文件还会被当成 ``mod_imagemap`` 的内存映像文件。

如果你只希望文件名用点分隔的最后部分映射成特定的元数据，则不要使用 ``Add*`` 指令。
比如，你想让 ``foo.html.cgi`` 被当成 CGI 脚本处理，但 ``bar.cgi.html`` 不会被当成 CGI 脚本，不能使用 ``AddHandler cgi-script .cgi``，可以这样设置：

.. code-block:: text

    <FilesMatch \.cgi$>
        SetHandler cgi-script
    </FilesMatch>

内容编码
--------

特定媒体类型的文件可能会额外采用特殊方式编码，来简化因特网的数据传输。
这种编码通常指压缩方式，比如 ``gzip`` 方式。
也可能是加密方法，比如 ``pgp`` 或者某种编码方式，比如 ``UUencoding``；
设计成二进制文件进行发送，文件以 ASCII 格式存储。

``HTTP/1.1 RFC`` 第 14.11 节这样描述::
 
 Content-Encoding 实体头字段作为媒体类型的标识符。
 如果存在，它的值表示实体消息的附加内容编码。
 因而为获取 ``Content-Type`` 头字段引用的媒体类型值，必须使用对应的解码机制。
 ``Content-Encoding`` 主要作用是将文档进行压缩，且不失实体的本来媒体类型。

通过多扩展名，可以指定文件的媒体类型和编码方式。

例如，你有一个 ``Microsoft Word`` 文档，采用 ``pkzipped`` 方式压缩。
如果 ``.doc`` 后缀和 ``Microsoft Word`` 文件关联，``.zip`` 后缀和 ``pkzip`` 编码关联，
则 ``Resume.doc.zip`` 会被当成 ``pkzip`` 压缩的 ``Microsoft Word`` 文档。

阿帕奇通过发送 ``Content-Encoding`` 头，告诉浏览器资源的编码方式。

``Content-Encoding: pkzip``

字符集和语言
------------

除了文件类型和文件编码，还有两条重要信息。
即文档的表达语言是什么，采用什么字符集来显示文件内容。
例如，文档用越南语或者西里尔语编写，因此必须采用对应的语言方式展示。
这些信息，同样包含在 HTTP 头中。

字符集，语言，编码方式以及 ``mime`` 类型是内容协商过程中都要用到的信息。
当有多个文档