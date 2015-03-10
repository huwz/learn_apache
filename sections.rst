.. _sections:

配置节点(Configuration Sections)
================================

配置文件中的指令可以作用于整个服务器，也可以限定作用范围，只对某些目录，某些文件，主机或者 URL 起作用。
这篇文档描述了如何使用配置节点容器（configuration section container，简称节点）或者 `.htacess` 文件改变其他配置指令的作用范围。

配置节点的类型
--------------

+----------------+-------------+
| 节点名         | 模块        |
+================+=============+
| Directory      | core        |
+----------------+-------------+
| DirectoryMatch | core        |
+----------------+-------------+
| Files          | core        |
+----------------+-------------+
| FilesMatch     | core        |
+----------------+-------------+
| If             | core        |
+----------------+-------------+
| IfDefine       | core        |
+----------------+-------------+
| IfModule       | core        |
+----------------+-------------+
| Location       | core        |
+----------------+-------------+
| LocationMatch  | core        |
+----------------+-------------+
| VirtualHost    | core        |
+----------------+-------------+
| IfVersion      | mod_version |
+----------------+-------------+
| Proxy          | mod_proxy   |
+----------------+-------------+
| ProxyMatch     | mod_proxy   |
+----------------+-------------+

节点分为两种基本类型。
大多数节点的作用是评估每个客户端请求。
节点中封闭的指令只作用于那些能匹配该节点的请求。
其他节点，如 IfDefine, IfModule 和 IfVersion，只是在服务器启动和重启的时候发挥作用。
在服务器启动的时候，如果条件为真，则节点中的封闭指令就会作用到所有请求上。
如果条件不为真，则节点指令会被忽略。

IfDefine 中的封闭指令是否生效，取决于 httpd 命令行参数。
比如以下配置的作用是，当服务器通过命令行 ``httpd -DClosedForNow`` 启动时，所有请求会重定向到另一个网站上。

.. code-block:: text

    <IfDefine ClosedForNow>
        Redirect / http://otherserver.example.com/
    </IfDefine>

IfModule 指令和 IfDefine 相似，只是生效的条件不同。
只有当某个模块存在时，才会执行 IfModule 中的指令。
该模块要么在服务器中静态编译，要么通过 LoadModule 指令加载和动态编译。
这个指令可以保证无论某个模块是否安装，你的配置文件都可以正常工作。
它不应该包含那些无条件执行的指令，因为该节点会将缺失模块的有用错误信息压制。

在下面的例子中，MimeMagicFile 指令只在 mod_mime_magic 模块存在条件下才起作用。

.. code-block:: text

    <IfModule mod_mime_magic>
        MimeMagicFile conf/magic
    </IfModule>

IfVersion 指令和 IfDefine 以及 IfModule 也很相似，不同之处在于执行条件。
只有满足某个服务器版本，才能执行该节点指令。
IfVersion 主要用于测试套件和大型网络，需要处理不同的 httpd 版本和配置。

.. code-block:: text

    <IfVersion >= 2.4>
    </IfVersion>

IfDefine, IfModule 和 IfVersion 可以使用 ``!`` 表示相反的条件。
为了实现更复杂的限制，这些节点可以相互嵌套使用。

文件系统，网站空间和布尔表达式
------------------------------

最常用的节点可以改变文件系统或网站空间特定位置的配置信息。
首先，理解文件系统和网站空间的区别是很重要的。
文件系统就是在操作系统中看到的磁盘目录空间。
比如，默认情况下，阿帕奇安装在 Unix 文件系统的 ``/usr/local/apache2/`` 目录下，
或在 Windows 文件系统的 ``C:/Program Files/Apache Group/Apache2`` 目录下。
（注意路径分隔符总是使用反斜杠，即使是 Windows 系统也是如此。）

与之相对的是网站空间。
网站空间是从网站服务器或者客户端看到的具体视图。

.. note:: 题外话

  网站空间可以看做是 URL 的集合

因此对于默认安装 httpd 的 UNIX 系统， URL 路径 ``/dir/`` 对应于文件路径 ``/usr/local/apache2/htdocs/dir/``。
网站空间不一定要对应文件系统，因为网页有可能是通过数据库或者其他方式动态产生的。

文件系统容器
^^^^^^^^^^^^

Directory 和 Files 节点，以及它们的正则副本（即 DirectoryMatch 和 FilesMatch），都是作用于文件系统的。
Directory 中的指令应用于指定文件系统目录以及该目录下的所有子目录和文件。
通过 .htacess 文件也可以达到同种效果。
比如以下配置信息的作用是对 ``/var/web/dir1`` 建立资源索引。

.. code-block:: text

    <Directory /var/web/dir1>
        Options +Indexes
    </Directory>

.. note:: 资源索引是指子目录和文件的列表。

Files 节点中的指令应用于某些特定名称的文件。

.. note:: 只要文件名匹配，不论文件处在什么位置，都会生效。

因此，以下配置指令一旦放在配置文件的主配置区，就会阻止一切针对 ``private.html`` 文件的访问请求。

.. code-block:: text

    <Files private.html>
        Require all denied
    </Files>

.. note::
 主配置区，一般是指主配置文件(httpd.conf 包括 Include 配置文件的内容）中非节点区域。
 主配置区的指令可以应用于所有的请求。

网站空间节点
^^^^^^^^^^^^

Location 节点和它的正则副本，可以修改网站空间的配置。
例如以下配置可以阻止任何 URL 路径以 ``/private`` 开始的请求。

.. code-block:: text

    <LocationMatch ^/private>
        Require all denied
    </LocationMatch>

同样，它也会对

``http://yoursite.example.com/private``
``http://yoursite.example.com/private123``
``http://yoursite.example.com/private/dir/file.html``

生效。

Location 指令不一定和文件系统有关。
比如以下配置为特定 URL 指定解析器 ``server-status`` (``mod_status``)。
但服务器的文件系统中没有 ``server-status`` 文件。

.. code-block:: text

    <Location /server-status>
        SetHandler server-status
    </Location>

网站空间重复
^^^^^^^^^^^^

为了让两个有重叠部分的 URL 共存，需要考虑节点或者指令执行的顺序。
对于 Location，可以这样：

.. code-block:: text

    <Location /foo>
    </Location>
    <Location /foo/bar>
    </Location>

Alias 刚好相反：

.. code-block:: text

    Alias /foo/bar /srv/www/uncommon/bar
    Alias /foo /srv/www/common/foo

ProxyPass 和 Alias 一样：

.. code-block:: text

    ProxyPass /special-area http://special.example.com smax=5 max=10
    ProxyPass / balancer://mycluster/ stickysession=JSESSIONID|jsessionid nofailover=On

通配符和正则表达式
^^^^^^^^^^^^^^^^^^

Directory, Files 和 Location 指令都可以用命令窗口风格的通配符。
``*`` 匹配所有字符序列，``?`` 匹配单个字符，``[seq]`` 匹配 seq 字符序列中的任意一个字符。
``/`` 不能匹配任何通配符，必须显式标明。

更灵活的方式，DirectoryMatch, FilesMatch 以及 LocationMatch 可以使用 perl 风格的正则表达式进行匹配。

一个使用通配符的例子，可以改变所有用户目录的配置：

.. code-block:: text

    <Directory /home/*/public_html>
        Options Indexes
    </Directory>

通过正则表达式，可以阻止访问多种类型的图片文件：

.. code-block:: text

    <FilesMatch \.(?i:gif|jpe?g|png)$>
        Require all denied
    </FilesMatch>

正则表达式中有分组和后向引用概念。
节点执行正则表达式时，分组名将会以大写的方式添加到环境变量。
这样就可以在后续表达式和模块（比如 mod_rewrite）中通过环境变量引用文件路径和 URL 中的元素。

.. code-block:: text

    <DirectoryMatch ^/var/www/combined/(?<SITENAME>[^/]+)>
        require ldap-group cn=%{env:MATCH_SITENAME},ou=combined,o=Example
    </DirectoryMatch>

布尔表达式
^^^^^^^^^^

If 节点通过布尔表达式的真假值来改变服务器配置。
例如以下配置将阻止任何 HTTP Refer 头不是以 ``http://www.example.com/`` 开始的请求。

.. code-block:: text

    <If "!(%{HTTP_REFERER} -strmatch 'http://www.example.com/*')">
        Require all denied
    </If>

何时何地使用
^^^^^^^^^^^^

选择文件系统节点还是网站空间节点是很容易的事情。
当指令针对的是文件系统中的对象，则使用 Directory 或 Files。
而对象不是在文件系统中（比如数据库产生的网页），则使用 Location。

限制客户端对文件系统对象的访问权限不能用 Location，这点是很重要的。
因为许多不同的网站地址 (URL) 可以映射到同一个文件系统位置，从而绕开限制。
例如：

.. code-block:: text

    <Location /dir/>
        Require all denied
    </Location>

当请求 url 是 ``http://yoursite.example.com/dir/`` 时，该配置可以正常工作。
但是如果是一个不区分大小写的文件系统呢？
那么限制可以轻松被绕开，如：``http://yoursite.example.com/DIR/``。
相比较之下，Directory 指令可以作用于该目录下的任何内容，无论名称是什么。
（唯一例外是文件系统的符号连接。
通过符号连接，同一个目录可以放置在文件系统的多个地方。
Directory 指令会跟随符号连接，不会修改路径名。
因此，对于安全级别高的环境，应该禁止使用符号连接。）

如果你认为这都不需要考虑，因为你使用的是大小写敏感的系统。
那么请注意有许多其他的方式，将多个网站地址映射到同一个文件系统目录。
因此尽量使用文件系统节点。
有一个例外， ``<Location />`` 指令可以绝对安全，因为它会作用于所有的请求。

节点嵌套
^^^^^^^^

某些节点类型可以相互嵌套。
如：Files 可以用于 Directory 中，If 可以用在 Directory, Location 以及 Files 中的元素。
正则副本行为也是如此。

合并时，含有嵌套的节点放在同类型的非嵌套节点后面。

虚拟主机
--------

VirtualHost 节点包围的指令作用于特定的主机。
同一台机器，可以提供多个配置信息不同主机。
关键是要使用 VirtualHost 节点。

代理
----

Proxy 和 ProxyMatch 节点指令只对 mod_proxy 代理服务器起作用。
例如，如下配置信息可以防止代理服务器访问 ``www.example.com``。

.. code-block:: text

    <Proxy http://www.example.com/*>
        Require all granted
    </Proxy>

指令权限
--------

哪些指令可以哪些节点中使用，可以查询指令的 Context 属性。
凡是可以用在 Directory 中的指令语法上也能用在 DirectoryMatch, Files, FilesMatch, Location, LocationMatch, Proxy 以及 ProxyMatch 中。
有一些例外：

* AllowOverride 指令只能用在 Directory 中
* ``Options FollowSymlinks/SymLinksIfOwnerMatch`` 只能用在 Directory 或 .htacess 文件中
* Options 指令不能用在 Files 和 FilesMatch 中

节点怎么合并
------------

节点合并的顺序是很特殊的。
因为这关系到指令的解析方法，所以理解它很关键。

合并顺序：

1. Directory 和 .htacess 同时执行（后者覆盖前者）
2. DirectoryMatch
3. Files 和 FilesMatch 同时执行
4. Location 和 LocationMatch 同时执行
5. If

除了 Directory 之外，相同节点按照配置文件中出现的位置由上到下执行。
Directory 执行顺序从最短的开始直到最长的。
比如 ``<Directory /var/web/dir>`` 和 ``<Directory /var/web/dir/subdir>``，先执行前者。
如果多个 Directory 节点作用于同一个目录，则按照出现顺序依次执行。
Include 指令加载的配置文件会被当成一直处在指令位置一样。

VirtualHost 中嵌套的节点在 VirtualHost 外部所有指令执行之后才开始解析。
因而 VirtualHost 可以修改主服务器配置信息。

当请求由 mod_proxy 受理时，需要将以上执行顺序中的 Directory 用 Proxy 替换。

后执行的节点可以覆盖先执行的节点。
覆盖方式由节点对应模块负责。
后执行的节点将先执行结果的部分或全部指令合并。
甚至整个替换。

.. note::
 Location/LocationMatch 序列必须在名称转换阶段之前执行（即指令 Aliases 之前）。
 如果名称转换结束，则 Location/LocationMatch 序列会被忽略。