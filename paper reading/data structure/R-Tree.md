## R-TREES: A DYNAMIC INDEX STRUCTURE FOR SPATIAL SEARCHING 

1. **问题**：空间数据对象 通常覆盖多维空间的一片区域，用点坐标来表示不是很好。比如常用的操作——搜索一个区域内的所有object，这些都需要能高效地获取到。而传统的索引，如hash(不能范围搜索)、B-tree和ISAM(空间是多维的)都不适用空间多维数据。

2. R树（R-tree）是一种将Ｂ树扩展到多维情况下得到的数据结构。

   R树表示由二维或者更高维区域组成的数据，我们把它们称为数据区。一个R树的 **内部结点** 对应于 某个**内部区域**，或称“区域”。原则上，区域可以是任意形状，虽然实际中它经常为矩形或其他简单形状。

   ![R树示例](https://img-blog.csdnimg.cn/20190504075541577.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9iYWltYWZ1amluamkuYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)

   上图（a）中的 内部结点(R3, R4, R5)就代表（b）中的一个区域，被包含在区域R1中，而R8, R9, R10就是空间数据对象。其中，区域被称为***Bounding Box*** (一般是矩形，可以不是矩形)。我们可以通过 一个object的***Minimum Bounding Box***来表示该object。

3. 示例：

   ![想表示的空间数据](http://www.mathcs.emory.edu/~cheung/Courses/554/Syllabus/3-index/FIGS/R-tree/R-tree05.gif)

   对于*house1*, *road1*, *road2*，它们被完全包在Bounding Box((0, 0), (60, 50))中。

   ![](http://www.mathcs.emory.edu/~cheung/Courses/554/Syllabus/3-index/FIGS/R-tree/R-tree05d.gif)

   注意：结点中的**bounding box**可以重叠 

4. 射线法：

    地理围栏一般是多边形，如何判断点在多边形内部呢？可以通过射线法来判断点是否在多边形内部。从该点出发沿着X轴画一条射线，依次判断该射线与每条边的交点，并统计交点个数，如果交点数为奇数，则在多边形内部；如果交点数是偶数，则在外部。