# PHP扩展开发学习笔记（五）：HashTable & Array及其他
---
这个迷你的扩展我已经post到我的[github](https://github.com/Raaaay/mcphessian)上了，当然我还会根据实际工作中的使用情况做一些定制和完善，但是基本的功能已经实现了。现在我们来看一下剩下的部分的实现。

其实我的前几篇博客基本上都把这个扩展所需的东西讲得差不多了，现在来看看具体的逻辑实现吧。

```c
static zval *mcpack2array(zval *data) {
	zval *params[] = { data };
	zval *function_name, *retval_ptr;
 
	MAKE_STD_ZVAL(retval_ptr);
	MAKE_STD_ZVAL(function_name);
	ZVAL_STRING(function_name, "mc_pack_pack2array", 0);
	if (call_user_function(CG(function_table), NULL, function_name, 
		retval_ptr, 1, params TSRMLS_CC) == SUCCESS) {
	} else {
		php_error(E_WARNING, "call function mc_pack_pack2array fail.");
	}
	return retval_ptr;
}

static zval *array2mcpack(zval *data) {
	if (Z_TYPE_P(data) != IS_ARRAY) {
	    php_error(E_ERROR, "parameter type should be array.");
	}
 
	zval *mc_pack_v;
	MAKE_STD_ZVAL(mc_pack_v);
	ZVAL_STRING(mc_pack_v, "PHP_MC_PACK_V2", 0);
	zval *params[] = { data, mc_pack_v };
	zval *function_name, *retval_ptr;
	MAKE_STD_ZVAL(function_name);
	ZVAL_STRING(function_name, "mc_pack_array2pack", 0);
	MAKE_STD_ZVAL(retval_ptr);
 	
	if (call_user_function(CG(function_table), NULL, function_name, 
	    retval_ptr, 1, params TSRMLS_CC) == SUCCESS) {
	} else {
	    php_error(E_WARNING, "call function mc_pack_array2pack fail.");
	}
	return retval_ptr;
}

/**
 * override __call()
 * it require two parameters, func_name and args
 **/
PHP_METHOD(McpackHessianClient, __call) {
	zend_class_entry *ce;
	zval *p_this, *args, *params, *result, *method, *tmp;
	char *func_name, *ret_str = NULL;
	int func_name_len = 0;
	size_t return_len = 0, max_len = 1024 * 1024 * 1024;
 
	p_this = getThis();
	if (zend_parse_method_parameters(ZEND_NUM_ARGS() TSRMLS_CC, p_this, 
	    "Osz", &ce, mcphessian_ce_ptr, &func_name, &func_name_len, &args) == FAILURE) {
	    php_error(E_ERROR, "parse parameters error.");
	    return;
	}
 
	// init params
	array_init(params);
	add_assoc_string(params, "jsonrpc", "2.0", 0);
	add_assoc_string(params, "method", func_name, 0);
	add_assoc_zval(params, "params", args);
	add_assoc_string(params, "id", "123456", 0);
	zval *pack = array2mcpack(params);
 
	// post data
	zval *z_url = zend_read_property(mcphessian_ce_ptr, p_this, ZEND_STRL	("url"), 1 TSRMLS_CC);
	convert_to_string(z_url);
	char *url = Z_STRVAL_P(z_url);
	php_stream_context *context = php_stream_context_alloc();
	MAKE_STD_ZVAL(method);
	ZVAL_STRING(method, "POST", 0);
	php_stream_context_set_option(context, "http", "method", method);
	php_stream_context_set_option(context, "http", "content", pack);
 
	// read data from stream
	php_stream *stream = php_stream_open_wrapper_ex(url, "rb", REPORT_ERRORS, 	NULL, context);
	ret_str = php_stream_get_line(stream, NULL, max_len, &return_len);
	MAKE_STD_ZVAL(tmp);
	ZVAL_STRINGL(tmp, ret_str, return_len, 0);
	result = mcpack2array(tmp);
	php_stream_close(stream);
	efree(ret_str);
 
	// get result from array
	zval **ret_val;
	if (zend_hash_exists(Z_ARRVAL_P(result), "result", 7)) {
	    zend_hash_find(Z_ARRVAL_P(result), "result", 7, (void **)&ret_val);
	    RETURN_ZVAL(*ret_val, 1, 0);
	} else {
	    php_error(E_ERROR, "return value illegal.");
	    RETURN_NULL();
	}
}
```

关于`mc_pack`的实现部分，我是用的厂内现成的实现（即`mc_pack_array2pack`和`mc_pack_pack2array`两个方法），在这里需要提一下如何调用用户函数。`call_user_function`即是用来做这个事的，它可以调用用户的其他实现方法，其函数定义如下

```c
int call_user_function(HashTable *function_table, zval **object_pp,
    zval *function_name, zval *retval_ptr,
    zend_uint param_count, zval *params[] TSRMLS_DC);
```

它把方法的返回结果放在了`retval_ptr`中，`function_table`是方法列表，一般来说用`CG(function_table)`就可以了，不是面向对象方法的话，`object_pp`传NULL就好；`function_name`是调用的方法名，传递的参数和参数列表则是在最后。

为了达到代理的作用，我们覆盖了`__call()`这个魔术方法，以保证正确的拿到调用的方法名和参数。

下面来介绍一下如何初始化一个数组并给它赋值。数组的初始化用`array_init`方法，向其中添加键值对的话有三个方法可以使用，分别是`add_assoc_*()`,`add_index_*()`和`add_next_index_*()`，其含义也很简单，分别代表了添加关联数组，索引数组以及按照索引顺序添加，后面的*是合法的类型，如`long`,`double`,`string`等。更详细的介绍请查询文档。

HashTable则是array的真正数据结构，所有HashTable上的方法都可以用在array上，如我这里用到的检查一个键是否存在`zend_hash_exists`以及获得某个键对应的值`zend_hash_find`。值得注意的是，这里key是包含了string后的”的，所以计算长度的时候记得要加1，否则会找不到该键。`zend_hash_find`的结果值会放在一个`(void **)`中。

与前一篇不同，这里我对流的读取用了`php_stream_get_line`，它相对于`php_stream_read`来说简单了不少，也更不容易出错。它的第2个参数是buffer数组，如果传入是NULL的话，则系统会为它动态分配一个。
编译后，执行一下测试代码（由于厂外没有mc\_pack包，所以没法使用了，呵呵）

```php
// Assuming we had a service http://www.baidu.com/api, 
// and had a method named proxyMethod which requires an array as it's parameter.
$obj = new McpackHessianClient('http://www.baidu.com/api');
$ret = $obj->proxyMethod(array('chrhust'));
var_dump($ret);
```
