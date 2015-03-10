指令属性
========

Context
-------

**应用场合**

``Context`` 属性表明指令放在配置文件的什么位置才是合法的。
该属性由以下列出的一个或多个值组合而成，值与值之间用逗号隔开。

* server config
  
  指令可以放在配置文件除 ``<VirtualHost>`` 或 ``<Directory>`` 外的任意位置。
  ``.htaccess`` 文件不允许使用。

* virtual host

  指令可用于 ``<VirtualHost>``。

* directory
  
  指令可用于 ``<Directory>``, ``<Location>``, ``<Files>``, ``<If>`` 以及 ``<Proxy>``。

* .htaccess

  表明指令可以用在每个目录下的 ``.htaccess`` 文件中。
  指令是否被处理，取决于配置是否覆盖。

Override
--------

**覆盖**

``Override`` 属性指明必须覆盖某些配置，才能使出现在 ``.htaccess`` 中的该指令得到处理。
如果 ``context`` 属性不允许指令出现在 ``.htaccess`` 文件中，则 ``context`` 列表为空。

配置覆盖行为由指令 ``AllowOverride`` 指令激活，并应用于某个范围（例如一个目录）及所有子孙；
除非被其他更低一级的 ``AllowOverride`` 指令修改了。
同样，它们的文档说明也会列出可能的覆盖名称。