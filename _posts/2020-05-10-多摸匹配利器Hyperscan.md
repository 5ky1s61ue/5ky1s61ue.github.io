---
layout: post
author: oy
tag: Hyperscan Regex NDA DFA 多模匹配 SIMD
---
## 铺垫知识

### Regex
>
正则表达式有主流的两套规则，分别为POSIX(BRE + ERE)和PCRE(Perl)，它们在定义正则表达式的规范时有些细微的差别比如POSIX是采用最左最长匹配而PCRE则是在找到第一个合法匹配后就停止，它们支持的字符组也不同。
>
* 匹配原理
	1. 从目标字符串起始位置尝试匹配。
>
	2. 如果当前位置尝试匹配失败，就从下一个字符位置重新尝试，直到匹配成功或完成所有匹配尝试。
>
>
* 回溯型语法
	* 回溯法是暴力搜索法的一种，其采用试错的思想，尝试分步地去解决一个问题，通过尝试发现现有的分步答案不能得到有效正确的解答时，它将取消上一步甚至上几步的计算，再通过其它可能的分步解答再次尝试。
>
	* 通常，回溯法采用最简单的递归方式来实现。
>
	* 在正则表达式的语法环境中，一些限定符号的使用如*,?,+,{n,m}在匹配失败时容易产生回溯。
>
>
* 固化分组：
>
	* 匹配进行到此结构后，此结构的备用状态都会被抛弃，可利用它抛弃不必要的备选分支，提高匹配效率。
>
>
* 匹配模式：
>
	* 贪婪，此模式下会根据前导字符去匹配尽量多的内容，在匹配失败时容易产生回溯。一般字符后限定符号?、*、+、{n}、{n,}、{n,m}
>
	* 懒惰，此模式下尽量少地重复匹配字符，匹配成功后会继续匹配剩余字符。在上述限定符号后加?
>
	* 独占，此模式下会尽量完成最长字符串匹配，匹配不成功就结束而不会回溯。在上述限定符后加+
>
>
* 用于分析正则匹配过程的工具：
>
	* RegexBuddy
>
	* regexper.com
	
	
### NFA & DFA
>
* 联系  
>
	它们都属于FSA，任何一个给定的NFA都可以构造成一个等价的DFA，但某些情况可能导致DFA大小会指数级增大。
>
>
* 区别
	* NFA  
		表达式主导，尝试找到正则表达式能匹配的文本的一部分。
		NFA最重要的性质是支持`回溯`
		匹配速度与正则表达式直接相关
>
	* DFA  
		文本主导，尝试找到与文本匹配的表达式分支。
		DFA不支持捕获型括号和回溯
		无须过多优化，本身已具备高效的匹配效率。
>
* Glushkov NFA & Thompson NFA
	* Glushkov算法构造NFA能解决Thompson算法引入的空状态及空转移问题，具体细节挺复杂，还需要重新看看编译原理中的相关内容和Glushkov算法论文`https://cs.nyu.edu/~mohri/pub/glush.pdf`


### Multi-String Matching & Multi-Pattern Matching
>
1. Multi-String Matching:
	* Bitap Algorithm  
		The bitap algorithm (also known as the shift-or, shift-and or Baeza-Yates–Gonnet algorithm) is an approximate string matching algorithm. The algorithm tells whether a given text contains a substring which is "approximately equal" to a given pattern, where approximate equality is defined in terms of Levenshtein distance – if the substring and pattern are within a given distance k of each other, then the algorithm considers them equal. The algorithm begins by precomputing a set of bitmasks containing one bit for each element of the pattern. Then it is able to do most of the work with bitwise operations, which are extremely fast.
>
		* Shift-And  
			S表示输入字符串，P表示模式串，D表示掩码集记录P的前缀匹配情况，辅助表B包含P中每个字符的分布情况，以bit位表示(与P方向相反，从右往左表示)，比如announce的辅助表可表示为:
> 	  
> 					a	00000001  
> 					n	00100110  
> 					o	00001000  
> 					u	00010000  
> 					c	01000000  
> 					e	10000000  
> 					*	00000000  
> 
> 			其中*表示不在辅助表B中的字符则以0表示，len(B[i])==len(P)
>	
			匹配过程如下
			1. 初始状态D=0;
			2. 遍历输入字符串S；计算P前缀匹配情况，D = (D\<\<1 \| 1) & B[S[i]] (用1表示前缀匹配情况，所以用`与`运算)
			3. 当 D & ( 1\<\<(len(P)-1) )结果非0时则表示命中，命中位置为i-len(P)+1
>
>		* Shift-Or  
>			算法基本原理与Shift-And相同，优化点在于匹配状态记录时减少了一步与1进行`或`的操作。
>			因此，构造辅助表时反过来以`0`表示P中字符的分布:
>
> 					a	11111110
> 					n	11011001
> 					o	11110111
> 					u	11101111
> 					c	10111111
> 					e	01111111
> 					*	11111111
>
>
>			匹配过程如下:
			1. 初始状态D= 1 \<\< len(P) - 1 (本例中即11111111)
			2. 遍历输入字符串S：计算P前缀匹配情况，D = (D \<\< 1) \| B[S[i]] (用0表示前缀匹配情况，所以用`或`运算)
			3. 当 D & ( 1\<\<(len(P)-1) )结果为0时则表示命中，命中位置为i-len(P)+1
>
* AC  
>
	频繁的缓存丢失缺陷，原因
>
	1. 过度的内存占用
>
	2. 随机的内存访问模式
>
>
### SIMD  
* Single Instruction Multiple Data，单指令流多数据流，一种采用一个控制器控制多个处理器，同时对一组数据中的每一个分别执行相同操作，从而实现空间上`并行`的技术。
>
* X86 CPU的MMX,SSE指令集，AMD的3DNow!指令集，ARM的NEON指令集，MIPS的MDMX指令集，等等，都支持SIMD；这样一来，HyperScan的匹配优势也能扩展到这些硬件平台上了。


## 优化背景
>
网络流量监控中(无论2-7层)，流量包检测都是关键的一环，以DPI(Deep packet inspection)为例，DIP结合了IDS、IPS和状态防火墙等功能，可以嗅探、检测2-3层以上的数据
而正则匹配，是DPI的核心要素，所以，正则匹配的效率和性能，尤为关键。

## 当前的最佳实践
>
基于 预过滤 的模式匹配  
即 keyword -> regex(signature)的匹配思路

## 预过滤模式匹配的问题
> 
1. keyword需要人工筛选(人力成本)
>
2. keyword选择未必合适(例如keyword选择过短，体现不出预过滤的优化效果)
>
3. 重复的keyword匹配(keyword匹配一次，regex又匹配一次)
>
4. 复杂的regex导致低效的NFA()(除非人工优化regex)

## HyperScan的针对性优化
>
针对上述问题：
	HyperScan使用与众不同的正则表达式分解(自动化)
>
当前匹配效率的问题：
>
1. 低效的多字符串匹配
2. 低效的NFA匹配
>	
针对匹配效率的优化：  
基于SIMD的模式匹配  
>
1. 高效多字符串匹配  
2. 基于bit位的高效NFA
>
达成的效果：  
Snort 8.7  
Multi-string matching: 3.2 over DFC  
Multi-regex matching: 13.5 over RE2

## 正则分解
>
* 基于分解的 匹配
	1. 将pattern分解为多个 字符串(STR)和子正则subregex(FA) 组件:
	HyperScan的整体思路是找到Regex中的STRs(也就是pre-filter Mode中的keyword)，但是不同点在于对于一些特殊类型Regex能高效而合适地提取STRs，例如：
>
		* 字符集类型 `b[i1l]l\s{0,10}`，普通的regex解析只会提取出b和l为keyword，而HyperScan可以提取出`bil`，`b1l`，`bll`这三个keyword。
>
		* 分支结构，例如`(.*\x2d)(h|H)(t|T)(t|T)(p|P)`，HyperScan是能够提取出`http`这样的keyword
>
		* 重复的绑定字符串，例如`[\x40\x90]{32,}`，HyperScan能够提取出这样64位长度的keyword
>
	2. 字符串匹配作为入口
>
	3. 所有组件的匹配需按序执行:即最先匹配STR，STR命中后再匹配它相邻的FA
>
	* 这么做的好处  
		1. 没有重复的string keyword匹配
>  
		2. 细粒度的FAs可以用高效的DFA进行匹配(因为FA会被尽量分解为最小的正则单元)  
>
		3. 促进/有利于 多正则(multi-regex) 匹配
>
	* 实现正则分解的关键问题：  
		1. 如何自动分解正则  
		2. 多少实际运用的正则可以被分解
>
* 基于`图`的正则分解
	* 为何要基于图？图结构更适合分析？正则解析的状态机也是基于图结构的
	* Regex -> Glushkov NFA -> Graph-based Decomposition
>	
	* 为了达到正则分解的预期效果(上述特殊类型Regex中的1,2,3)，HyperScan梳理出了4个基本原则来达到这一效果：
>
	1. 找到一个能把Glushkov NFA graph分割为两个子图的`字符串`，一个子图包含开始状态，另一个包含接受(结束)状态。
	如果开始和结束状态最后在一个图里，那么即便STR命中了FA也得执行。
>
	2. 避免短字符串(HyperScan的底线设置在2-3个字符。)
	因为短STR意味着可能更多次数的匹配，而且容易触发更多失败的FA匹配。
>
	3. 把小字符集展开为多个字符串来促进正则分解。(HyperScan把展开为字符串后长度小于11的字符集 定义为 小字符集)  
	这样不仅能提升成功分解的几率，而且在字符集截断字符串序列的场景中(如`document[\x22\x27]object`)能直接转换成一个更长的STR。
>
	4. 避免保持过多的字符串。  
	过多的字符串匹配会使得匹配器过载同时降低整个匹配系统的性能。所以，找到一些合适的STR来提升匹配效率是很重要的。
>
* 三种图分析技术来发现关键匹配路径的STRs，依次尝试，来分解得到STRs和FAs
	* 1.主导路径分析 -> 2.主导区域分析 -> 3.网络流分析
>
	* `主导路径`分析算法
		1. 若从起点开始，到达顶点v必须经过顶点u，那么u为v的主导顶点。
>
		2. 对应的，顶点v在图中的主导路径为其主导顶点在同一条单一路径上的集合。
>
		3. 本算法 用于查找 对于接受态(终点)顶点的所有主导路径中最长的字符串。
>
		4. 本算法 查找出的 STR，能清晰明确地把开始和接受状态分为两个独立的子图，这就完美满足了基本原则1。
>
		5. 本算法对每个接受态计算出主导路径，然后找到所有主导路径中有最长通用前缀的主导路径，接着再提取出STR。若路径上的顶点是一个小字符集，那么会对其进行扩展然后得到多个STR。
>
>
	* `主导区域`分析算法  
	若主导路径分析算法提取STR失败了，则采用本算法。
>
		1. `主导区域`更正式的定义可以表述为：  
			图G=(V, E)中V的一个子集，满足下列特性：
			* 这个区域点集构成的所有边，在图G中加入或去除这样的边，都能构成这个图的一个 [割集](https://en.wikipedia.org/wiki/Cut_(graph_theory))
>			
			* 对于该区域的每条入边(u, v)(其中u为区域外的入方向的源顶点，v为区域内的入方向目标顶点)，总存在边(u, w)(w在区域内且w有入边)
>			
			* 同样，对该区域的每条出边(u, v)(u为区域内出方向的源顶点，v为区域外出方向的目标顶点)，总存在边(w, v)(w在区域内且w有出边)
>	
		2. 若主导区域只是由字符串或小字符集 这样的顶点构成，本算法会把主导区域转化为一个字符串集。  
	然后这个字符串集 就能把原始图分隔为两个不连通的子图，在正则匹配的时候就必须先匹配命中字符串集中的任一字符串。
>	
		3. 本算法会在Glushkov NFA图基础上构造一个DAG(有向无环图)来避免分析过程收到后面的边的干扰；接着，对DAG进行拓扑排序，遍历每个顶点看是否已加入当前候选区域，它的边界边构成一个有效的割集。
>	
		4. 重复这个步骤来发现图里的所有主导区域。
>
		5. 因为只分析了DAG，所以原图里的`回边(back edge)`可能影响准确性。因此，对每个回边，如果它的源顶点和目标顶点分别在不同的主导区域，本算法会将这些相关主导区域合并为一个主导区域。
>	
		6. 最终，从主导区域里提取STRs
>
>
	* `网络流`分析算法  
	由于主导路径和主导区域 分析算法都是基于特殊的图结构实现，即原来的NFA图需满足一定的条件，所以有些场景它们可能发挥不了作用；因此，网络流 分析算法用于解决更通用的NFA图正则分解问题。
>
		1. 对每条边，分析算法查找到在这条边结束的一个或多个字符串；
>
		2. 然后，以在此边结束的字符串长度成反比的方式给这条边分配分数，即此边结束的字符串越长，此边得分越低。
>
		3. 对每条打分后的边，使用'max-flow min-cut'算法来找到最小的割集能把原图分割为开始态和接受态的两个子图。
>
		4. 最后，用'max-flow min-cut'算法发现 `原图中能生成最长字符串` 的边割集

## 基于 SIMD加速的 模式匹配
>
### Multi-string Pattern Matching
	FDR -> Hyperscan的高效多字符串匹配器  
	具体的优化内容  
>
	* extended shift-or matching -> candidate input strings
>	
	* verification -> by exact matching
>
* 传统shift-or matching的限制
	1. 只支持一个string pattern
>
	2. 尽管是采用bit位级别的运算，除了处理长的string pattern外，它的实现方式并没有利用好SIMD的CPU架构
>
* multi-string shift-or matching
	1. 区别与传统的shift-or匹配，对于模式串如announce是从左往右计算每个不同字符的分布位置(用bit表示时是从右往左对应位置)，FDR则采用从右往左计算位置(这样可以利用'SIMD加速'特性,在一个CPU时钟周期内并行执行多个CPU指令)。
>
	2. 辅助表B的字节段 长度 应`大于等于`最长模式串的长度
>
	3. 若辅助表B的字节段长度 大于 某个bucket内最长模式串的长度，那么把该bucket id编码进padding byte中，而padding byte就为B字节段长度减去某bucket最长模式串长度的这些字节段(从右往左)。
>
	4. 移位操作针对辅助表B而非状态表D，状态表D初始值为0(除了字节位长度小于最短模式串的)
>
	5. 遍历辅助表B，执行状态转移计算 D = D \| (B[s[i]] \<\< k) (k=i*n) n = len(bucket)
>
	6. 当遍历完毕时，匹配器查看D中是否有0bit存在，存在即表明某个桶的pattern可能命中，后面的命中验证则交由另一套逻辑负责。
>
* 模式串分组  
	根据上述的匹配算法思路，将模式串分配到不同的bucket里的策略，将会影响匹配的性能。  
	而分组策略的另一个主要目标，是最大限度地降低误报率。
>
	基于上述的目标，Hyperscan在模式串分组算法设计时会基于两个原则  
>
	1. 把长度近似的模式串归为同一个bucket；这样是为了让更长模式串的信息丢失最小化，因为输入字符的匹配命中率 只与bucket中最短模式串的长度相关。
>
	2. 避免把过多的短模式串放入同一个bucket；因为总体看来，越短的模式串越容易产生误报。
>
	基于上述原则  
>
	1. 把模式串按其长度升序排列，并且根据序列分别从0 ~ (len(p)-1)给模式串编号。
>
	2. 使用一种动态规划算法来计算 把模式串分配到 n个bucket 的 最小损耗，算法可总结为两个等式  
>
> 等式一
> 
					s-1
		t[i][j] = min (COSTik + t[k+1][j-1])
					k=i+1
>
* * 其中，s表示模式串的数量，t[i][j]存储分配编号为(i ~ s-1)的模式串 到 j+1 个bucket的最小损耗  
		对于这个等式，含义可表述为：
		t[i][j]取 `COSTik(模式串i到k分配到一个bucket的消耗)的总和` 与 `把剩余的k+1到s-1的模式串分配到j个bucket的最小消耗` 两者的最小值
>
等式二
>
		COSTik = (k-i+1)ᵃ / Liᵇ
>
* * 其中，COSTik表示对编号i~k的模式串分配到一个bucket里的损耗，Li是序号为i的模式串的长度，ᵃ和ᵇ是常量参数。
>
>	bucket中含有的模式串越长，那COSTik则越小，这样就允许更多模式串存入到这个bucket；
>
>	而bucket中含有更短的模式串，那么COSTik则越大；由此限制bucket中这类模式串的数量。
>
>	当前,hyperscan中的常数值设置为α = 1.05 , β = 3。
>
>	然后取t[0][7]来分配所有模式串到8个bucket中，并在这个过程对每个模式串记录其对应的bucket id。
>	在实际应用中，这套模式串分组的算法看起来表现挺不错，达到了设计预期。
>
* 超级字符
	* 对于上述基于bucket的匹配引入的一个问题是，在同一个bucket里的多个模式串的误报问题。  
	比如，一个bucket里包含`ab` `cd`两个模式串，那么上述算法也会命中目标串比如`ad` `cb`。  
>
		为了解决这个问题，hyperscan使用m-bit(m>8)，这里定义为`超级字符`(对应着ASCII字符为8bit)来构造辅助表B和处理输入字符(相当于重构了每个字符的bit表示方法，来运用上述的算法)。
	* 超级字符  
	由`放在低8位的ascii字符(字符本身的8bit值)`和`放在高位的下一个字符的m-8bit(从下一字符的低位开始取bit值)`，若是模式串或输入串的最后一个字符，则用空字符填充作为下一个字符。 
> 
	这里的核心思想是，对当前模式串的字符构造辅助表时，包含下一个字符的一些信息。  
>
	这样一来，只有两个字符都在输入中出现时，才构成命中条件(当然由于m-8bit不能完全涵盖下一个字符的信息，依然会有少许误报)，这样就能显著减少误报而只需消耗略多构建辅助表的内存。 
> 
	在实际应用中，m取9-15之间。
* SIMD加速  
	FDR重度利用了SIMD特性，这也是HyperScan高效的一个重要原因；  
	为什么SIMD对提升FDR如此显著，有以下两个方面：
	1. HyperScan使用128bit的状态表所以能直接利用上128bit的SIMD相关指令(比如x86-64指令集里的左移指令`pslldq`和或操作指令`por`)来更新状态表，而这种`移位` `或`又是FDR中最频繁使用的操作，所以性能由此大幅提升。
>
	2. 利用了多CPU执行端口的并行执行能力；原始的shift-or匹配，由于依赖上一步结果，所以是完全串行的；即便它能在一个CPU时钟周期执行多条指令，也没有充分利用上现代CPU特性。反之，FDR通过读取多个输入字符 并行地预先移位辅助表 来充分利用现代CPU指令级别的并行能力(值得注意的是能达到这种效果也是由于算法逻辑与原始的shift-or不同了)。
>
		这种并行执行效率提升了IPC(instructions per cycle)从而显著提升效率。
>
		为了满足这样的并行移位，FDR限制了模式串为8字节，若模式串超过8字节则取低位开始的8字节；这是因为并行移位至少需要8字节的掩码和最多7字节的移位，因为超过7字节移位128bit掩码就会在左移过程丢失信息了。
>
		在实际匹配中，FDR一次处理8字节的输入；为了能连续匹配这种每8个字节的输入方式，上一轮匹配过程的状态表D需要右移8字节给下一轮匹配使用。
* 验证  
	经过上述算法后，仍然可能会产生误报，因此需要再进行一次精确匹配。  
>
	这个阶段主要包含hash和精确字符串对比，为了最小化hash碰撞，hyperscan为每个bucket构造了一个独立的hashtable，然后根据上面匹配命中的位置，来把输入和hash bucket里的每个string进行比较。  
>
	在实际使用中，通过这个验证过滤过程，确实能屏蔽掉很多误报情况。

### Finite Automata Pattern Matching
>
hyperscan的策略是尽量使用DFA，只有DFA状态数量过多时，才回来使用NFA。 
> 
由此，hyperscan实现了一个基于NFA的快速匹配算法，其中利用了SIMD相关操作。



## 主要参考
1. https://en.wikipedia.org/wiki/Bitap_algorithm
2. https://www.usenix.org/sites/default/files/conference/protected-files/nsdi19_slides_wang_xiang.pdf
3. https://www.usenix.org/system/files/nsdi19-wang-xiang.pdf
