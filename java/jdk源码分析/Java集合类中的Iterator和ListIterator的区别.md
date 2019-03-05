本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

## Java之集合遍历器分析



最近看到集合类，知道凡是实现了Collection接口的集合类，都有一个Iterator方法，用于返回一个实现了Iterator接口的对象，用于遍历集合；

（Iterator接口定义了3个方法分别是hasNext（），next（），remove（）；）　　

　　我们在使用List,Set的时候，为了实现对其数据的遍历，我们经常使用到了Iterator(迭代器)。使用迭代器，你不需要干涉其遍历的过程，只需要每次取出一个你想要的数据进行处理就可以了。

　　但是在使用的时候也是有不同的。List和Set都有iterator()来取得其迭代器。对List来说，你也可以通过listIterator()取得其迭代器，两种迭代器在有些时候是不能通用的，Iterator和ListIterator主要区别在以下方面：

　　1. iterator()方法在set和list接口中都有定义，但是ListIterator（）仅存在于list接口中（或实现类中）；

　　2. ListIterator有add()方法，可以向List中添加对象，而Iterator不能

　　3. ListIterator和Iterator都有hasNext()和next()方法，可以实现顺序向后遍历，但是ListIterator有hasPrevious()和previous()方法，可以实现逆向（顺序向前）遍历。Iterator就不可以。

　　4. ListIterator可以定位当前的索引位置，nextIndex()和previousIndex()可以实现。Iterator没有此功能。

　　5. 都可实现删除对象，但是ListIterator可以实现对象的修改，set()方法可以实现。Iierator仅能遍历，不能修改。　　

　　因为ListIterator的这些功能，可以实现对LinkedList等List数据结构的操作。其实，数组对象也可以用迭代器来实现。