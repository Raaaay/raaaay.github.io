# Nginx中的location匹配顺序
---
近年来Nginx越来越流行，以前的LAMP架构很多都转向了LNMP架构，对高并发的高效支持，灵活的配置形式以及对扩展开发的支持，都使我们有足够的理由抛弃Apache投入nginx的怀抱。

location作为http模块中的一个重要部分，其配置方法是每个nginx使用者都必须要掌握的，但是当我们的Request URI同时匹配到了多个location之后，nginx会选择哪一个呢？其中的匹配方式会成为许多初学者的迷惑。本文只不过是把官方文档中的相关部分摘录下来，供大家参考。

首先，location中的modifier（不知道该如何翻译）有两种类型：

1. 前缀字符串（prefix string）匹配，其中又有完全匹配（=）和前缀匹配（^~）之分。
2. 正则表达式（regular expression）匹配，正则分为大小写敏感（~）和大小写不敏感（~*）两种。

当nginx为一个Request寻找配对的location时，它会

1. 首先在前缀字符串匹配的location中找到匹配的location，如果有完全匹配的，立即结束查找并返回；否则记录下前缀匹配长度最长的那个。
2. 然后在正则表达式的location中找，查找的顺序是它们在配置文件中定义的顺序，一旦找到匹配的则立即结束查找并返回。
3. 若在正则表达式中没有找到匹配的，则返回在步骤1中记录下的那个location。

**注：没有指定modifier的location优先级低于完全匹配但高于前缀匹配。**

下面来看一个例子：

```nginx
location ~ /test/ {
	more_set_headers "Content-Type: text/html; charset=utf-8";
	return 200 'Regular Expression case sensitive.';
}
 
location ~* /test/ {
	more_set_headers "Content-Type: text/html; charset=utf-8";
	return 200 'Regular Expression case insensitive.';
}
 
location /testmore/ {
	more_set_headers "Content-Type: text/html; charset=utf-8";
	return 200 'No Modifier.';
}
 
location = /test/ { 
	more_set_headers "Content-Type: text/html; charset=utf-8";
	return 200 'Exact Match.';
}
 
location ^~ /test {
	more_set_headers "Content-Type: text/html; charset=utf-8";
	return 200 'Prefix String.';
}
```

当我们访问***http://hostname/test/***时，返回”Exact Match.”；

当我们访问***http://hostname/test/more***时，返回”Regular Expression case sensitive.”；

当我们访问***http://hostname/TEST/MORE***时，返回”Regular Expression case insensitive.”；

当我们访问***http://hostname/testmore***时，返回”Prefix String.”；

当我们访问***http://hostname/testmore/***时，返回”No Modifier.”；
