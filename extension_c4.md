# PHP扩展开发学习笔记（四）：流
---
Stream是个很重要的概念，PHP扩展中的Stream layer可以使你进行透明的使用一些内置的包装器，比如说HTTP，FTP以及GZIP等，所以你无需再去实现这些协议。在扩展中进行流操作也很方便，zend提供了一系列的stream操作函数可供调用，如`php_stream_open_wrapper`、`php_stream_read`、`php_stream_write`等。

以下是我们实现的一个HTTP POST请求方法（Hessian是基于HTTP协议的），用在我们的HessianClient中。这个方法postData接受一个字符串参数，形如”key1=value1&key2=value”，并将该数据通过POST方法发送到指定接口上(url)，并返回接口数据。

```c
PHP_METHOD(McpackHessianClient, postData) {
    zend_class_entry *ce;
    zval *p_this, *z_url, *method, *content, *result;
    php_stream *stream;
    php_stream_context *context = NULL;
    p_this = getThis();
 
    if (zend_parse_method_parameters(ZEND_NUM_ARGS() TSRMLS_CC, p_this,
	    "Oz", &ce, mcphessian_ce_ptr, &content) == FAILURE) {
    return;
    }   
 
    p_this = getThis();
    z_url = zend_read_property(mcphessian_ce_ptr, p_this, ZEND_STRL("url"), 1 TSRMLS_CC);
    convert_to_string(z_url);
    char *url = Z_STRVAL_P(z_url);
 
    // set options of request
    context = php_stream_context_alloc();
    MAKE_STD_ZVAL(method);
    ZVAL_STRING(method, "POST", 0); 
    php_stream_context_set_option(context, "http", "method", method);
    php_stream_context_set_option(context, "http", "content", content);
    // read data from stream
    stream = php_stream_open_wrapper_ex(url, "rb", REPORT_ERRORS, NULL, context);
    if (stream) {
	    size_t read_len = 0, step_size = 256;
	    char *tmp = (char *) emalloc(step_size);
	    size_t total = sizeof(tmp);
	    char *ptr = tmp;
	    while (!php_stream_eof(stream)) {
		    char buffer[step_size];
		    size_t len = php_stream_read(stream, buffer, step_size);
		    if (len > 0) {
			    if (len > (total - read_len)) {
				    total += step_size;
				    tmp = (char *) erealloc(tmp, total);
				    ptr = tmp + read_len;
			    }   
			    memcpy(ptr, buffer, len);
			    ptr += len;
			    read_len += len;
		    } else {
			    break;
		    }   
	    }   
	    *ptr = '\0';
	    MAKE_STD_ZVAL(result);
	    ZVAL_STRING(result, tmp, 0);
	    RETURN_ZVAL(result, 1, 0);
	    php_stream_close(stream);
    }
    RETURN_NULL();
}
```
在这里我们从stream中打开stream用的是`php_stream_open_wrapper_ex`这个方法，相对于`php_stream_open_wrapper`而言，前者多了一个context参数，可以方便我们传递上下文。

context是一个`php_stream_context`对象，通过`php_stream_context_alloc`方法生成，我们可以调用`php_stream_context_set_option`方法进行上下文设定，如http的method、content(表单数据)、header等。在最后记得使用`php_stream_close`方法关闭stream。不过令人费解的是，不要在最后使用`php_stream_context_free`方法来free掉context，因为会导致segfault。

流数据的读取我们用的是`php_stream_read`方法，该方法返回的是读入buffer的数据长度，当然，我们也可以用另外的方法如`php_stream_gets`等来进行读取。

编译后，假定我们已经有一个server接收POST请求，并返回一些数据，如

```php
function service() {
	echo $_POST['name'];
}
```

使用以下代码进行测试：
​    
```php
$mcpack = new McpackHessianClient("http://host/api");
echo $mcpack->postData('name=chrhust&password=123456'); // print 'chrhust'
```

