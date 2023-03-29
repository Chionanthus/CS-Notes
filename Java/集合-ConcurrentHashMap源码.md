#### 结构

1.7以前是由多个Segment组合，Segment大小初始化了就不能改变

put的时候根据key计算位置，获取指定位置的Segment，如Segment不存在则初始化

1.8后，变成Node+链表+红黑树


#### initTable


#### put

* 根据Key算出hashcode
* 判断是否需要初始化
*
