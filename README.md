> 首次低代码探索

# 1. 背景
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前公司一个几年前使用传统开发模式（tomcat+mysql+Jersey+mybatis）研发的一个应用服务在ToB市场卖了好几年。ToB市场最恼火的是各种定制化开发，这也是大量以项目制形式运作的外包公司和Team哪哪都是的根本原因。对于ToB的定制化开发，公司层面的应对也是左左右右：有时使用自己的研发人员进行定制化需求的研发，以保障交付的可控度。有时候交给服务商（合作伙伴）或者短期聘用外包人员。在开发界大家都有一个共识：改别人的代码往往纠结更多。老服务的问题不仅如此，性能不高使得在面对大中型企业应用时感觉身体被掏空。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第一季度年度规划时，领导有意研发应用服务2.0，摆在眼前的问题非常明确：1、二开门槛高；2、性能需要提升。性能问题基本难不倒人（分布式常用套路：一致性哈希去分而治之，LRU等缓存策略，IO优化考虑等），二开门槛高该如何应对呢？思前想后也只有一个模糊的感觉：需要一个与业务无关的业务托管平台，并且一切皆可配。基于此，首先考虑两个最基本的方面：1、数据结构可配；2、接口可配；由此，前几个月开始了各种思路发散、方案验证、方案变更的反复，经历了2个多季度纠结，第四季度开始思路逐步清晰和收拢，研发越来越聚焦和收拢。然后昨天给BOSS进行了演示，然后就没有然后了，当即正式叫停该项目。原因很简单：咱家一直搞传统火电站，你们整个核电站，危险、危险、危险……

# 2. 整体功能
![IMG](https://s4.ax1x.com/2022/01/06/7SraRA.png)  

* 数据：配置数据结构（包括表、字段、字段类型）、配置索引、配置SQL等。
* 接口：配置接口信息（包括接口方法、路径、参数等）、接口调式等。
* 函数：配置函数、接口母版（类似ASP.Net中的母版概念）。
* 任务：配置定时任务。
* 依赖：项目依赖管理（例如：企业通讯录项目依赖用户项目）。
* 路由：配置对外HTTP交互模板路由（模板引擎结合脚本）。
* 测试：配置接口测试用例（一个接口支持配置多个用例）、配置回归测试计划、自动化拨测计划。
* 工具：数据库文档生成、API文档生成、客户端胶水层代码生成、配置导入、导出等。

# 3. 数据
![IMG](https://s4.ax1x.com/2022/01/06/7SsUTU.png)  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;动态建表显然简单，但是如果想做Severless的aPaaS，这样干就是找死，业务变更直接导致物理表结构的变更显然不可接受。可能此时有人会这样想：那么好吧，不管3721给表预建百十个字段不就行了（有些自定表单的系统好像确实就是这样干的）。但是这样干引发的其他弊端和约束又如何解决呢？例如：表长太长影响性能、要根据预建字段进行查询怎么搞？难道强制约束为：预建字段只允许显示？……  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;基于此，我们首先想到纵向进行数据存储，但如果想法仅仅这样简单，纵表带来的其他问题如何应对？例如纵表使用、理解复杂度高。纵表数据检索如何搞？在这两个问题上我们经历的曲折、反复、纠结不做介绍，最终方案的要领为：

* **数据本身纵向存储**  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据本身类型约束为：字符串（varchar）、数字（bigint）、浮点数字（double）三者之一，不能一切皆String的原因很简单：数字1,2,11你用String去表达，最后排序你咋搞？

* **数据索引横向存储**  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据索引表动态建表。例如给类型为String和Int的两个字段建立一个普通索，首次系统会建一张：普通索引_String_Int的索引横表，并给String和Int这2个字段建立一个普通索引。那么其他情况大家类推了。


* **先查索引再查数据**  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;类似Mysql自身引擎的处理方式，先查索引再查数据。我们在实际的处理中采取的是索引数据和数据本身通过真实PK关联，可以把这里说的真实PK理解为一个行号，那么数据查询过程就是：先查一个数据索引横表：pk,field1,field2，其中field1,field2建立索引（PK，UNIQUE、INDEX之一）得到一个行号，然后再用这个行号去查数据纵表（数据纵表的行号字段为物理PK）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可能这样说大家还是不好理解，那么大家可以按照：数据本身的存储其实聚簇索引（将数据存储和索引放在一起、并且是按照一定的顺序组织的，找到索引也就找到了数据），而上面提到的索引数据是非聚簇索引（不存储数据，存储的是数据行地址），如果查询结果需要返回索引字段之外的其他字段，则需要走一个通过非聚簇索引 ->数据本身的回表过程（不过现在回想起来，我们的整个系统的这个rowId是用飘雪算法生成的8字节long，mysql如果在没有PK和UIndex的情况下，系统生成的rowId是6字节的东西，基于我们系统设计和验证绕了很多弯路后得出的结论：不要轻易自己创造什么概念和假定什么结论，如果有类似成熟系统可以参考，则应该优先使用其概念和结论。那么这里我们用8字节表达rowid，显然存在2字节的浪费，在这个以云服务器为主的年代，内存和磁盘的占用都是钱哪！这种基础的东西如果能省一点，抠一点，最终产生的收益往往非常可观，反之，浪费可观！）。这里的提到的：聚簇索引、非聚簇索引、回表其含义及做法与Mysql是相通的，只是说这些过程以前大家可以不需要关注，Mysql自身就帮你完成了。同样，在当前这套系统的使用过程中，你同样可以不需要关注，直接写业务SQL即可。另外需要说明是：考虑到性能，我们不做全表扫描，因此在后台中提交SQL时，如果条件没有命中任何索引，系统会强制拒绝该条SQL的提交。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可能大家有一个印象mysql这玩意单表数据超过1k万就完犊子了，但是如果完全基于PK查询，十亿级别下效率也是感人的毫秒级。演示之前我做了一个压力测试：单台8C16G服务器下，数据表在接近十亿的情况下调用读写接口的TPS性能为： 

* 写性能：  
新增数据(18个字段)写：1536.13    
新增数据(5个字段)：3498.46  

* 读性能（存量数据6-8亿）  
单表查询-无本地缓存-不命中SQL缓存： 560.7
单表查询-无本地缓存-命中SQL缓存 ： 4153.7  
两表联合查询-无本地缓存-不命中SQL缓存： 241.23  
两表联合查询-无本地缓存-命中SQL缓存： 3677.78   


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到这里，解决了读写问题，但还一个不可回避的问题：这样搞使用起来太复杂了！别说外包人员了，就是公司内部自己的高级研发也得骂娘。搞玩意的初衷不就是想给外面的人改起来简单方便吗？那就更一步吧：使用上对外屏蔽纵表读写的复杂度。做法是：解析SQL，然后转成纵表读写语句执行。谈到语法解析、AST什么的，我当时也是觉得这玩意太高深。还好有现成的SQL -> AST 东西可以直接用。不过接下来实现接口配置的时候，我们不得不面对。还有我们的首席架构师LM大神搞定了这块最难啃的骨头（LM大神早生个十年，搞不好是语言红皮书的封面人物了，这项目黄还得赖LM大神，根据作者胡子长短与语言发展好坏成正比定律来看，没胡子的LM大神早已注定了昨天的结果）。 昨天演示了一个企业通讯录项目，其中的一个SQL配置为以下，对外使用完全可以直接当横表去用。
```
SELECT relation.*,org.organizationName AS organizationName
FROM organization_relation AS relation
INNER JOIN organization AS org ON relation.objectId = org.organizationId
WHERE relation.parentId = #{parentId} 
ORDER BY relation.objectOrder ; 
```

收一收，对数据配置总结一下：  
**纵向数据存储引擎**   
使用简单：以横表方式直接进行使用，对外隔离纵向存储复杂度。  
读写高效：亿级数据量下读写表现良好。  
约束性强：通过不允许执行无索引的查询约束使得低效SQL无法执行。  

# 4. 接口
![IMG](https://s4.ax1x.com/2022/01/06/7SsDp9.png)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;下面说说接口，接口配置中配置接口信息（包括接口方法、路径、参数等）没啥难度，可是接口的逻辑也要实现可配置如何搞？这时万恶的词法解析、语法解析，最终还是没能避开，其实也不是没有进行过尝试去避开：最开始使用现成的脚本引擎去搞，后来还用过逆波兰表达式处理，因不完全可控、达不到预期、扩展太麻烦等原因而放弃。下面简单说下脚本引擎的实现思路：  

* **词法分析**   
1、扫描源代码文本，从左到右扫描文本，把文本拆成一些单词。  
2、分析出拆出来的单词是什么：关键字、标识符、符号、，注释……，其结果产物为Token。   

上面的话不太好懂，举个栗子：
```
var a = 1 + 1;  // Comment

var ：关键字
a ： 标识符
= ： 符号
1 ： 数字
// Comment ： 注释

```

* **语法分析**   
1、token序列会经过语法解析器识别出文本中的各类短语。  
2、根据语言的文法规则输出解析树（AST）。  

需要说明的是，我们在实现过程中文法定义由配置文件表达，这样做的好处是在实现脚本多语言时，只需要定义新文法配置文件，然后写对应的statementHandler即可。目前我们的脚本语法选择高度贴合JavaScript，如果未来想搞套其他语言的语法实现则会相对方便（例如.Net支持 C#、VB.Net、JS.Net 3种语法，但编译后生成的中间语言相同）。

* **脚本编译**  
1、根据AST生成四元式。  
2、序列化四元式（有一定压缩优化，如果满大街的很长的标识符，显然很容易对其优化）。  
关于四元式举个栗子（当然现在大家基本上都是用带返回的用三元式去搞）：
```
5 + 6 -> 四元式:

5 ： p1
6 ： p2
+ ： operator
11 ： result
```

* **脚本运行时**   
1、 简单来说就是按照顺序执行四元式数组，当然还有jump（想想for -> do while, if else 嘛）。  
2、 目前和JAVA原生的执行效率对比是4点几 : 200多（冒泡排序Java原生执行和脚本运行1万次的用时对比，单位：毫秒；当然这是公司发的笔记本电脑跑出的结果），为了赶演示导致没有时间继续优化（1周半完成接口、函数、项目等的后台及联调和问题解决），这块还有提升空间，应该能做到跟JAVA原生差1一个量级。  

* **脚本能力**  
1、支持JavaScript语法和数据结构，例如：运算、条件、循环、判断等；String、Number、数组、Map等；函数（匿名函数）、对象等。   
2、此外内置提供应用开发中常用函数，例如，DB操作、Redis操作、LRU缓存等。  
3、脚本支持分布式（一个接口如果因为业务逻辑确实很重时通过分布式执行方式可以充分发挥服务器集群能力），同时支持非阻塞旁路分支逻辑（例如，创建用户的同时需要请求其他系统获取Token，获取Token走旁路逻辑则不影响创建用户接口自身响应时间）。  

收一收，对接口配置中涉及的脚本引擎总结一下：  
**自定义脚本引擎**  
简洁易用：高度贴合JavaScript，内置应用开发所需的常用函数。  
扩展性强：脚本增加新语法、内置函数快速简单。  
执行高效：通过基于非解释性元组的执行方式使得执行效率非常高效，同时编译产物有一定压缩率。  
支持分布式：一段脚本执行到任意一行之后支持交给其他节点服务器继续进行处理。

# 5. 其他
* **定时任务**  
定时任务的实现是将quartz与脚本运行结合实现的，不想花额外的精力，况且cron表达式已经是大家共识的规范。  

* **复用性**  
1、依赖，真正意义上实现了应用开发中经常提起的模块化能力，演示中展示的就是：“通讯录项目”依赖“用户项目”。 简单来说就是支持已有项目组合，然后再去加已有项目能力之外的东西，已有的能力和新加能力在一起就全部可用了。  
2、通过import方式支持函数复用、通过母版的方式支持应用内全局复用。

* **文档及客户端代码生成**  
1、DB和API所有定义规则服务端均有明确表达，具备生成这2个文档的能力理所当然。  
2、同样客户端接口调用胶水层代码生成也是同一个道理。  


* **开发环境**   
1、WEB IDE插件目前不是鲜见的东西了，早在2019年5月7日微软就推出了WEB版本VS Code，目前支持代码提示、代码格式化、查找定义等。    
2、此外还支持Debug调试（后台具备能力，WEB后台没加）、日志获取等。  
3、参考VS Code后，我新设计的配置后台原型个人认为更加合理。

![IMG](https://s4.ax1x.com/2022/01/06/7Sss61.png)  

* **客户端逻辑植入（当前未实现）**   
1、 基于对脚本引擎的介绍大家可以知道整个运行机制其实就是执行四元式数组，那么只要实现安卓、OC、WEB的执行器后就可以做到服务端定义客户端逻辑后下发给客户端去执行。那么终极的效果可以做到：客户端UI稳定之后，客户端逻辑变更将不再需要客户端发版（客户端发出去的版就是泼出去的水，就问你微信让你强制升级你恼火不？）。  

* **方案可行性证明**   
1、 配置WEB后台调用的接口就是基于目前的这套游戏规则手工配置出来的（手工把接口相关信息写到配置库里，然后编译嘛）。就是说自己后台的接口用自己的游戏规则去配出来先验证一次。  
2、 演示企业通讯录项目：用户项目包括创建用户、用户登录、各种获取/修改用户信息等功能。通讯录项目包括创建公司、部门及其关系、全量获取通讯录信息、按需增量获取通讯录信息、加人、删人等等。基本上花1天时间完成通讯录的主体功能。Github上某著名项目：某API对外称相对传统开发模式效率提升25倍以上， 那么我们搞的这玩意不知道说得说多少倍合适了。

# 6. 低代码能力的五个级别
* **L1. 基础表单工具**   
支持用户自定义表单，表单支持基础权限管理，支持基础的主从表结构，面向非编码人员，基本不支持函数，也不支持编程。适用场景，调查登记表, 问卷等。

* **L2. 低代码平台**   
支持用户自定义表单，支持综合性的主从表单，支持丰富的权限体系，支持完全自定义的用户角色，支持流程与基础报表分析。 支持部分函数，支持部分的前端编码干预，开放通用的数据接口, pc端移动端双向适配。 不支持复杂业务场景。适用基础客户关系管理， 部分的销售、采购管理等。

* **L3. 企业级低代码工具**   
支持完全的用户自定义数据结构，生成相应的综合性表单和接口, 支持复杂的角色权限体系设计， 支持所有的前后端编码 。 非内置的复杂场景，需要自行编码完成。 相对完整的组件级封装，pc端移动端双向适配。 有完整的架构设计，大概包括数据结构、权限、元数据、流程、菜单、业务分层、系统配置、路由、日志等。基本无内置完整应用。 架构相对完整，适合初级人员快速开发独立的项目，组件级封装大大减少前端编码量。 复杂业务场景未能有效简化。

* **L4. 高级企业级低代码工具**   
支持用户自定义数据结构，生成全能型表单和接口, 支持完整的权限体系设计，支持前后端编码， 内置处理大部分复杂场景。 自动化处理大部分CRUD工作，全面业务封装，pc端移动端双向适配。 覆盖完整的架构设计，内置成熟的应用。适合定制化的综合性项目，整体代码量非常少。

* **L5. 与机器沟通完成编程**  
无需任何编码知识，只需跟机器沟通需求，机器自己组装代码完成编码工作。 非完全人机对话可实现， 基于庞大的代码库和AI技术， 人类面向代码引擎，描述要解决的问题，机器调用代码库，用户根据代码描述选择就行。 所有的代码抽象为过程，同时生成可被普通人认知的描述， 普通人根据描述认知每个过程，并组装。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其实我曾对低代码未来趋势、现状分析等做过调研和向BOSS汇报，目的很简单：希望这个项目能活下去。曾经写过2个汇报PPT，一个是XX公司提供 aPaaS 云服务调研分析，从历史发展趋势的：物理机-虚拟机-云-到Serverless，再到最后的低代码aPasS，然后谈及Gartner预测：2024年所有应用程序开发活动当中的65%将通过低代码的方式完成、未来5年中国企业数字化转型至少需要开发5亿个新应用。再到资本风向、大厂布局、低代码行业现状，利弊分析等方面给BOSS吹风。然后还写了一个关于将来对外推广宣讲材料草稿及思路的PPT，里面对于平台介绍、未来产品规划、商业模式（运行时开源、用户可选择全云上Serverless使用，也可以选择云上开发，私有运行和全私有方式）等均做了具体规划和设想。无奈公司战略级问题，岂是我等鼠辈能左右。假如有老板喊一个Y，哪我肯定秒跟了（不让做白日梦，黑日梦总可以吧？）。目前DD-YD虽然背靠实力庞大的AL，而且其目的也是做生态，但从本质上来说还是基于表单的数据获取、展现及分析的低代码，就算将来具备了良好的生态，也不能真正达到L5。

# 7. 结束语  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在2021年的最后1天稀里哗啦写那么多，一方面回顾今年主要工作，另一方面，希望给低代码后来之人提供一些思路参考。需要说明的是，这玩意纯属于闭门造车，我们Team中的这几个人之前在低代码领域都是小白，方案中的不妥之处，还请各位看官轻点。 以前真不懂什么叫做集体的智慧，什么事情都是自己闷想，经过该项目后，才真正体会到什么是集体的智慧。我在这个项目中做的事情比较杂，面上的负责人，实际上，架构师的系统设计、程序员的代码编写、产品经理的UI原型设计、项目经理的计划排期都搞了。希望L4的低代码+AI+良好生态的Level5级产品早日问世（生态这玩意已经不是技术问题，也不是个人能推动和解决的事情了）。

by chenx 2021-12-31 夜
