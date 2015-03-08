cgi 脚本
========

阿帕奇 HTTP 服务器通过配置 ``Options ExecCGI`` 执行 cgi 脚本。

cgi 脚本就是简单的文本文件，可以用任何文本编辑器编写。
对于 Windows 操作系统，可以用文本文档或者 ``Notepad``。

原理
----

服务器和脚本的通信是过标准输入输出缓冲区进行的。

* 服务器接收到客户端请求后，将参数放入标准输入缓冲区；
* 脚本通过输入缓冲区获取用户请求参数；
* 脚本处理完之后将所有信息输出到标准输出缓冲区，服务器从中获取响应头和响应体
  
脚本输出的信息分为三类：

* 正常输出响应头，如 ``Content-Type``
* 正常输出响应数据，可以是任意类型。
* 错误输出响应头，为 ``Status``

编写 cgi 脚本没有固定的脚本语言，只要满足 cgi 脚本相关格式即可。
你可以用 Python，也可以用 Perl，还可以用批处理脚本，根据个人情况而定。

本文选择 Python cgi 脚本进行说明。

脚本解析器路径
--------------

任何脚本都要通过特定的脚本解析器执行，Python 脚本也不例外。
Python 脚本通过 ``python.exe`` 解释执行，所以脚本第一行需要给出解析器路径：

.. code-block:: python

 #!<your Python's path>

依据个人安装 Python 的情况不同，填写相应路径。
比如在 Windows 操作系统中，以默认路径安装 Python 2.7.2：``#!C:/Python27/python.exe``。
如果是 Linux 操作系统，则可能是：``!/usr/bin/python``。

.. note:: Perl 脚本也一样：``#!/usr/bin/perl``

实际上服务器就是通过第一行构造 shell 命令行，执行脚本的。
所以可以在第一行加上一些解析器选项参数。
例如：

``#!C:/Python/Python27/python.exe -s -t``

服务器执行 cgi.py 脚本时，执行命令行 ``C:/Python/Python27/python.exe -s -t cgi.py``。

HTTP 响应头
-----------

HTTP 头是一些简单的文本行，放在服务器响应体前面发送给浏览器。
浏览器不会显示这些头信息，会依据头信息决定如何解析响应内容并展示。

``Content-Type`` 头一般是 cgi 脚本输出的第一项内容，例如：

.. code-block:: python
 
 print 'Content-Type: text/html\n'

以上示例指定响应内容以 HTML 文本格式展示。

此外，还可以有其他 HTTP 头，如：``Content-Language`` 和 ``Content-Length`` 等。
需要注意的是，HTTP 头信息之后需要有一个空行，之后才可以添加响应正文。

打印网页内容
------------

对于媒体类型为 ``text/html`` 响应，只要将响应内容依据 HTML 格式打印到标准输出区即可。
例如：

.. code-block:: python

 print '<html>'
 print '<body>Hello,world</body>'
 print '</html>'

至此，一个简单的 Python cgi 脚本就完成了。

完整代码如下：

.. code-block:: python
 :linenos:

 #!C:/Python27/python.exe

 print 'Content-Type: text/html\n'
 print '<html>'
 print '<body>Hello,world</body>'
 print '</html>'

对于其他媒体类型，比如图片 (``image/*``)，二进制数据(``application/octet-stream``)等，做法是一样的。

举个例子，现在需要将 `/cgi-bin/`` 目录下的图片 ``beauty.png`` 发给客户端。
脚本可以这样写：

.. code-block:: python

 #!C:/Python27/python.exe -u
 # script_name: show_image.py

 print 'Content-Type: image/png\n'

 print file('beauty.png', 'rb').read()

在浏览器上运行 ``http://<servername>/cgi-bin/show_image.py`` 即可显示该图片了。

这里注意几个问题：

* 在指定 ``python.exe`` 的路径时，加上了 ``-u`` 选项，表示不缓存二进制的标准输出流和错误流，直接发送给客户端。
  缓存的话，可能因转码的问题导致丢失数据，引起图片失真。
* 在读取文件的时候注意指明为 ``rb``，采用二进制的方式读取。

另外为了保险起见，最好使用 ``shutil.copyfileobj()`` 代替 file.read()。
因为 ``file.read()`` 在文件很大的情况下，可能会出现读取异常。

其实还有一种更简单的方法，利用 HTML 的 ``img`` 元素。

.. code-block:: python
 
 print 'Content-Type: text/html\n'

 print "<html><body><img src = 'http://<servername>/beauty.png' alt='beautiful girl' /></body></html>"

传递参数
--------

前面介绍了 cgi 脚本的一个简单例子。
现在有个问题，如何通过 url 传递参数给 cgi 脚本呢？
可以使用 ``?`` 和 ``+``。

例如：

``http://<servername>/login.py?hob+123456``

服务器收到请求后，将参数 ``hob``，``123456`` 传递给脚本。
传递方式和命令行传递参数一样。

Python 脚本可以通过 ``os.argv`` 获取参数。
Perl 脚本通过 ``${n}`` (n=1,2,3,...) 获取参数。 

返回服务器文件
--------------

将一个已经存在的文件发回客户端，
例如：``hello_world.html`` 和 cgi 同目录。
可以在 cgi 脚本中写：

.. code-block:: python
 
 print 'Location: hello_world.html' # 等价于 print 'http://<servername>/cgi-bin/hello_world.html'

返回错误响应
------------

``print 'Status: <statusCode> <message>\n'``

.. note:: 后面跟一个空行

e.g: ``print 'status: 204 No Response\n``

服务器配置
----------

完成 cgi 脚本之后，需要进行相应的配置，使阿帕奇服务器可以识别并执行该脚本。

1. 加载模块 ``cgi_module.so``，这是前提条件；
2. 让服务器能识别 cgi 脚本。
   
   利用 ``ScriptAlias``, ``AddHandler``, ``SetHandler``等指实现。   

   如：

   .. code-block:: html

    <IfModule alias_module>
        ScriptAlias /cgi-bin/ "F:/my_cgi_bin/"
    </IfModule>

    <Directory "F:/my_cgi_bin">
        SetHandler cgi-script # AddHandler cgi-script .py
        Options ExecCGI
    </Directory>

   将 `hello_world.py` 放在目录 ``F:/my_cgi_bin/`` 下。

   如果要求 ``.py`` 文件才是 cgi 脚本，可以这样：
   
  .. code-block:: html

     <LoctionMatch "/.py$">
         SetHandler cgi-script
     </LoctionMatch>
     
3. 编写 cgi 脚本。

