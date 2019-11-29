# 小叙

在之前文章中最后的结尾，我贴了一段代码，不知道有没有师傅遇上了那种情况，因为这其中影响它出现的因素有很多，略微的变化有可能输出的结果就截然不同，为了更好的解释这其中的为什么，我重新从php5的编译器看起，因为之前我一直看的是php7，而php5和php7有很多不一样，比如底层zval结构的变化，php7语法分析过程中生成间接的语法树，执行器handler调度的多种方式，为了让写出来的东西更加的严谨和细致，我看了很多关于本文之外php5其他方面的东西，这是本文迟迟没有写的原因。

回到本文主题，确实这里算是一块php内核中比较庞大的一块内容，从文章的题目也可以猜测到后面的一切内容都关系到了内存，废话不多说进入正题。

# 简要分析
我把之前遗留问题的代码贴上来：
```php
$a="phpinfo();"; 
$b=$a; 
$serialized_string = 'a:1:{i:1;C:11:"ArrayObject":37:{x:i:0;a:2:{i:1;R:4;i:2;r:1;};m:a:0:{}}}'; 
$outer_array = unserialize($serialized_string); 
gc_collect_cycles(); 
$filler1 = "aaaa"; 
$filler2 = &$a; 
var_dump($outer_array);
```
这里逻辑上是完全没有问题的，`$filler2 = &$a; `这一步的分裂过程应该是可以拿到`$outer_array `指向已经被释放的zval结构大小的内存。但是奇怪的是输出`$outer_array`还是NULL。

为了更好的让师傅了解整个过程，讲一下之前没有提到的，为什么还存在`$filler1 = "aaaa"`这个赋值过程，在语法分析过程中，通常存在着一个叫三地址中间代码，什么叫三地址呢？比如：`x = y op z` ,其中op是一个二目运算符， y 和 z是运算分量的地址，也是我们经常说的曹操作数，而x是运算结果存放的地址。这种三地址指令最多只执行一个运算，操作对象也是最基础的地址，叫三地址其实并非会完全用到x y z三个地址，但是至少有一个。

我们来看`$outer_array = unserialize($serialized_string);`这一步我们尝试用三地址代码思想来分解一下：
```php
send_var $serialized_string   //第一步函数传参: op=send_var,y=$serialized_string
$tmp = do_fcall 'unserialize' //第二步函数调用: op=do_fcall,x=$tmp,y='unserialize'
$outer_array assign $tmp      //第三步讲函数返回值的赋值：op=assign,y=$outer_array,z=$tmp
```
现在用三地址代码将原php代码分解为了3行，其中多出了一个临时变量`$tmp`用来保存函数的返回值，这个`$tmp`也是指向一块zval的结构，所以在第三步完成赋值以后，会将其释放掉，放到内存池，

我们来看一下`gc_collect_cycles();`这一步如果用三地址代码来分解的话：
```php
do_fcall 'gc_collect_cycles'
```
这里看起来应该算个单目运算符，因为在php里面有一个执行栈，配合`send_var` opcode来完成参数调用的。但是在php的函数调用里面，无论是否会用到他们的返回值，这个时候都会先初始化一个`$tmp`,用来保存函数返回值，如果该返回值并没有用到的话，再进行释放，`$tmp`保存结构同样是zval，所以这里会释放掉一个zval大小的内存。
```php
$tmp = do_fcall 'gc_collect_cycles'
```
这里释放掉`$tmp`以后zval内存块就刚好'盖'在原先gc释放以外释放掉的$out_array上面。这里用了一个比较形象的'盖'字，所以这里我们需要先把`$tmp`拿走，再申请zval的时候就可以拿到$out_array指向的zval。

出现这种奇怪的现象，第一感觉可能在复制分裂申请zval之前，$out_array这个目标zval已经被拿走了，那么是被'谁'拿走了呢?

# 具体分析

在我拿gdb调之前，我做了一些细微操作，$out_array却正常输出了，比如新增一条语句：`$randman="unit"` 或者把上面的`var_dump`换成`echo`,都能正常输出，是不是感觉非常的不可思议，并且难以理解。

我们进一步缩小问题，当我把简单把`var_dump`换成了`echo`, 这是最后一步肯定对面执行过程没有干扰的，但是还是影响了输出，如果说执行过程没有影响，那么最后的不同地方就发生在php代码的编译阶段！

现在我们需要确保一下`$out_array`释放伴随着gc正常释放了，这时候就需要用gdb来动态调了，第一次断在`gc_collect_cycles`其中的：
```c
/* Free zvals */
		p = GC_G(free_list);
		while (p != FREE_LIST_END) {
			q = p->u.next;
			FREE_ZVAL_EX(&p->z);
			p = q;
		}
```
首先我们确保释放顺序：
```c
GC buffer content:
[0x7ffff7c828b0] (refcount=2) array(1): {
    1 => [0x7ffff7c82aa8] (refcount=1) object(ArrayObject) #1 //ArrayObject
  } //outer_array
[0x7ffff7c818e0] (refcount=2,is_ref) array(2): {
    1 => [0x7ffff7c818e0] (refcount=2,is_ref) array(2): 
    2 => [0x7ffff7c828b0] (refcount=2) array(1): 
  }//inner_array
```
这里的释放顺序应该是inner_array ，ArrayObject，outer_array. 关于这里的过程可以去看gc遍历结点的过程，也可以参考我之前的文章，在这里不累述。

确保了释放的过程，再来看之后的分配过程，这里可以在`_emalloc`下个断，预想其过程，`$filler1= 'aaaa'`拿走了`gc_collect_cycles`释放的`$tmp`, 在这过程gdb 一直continue即可，第二个zval的申请确实发生在了引用复制分裂的过程中，但是申请到的内存却是之前ArrayObject对应的zval，这过程中也没有其他操作申请了zval，可out_array对应的zval去哪了呢？

之前dumpgc里面还有outer_array的地址，查看其内容，这里有一个tip，因为我这里使用的php-5.6.20开了debug，开了debug之后，默认把内存保护也开了，内存保护会把释放掉的内存块清零，所以这里我肯定它还在内存池里面。

为什么它还躺在内存池里面呢？引入今天的主角php的内存管理


# php内存管理

玩PWN的师傅都会非常熟悉glibc里面ptmalloc内存模型，同样php里面也有自己的一套内存管理与ptmalloc有些不一样，它和google的tcmalloc是想对应的。这里我画了一张图，我会用图来描述整个过程，拒绝贴代码！

![enter image description here](https://s4.aconvert.com/convert/p3r68-cdx67/b6crn-0wk8n.png)
可以看到管理整个内存是一个zend_mm_heap结构，初始化内存可以通过malloc或者mmap来完成，其申请的粒度zend_mm_heap->block来决定的，也就是每次向系统申请的内存大小是zend_mm_heap->block的整数倍

每次向系统申请的内存，都会通过一个zend_mm_segment结构来管理，向系统的申请内存我们称之为segment段，想系统申请的segments，zend_mm_heap->segments_list这个字段组成了一个链表。可以看到segment起始位置保存着zend_mm_segment的结构，而后的位置被分成了一个又一个的block块，看绿箭头的位置，这里blocks来自于4种不同的结构上。我们就这四个结构来分别描述一下：

1.cache:
cache是第一层的缓存结构，可以看到长度是ZEND_MM_NUM_BUCKETS,上面还有一串宏就是计算不同大小的内存块对应的cache里面的index。
cache里面内存块可用的大小 为`8 * index`，最小是可以为0，而最大是`8*(ZEND_MM_NUM_BUCKETS-1)`，这里为什么有一个0呢，因为这里
