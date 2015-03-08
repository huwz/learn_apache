Appache 阿帕奇
==============

新手教学
--------

.. note::
 http://httpd.apache.org/docs/current/getting-started.html

* ``Clients, Server, URL``

  网络地址通过 URL [1]_ 描述；
  一个完整的 URL 从左到右依次分解为网络协议，服务器名，URL 路径以及可能的查询串。

  .. note::
   http://127.0.0.1:8000/query?name=Appache，其中:
   协议：http
   服务器名：127.0.0.1:8000
   URL 路径：/query
   查询串 name=Appache
  
  
  客户端通过 URL 向服务器发起请求，索取 URL 路径指定的资源。
  该资源可能是服务器上的文本文档（如 ``HTML`` 文档），可执行程序，或者其他类型文件（比如 ``index.php``）。

  .. note:: 在服务器上创建 ``HTML`` 文件，并用 URL 路径链接该文件，浏览器可以通过 URL 路径访问 ``HTML`` 文件。
   服务器主页一般都是通过这种方式加载的。

  服务器将响应发给客户端，包括响应状态码和响应消息体（``response body``）。
  服务器通过状态码告诉客户端是否成功受理请求；如果没有，则指定错误码和错误信息。

* 主机名和域名服务器
  
  为连接到服务器，客户端首先要知道服务器的名称和 IP 地址。
  要想在因特网中访问服务器名，必须保证服务器名处在 DNS 中。

  一个 IP 地址可能对应多个服务器名称，而同一个物理服务器可能具有多个 IP 地址。
  因此在同一个物理服务器上可以建立多个网站，使用的是“虚拟主机”。

  如果你想测试一个本地服务器（不能被因特网访问），可以修改 hosts 文件，例如添加记录：``127.0.0.1 www.example.com``

* 配置文件和指令
  
  阿帕奇 HTTP 服务器通过文本文件存储配置信息。
  配置文件可以出现在服务器文件系统的任意位置，取决于组建服务器的方式。
  如果阿帕奇应用是通过源文件编译得到的，则默认的配置文件路径为：``/usr/local/apache2/conf/httpd.conf``。

  配置文件可以分解为若干更小的子文件，再通过 ``Include`` 指令加载::

    Include /usr/local/apache2/conf/ssl.conf
    Include /usr/local/apache2/conf/vhosts/*.conf

  .. note::
   apache 服务器由配置指令设置；配置指令由关键字和参数构成。

  如果配置指令应用于全局，应该放在配置文件节点之外（如 ``<Directory>``）；
  如果作用于某个目录，则应该放在指向该目录的 ``Directory`` 节点中。
  
  除了配置文件（.conf）外，某些指令也可以放在 ``.htaccess`` 文件中。

  .. note::
   ``.htaccess`` 文件主要是为无权访问主站配置文件的用户设置的。

配置节点
--------

.. note::
 http://httpd.apache.org/docs/current/sections.html

配置文件中的节点类似于 HTML 源码中的元素对象，格式如下：

.. code-block:: html

 <section>
    directives
 </section>

配置指令(directive)可以应用于整个服务器，也可以应用于某个特定的目录，文件，主机或者 URL。

.. note:: 相对路径/目录以 ``/`` 开始，绝对路径/目录需要加上双引号

* 配置节点类型
  
  配置节点包括：``<Directory>`` ``<DirectoryMatch>`` ``<Files>`` ``<FilesMatch>`` ``<If>`` 
  ``<IfDefine>`` ``<IfModule>`` ``<Loaction>`` ``<LocationMatch>`` ``<Proxy>`` 
  ``<ProxyMatch>`` ``<VirtualHost>``

  有两种基本类型：

  * 只对客户端请求起作用；
  * ``<IfDefine>``，``<IfModule>`` 和 ``<IfVerion>`` 服务器启动时，如果条件为真，则处于节点中的指令作用于所有客户端请求，否则节点中的指令被忽略。

  ``<IfDefine>`` 针对 ``httpd`` 指令的参数。

  .. note::
   ``httpd -D name`` 指定一个 name，一般是 ``<IfDefine name>`` 指令中的 name。

  例如：

  .. code-block:: html
  
   <IfDefine ClosedForNow>
       Redirect / http://otherserver.example.com/
   </IfDefine>

  以上配置在执行 ``httpd -DClosedForNow`` 启动服务器时，将所有请求进行重定向。

  ``<IfModule>`` 用于判断模块是否可用。

  .. note::
   所谓模块，就是执行特定功能的独立代码块。
  
  用得最多的模块是动态模块（``so`` 文件），在 ``modules`` 目录下就有大量动态模块。
  
  
  模块一般是服务器静态编译的；
  如果是动态编译的，则要借助 ``LoadModule`` 先加载该模块。 
  
  若不确定某个模块是否加载，可以使用 ``<IfModule module>`` 指令做判断。
  例如在下面的例子中， ``MimeMagicFile`` 指令只有在 ``mod_mime_magic.c`` 模块存在的情况下起作用。

  .. code-block:: html
  
   <IfModule mod_mime_magic.c>
       MimeMagicFile conf/magic
   </IfModule>

  ``<IfVerion>`` 限定服务器的版本。

  .. code-block:: html
   
   <IfVerion >= 2.4>
       # this happens only in versions greator or 
       # equal 2.4.0.
   </IfVerion>

  ``<IfDefine>``， ``<IfModule>`` 和 ``<IfVerion>`` 可以用 ``!`` 表示相反条件；
  还可以进行嵌套处理，以实现一些更复杂的限制。

文件系统，网站空间和布尔表达式
------------------------------
  
改变文件系统或者网站空间位置的节点类型最常用。
  
* 文件系统就是从操作系统角度看到的磁盘视图

  例如，对于默认的安装，阿帕奇的 ``httpd`` 工具在 ``/usr/local/Apache2`` 目录下 (Linux) 或者 ``c:/Program Files/Apache Group/Apache2`` (Windows)。
    
  .. note:: 
   反斜杠总是作为阿帕奇 ``httpd`` 配置文件的路径分隔符，无论是 Linux 还是 Windows OS。

* 网站空间是由服务器发送的在客户端看到的地址视图
   
   .. note:: 可以认为网站空间是 URL 集合
    
  因此网站空间中的 ``/dir/`` 对应于文件系统中 ``/usr/local/apache2/htdocs/dir/`` （UNIX）。
   
  网页可能是数据库或者其他程序动态产生的，所以网站空间不一定会映射到文件系统。

文件系统节点
^^^^^^^^^^^^
   
包括 ``<Directory>``， ``<Files>``， ``<DirectoryMatch>``， ``<FilesMatch>``，作用于文件系统的某个目录或文件。

.. note:: ``<*Match>`` 表示支持正则的节点
  
``<Directory>`` 节点包围的指令作用于指定目录，包括子目录和文件
例如，以下配置，将会检索 ``/var/web/dir1`` 目录和所有子目录：

.. code-block:: html
  
 <Directory /var/web/dir1>
     options +Indexes
 </Directory>

``<Files>`` 中的指令应用于指定文件，无论文件处在哪个目录。
例如，以下配置禁止访问 ``private.html``，无论该文件在哪个位置：

.. code-block:: html

 <Files private.html>
     Require all denied
 </Files>

为检索文件系统指定目录下的某些文件，可以将 ``<Files>`` 和 ``<Directory>`` 联合使用。
例如：

.. code-block:: html

 <Directory /var/web/dir1>
     <Files private.html>
         Require all denied
     </Files private.html>
 </Directory>
        
限定 ``/var/web/dir1`` 以及它的所有子目录下的 ``private.html`` 都不可访问。

网站空间节点
^^^^^^^^^^^^
   
``<Loaction>`` 和正则副本 ``<LocationMatch>``，作用于 URL-path。
例如：

.. code-block:: html

 # 拒绝所有 URL-path 以 /private 开始的请求。
 <LocationMatch ^/private>
     Require all denied
 </LocationMatch>


.. code-block:: html

 # 将特殊 URL 映射到阿帕奇内部的 HTTP 请求处理器。
 <Location /server-status>
     setHandler server-status
 </Location>

注意，server-status 不一定是文件夹或者文件
 
网站空间重叠
^^^^^^^^^^^^

若 URL 有重叠，则需要考虑节点执行顺序。

如 ``<Location>``：

.. code-block:: html
 
 <Loaction /foo>
 </Loaction>
 <Loaction /foo/bar>
 </Loaction>

如 ``Alias``：

.. code-block:: html

 Alias /foo/bar /srv/www/uncommon/bar
 Alias /foo /srv/www/common/foo

如 ``ProxyPass``：

.. code-block:: html
   
 ProxyPass /special-area http://special.example.com smax=5 max=10
 ProxyPass / balancer://mycluster/ stickysession=JSESSIONID|jessionid nofailover=On

这里空格表示 URL 根目录，special-area 是其子目录。

通配符和正则表达式
------------------
    
``<Directory>``, ``<Files>`` 和 ``<Location>`` 指令可以使用通配符：

* ``*`` 表示匹配任何字符串
* ``?`` 表示匹配任何字符
* ``[seq]`` 表示匹配 seq 中的任何字符
      
如果使用更复杂的匹配规则，则需要借助节点 ``<*Match>``。
如：``<DirectoryMatch>``, ``<FilesMatch>`` 以及 ``<LocationMatch>``，允许使用 ``perl`` 正则表达式。

不使用正则：

.. code-block:: html

 <Directory /home/*/public_html>
     Options Indexes
 </Directory>

使用正则：

.. code-block:: html

 <FilesMatch \.(?i:gif|je?g|png)$>
     Require all denied
 </FilesMatch>

以上指令可以阻止访问许多类型的图片文件。

.. note:: 
 ``(?option pattern)`` is equivalent to ``pattern/option``;
 ``(?i:gif|je?g|png)$`` alternative: ``(?:gif|je?g|png)$/i`` indicates that the pattern matching is insensitive

正则表达式如果包含命名的分组和后向引用，则组名会添加到环境变量中；

如：

.. code-block:: html

 <DirectoryMatch ^/var/www/combined/(?<SITENAME>[^/]+)>
     require ldap-group cn=%{env:MATCH_SITENAME},ou=combined,o=Example
 </DirectoryMatch>

布尔表达式
----------

``<If>`` 节点修改配置信息，取决于布尔表达式的值。
例如，如下配置的作用是，如果 ``HTTP_REFERER`` 不是以 ``http://www.example.com`` 开头，
则拒绝访问。

.. code-block:: html

 <If "!(%{HTTP_REFERER} -strmatch 'http://www.example.com/*')">
      Require all denied
 </If>

节点选择策略
------------

选择文件系统节点还是网站空间节点的策略很简单。
如果指令应用于文件系统中的对象，则使用 ``<Directory>`` 或者 ``<Files>``；
如果指令应用的对象不在文件系统中（如由数据库产生的网页），则使用 ``<Location>``。

如果限制访问文件系统中的对象，一定不能用 ``<Loaction>``。
因为许多不同的网站空间（URL）可能映射到同一个文件系统位置，导致限制被绕过。
例如：

.. code-block:: html

 <Loaction /dir/>
     Require all denied
 </Location>

如果请求是 ``http://yoursite.example.com/dir/``，会被禁止。
但如果是一个大小写不敏感的文件系统呢？
``http://yoursite.example.com/DIR/`` 会轻易绕开禁止限制。

相反，``<Directory>`` 指令，会应用到该位置处的任意文件，无论大小写和调用方式。
所以还是推荐大家使用文件系统节点。

.. note:: 
 通过符号链接，同一个目录可能映射文件系统的多个位置。
 ``<Directory>`` 使用的是符号链接，而不是路径名。
 因而限制同样可能被绕开。
 所以，若要工作在最高安全级别下，符号链接应该被 ``<Options>`` 指令禁止。

当然如果这样限定：``<Loaction />`` 则非常安全，该节点可以应用于所有请求。

节点嵌套
--------
  
一些节点类型可以在其他节点类型中嵌套使用：

* ``<Files>`` 可以在 ``<Directory>`` 中使用；
* ``<If>`` 可以在 ``<Directory>``， ``<Location>`` 和 ``<Files>`` 中使用。
* 正则副本也遵循上述规则

虚拟主机
--------

``<VirtualHost>`` 节点包含的指令应用于特地的主机。
这个指令对于在同一台机器上有多个主机，每个主机的配置信息不同。

代理
----

``<Proxy>`` 和 ``<ProxyMatch>`` 在设定了代理服务器时生效。

.. note:: 可以通过指令 ``mod_proxy`` 设定代理服务器。

.. code-block:: html

 <Proxy http://www.example.com/*>
     Require all granted
 </Proxy>

若设置了代理服务器，以上配置将阻止代理服务器访问 www.example.com 网站。  

节点指令
--------

对于不同类型的节点，哪些指令是允许的呢？

记住：凡是 ``<Directory>`` 中允许的指令都可以使用在其他节点中。
也有例外：

* ``AllowOverride`` 指令只能工作在 ``<Directory>``
* ``Options FollowSymLinks/SymLinksIfOwnerMatch`` 只能用在 ``<Directory>`` 或者 ``.htaccess`` 文件中。
* ``Options`` 指令不能用在 ``<Files>`` 和 ``<FilesMatch>`` 中。

节点执行顺序
------------

节点的执行顺序如下：

* ``<Directory>`` （除正则表达式）和 ``.htaccess`` 同时执行（``.htaccess`` 会覆盖 ``<Directory>``）
* ``<DirectoryMatch>`` (``<Directory ~>``)
* ``<Files>`` 和 ``<FilesMatch>`` 同时执行
* ``<Location>`` 和 ``<LocationMatch>`` 同时执行
* ``<If>``
  
除了 ``<Directory>`` 之外，上述节点按照出现在配置文件中的位置处理。
``<Directory>`` 按照目录最短到最长的顺序处理。
如：``<Directory /var/web/dir>`` 将在 ``<Directory /var/wev/dir/subdir>`` 之前处理。
如果有多个 ``<Directory>`` 节点应用到用一个目录，则按照出现顺序处理。
通过 ``Include`` 指令包含的配置文件会被加载到指令位置。

在所有外部节点应用之后，``<VirtualHost>`` 内部节点开始应用。
该节点允许虚拟主机修改服务器的配置。

如果请求由 ``mod_proxy`` 设定的代理服务器受理，则 ``<Proxy>`` 将取代 ``<Directory>`` 作为第一个应用。

后应用的节点可以改写先应用的节点，然而每个模块负责解析改写的采取的方式。
后应用的节点可能需要对某些指令进行“概念”合并。

.. [1] `Uniform Resource Locators` 资源唯一标识符