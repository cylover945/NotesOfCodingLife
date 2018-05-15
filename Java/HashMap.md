# HashMap原理分析

		HashMap作为日常开发中常用的集合类，key-value的方式存储数据，底层通过数组加链表/红黑树的数据结构，使用hashcode和equals方法来实现的。
		俗话说，工欲善其事必先利其器。除了知道如何使用外，更需要了解其实现原理，设计思想。接下来本人基于JDK1.8来分析HashMap的原理。

## 数据结构介绍


![hashmap.jpg](http://47.98.217.105:8081/blog/articles/img?filename=hashmap.jpg)

+ 简单介绍

	HashMap是基于Map接口实现的一种键-值对<key,value>的存储结构，允许null值，无序，不同步(即线程不安全)。HashMap的底层实现是数组 + 链表 + 红黑树（JDK1.8增加了红黑树部分）。它存储和查找数据时，是根据键key的hashCode的值计算出具体的存储位置。HashMap最多只允许一条记录的键key为null，HashMap增删改查等常规操作都有不错的执行效率，是ArrayList和LinkedList等数据结构的一种折中实现。

+ 重要属性

	**int size**;用于记录HashMap实际存储元素的个数；

	**float loadFactor**;负载因子（默认是0.75，表示哈希表空间的利用率）。

		1. 当负载因子越大，则HashMap的装载程度就越高。也就是能容纳更多的元素，
		   元素多了，发生hash碰撞的几率就会加大，从而链表就会拉长，此时的查询效率就会降低。
	
		2. 当负载因子越小，则链表中的数据量就越稀疏，此时会对空间造成浪费，但是此时查询效率高。
	
		3. 我们可以在创建HashMap 时根据实际需要适当地调整load factor 的值；
		   如果程序比较关心空间开销、内存比较紧张，可以适当地增加负载因子；
		   如果程序比较关心时间开销，内存比较宽裕则可以适当的减少负载因子。
		   通常情况下，默认负载因子 (0.75) 在时间和空间成本上寻求一种折衷，程序员无需改变负载因子的值。
	
		4. 因此，如果我们在初始化HashMap时，就预估知道需要装载key-value键值对的容量size，
		   我们可以通过size / load factor 计算出我们需要初始化的容量大小initialCapacity，
  		   这样就可以避免HashMap因为存放的元素达到阈值threshold而频繁调用resize()方法进行扩容。从而保证了较好的性能。

	**int threshold**;下一次扩容时的阈值，达到阈值便会触发扩容机制resize（阈值 threshold = 容器容量 capacity * 负载因子 load factor）。也就是说，在容器定义好容量之后，负载因子越大，所能容纳的键值对元素个数就越多。

	**Node<K,V>[] table**; 底层数组，充当哈希表的作用，用于存储对应hash位置的元素Node<K,V>，此数组长度总是2的N次幂。



## 常用方法原理介绍

+ **hash()**
	
		static final int hash(Object key) {
		        int h;
				//做一次16位右位移异或混合
				//右位移16位，正好是32bit的一半，
				//自己的高半区和低半区做异或，就是为了混合原始哈希码的高位和低位，
				//以此来加大低位的随机性。减少元素之间碰撞的几率。
		        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
		    }

+ **put()**

		/**
		     * put方法实际实现过程
		     * 1.计算新增元素索引 
		     * 2.判断新增元素位置是否发生hash冲突  
		     * 3.发生hash冲突后，修改value/按红黑树插入/按链表插入
			 * 4.链表插入是否会有转化成红黑树的操作 
			 * 5.完成操作后是否需要扩容
			 * 
		     * @param hash 键的hash值
		     * @param key 键
		     * @param value 值
		     * @param onlyIfAbsent 如果是true，如果此key存在value，不执行修改操作
		     * @param evict 如果为false，哈希表在初始化
		     * @return 返回原来的值，如果是新增操作，返回null
		     */
		    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
		                   boolean evict) {
		        //用于记录当前的hash表
		        Node<K,V>[] tab;
		        //用于记录当前的链表节点
		        Node<K,V> p;
		        //n用于记录hash表的长度，i表示当前操作的索引index
		        int n, i;
		        //当前hash表为空时
		        if ((tab = table) == null || (n = tab.length) == 0)
		            //初始化hash表，并把初始化后hash表的长度赋值给n
		            n = (tab = resize()).length;
		        //通过(n-1) & hash获取到当前key所计算出的位置
		        //判断当前位置没有元素存在
		        if ((p = tab[i = (n - 1) & hash]) == null)
		            //新建一个node节点，在该位置存储元素
		            tab[i] = newNode(hash, key, value, null);
		        //如果当前位置有元素存在，说明已经发生了hash冲突
		        else {
		            //用于存放新增节点
		            Node<K,V> e;
		            //用于临时存在某个key值
		            K k;
		            //1.如果当前位置元素hash值与新增元素的hash值相等
		            //2.当前位置元素key与新增元素key也相等，则执行value修改操作
		            if (p.hash == hash &&
		                ((k = p.key) == key || (key != null && key.equals(k))))
		                //将当前节点的引用赋值给e
		                e = p;
		            //如果当前节点是红黑树节点
		            else if (p instanceof TreeNode)
		                //按照红黑树的规则添加节点
		                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		            //排出上述两种情况，说明是在单链表中发生了hash冲突
		            else {
		                //遍历单链表，把新增元素放在此链表最后
		                for (int binCount = 0; ; ++binCount) {
		                    if ((e = p.next) == null) {
		                        //将新节点放到链表最后
		                        p.next = newNode(hash, key, value, null);
		                        //新增节点后，判断当前节点数量是否大于阈值8
		                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
		                            //大于等于8就将链表转化成红黑树
		                            treeifyBin(tab, hash);
		                        break;
		                    }
		                    //如果链表中已经存在key，则修改value
		                    if (e.hash == hash &&
		                        ((k = e.key) == key || (key != null && key.equals(k))))
		                        break;
		                    p = e;
		                }
		            }
		            if (e != null) { // 如果可以存在
		                V oldValue = e.value;
		                //允许修改value，则修改value为新值
		                if (!onlyIfAbsent || oldValue == null)
		                    e.value = value;
		                afterNodeAccess(e);
		                return oldValue;
		            }
		        }
		        ++modCount;
		        //如果当前存储的键值对大于阈值，则扩容
		        if (++size > threshold)
		            resize();
		        afterNodeInsertion(evict);
		        //如果返回null,则是新增操作。
		        return null;
		    }


+ **get()**

		 /**
		     * get方法实际实现
		     * 1.计算后得到索引不为空，判断要找的是否是第一个节点
		     * 2.不是第一个节点时，判断节点什么类型
		     * 3.红黑树节点按红黑树方式查找，单链表节点遍历查找
		     * 4.如果均未查找到，则返回null
		     *
		     * @param hash 键的hash值
		     * @param key 键
		     * @return 查找的节点，如果为空返回null
		     */
		    final Node<K,V> getNode(int hash, Object key) {
		        //用于记录当前的hash表
		        Node<K,V>[] tab;
		        //first是对应hash位置的第一个节点，e是工作节点
		        Node<K,V> first, e;
		        //hash表的长度
		        int n;
		        //存放临时的key
		        K k;
		        //通过(n-1)&hash计算位置
		        //判断当前位置是否有元素存在
		        if ((tab = table) != null && (n = tab.length) > 0 &&
		            (first = tab[(n - 1) & hash]) != null) {
		            //元素存在
		            //如果头节点的key和hash与要获取的相等
		            if (first.hash == hash && // always check first node
		                ((k = first.key) == key || (key != null && key.equals(k))))
		                //则返回头结点
		                return first;
		            //获取的值不是头结点的情况
		            if ((e = first.next) != null) {
		                //如果头结点是红黑树节点
		                if (first instanceof TreeNode)
		                    //按照红黑树的方式查找要获取的节点
		                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
		                do {//当前数据结构是单链表的情况
		                    //遍历查找，key和hash都与要获取节点相等的
		                    if (e.hash == hash &&
		                        ((k = e.key) == key || (key != null && key.equals(k))))
		                        //找到对应节点返回
		                        return e;
		                } while ((e = e.next) != null);
		            }
		        }
		        //按照上述方式均未查找到，则返回null
		        return null;
		    }

+ **remove()**

		/**
		     * remove的实际实现
		     * 1.判断删除节点是否头节点
		     * 2.不是头结点时，判断该节点的类型是红黑树节点还是单链表节点
		     * 3.删除时，节点指向的变化
		     *
		     * @param hash 键的hash
		     * @param key 键
		     * @param value 比较value值，当matchvalue是true时才有效，否则忽略
		     * @param matchValue 如果为true，则只有当value相等才会移除
		     * @param movable 如果是false，当执行删除操作时，不删除其他节点
		     * @return 返回删除的节点，如果节点不存在，则返回null
		     */
		    final Node<K,V> removeNode(int hash, Object key, Object value,
		                               boolean matchValue, boolean movable) {
		        //当前的hash表
		        Node<K,V>[] tab;
		        //当前的链表节点
		        Node<K,V> p;
		        //n表示hash表长度，index表示当前操作索引
		        int n, index;
		        //判断计算得到的位置是否存在元素
		        if ((tab = table) != null && (n = tab.length) > 0 &&
		            (p = tab[index = (n - 1) & hash]) != null) {
		            //node表示找到的节点，e为工作节点
		            Node<K,V> node = null, e;
		            K k;V v;
		            //判断头结点是否是要删除的节点，如果是，就删除
		            if (p.hash == hash &&
		                ((k = p.key) == key || (key != null && key.equals(k))))
		                //记录要删除的节点的引用地址到node中
		                node = p;
		            //如果要删除的不是头结点
		            else if ((e = p.next) != null) {
		                //如果是红黑树节点
		                if (p instanceof TreeNode)
		                    ////记录要删除的节点的引用地址到node中
		                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
		                //如果是单链表的节点
		                else {
		                    do {
		                        if (e.hash == hash &&
		                            ((k = e.key) == key ||
		                             (key != null && key.equals(k)))) {
		                            //找到要删除的节点，记录信息后，中断遍历
		                            node = e;
		                            break;
		                        }
		                        p = e;
		                    } while ((e = e.next) != null);
		                }
		            }
		            //如果找到要删除的节点，则判断是否需要value也一致才删除
		            if (node != null && (!matchValue || (v = node.value) == value ||
		                                 (value != null && value.equals(v)))) {
		                //value一致或者不需要比较value的，则执行删除操作
		                if (node instanceof TreeNode)
		                    //如果是红黑树节点，按红黑树方式删除
		                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
		                //如果删除的是头结点
		                else if (node == p)
		                    //当前存储位置头结点指向被删除节点的下一个节点
		                    tab[index] = node.next;
		                else//删除的不是头节点
		                    //就把被删除节点的前一个节点指向它的后一个节点
		                    p.next = node.next;
		                ++modCount;
		                //当前存储的键值对数量减少1
		                --size;
		                afterNodeRemoval(node);
		                //返回被删除的节点
		                return node;
		            }
		        }
		        return null;
		    }



## hash冲突与解决方法

+ hash冲突

	当我们调用put(K key, V value)操作添加key-value键值对，这个key-value键值对存放在的位置是通过扰动函数(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)计算键key的hash值。随后将 这个hash值 % 模上 哈希表Node<K,V>[] table的长度 得到具体的存放位置。所以put(K key, V value)多个元素，是有可能计算出相同的存放位置。此现象就是hash冲突或者叫hash碰撞。

+ 解决方法：

	开发定址法，再散列法，链地址法，公共溢出区法。

	HashMap是使用<font color=#DC143C size=4>链地址法</font>解决hash冲突的。
	当有冲突元素放进来时，会将此元素插入至此位置链表的最后一位，形成单链表。但是由于是单链表的缘故，每当通过hash % length找到该位置的元素时，均需要从头遍历链表，通过逐一比较hash值，找到对应元素。如果此位置元素过多，造成链表过长，遍历时间会大大增加，最坏情况下的时间复杂度为O(N)，造成查找效率过低。所以当存在位置的链表长度 大于等于 8 时，HashMap会将链表 转变为 红黑树，红黑树最坏情况下的时间复杂度为O(logn)。以此提高查找效率。

## HashMap容量是2的N次方的原因

+ 巧妙的设计

	因为调用put(K key, V value)操作添加key-value键值对时，具体确定此元素的位置是通过 hash值 % 模上 哈希表Node<K,V>[] table的长度 hash % length 计算的。但是"模"运算的消耗相对较大，通过位运算h & (length-1)也可以得到取模后的存放位置，而位运算的运行效率高，但只有length的长度是2的n次方时，h & (length-1) 才等价于 h % length。

+ 效率的考量

	当数组长度为2的n次幂的时候，不同的key算出的index相同的几率较小，那么数据在数组上分布就比较均匀，也就是说碰撞的几率小，相对的，查询的时候就不用遍历某个位置上的链表，这样查询效率也就较高了。



## key值的选取
+ <font color=#0099ff size=4>我们在使用HashMap时，最好选择不可变对象作为key。例如String，Integer等不可变类型。</font>

+ 可变对象：是指该对象在创建后它的哈希值可能被改变。例如自己创建的对象，通过set方法改变了该对象的属性。

+ 如果key对象是可变的，那么key的哈希值就可能改变。在HashMap中可变对象作为Key会造成数据丢失。因为我们再进行hash & (length - 1)取模运算计算位置查找对应元素时，位置可能已经发生改变，导致数据丢失。

## 线程安全性问题
1. **Hashmap与Hashtable**

	1）容器整体结构
	
	HashMap的key和value都允许为null，HashMap遇到key为null的时候，调用putForNullKey方法进行处理，而对value没有处理。

	Hashtable的key和value都不允许为null。Hashtable遇到null，直接返回NullPointerException。
	
	2）容量设定与扩容机制
	
	HashMap默认初始化容量为 16，并且容器容量一定是2的n次方，扩容时，是以原容量 2倍 的方式 进行扩容。
	
	Hashtable默认初始化容量为 11，扩容时，是以原容量 2倍 再加 1的方式进行扩容。即int newCapacity = (oldCapacity << 1) + 1;。
	
	3）散列分布方式（计算存储位置）
	
	HashMap是先将key的hashCode经过扰动函数扰动后得到hash值，然后再利用 hash & (length - 1)的方式代替取模，得到元素的存储位置。
	
	Hashtable则是除留余数法进行计算存储位置的（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算），int index = (hash & 0x7FFFFFFF) % tab.length;。
	
	由于HashMap的容器容量一定是2的n次方，所以能使用hash & (length - 1)的方式代替取模的方式计算元素的位置提高运算效率，但Hashtable的容器容量不一定是2的n次方，所以不能使用此运算方式代替。
	
	4）线程安全
	
	HashMap 不是线程安全，如果想线程安全，可以通过调用synchronizedMap(Map<K,V> m)使其线程安全，但是使用时的运行效率会下降。
	
	Hashtable则是线程安全的，每个操作方法前都有synchronized修饰使其同步，但运行效率也不高。
	
	因此，Hashtable是一个遗留容器，如果我们不需要线程同步，则建议使用HashMap，如果需要线程同步，则建议使用ConcurrentHashMap。

2. **HashMap为什么线程不安全**


	1）fail-fast（快速失败）机制

	如果多个线程同时对同一个HashMap更改数据的话，会导致数据不一致或者数据污染。

	在出现线程不安全的操作时，HashMap会尽可能的抛出ConcurrentModificationException防止数据异常，当我们在对一个HashMap进行遍历时，我们是不能对HashMap进行添加，删除等更改数据的操作的，否则也会抛出ConcurrentModificationException异常。

	从源码上分析，我们在put,remove等更改HashMap数据时，都会导致modCount的改变，当expectedModCount != modCount时，则抛出ConcurrentModificationException。

	2） resize导致死循环
	
	在多线程下操作HashMap，由于存在扩容机制，当HashMap调用resize()进行自动扩容时，可能会导致死循环的发生。

	原因：如果扩容前相邻的两个Entry在扩容后还是分配到相同的table位置上，就会出现死循环的BUG
	https://www.cnblogs.com/dongguacai/p/5599100.html


## 总结

+ HashMap是基于数组+链表/红黑树实现的key-value存储结构；

+ HashMap定位元素位置是通过键key经过扰动函数扰动后得到hash值，然后再通过hash & (length - 1)代替取模的方式进行元素定位；

+ HashMap中比较重要的方法get/put/remove/replace/hash等；

+ HashMap的key最好选择String，Integer等不可变对象；

+ HashMap是线程不安全的，如果需要线程安全，则建议使用ConcurrentHashMap.