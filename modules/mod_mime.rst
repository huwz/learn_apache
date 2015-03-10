mod_mime 模块
=============

总结
----

模块将 URI 中的文件模式或文件名映射为元类型，作为 HTTP 响应的内容。
例如，内容文件的扩展名通常对应该文件的媒体类型，语言，字符集和编码方式。
HTTP 发送消息包含了这些内容，用于内容协商，在多个可能的选择中挑出一个满足用于偏好。

指令 ``AddCharset``, ``AddEncoding``, ``AddLanguage`` 以及 ``AddType`` 的作用都是将文件扩展名映射为某个元类型。
分别对应字符集，编码方式，语言以及文档的媒体类型。
指令 ``typesConfig`` 指定一个文件，将扩展名映射成媒体类型。

另外，``mod_mime`` 可以定义解析器和过滤器，用于初始化和处理内容。
指令 ``AddHandler``, ``AddOutputFilter`` 以及 ``AddInputFilter`` 对模块或脚本进行控制。
``MultiViewsMatch`` 允许模块 ``mod_negotiation`` 在测试 ``MultiViews`` 搜索时，考虑加载哪些扩展名。

模块 ``mod_mime`` 将元类型和文件扩展名联系起来；
而 ``core`` 指令将给定节点（``<Location>``, ``<Directory>`` 或 ``<Files>``）中的所有文件与某个元类型联系起来。
指令分别是 ``ForceType``, ``SetHandler``, ``SetInputFilter`` 和 ``SetOutputFilter``。
``core`` 模块指令会覆盖 ``mod_mime`` 定义的扩展名映射。

注意，修改文件的元类型，不会改变 ``Last_Modified`` 头。
因此，用户或者代理依然可以使用之前缓存的副本。
如果你改了元类型（语言，媒体类型，字符集或者编码），你还需要修改文件的时间戳（即最近修改日期），来保证所有来访者可以获得正确的响应头。

多扩展名文件
------------

文件可以拥有不止一个扩展名，扩展名的顺序一般是无关紧要的。
例如，通过扩展名映射，文件 ``welcome.html.fr`` 的媒体类型是 ``text/html``，语言是法语；
改变扩展名顺序，``welcome.fr.html`` 的映射结果是一样的。
如果多个扩展名都映射到同一种元类型，则选择排在最右边的映射结果。
这个约定对语言和编码方式不适用。
例如， ``.gif`` 的媒体类型是 ``image/gif``，``.html`` 的媒体类型是 ``text/html``，则文件 ``welcome.gif.html`` 的媒体类型是 ``text/html``。

我们需要注意多扩展名文件与某个媒体类型、解析器相关联的情况。
这通常表示请求会由该解析器对应模块处理。
比如说，如果通过扩展名映射，``.imap`` 后缀由 ``imap-file`` 解析（``mod_imagemap``），``.html`` 的媒体类型是 ``text/html``，则文件 ``world.imap.html`` 会被服务器关联到 ``imap-file`` 和 ``text/html``。
处理时，会使用 ``imap-file`` 解析器解析文件，而且 ``world.imap.html`` 会被当成 ``mod_imagemap`` 内存映像文件。

如果你更希望文件名的最后一部分映射成某个元类型，则不要使用指令 ``Add*``。
例如，如果希望 ``foo.html.cgi`` 在处理时当成一个 CGI 脚本，而 ``bar.cgi.html`` 则不会，你可以使用以下配置方法来代替 ``AddHandler cgi-script .cgi``:

.. code-block:: text

    <FilesMatch \.cgi$>
        SetHandler cgi-script
    </FilesMatch>

内容编码
--------

特定媒体类型的文件可能会额外采用特殊的编码方式，来简化因特网的数据传输。
这通常指压缩方式，比如 ``gzip`` 方式，当然也包括加密方法，比如 ``pgp`` 或者 ``UUencoding``。
后者采用 ASCII 码存储格式的二进制文件进行传输。

``HTTP/1.1 RFC`` 第 14.11 节点是这样描述的::
 
 Content-Encoding 头实体字段，作为媒体类型的标识。
 在进行资源显示时，该值表示主体实体应该采用何种额外的编码方式。
 进而采用什么样的解码机制，以获取 Content-Type 头字段指定的媒体类型。
 Content-Encoding 头主要作用是允许文档在不丢本来的媒体类型实的前提下进行压缩处理。

通过多个扩展名，可以指定文件的特定媒体类型以及特定的编码方式。

例如，你可以使用 ``Microsoft Word`` 文档文件，采用 ``pkzipped`` 方式压缩。
如果 ``.doc`` 后缀和 ``Microsoft Word`` 文件关联，``.zip`` 后缀和 ``pkzip`` 编码关联，
则 ``Resume.doc.zip`` 会被当成 ``pkzip`` 压缩的 ``Microsoft Word`` 文档。

阿帕奇通过对资源发送 ``Content-Encoding`` 头，告诉浏览器编码方式。

``Content-Encoding: pkzip``

字符集和语言
------------

除了文件类型和文件编码，还有两条重要信息。
即文档的表达语言是什么，采用什么字符集来显示文件内容。
例如，文档用越南语或者西里尔语编写，因此必须采用对应的语言方式展示。
这些信息，同样包含在 HTTP 头中。

字符集，语言，编码方式以及 ``mime`` 类型是内容协商过程中都要用到的信息。
当有多个文档