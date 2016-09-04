# PHP扩展开发学习笔记（三）：为对象添加属性
---
在上一篇文章中我们创建了一个简单的class，并且实现了两个简单的方法，但是这个class还没有任何的成员变量，本章我们来尝试为它添加一些属性。 我们希望能用以下的方式来进行实例的创建、成员的赋值和访问。

```php
$obj = new McpackHessianClient('http://www.baidu.com', array('username' => 'Ray'));
echo $obj->getUrl(); // print 'http://www.baidu.com'
```
`McpackHessianClient`的构造方法有两个参数`$url`和`$options`，其中`$url`是必须的，`$options`是可选参数。`McpackHessianClient`还有一个方法`getUrl`返回url的值。

首先我们在mcphessian.c中__定义`McpackHessianClient`及其两个属性`url`和`options`__。`zend_declare_property_xxx`方法可以声明属性，xxx可以是合法的类型，如null、string、bool等。

这里声明的两个属性都是protected的，如果想要定义为private或者public的，需要把`ZEND_ACC_PROTECTED`替换成`ZEND_ACC_PRIVATE`或者`ZEND_ACC_PUBLIC`。值得注意的是private的属性只能在它从属的class_entry中才能访问。代码如下：

```c
PHP_MINIT_FUNCTION(mcphessian)
{
	zend_class_entry ce; 
	INIT_CLASS_ENTRY(ce, "McpackHessianClient", mcphessian_methods);
	mcphessian_ce_ptr = zend_register_internal_class(&ce);
	zend_declare_property_null(mcphessian_ce_ptr, ZEND_STRL("url"), ZEND_ACC_PROTECTED TSRMLS_CC);
	zend_declare_property_null(mcphessian_ce_ptr, ZEND_STRL("options"), ZEND_ACC_PROTECTED TSRMLS_CC);
	return SUCCESS;
}
```
定义好了类和属性之后，我们__定义方法并添加到`zend_function_entry`中去__。需要注意的是属性的赋值我们用了`zend_update_property`这个方法，属性的读取则是`zend_read_property`，这两个方法都有根据类型区分的特定方法，具体可以查询documents。由于options是可选参数，因此这里我们对其进行了初始化成为一个空array，否则需要在赋值之前对其进行为空判断，避免出现segfault。

还有一个需要注意的点是在`PHP_METHOD`中是调用`zend_parse_method_parameters`来获取参数的，在`PHP_FUNCTION`中调用的是`zend_parse_parameters`，其区别在于前者需要传递一个zval作为第一个参数（这里是`pThis`，相当于PHP中的`$this`），并用”O”指定为当前`class_entry`的相同类型。

```c
PHP_METHOD(McpackHessianClient, __construct) {
	zend_class_entry *ce;
	zval *url, *options, *pThis;
 
	MAKE_STD_ZVAL(options);
	array_init(options);
	pThis = getThis();
 
	if (zend_parse_method_parameters(ZEND_NUM_ARGS() TSRMLS_CC, pThis,
            "Oz|z", &ce, mcphessian_ce_ptr, &url, &options) == FAILURE) {
    	return;
	}   
	// set the properties of instance
	zend_update_property(mcphessian_ce_ptr, pThis, ZEND_STRL("url"), url TSRMLS_CC);
	zend_update_property(mcphessian_ce_ptr, pThis, ZEND_STRL("options"), options TSRMLS_CC);

PHP_METHOD(McpackHessianClient, getUrl) {
	zval *url, *pThis;
     
	pThis = getThis();
	url = zend_read_property(mcphessian_ce_ptr, pThis, ZEND_STRL("url"), 1 TSRMLS_CC);
	RETURN_ZVAL(url, 1, 0); 
}

zend_function_entry mcphessian_methods[] = { 
	PHP_ME(McpackHessianClient, __construct, NULL, ZEND_ACC_PUBLIC)
	PHP_ME(McpackHessianClient, getUrl, NULL, ZEND_ACC_PUBLIC)
	{NULL, NULL, NULL}
};
```

至此，`McpackHessianClient`的属性和方法我们就定义完了，make && make install之后用开头的测试代码进行测试吧。
