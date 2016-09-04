# PHP扩展开发学习笔记（二）：面向对象初步
---
OO是PHP中的重要特性，因此知道如何创建一个类对于要做面向对象的扩展开发来说，是必须的。由于工作上经常需要接触到基于Hessian+Mcpack的接口形式，因此决定先做一个基于扩展的`McpackHessianClient`对象作为练手(Mcpack是厂内使用的一个简单的对象二进制序列化协议)。平时使用接口的形式基本上是这样的：

```php
$proxy = new McpackHessianClient('http://127.0.0.1/xxx.php', $arrOptions);
$result = $proxy->xxxMethod();
```
接下来几篇笔记将围绕着扩展开发一个McpackHessianClient，用以替代基于php开发的客户端来进行学习。

__本次要创建的扩展名称叫mcphessian__，首先使用`ext_skel -extname=mcphessian`创建扩展框架。
打开mcphessian.c文件，在`PHP_MINIT_FUNCTION`前添加以下代码：

```c
PHP_METHOD(McpackHessianClient, sayHello) {
	php_printf("hello world\n");
}

PHP_METHOD(McpackHessianClient, __construct) {
	php_printf("constructing...\n");
}
 
zend_function_entry mcphessian_methods[] = {
	PHP_ME(McpackHessianClient, __construct, NULL, ZEND_ACC_PUBLIC)
	PHP_ME(McpackHessianClient, sayHello, NULL, ZEND_ACC_PUBLIC)
	{NULL, NULL, NULL}
};
```
其中`PHP_METHOD`宏用于定义一个类方法，其接受两个参数类名和方法名。`PHP_ME`用于将类方法添加到类的方法列表中，接受四个参数，分别是类名、方法名、`ARG_INFO`和可见范围（可以是`ZEND_ACC_PUBLIC`、`ZEND_ADD_PROTECTED`、`ZEND_ACC_PRIVATE`）。在这里我们定义了两个类方法`__construct`和`sayHello`，它们都不需要任何参数，仅仅在执行的时候打印出对应的字符串。

__在`PHP_MINIT_FUNCTION`中加入以下代码定义我们的class__：

```c
zend_class_entry ce;
INIT_CLASS_ENTRY(ce, "McpackHessianClient", mcphessian_methods);
mcphessian_ce_ptr = zend_register_internal_class(&ce);
```
其中`INIT_CLASS_ENTRY`宏用于定义我们的class入口。在这里我们定义了一个名叫`McpackHessianClient`的class，该class没有任何的成员变量，拥有两个类方法`__construct`和`sayHello`，这两个方法非常简单，不需要任何的参数，仅仅打印对应的字符串。

至此，__我们简单的的class定义就完成了__。现在进行make和make install，添加完so包之后，执行以下的测试代码吧。

```php
$mcpack = new McpackHessianClient(); // print "constructing..."
$mcpack->sayHello(); // print "hello world"
```
