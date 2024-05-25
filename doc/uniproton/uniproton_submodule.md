# 如何在 UniProton 模块下添加子管理模块



## UniProton 模块说明

- UniProton 模块是 oerv-rtos 工作的项目模块之一， 对应路径在 /work/uniproton 下面
- UniProton 模块的介绍在 /work/uniproton/README.md 中
- UniProton 模块需知的文档和发布的issue 均在 /work/README.md 中介绍



## UniProton 子模块管理目录添加说明

#### 添加子模块到工作模块

- 添加对应模块名字 xxx [此处以 `rpmsglite` 为例子], 在 /work/README.md 中关于 UniProton 的添加对应子模块名字, `rpmsglite`，**同时添加超链接指向该模块的工作指南**

#### 为子模块创建管理目录

- 在 /work/uniproton 下面创建目录，名字必须为**对应模块的名字**，比如为可以在 **/work/uniproton** 下面创建 `rpmsglite`目录，作为子模块的管理目录 .[1] 
- 子模块的管理目录下面必须有 **README.md** 说明文档，同时对应README.md 需要满足 UniProton 模块格式规范
- 子模块的相关内容，除了**发布的issue 和 发布的 case** ，其他的内容**只能出现在对应的管理目录里面**，如该模块需要使用的图片，该模块的一些入门文档，和该模块的细节内容等等。 

#### UniProton 子模块README.md 规范

- [UniProton 子模块 README.md 格式规范](./uniproton_submodule_rdme.md)

#### 子模块 issue 发布说明

- 所有 UniProton 子模块issue 应当**直接发布在 UniProton 大模块中进行统一 .[2]**
- 子模块不需要有独立的 issue 发布体系 
- [UniProton issue 发布规则](./uniproton/uniproton_issue.md)

#### 子模块 case 发布说明

- 所有 UniProton 子模块 case 应当**直接发布在 UniProton 大模块中进行统一**
- 子模块不需要有独立的 case 体系
- [UniProton case 规则](./uniproton_case.md)

####  子模块管理者说明

- UniProton 模块会对**所有要维护的子模块**创建管理目录
- 每个**需要维护的子模块**可以有0 - n 个维护者 , 0 即为无人关注, n 没有具体上限，成为子模块的维护者的评判准则为对该模块的贡献度.[3]
- 子模块管理者动态更新，若子模块维护者不再维护，则退出对应子模块的维护者序列



# QA

- [1] : 子模块管理目录需要有对应的开发者进行维护，如果没有开发者进行维护，或者以前有过开发者针对对应子模块进行开发，但是由于其他原因不再维护，oerv-rtos 小队需要根据对该模块是否继续维护进行评估，若不再需要维护则删除对应子模块，否则会将其的主要权限开发者置为空
- [2] : 统一的issue 管理体系，可以让其他开发者更方便直观的看到可以做的内容，不需要进入对应子模块管理目录了解其新的体系内容
- [3] : 评判的内容为实际对该模块的 pr 量和 pr 的质量，由UniProton大模块的维护者进行评判