# PHP扩展开发学习笔记（一）：第一个扩展
---
本系列的文章只为记录本人的PHP扩展开发学习过程，基本上也是按照网络上的教程按部就班，谈不上什么技术或者技巧，只希望能为有相同兴趣的初学者做一些参考。废话不多说，马上开始。

* __准备工作__：下载PHP源代码（路径为PHP_SOURCE），安装PHP环境（路径为PHP_HOME）
* __明确需求__。第一个程序非常简单，实现一个简单的字符串repeat功能，其定义如下：

```c
string repeat(string content, int n)
```

该方法接受两个参数，content是要重复的内容，n是要重复的次数，返回将content重复n次之后的字符串。如：

```php
$str = repeat('abc', 3);
echo $str; // print 'abcabcabc'
```

* __使用ext\_skel工具生成扩展框架__。我为我的第一个程序取名为aya。
  在PHP_SOURCE/ext目录下新建一个文件aya.def，内容为

```c
string repeat(string content, int n)
```

保存退出。下面使用ext_skel生成代码，其中extname是我们的扩展名，proto是要实现的扩展方法的定义。

```shell
cd $PHP_SOURCE/ext
./ext_skel --extname=aya --proto=aya.def
```
执行完之后，ext目录中生成了aya文件夹，其中包含了aya.c,aya.h,config.m4等文件，我们打开config.m4文件，修改一下，去掉下面三行前面的dnl

```ini
dnl PHP_ARG_WITH(aya, for aya support,
dnl Make sure that the comment is aligned:
dnl [ --with-aya Include aya support])
```
变为：

```ini
PHP_ARG_WITH(aya, for aya support,
Make sure that the comment is aligned:
[ --with-aya Include aya support])
```
保存退出。

* __编写函数实现代码__。找到aya.c文件，打开找到repeat方法的定义`PHP_FUNCTION(repeat)`，修改其实现如下，其中`zend_parse_parameters`函数是用于读取参数到本地变量中。

代码如下:

```c
PHP_FUNCTION(repeat)
{
    char *content = NULL;
	int argc = ZEND_NUM_ARGS();
	int content_len;
	long n;
	int ret_len;
	char *ptr;
	char *result;     
	if (zend_parse_parameters(argc TSRMLS_CC, "sl", &content, &content_len, &n) == FAILURE)
		return;
	ret_len = (content_len * n);
	result = (char *) emalloc(ret_len + 1);
	ptr = result;
	while (n--) {
		memcpy(ptr, content, content_len);
		ptr += content_len;
	}
	*ptr = '';
	RETURN_STRINGL(result, ret_len, 0);
}
```

* __编译扩展__。

命令如下：

```shell
cd ext/aya
$PHP_HOME/bin/phpize
./configure --with-php-config=$PHP_HOME/bin/php-config
make && make install
```

编译生成so包后在php.ini文件中添加扩展，使用步骤1中的测试代码测试OK。
