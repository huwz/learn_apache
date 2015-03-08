.. _server_side_includes:

Server Side Includes
====================

“服务端嵌入” 提供了一种方法，可以给现有的 HTML 文档动态添加新的内容。

简介
----

“服务端嵌入” 一般简称为 ``SSI``。

``SSI``
-------

``SSI`` 是放在 HTML 网页中的指令，当网页被加载的时候，执行这些指令。
这些指令可以让你给一个已有的 HTML 页面动态增加新的内容，不用经过 CGI 程序或者其他动态嵌入技术。

例如，你可能给网页加了这样一条指令：``<!--#echo var="DATE_LOCAL" -->``。
当网页加载的时候，该代码片段会被执行为：``Wensday, 11-Feb-2015 14:48:10 GMT``。

配置服务器
----------

为使服务器的 ``SSI`` 生效，需要在服务器配置文件或者 ``.htacess``  文件中配置：

.. code-block:: html

 Options +Includes

该指令告诉阿帕奇允许解析 ``SSI`` 指令。
注意，绝大多数的配置里包含多个 ``Options`` 指令，这些指令会相互覆盖。
若你想让某个特定目录支持 ``SSI`` 服务器特性，在写 ``Options``  指令的时候，最好保证它在最后执行。

不是所有的文件都需要解析 ``SSI`` 指令，只需要将需要解析 ``SSI`` 指令的文件告诉阿帕奇即可。
有两种方法：

* 让阿帕奇解析任何带有特定后缀的文件，如 ``.shtml``：
  
  .. code-block:: html
  
   AddTytpe text/html .shtml
   AddOutputFilter INCLUDES .shtml

 这种方法的缺点是：如果想给某个网页增加 ``SSI`` 指令，必须改变该网页的名称，以及所有对该网页的链接。
 因为它只支持 ``.shtml`` 后缀的文件。

* 使用 ``XBitHack`` 指令
  
  .. code-block:: html
   
   XBitHack on

  该指令告诉阿帕奇，如果文件存在“可执行的位集合”，则解析该文件的 ``SSI`` 指令。
  这个方法不用修改文件名，只要使用 ``chmod`` 指令修改文件的读写权限即可：
  ``chmod +x pagename.html``

  .. note:: 该方法只在 Unix/Linux 系统中使用

  简单说下禁忌事项。
  你可能会想，只要让阿帕奇对所有的 ``.html`` 文件进行 ``SSI`` 解析即可，不用关心 ``.shtml`` 文件名。
  但对 ``XBitHack`` 也这样做的是行不通的。
  很简单，如果要求阿帕奇对每个传送给客户端的文件都读一遍，不管这些文件中是否有 ``SSI`` 指令，
  则会严重降低服务器的运行效率，因而不推荐这么做。

  Windows 中，在默认配置下，阿帕奇发送 ``SSI`` 网页时，HTTP 头中省略了最近修改日期或者内容长度值，
  因为这些字段是可变的，无法确定。
  这会导致 ``SSI`` 网页不能被浏览器缓存，降低客户端接收性能。

  有两种方法解决这个问题：

  * 使用 ``XBitHack Full`` 配置。
    告诉阿帕奇使用请求文件的原始日期作为最近修改日期，忽略动态修改（Unix/Linux）
  * 使用 ``mod_expires`` 指令显示地设置文件激活时间，使文件能被浏览器和代理服务器缓存
    
基础 ``SSI`` 指令
-----------------

``SSI`` 语法：

.. code-block:: html

 <!--#function attribute=value attribute=value ... -->

格式与 HTML 源码的注释相似，如果 ``SSI`` 格式不对，浏览器会忽略它，但指令在 HTML 源码中仍然可见。
如果格式正确，则会被替换为指令的结果。

* 今天的日期

  .. code-block:: html
  
   <!--#echo var="DATE_LOCAL" -->

  ``echo`` 的作用是将变量值发出来。
  有许多标准变量，包括所有 CGI 程序可用的环境变量。
  你也可以用 ``set`` 函数定义你自己的环境变量。

  如果想改变打印格式，可以通过 ``config`` 函数设置：

  .. code-block:: html
   
   <!--#config timefmt="%A %B %d,  %Y" -->
   Today is <!--#echo var="DATE_LOCAL" -->

* 修改文件日期
  
  .. code-block:: html
   
   This document last modified <!--#flastmod file="index.html" -->

  ``flastmod`` 函数有一个属性，作用是修改并打印最近修改日期，日期格式受 ``config`` 函数的设置控制

* 包含 CGI 程序的结果
   
  这是一种常用方式：

  .. code-block:: html

   <!--#include virtual="/cgi-bin/counter.pl" -->

  ``include`` 函数将程序执行结果包含进来。

更多的例子
----------

* 文档修改时间
  
  前面提到，可以使用 ``SSI`` 给出最近的修改时间：

  .. code-block:: html

   <!--#config timefmt="%A %B %d, %Y" -->
   This file last modified <!--#flastmod file="your_file.shtml" --> 

  是否有更通用的代码，可以粘贴到任何文件中呢？
  使用标准变量就可以了：

  .. code-block:: html

   <!--#config timefmt="%D" -->
   This file last modified <!--#echo var="LAST_MODIFIED" --> 

* 嵌入标准页脚
  
  如果需要管理的网页很多，则对所有网页做修改是很痛苦的；而对所有网页维护一个标准搜索功能更困难。

  加载一个页眉文件或页脚文件可以减轻更新的压力。
  你只要编辑一个页脚文件，然后使用 ``include`` ``SSI`` 命令加载到网页中即可。
  ``include`` 函数可以使用 ``file`` 或者 ``virtual`` 属性加载文件。
  ``file`` 是文件路径，和当前目录有关；
  意味着它不能是绝对文件路径（即不能以 ``/`` 开始），也不能包含 ``../``。
  ``virtual`` 可能更有用处，必须指定一个和加载文档相关的 URL。
  它可以是 ``/`` 开始，但必须和加载的文档处在同一个服务器上。

  例如：``<!--#include virtual="/footer.html">``

  可以把 ``LAST_MODIFIED`` 指令放到页脚文件中。
  ``SSI`` 指令可以放到被加载文件中，加载还可以嵌套。

其他配置
--------

除了可以 ``config`` 时间格式，你可以 ``config`` 其他的：

* 配置 ``SSI`` 指令执行出错信息
  
  .. code-block:: html
  
   <!--#config errmsg="[It appears that you don't know how to use SSI]" -->

* 设置文件大小显示格式 ``sizefmt``:
  
  * ``bytes`` 字节数
  * ``abbrev`` 按照数值大小自动转换单位 ``Kb`` 或者 ``Mb`` 等

执行命令行
----------

Linux:

.. code-block:: html

 <pre>
 <!--#exec cmd="ls" -->
 </pre>

Windows:

.. code-block:: html

 <pre>
 <!--#exec cmd="dir" -->
 </pre>

.. note:: 
 这样做是很危险的，因为打印的内容会嵌入到 HTML 网页中，可能产生错误，导致浏览器无法解析

你可以用 ``includeNOEXEC`` 禁用 ``exec`` ``SSI`` 指令。

更高级的 ``SSI`` 技术
---------------------

除了输出内容，阿帕奇的 ``SSI`` 允许你设置变量，并在比较表达式和条件语句中使用。

* 设置变量
  
  使用 ``set`` ：

  .. code-block:: html

   <!--#set var="name" value="huwz" -->
   <!--#set var="modified" value="$LAST_MODIFIED" -->
   <!--#set var="cost" value="\$100" -->
   <!--#set var="date" value="${DATE_LOCAL}_${DATE_GMT}" -->

* 条件表达式
  
  既然可以使用变量，就可以用变量构造条件表达式，格式如下：

  .. code-block:: html

   <!--#if expr="test_condition" -->
   <!--#elif expr="test_condition" -->
   <!--#else -->
   <!--#endif -->

  例如：

  .. code-block:: html
  
   Good <!--#if expr="%{TIME_HOUR} < 12" -->
   morning!
   <!--#else -->
   afternoon!
   <!--#endif -->

  .. note:: 
   在条件表达式中，获取变量值的写法：``%{varname}``，和 ``set`` 引用变量值不一样