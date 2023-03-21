
# 自动化微服务拆分

## 定位

为已经构建完成不希望改动范围大的单体应用提供简便的自动化拆分工具，对云计算领域并不熟悉的单体应用开发者能够轻松使用。

本项目现阶段针对使用C++面向对象开发的单体应用改造，面向Java等其余面向对象语言的单体应用可能使用其他的技术路线有更好的效果。

因开发时间有限，本项目采取简化问题的处理方式，使用最易的实现方式。

## 形式

### C++面向对象单体应用开发建议

#### 单体应用项目架构

一个常见使用CMake构建的C++项目目录如下：

- `bin`: 二进制目标所在目录
- `cmake`: 部分库的编译指导
- `config`: 项目配置
- `doc`: 项目文档库
- `examples`: 项目使用范例
- `src`: 项目主要代码，在单体应用分层构建时可能有各种层次结构，模块由`module`组成，`module`可由无限级的`submodule`组成，我们关心三种最关键的模块代码：
	- `data`: 关于数据的存储、管理与使用
	- `business`: 处理业务逻辑的模块
	- `calculation`: 计算密集的应用模块
- `test`: 项目测试程序
- `thirdparty`: 项目使用的第三方库


### 项目目录架构

为简便实现，本项目主要采用Python，遵照Python项目的结构。

- `config`: 项目面向配置，包含微服务拆分的指导配置
	- `.clang-format`: 采用Google Code Style，将用户原C++工程格式重构，利用这种方式省去利用clang编译器重构项目的必要性
	- `architecture`: 本项目将设计新的语言或者使用json等描述文件刻画用户的微服务组织架构图，项目运行将以此作为重要参考
	- `tree_module_map`: 表明文件树中文件夹所代表的模块名，模块都应在微服务`architecture`中得到体现
	- `tutorial.md`: 该教程为组织架构图的描述语言介绍
- `micro_service_app`: 生成的微服务程序
- `raw_app`: 用户源代码，需要用户在源码中添加编译制导指令`#pragma ms`来指导本项目生成分布式共享数据库
	- `tutorial.md`: 微服务编译制导指令的格式与介绍
- `source_libs`: 包含已写好可直接搬运使用的微服务源码
- `src`: 项目源代码主目录
- `requirements.txt`: 依赖包安装
- `setup.py`: 项目安装

### 微服务项目组织架构图描述语言

欲描述一个微服务项目的组织架构图，需要明确相关的定义：

- `API Server`: 作为用户使用微服务的接口，负责与单用户流程的主节点通信
- `Node`: 以docker为单位的节点，为调度和微服务基本单位
	- `Components`: `Node`由组件`Components`构成，组件有三种角色(`role`)：
		- `inNode`为不与外界交互的逻辑模块，只对内提供服务
		- `Server`为向外界提供服务的模块
		- `Client`为请求外界服务的模块
	- `Functions`: `Node`有特定功能，后两者可与前三者组合
		- `File`为文件处理节点
		- `Calculate`为计算节点
		- `Passer`为中间数据处理、传递节点
		- `Master`可以是全局主节点，也能够是特定任务的局部主节点，全局主节点仅能有一个
		- `Child`为承担并行任务的子节点
- `Layer`: 当若干`Node`运行不存在冲突时，可并行运行，则可在同一`Layer`
- `RequestEdge`: 当存在一个`Node`的`Client Component`对另一个`Node`的`Server Component`有动作请求，则使用`RequestEdge`表明这种关系。不需要`ResponseEdge`，因为请求必被允许，可以理解为若两`Node`之间存在有向边`Edge`，则方向代表请求。`RequestEdge`落到`Components`

在明确上述定义之后便可以书写组织架构图了。

使用yaml描述微服务项目架构，yaml本身无naming style，因为是python项目，所以采取python `lower_case_with_underscore`命名规范：

```yaml
architecture:
---
# Layer
---
  - layer:
    name: "layer_layer_name" # must be unique
    # nodes
	nodes:
	  - node:
	    name: "node_node_name"
	    # functions
	    functions: 
	      - ["File", "Calculate", "Passer"] # Not a list, only chose one
	      - ["Master", "Child"] # Not a list, only chose one
	    # components
	    components:
	      - component:
	        name: "module_module_name" # match with module in raw project belew
	        role: ["inNode", "Server", "Client"] # Not a list, only chose one
	        request_edges: # only Servers have this field
	          - target:
	            layer_name: "other_layer_name"
	            node_name: "other_node_name"
	            component: "other_component_name"
	          - target:
	          # same below
	      - component:
	      # same below
	  - node:
	  # same below
  - layer:
  # same below
```

### 项目树-模块映射

需要将`architecture`中出现的组件名与项目树中的文件夹对应映射，即`Components`$$\leftrightarrow$$`Module`。

按照开发建议中[C++单体应用的项目结构](#单体应用项目架构)，本项目也使用这种模板来对源程序改造，因此在`src`目录下，需要能够找到在架构图中所使用的各个组件（模块），因此需要一种映射关系，我们直接借用`tree`命令并使用`.gitignore`过滤干扰文件夹：

```sh
cd src/.../modules_top_dir && tree -d -f --gitignore -o /path/to/config/project_module_tree.cfg
```

`.gitignore`内容如下：

```gitignore
.DS_Store
include
doc*
example*
api
config
test*
third*party
```

排除的文件中例如`thirdparty`第三方库会被集合进`Common`库，为每个微服务自动链接。

在运行完成此前命令后，打开`config`下`project_module_tree.cfg`文件，在模块名与文件夹名不同的项后添加模块名，并且需要将模块特征附后（所有条目），使用空格分隔，模块特征有以下几种：

- `ignore`: 不属于模块，可以在此标定也可以在`.gitignore`中添加
- `topdir`: 仅作为单体应用封装的顶层文件夹，不会在架构图中出现
- `subdir`: 仅为子文件夹，将与上层`module`合并
- `third-party`: 未排除的第三方库
	- 若需要在common中加入，则在后面加`common`
	- 若只需要链接某些模块或目录，则在后面加`link`，而后将模块或目录的顶层文件夹路径
- `module`: 为架构图中模块
- `submodule`: 对应架构图中模块，作为独立功能的子结构存在于`module`中

举例如下（STA应用）：

```sh
.
├── ./cmd ignore
├── ./delay spef module
├── ./liberty module
├── ./netlist module
├── ./parser-spef module
│   └── ./parser-spef/pegtl subdir
│       └── ./parser-spef/pegtl/pegtl subdir
│           ├── ./parser-spef/pegtl/pegtl/analysis subdir
│           ├── ./parser-spef/pegtl/pegtl/contrib subdir
│           │   └── ./parser-spef/pegtl/pegtl/contrib/icu subdir
│           └── ./parser-spef/pegtl/pegtl/internal subdir
├── ./sdc module
├── ./sdc-cmd module
├── ./sdf-parser ignore
├── ./shell-cmd ignore
├── ./sta module
├── ./user-shell ignore
├── ./utility third-party link ./delay
└── ./verilog-parser verilog module
```

由该指引本项目能够将微服务应用快速组织起来。

### 编译制导指令格式

本项目使用编译制导指令对源代码进行解读和改造，针对C++面向对象的程序，请尽量遵照一些单体应用的开发建议，从而能够通过本工具进行快速开发，否则需要人工进行一些程序正确性检查。

为方便对数据结构进行分布式存储与共享访问，需要用户提供需要远程存储的数据资源指导，参照C++`pragma`编译制导宏，本项目新定义了`ms`(即microservice) keyword来作为源程序向分布式微服务的编译制导，语法规范将在后面介绍。

#### 关键词语义

仍然需要将编译制导所需要用到关键词作较为清晰的阐释：

##### 数据结构

在理想状态下，每个模块的数据结构组织都是森林（或树），允许有回边。

- 首先在每个模块中需要将森林中树的顶点类明确标注，定义为`root`，在本项目的代码运行中也会形成森林结构，只需指明需要节点间共享的树定点程序即可了解某一棵树需要远端共享存储
	- 工作量略多，现阶段可以尝试标注所有树枝和树叶，但未来考虑用户易用性需要程序自动构建森林；现阶段标注树叶使用`leaf`关键字
- 为了性能考虑，对于C++类中的一些optional fields，即不一定在对象生成周期会存在的域，可以在代码中使用`std::optional`指明，或者使用编译制导`optional`进行定义
- 若类中存在无需使用的域，则使用关键字`ignore`来标注
- 对于某些细粒度的类，若没有必要单独存储，则可以通过编译制导`fusion`，此举将类的数据合并至上层。以这种方式来控制数据结构存储的粒度

##### 运行逻辑

如果只是按照模块拆分单体应用，分布式运行逻辑的标注将非常简单，但是本项目还支持分布式并行加速应用实现，所以需要提供以下关键词的解释：

###### 远程调用函数

在前文[微服务项目组织架构图描述语言](#微服务项目组织架构图描述语言)部分介绍了微服务拆分的组织架构图，要想实现这种微服务的联系，节点或服务之间需要有调用关系，本项目需要知道服务之间调用对应于单体应用的调用函数，使用`call`关键字定义这种关系。

- 因为目前使用`grpc`作为服务间通信协议，因此还需要指明服务调用的参数`argumemt`（现阶段不可重复，后期支持重复）与返回值`return`类型，可以是C++的基本数据结构也可以是远端存储的类
- 可以为稍复杂的数据传输定义类，但不推荐这么做，节点间大型的数据传输都需要通过Redis网络

###### 分布式并行

对于一些类似`for`循环的可并行步骤，若在本地能够通过`openmp`框架运行，且不存在较多的同步需求，则可以进行分布式并行执行，利用更多机器的资源更快的完成任务。

本项目所采用的分布式并行设计源于MapReduce，或者分布式的clone/fork，分发与收集的数据量很少，由各个节点自行准备数据。

- 关键字`parallel`描述了这一行为，本项目将自动提取函数逻辑，分发至不同节点运算。

#### ms编译制导语法规范

##### share

###### root

```cpp
#pragma ms share root
class Root {}
```

此类将作为模块中森林的根向下组织数据结构

###### leaf (temporary)

```cpp
#pragma ms share leaf [fusion]
class Leaf {}
```

将作为根节点的下属，且数据结构稍复杂，需要独立存储

###### fusion

```cpp
#pragma ms share fusion
class FusionLeaf {}
```

此叶子类较小，无需单独存储，将与父数据结构合并存储

##### field

###### optional

```cpp
class A() {
private:
	#pragma ms field optional
	int m;
}
```

在A的生成过程中可能不会生成m域

###### ignore

```cpp
class A() {
private:
	#pragma ms field ignore
	int m;
}
```

无需使用类A的域m