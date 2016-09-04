# PHP中的foreach谜题及其他
---
说是谜题其实谈不上，只是这个问题个人觉得其实是有点意思的，所以在这里贴一下，对于了解foreach的机制及使用场景还是有些帮助的。下面先来说说问题吧，看以下的代码。
```php
$arr = array('a', 'b', 'c');
foreach ($arr as $item) {
	echo current($arr); // print what here?
	next($arr);
}
```

问题很简单，以上这段代码会输出什么？我们当然期望输出’abc’，可似乎代码并没有按照我们预期的输出，结果是’bc’，当然，任何一个正常的程序员都不会写出这样的代码，但是，我们还是要问，why？
先来看看foreach对一个Iterator对象的执行顺序（来自[官方文档](http://php.net/manual/en/class.iterator.php)）：
> Order of operations when using a foreach loop:
>
> 1. Before the first iteration of the loop, Iterator::rewind() is called.
>
> 2. Before each iteration of the loop, Iterator::valid() is called.
>
> 3. It Iterator::valid() returns false, the loop is terminated.
>    If Iterator::valid() returns true, Iterator::current() and
>    Iterator::key() are called.
>
> 4. The loop body is evaluated.
>
> 5. After each iteration of the loop, Iterator::next() is called and we repeat from step 2 above.

这并不能解释我们的疑问，因为a不见了，为什么第一个输出的不是a而是b呢？毕竟调用next()是在一次iteration的结束而不是开始。
仔细查看一下官方文档中对于foreach的一个[Note](http://php.net/manual/en/control-structures.foreach.php)：
> When foreach first starts executing, the internal array pointer is automatically reset to the first element of the array. This means that you do not need to call reset() before a foreach loop.
> As foreach relies on the internal array pointer, changing it within the loop may lead to unexpected behavior.

还有另外一段：
> Unless the array is referenced, foreach operates on a copy of the specified array and not the array itself. foreach has some side effects on the array pointer. Don’t rely on the array pointer during or after the foreach without resetting it.

至此终于明白，foreach操作的只是$arr的副本，并且由于它的**side effects on the array pointer**，导致了我们的程序出现了非预期的结果。

最后再说一遍，**永远不要写这样危险的代码**。
