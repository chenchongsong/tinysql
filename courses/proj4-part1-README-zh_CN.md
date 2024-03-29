# 搜索框架 System R 和 Cascades

## 概览

在这一小节，我们将详细描述 System R 和 Cascades 两种优化器框架的设计理念以及 TinySQL 中对应的代码实现。

## TiDB 中基于 System R 框架的优化器

System R 两阶段模型可以说是现代数据库基于代价优化（Cost based optimize）的优化器的鼻祖。TiDB 目前生产环境中仍然采用的这一框架。

[DoOptimize](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/optimizer.go#L76) 是优化器的入口。这里我们会传入原始的 Plan tree。然后在经过逻辑优化和物理优化后，返回一个最终的 Physical plan tree。

### 逻辑优化

[logicalOpimize](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/optimizer.go#L95) 是逻辑优化的入口。我们会顺序遍历所有的优化规则，每个优化规则会遍历整个 plan tree，同时对 plan tree 做一些修改，最后返回修改过的 plan tree。


### 物理优化

[physicalOptimize](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/optimizer.go#L112) 是物理优化的入口。这个过程实际上是一个记忆化搜索的过程。

其记忆化搜索的过程大致可以用如下伪代码表示：

```
// The OrderProp tells whether the output data should be ordered by some column or expression. (e.g. For select * from t order by a, we need to make the data ordered by column a, that is the exactly information that OrderProp should store)
func findBestTask(p LogicalPlan, prop OrderProp) PhysicalPlan {

	if retP, ok := dpTable.Find(p, prop); ok {
		return retP
	}
	
	selfPhysicalPlans := p.getPhysicalPlans()
	
	bestPlanTree := a plan with maximum cost
	
	for _, pp := range selfPhysicalPlans {  // 枚举当前树节点的所有物理执行计划
	
		childProps := pp.GetChildProps(prop)
		childPlans := make([]PhysicalPlan, 0, len(p.children))
		for i, cp := range p.children {
			childPlans = append(childPlans, findBestTask(cp, childProps[i])
		}
		physicalPlanTree, cost := connect(pp, childPlans)
		
		if physicalPlanTree.cost < bestPlanTree.cost {
			bestPlanTree = physicalPlanTree
		}
	}
	return bestPlanTree
}
```

实际的执行代码可以在[findBestTask](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/find_best_task.go#L95) 中查看，其逻辑和上述伪代码基本一致。

## Cascades 框架

[最初的论文](https://15721.courses.cs.cmu.edu/spring2018/papers/15-optimizer1/graefe-ieee1995.pdf)里对 Cascades 架构的设计理念做了一个比较细致的讲解。

[TiDB Cascades Planner 原理解析](https://pingcap.com/blog-cn/tidb-cascades-planner/) 这篇文章对 Cascades 在 TiDB 中的实现做了比较细致的讲解，大家可以通过这篇博客结合 TinySQL 的代码进行学习。TiDB 和 TinySQL 在关键概念释义上是完全一样的。概念名词可以直接在 TinySQL 找到。

这篇文章主要讲了Volcano Optimizer与Cascades Optmizer

### Volcano Optimizer
1. 使用两阶段的优化，即逻辑优化[DoOptimize](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/optimizer.go#L76) 与物理优化[physicalOptimize (findBestTask)](https://github.com/pingcap-incubator/tinysql/blob/master/planner/core/optimizer.go#L112) 。
2. 实现的代码位于`planner/core`目录下面
3. 逻辑优化阶段，所有的优化规则都按照同一个固定的顺序依次去看是否能够作用于当前的逻辑执行计划，例如最先执行的规则总是列剪裁。且每个规则至多只会在被顺序遍历到的时候执行一次
4. 采用自顶向下的动态规划算法（记忆化搜索）

### Cascades Optimizer
1. 以Group为基础建立搜索空间，可以重复应用同一个Pattern & Rule来多次优化。TiDB的实现中包含`Preprocessing Phase`、`Exploration Phase`、`Implementation Phase`三个阶段
2. 实现的代码位于`planner/cascades`目录下面

## 作业

1. 完成 `planner/core/rule_predicate_push_down.go` 中的 TODO 内容。 `PredicatePushDown`

2. 完成 `planner/cascades/transformation_rules.go` 中的 TODO 内容。`PushSelDownAggregation.OnTransform`与`MergeAggregationProjection.OnTransform`

## 评分

通过 `transformation_rules_test.go` 中的测试以及 package `core` 下 `TestPredicatePushDown` 的测试

```
cd planner/cascades
go test -v -check.f="TestRule*" .
```

```
cd planner/core
go test -v -check.f TestPredicatePushDown .
```
