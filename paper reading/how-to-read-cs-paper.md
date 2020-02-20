## [How to Read a CS Research Paper](https://people.cs.pitt.edu/~litman/courses/cs2710/papers/howtoreadacspaper.pdf)

### 论文介绍

1. CS论文可发表为

   - **technical reports** 
   - **conference papers**
   - **journal papers**
   - **book chapters**

   通常作者会将一篇会议论文的信息进行扩展，写成一篇技术报告。几篇会议论文的结果可以合并再扩展成一篇期刊论文。会议论文也可以被扩展，写成书的一个章节。

2. CS论文三种类型

   - theoretical

     描述一个理论或算法，为某个假说提供数学证明。

   - engineering

     介绍某种算法、系统或应用的实现。通常需要包括 对系统的评估。

   - empirical

     描述验证某个假说的一个实验。

3. 好的论文应该有
   - 论文研究的问题陈述清晰，描述了其重要性 以及 更广泛的影响。
   - 对实验、系统或理论的清晰介绍
   - 对实验结果的详细描述与**分析**
   - 作者对 future work 有独到的见解
   - 描述了相关研究，以及正确的引用

------

### 如何读

1. 读 introduction，找到**问题陈述**、**理论重要性**、**广泛影响**。
2. 读 method (experiment/ system description)，提问：
   - 对于文章中的例子 是否能给出反例？
   - 方法是否描述清楚了？能否总结方法的步骤？
   - work是否解决了问题？
   - 方法是否合理？
   - 方法是否客观？

3. 语言问题

> Computer science papers are often written in English by non-native speakers of English. Syntactic errors or awkwardness of phrasing do not indicate that the research is bad; you should try distinguish between the writing style and the research itself. 

----

[Three-Pass Approach](https://web.stanford.edu/class/ee384m/Handouts/HowtoReadPaper.pdf)

1. **第一遍：**（5分钟）

   - 仔细读 title, abstract, and introduction
   - 只读每小节标题
   - 读conclusion
   - 浏览引用文献

   回答5C：

   - **Category**：什么类型的文章？
   - **Context**：基于什么理论？
   - **Correctness**：是否成立？
   - **Contributions**：主要贡献是什么？
   - **Clarity**：写得好吗？

   再决定是否要更深入阅读。

   通常一篇文章 需要在第一遍阅读 就让读者把握大意，五分钟内知道亮度在哪。

2. **第二遍：**（1小时）
   - 仔细看图表
   - 标记没有读过的引用文献，用于更深入阅读

3. **第三遍：**（2-5小时）
   - 可以根据记忆重建文章结构
   - 知道该方法的优势和劣势
   - 指出文章中 隐含的假设、缺少对相关研究的引用

----

### 做文献综述 LITERATURE SURVEY

做文献综述可以检验阅读论文的水平。

1. 使用 [Google Scholar](http://scholar.hedasudi.com/) 或 [CiteSeer](https://citeseerx.ist.psu.edu/) 搜索3-5篇该领域**近期的**论文，运气好可以找到综述。
2. 在文献中，找到 共用的引用、重复出现的作者名；再根据这些信息 下载关键的论文。去作者官网看看最近的进展，同时也可以找到该领域的顶级会议top conferences。
3. 去这些顶级会议的网站，浏览最近的进展。

---

### 相关阅读

[Writing reviews for systems conferences](http://people.inf.ethz.ch/troscoe/pubs/review-writing.pdf)

[Whitesides’ Group: Writing a Paper](http://www.che.iitm.ac.in/misc/dd/writepaper.pdf)

[Research Skills](http://research.microsoft.com/simonpj/Papers/givinga-talk/giving-a-talk.htm)

-----

### 三大顶会

1. **[SIGMOD](http://www.sigmod.org/)**（Special Interest Group On Management Of Data）——美国计算机协会ACM
2. **[VLDB](http://www.vldb.org/)**（Very Large Data Base）——欧洲的数据库会议
3. **[ICDE](http://www.icde.org/)**（International Conference On Data Engineering）——IEEE的数据库会议