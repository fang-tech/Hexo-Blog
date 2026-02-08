现在的skill使用流程
1. 准备: skills文件夹
2. 加载: 通过FileSystemSkillRepository指定这个文件夹为baseUrl -> 从这个文件夹中load skills -> 注册进SkillBox中
3. 加载结束: 将SkillBox注册到ReActAgent中
4. 运行时动态管理: 通过skillBox提供的API管理(CRUD)skill并加载进系统
5. 对话前: 通过SkillHook将skill的instruction和skill (name, desc)二元组注入到system_prmopt
6. 对话时: 通过SkillLoaderTool(skillId, path(包含SKILL.md)), load对应的skill的特定的内容

拓展后的流程
1. 准备: skills数据源
2. 加载: **使用数据源创建repo, 将repo注册进skillBox**
	1. 可以**直接将repo注册进skillBox, 交给skillBox管理**
		1. 不再将需要从repo中load数据, 注册成AgentSkill再注册到SkillBox中(repo_name, repo_instance). repo只作为Skill的数据源存在, 那么实际上只和skillBox耦合
	2. repo现在维护一个**metaData**字段和一个**可配置大小的缓存队列缓存SKILL**, 以及refresh()
		1. **metaData会在repo构造的时候初始化**, 里面存储了所有skill的(name, desc)二元组, 用于注入skill提示词
		2. **repo的upload/load**
			1. upload: upload会更新metaData, 并且将skill持久化
			2. load(skillId): 优先从缓存中获取, 如果没有的话, 从数据源中加载
		3. **refresh**: 将缓存清空, 并且重新加载metaData
	3. **同样支持原来的创建模式**
3. 加载结束: 将SkillBox注册到ReActAgent
4. 运行时动态管理: 
	1. SkillBox自己持有一个HashMap, 通过SkillBox创建的都会保存在这里面
	2. SkillBox提供upload(repo_name, List.of(Skills)), 将skill持久化
	3. 并不在SkillBox中完全管理repository, 把repository的持久化嵌套一层到SkillBox中, 而是实现懒加载, 只有在要使用这个skill之前, 才会将这个skill实例化
5. SkillBox内部持有的只有skillMetaData, 完整的skill归repository持有(对于原先skillBox中的默认的内存的存储, 提供memoryRepository)
6. 对话前: 
7. 对话时: 将这次对话的skill的scripts的内容下载到skillWorkPlace(这个workplace是创建的一个;临时目录模拟沙箱隔离).  这个workplace的路径就是skill存放的路径, 也会是代码执行(shellCommmandTool)的base路径
	1. 提供一个API供销毁/创建新的workpalce
		1. 创建: 返回ID
		2. 销毁: true/false
	2. 提供一个API供连接上对应ID的workplace
		1. 对于沙箱就是是否和沙箱链接成功(不需要实现)
		2. 对于现在的临时目录, 则检测这个目录是不是存在
	3. 这个workplace的生命周期应该是session/会话级别的(具体方案你来考量, 和实现后我来决定)), 我期望的是
		1. 用户一段时间没有操作后, 这个workplace能执行自销毁(这个销毁策略我觉得用户可以自己设置, 什么情况会触发销毁, 销毁的动作是怎么样的)
		2. 如果要load的skills列表发生了变化(比如新增了skill, 或者删除了skill, 则需要清除过去的workplace中的skills, 重新load)
8. 并且存在代码执行路径的问题, 比如我在skill.md文件中要执行的代码是`python ./scripts/validate.py`, 但是我们shelltool的pwd并不是在这个skill的路径中, 这会导致要执行的脚本文件路径出问题(这里的你需要自己解决, 给出我解决方案)
9. 同样的问题之一是, 我的skill的代码常常需要考虑执行的脚本本身是携带命令行参数的, 比如我有一个parse_pdf_to_md.py, 这个脚本的命令行参数是携带了文件的, 这个时候很明显这个文件也要传入到workplace里面, 我怎么通过ID将这个文件upload到这个workplace, 这也是一个问题, 这个问题我并不知道实际的应用场景, 也就是我不知道这个机制该以什么样的形式提供能怎么解决在对话中的这个问题, 比如要用脚本解析某个文件的需求, 所以我需要你帮我设计, 以及帮我打通完整的链路, 完整的使用skill来分析我上传的文件的链路