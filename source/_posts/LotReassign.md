---
title: LotReassign
date: 2022-05-09 16:30:19
tags: Design
---



在北京中芯国际出差这段日子，主要负责了 Lot Reassign 这个功能模块，同时也了解了 EDC 这个功能模块，我会在之后的文章介绍。

### 功能描述

#### Lot Reassign 分为三个功能模块：

1. Only Product Change
2. Lot Reassign
3. Only Version Up



#### Lot Reassign

Lot Reassign 主要是对当前 Lot 进行 Product、Process 、Route、 Step 进行更改。

下图是 Process、Route 及 Step 的关系图，可以理解为工艺、工步、工序。

Process、Route、Step 在 MDS 进行配置。

![a2be5408ea04bcf4ce341d0a10809ff7.png](https://img.gejiba.com/images/a2be5408ea04bcf4ce341d0a10809ff7.png)

#### Only Product Change 

对 Lot 的 Product 信息进行改变，Product 信息 在 MDS 系统进行配置，可以配置当前 Product 和 升版的 Product 及 Active 的 Product，eg：ProductA.1、ProductA.2（Active）。如果对某个 Lot 进行 Product Change 那么默认取 Active 的最高版本的 Product。



如果 Product 对应的 Process、Route 及 Step 有 Active 并且升版的数据，那么默认也要取这些数据进行Update。



#### Only Version Up 

仅仅对 Lot 的 Product、Process、Route、Step 进行升版（版本应该处于 Active 状态）。



### 详细设计

#### Future Action

针对这三个功能，它们有着公共的需要处理的业务场景就是 Future Action，

eg：Future Hold，Future Rework，Future Merger、Future Split、Future Reassign。

如下图：绿色区域表示 Lot 所在 Step ，红色区域表示未来发生的动作。如果用户勾选删除 Future Action 复选框，进行 Reassign 之前应当提示用户 Lot 的 Process 有哪些 Future Action 会失效。Process A 切换到 Process B ，400 站点的 Future Action 会失效，如果用户未勾选，那么将进行 Copy，Process B 的 400 站点也将设置相同的 Future Action。

![c9203e83b558854ceee83791060af954.png](https://img.gejiba.com/images/c9203e83b558854ceee83791060af954.png)



#### Check

在执行这三个功能的前置动作除了 Future Action 提示外，还将进行一系列的 Check 操作，eg：检查 Lot 的状态是否处于 Hold 状态，是否是要生产的 Lot（可能已经报废，终止，完成）等，是否正在进行加工中，是否处于异步执行等逻辑。这些检查逻辑需要详细的罗列出来，并且检查顺序要有一个优先级。当不满足某个条件，应当立即返回提示信息。因为界面支持批量，对多个 Lot 进行 Reassign 操作，所以 Check 之后应当提示哪些 Lot 是成功的，哪些是失败的及失败原因。这里存在性能问题，因为是批量，并且需要检查很多业务逻辑，调用很多不同的查询方法，接口性能需控制在 300ms，所以应当控制多少个 Lot 去处理。



#### Do Action

当经过一系列的检查逻辑之后，该 Lot 满足执行 Reassign 操作，在执行 Reassign 之前还将触发某些 Action ，eg：当 Process A 100 直接 Reassign to Process A 400。具体可能触发  Hold，Future Reassign 等，这些 Action 需具体罗列出来，并且排列优先级去执行。当前站点可能多个 Action ，应当考虑先触发哪个，后触发哪个，或者哪些不会触发。 eg：400 站点有 Hold Action、Rework Action、Reassign Action。必须先触发 Hold，

然而 Rework 和 Reassign 哪个先执行呢，这个很让人头大，因为这两个 Action 只会先执行其中一个，如果先执行 Rework，那么回到了 400 前的站点，同时删除 Rework Action，当又到达 400 站点的时候，Rework Action 不存在，这个时候就会执行 Reassign。那假设先执行的 Reassign ，Lot 可能已经不在当前 Flow 了，如果用户勾选了删除 Future Action，那么 Rework 永远不会执行。如果未勾选会 Copy Rework Action，参考下图：

但如果 A 400 到 B 400 之后是不会触发 Copy 的 Action，此时 Copy 对当前 Lot 没有意义。关于执行优先级需要沟通，根据业务场景定好优先级。因为触发顺序不一样，结果也不一样。

![1fb33840e007d2cb619fc2ace11dd958.png](https://img.gejiba.com/images/1fb33840e007d2cb619fc2ace11dd958.png)

如果 Process A 的 100 站点 Reassign to Process B 的 200 站点设置了 Future Action，那么将触发该 Action。

有一种特殊情况，就是 B Flow 的 200 站点 Action 是 Future Reassign 并且指向的 A Flow 的 100 站点，那么又回到 Reassign 之前的 Flow A，这种操作就会毫无意义，界面应该给予友好提示。更糟糕的事是如果 A Flow 的 100 站点同样设置了 Future Reassign to B  Flow 200，那么将导致死循环。B 200 跳 A 100，A 100 跳 B 200。为了避免这种事情发生，如果 A 100 设置了 Future Reassign to B 200，那么 B 200 是不能设置 Future Reassign to A 100。

![c9203e83b558854ceee83791060af954.png](https://img.gejiba.com/images/c9203e83b558854ceee83791060af954.png)

这里模拟了几种执行 Action 情况，还有很多情况需要考虑，例如 FSM（Future Split & Merger）等。



#### Copy Action

参考上图，如果原 Flow 的站点和目标 Flow 的站点对应，将原 Flow 站点的 Action 进行 Copy。Copy 的时候需要检查目标站点是否发生变化，如果此时站点已经更新了，400 站点不存在，那么应 Hold 原 Flow 当站 100，并且给予提示。



#### Do Reassign

这一步之前仍然需要 Check 目标信息是否存在，如果不存在 Hold 当站，并且给予提示。将进行 Product Change 或者 Version Up 或者 Flow 的变更操作，之后就是写入历史记录。如果某个 Lot 正在加工中，具有 Control Job

那么会创建一条 Future Reassign，也就是 Post Reassign。

![71ae22434c908489f4eb8106ff4d4953.png](https://img.gejiba.com/images/71ae22434c908489f4eb8106ff4d4953.png) 

#### Post Reassign

当前 Lot 已经进入机台正在加工中，这个时候是不能进行 Reassign 操作的，但是会默认创建一条 Future Reassign 的数据，Lot Move out 的时候再去执行这条数据。这里有几点要注意，因为这种情况 200 站点已经创建了一条 Future Reassign ，那么界面上不能手动再去创建 200 站点的 Future Reassign。一个站点只能有一条 Future Reassign 数据。因为是先创建数据，然后再去执行，所以执行的时候需要 Check 数据信息的正确性。如果目标站点不存在那么需要 Hold 当站，并且给予提示。

![437b42ecaedae43ee2c623c55ad9db0f.png](https://img.gejiba.com/images/437b42ecaedae43ee2c623c55ad9db0f.png)



#### Future Reassign

未来执行的 Reassign 操作，可以提前设置。当 Lot 进行 Move Out （移出机台）或者 Pass Thu （过站）等操作的时候，会异步执行 Reassign 操作。有些 Main Process （主要的业务操作）有很多 Post Process（异步操作），例如 Move Out 可能会有以下异步操作。

![dc0b99d09e34722f9f0cf444997e5aca.png](https://img.gejiba.com/images/dc0b99d09e34722f9f0cf444997e5aca.png)

Future Reassign 也属于其中一种，如果 Future Reassign 在 Check 阶段失败了，那么应当怎么处理。因为它是

异步操作，没有和主逻辑在同一个事务，Future Reassign 失败了，Move Out 还是成功的。并且如果 Process Move 这个操作在 Future Reassign 之前，那么还会到下个站点。这种情况不符合业务场景，Future Reassign 失败了应该 Hold 当站，不能移动到下个站点，而且界面应当具备可以查看哪些异步操作是成功，哪些是失败，失败原因是什么。在 Future Reassign 中 Hold 方法和 Check 方法是同一个事务，Check 方法失败抛异常会导致 Hold 方法事务回滚，所以没有办法去 Hold 当站。（可以通过环境变量去控制，这个我会在之后的 Post Process 文章详细介绍）。

界面支持手动创建和删除 Future Reassign。

#### History

NA

### 界面设计

#### 条件查询区域

From 区域：用户可以根据 Product、Process、Route、Lot 四个之一去查询，例如 Process 下拉框，下拉会查询所有的 Process 数据，如果用户先选择 Product，那么需联动出来对应的 Process 和 Route 数据，如果有一对多关系，需用户手动选择，如果只有一个，默认带出。可以查询 Active 和非 Active 的数据。

To 区域：只能查询 Active 的数据，并且只能联动，不能单个下拉查询。



#### 是否删除 Future Action

Product Change 和 Version Up 默认不勾选，因为这两种情况一般不会改变 Flow ，所以保持原样。

Reassign 默认勾选删除，因为会从当前 Flow 改变到其它 Flow。当前 Flow 的 Action 对应该 Lot 是

没有意义的，需要 Copy 到新的 Flow 上。



####  查询的数据列表区域

默认查所有，暂时不支持分页。可以下拉选择然后添加到右边列表。



#### 添加

如果单选框选择的是 Product Change，那么添加的时候只能添加相同 Process 的数据。

保持 Flow 不变，仅仅改变 Product 。

如果选择的是 Version Up，那么 Lot 的 Product、Process、Route 必须一致才能添加。

如果选择的是 Reassign ，无需卡控，查询出来的数据都能添加到右边列表中。



![fc89d8b2b53d567e77dc9c03386f4d04.png](https://img.gejiba.com/images/fc89d8b2b53d567e77dc9c03386f4d04.png)

### 接口设计

NA

### 数据库设计


#### PD 

图中一个个原点表示（Process Define 数据）

#### PF

一个原点与原点直接的黑色实线表示 PF（记录 PD 的下级所属关系表）

其中有个字段 LEVEL，有以下三种值

- Main_Mod（Process 与 Route 关系）
- Main_Ope（Process 与 Step 关系）
- Module（Route 与 Step 关系）

PF_Route 、PF_Step 表示具体的信息。

#### POS 

Step 配置信息

#### PO

Process Operation 运行时数据 





![a2be5408ea04bcf4ce341d0a10809ff7.png](https://img.gejiba.com/images/a2be5408ea04bcf4ce341d0a10809ff7.png)
