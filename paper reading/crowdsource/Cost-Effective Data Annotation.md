## Cost-Effective Data Annotation using Game-Based Crowdsourcing

1. **问题**：现存的数据标注方案，要么对于大数据集的开销很高，要么容易产生噪声结果。

2. **Contribution**: 

   - 提出一种性价比高的标注方法，生成高质量的规则来 在保证label质量的情况下 减少cost。先生成候选规则，再设计CrowdGame 基于**覆盖率**和**准确率**来选出高质量的规则。引入两组crowd worker: 一组作为**rule generator** 负责检验 规则是否有效；另一组作为**rule refuter** 负责检验 一条数据标注的标签是否正确，从而可以得出某些规则的coverage是否足够

   - 引入 规则的损失，来平衡**覆盖率**和**准确率***

   - 以 Entity matching 和 relation extraction作为实验数据集，证明了方法比  state-of-the-art方案 在元组级别众包  tuple-level
     crowdsourcing  以及 基于机器学习的标签规则  ML-based consolidation of labeling rules 上 的优越性。
     
   - 框架：
     
     ![](https://s2.ax1x.com/2020/02/20/3ZNgr4.md.png)

3. **Challenge**: 

   - *coverage* 和 *precision*的tradeoff —— 定义包含两个因素的损失函数 loss
   - *rule precision estimation* —— 使用贝叶斯估计将 **rule validation** 和 **tuple checking** 结合起来 （把rule validation的结果作为先验概率，再通过实际的tuple checking作为后验概率来修正）
   - 选择众包任务 来最小化loss —— 采用 minimax策略，提出了高效的task selection算法。**Rule generator**作为*minimizer*，使loss最小化；**Rule refuter**作为*maximizer*，使检验数据的loss最大化（反复进行这两项，直到众包预算用尽）

   > Minimax算法又名极小化极大算法，是一种找出失败的最大可能性中的最小值的算法。Minimax算法常用于棋类等由两方较量的游戏和程序，这类程序由两个游戏者轮流，每次执行一个步骤。我们众所周知的五子棋、象棋等都属于这类程序，所以说Minimax算法是基于搜索的博弈算法的基础。该算法是一种零总和算法，即一方要在可选的选项中选择将其优势最大化的选择，而另一方则选择令对手优势最小化的方法。

4. 实现细节：
   
   - **Labeling Rule**: 
   
     Labeling Rule 即 将元组 映射为 标签或nil 的一个函数，一个规则 *r<sub>j</sub>*所有映射值不为nil的输入元组集合 即为该规则的 covered set。
   
     使用Labeling Rule来标注 输入的元组 可分为两个阶段：
   
     1. **Crowdsourced rule generation**
   
        旨在生成高质量的规则（覆盖率 和 准确率）。首先构建候选规则，可以通过 hand-crafted(由领域专家生成，对于大数据集不适合) 或 *weak-supervision*(由算法生成，如Distant Supervision) 来构建。
   
        同时 覆盖率 和 准确率之间存在tradeoff：有些场景更需要precision，比如在Entity Matching任务中，正确标签可能偏斜度 skew很大（比如7个元组标签为1，只有1个标签为-1），此时规则就不能有太大误差；另一些场景可能为了减少开支，尽量提高coverage。因此，引入 规则集*R*的损失函数（里面有一个 0≤ *γ* ≤1，作为人为定义的 覆盖率和准确率的preference）。
   
        最终规则的生成 可视为一个最优化问题。
   
     2. **Data annotation using rules**
   
        使用Labeling Rule来标注数据，对于没有被cover的元组，使用传统的众包方法来进行元组级别的标注。
   
   - **Two-pronged task scheme**
   
     双支众包任务方案——先利用crowd直接验证一条规则的有效性，作为粗糙的预估；再通过 基于已验证的规则进行*tuple checking*任务，作为更精确的后验评估。为了达到此目的，提出了*rule validation*任务类型。因为 crowd很可能产生 False Positive（错的当做对的）的误报（因为 当规则不适用时，很容易忽略 negative的例子），所以同样需要 细粒度的tuple checking任务。
   
   - **A game-based crowdsourcing approach**
   
     预算是固定的，所以要平衡 *rule validation*和*tuple checking* 两种类型任务。引入一种基于博弈的众包方法*CROWDAGME*，如框架图中心所示，两组worker，一组负责*rule validation*的称为*RuleGen*，一组负责*tuple checking*的称为*RuleRef*。因此可以视为一个 基于规则集的损失函数的 两方博弈*two-player game*；*RuleGen*作为minimizer，最小化损失；*RuleRef*作为maximizer，最大化损失。
   
     (GAN 对抗式生成网络 在训练时也使用了minimax框架，用于图像和文本处理中)
   
   - **Task selection**
   
     迭代式的众包算法，每轮迭代包含两步：*RuleGen*和*RuleRef*。使用 通用的 *batch mode*，每次选取一批任务，一起放上众包平台。
   
     算法输入为 
   
     - **candidate rules**
     - **tuples to be labeled**
     - **budget**
     - **crowdsourcing batch**
   
     输出为 a set of generated rules 生成规则集。
   
   - **Candidate Rules for Entity Matching**
   
     构造 candidate *blocking rules*，目前大部分方法是基于结构化的数据。提出方法 自动识别 关键词对*keyword pair*，可以有效地 从未处理文本中分别出 *record pairs*。
   
     通过 词移距离*Word Mover’s Distance* (基于NLP中的词嵌入，词映射为一个词向量，在这个向量空间中，语义相似的词之间距离会比较小)，从元组对中 可识别出 关键词对。
   
   - **Candidate Rules for Relation Extraction** 
   
     使用 *Distant Supervision*来找到规则，使用 point-wise mutual information 来决定两个连续的词能否形成一个词组。
   
   - 实验的指标
     - rule coverage
     - FN rate
     - precision
     - recall
     - F1 score
     - crowd cost 