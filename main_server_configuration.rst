.. _main_server_configure:

主服务器配置
============

主服务器配置，是指针对主服务器的配置。
所有不在 ``<VirtualHost>`` 中解析的请求都交由主服务器解析。
换句话说主服务器就是真实主机，不是由 ``<VirtualHost>`` 定义的虚拟主机。

.. note::
 主服务器中的指令可以在 ``<VirtualHost>`` 中修改。