---
title: golang 用策略模式优化一段代码
description: golang 用策略模式优化一段代码
date: 2024-10-15
tags:
  - 后端开发
  - golang
---


背景是小组内组织代码 review 后分配了一个任务给我，优化：**在不同场景下获取不同的推荐分配方案的函数。**

# 直接上优化前和优化后的代码对比
## 代码对比
优化前代码：

```go
const (
    AlgorithmRandom             = 11
    AlgorithmItemCf             = 22
    AlgorithmContentBased       = 33
    AlgorithmRankingLatestHot   = 44
    AlgorithmRankingBestSell    = 55
    AlgorithmHistoricalBestSell = 66

    RecommendCategoryLatestHotGroup = 10001 

    DefaultFixedCount = -1
)

type AlgorithmAllocation struct {
    Algorithm     int32 // 算法
    FixedQuantity int32 // 固定数量
}

type mockDomainRecommendService struct{}

func (r *mockDomainRecommendService) getAlgorithmAllocationSchemesV1(categoryId int64, enablePersonalizedService bool, index int) (schemes []*AlgorithmAllocation) {
    if !enablePersonalizedService {
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount})
        return schemes
    }
    if categoryId == RecommendCategoryLatestHotGroup {
        if index < 1 {
            // 第0页需要强行随机插入三个卖的最好的物品
            schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmHistoricalBestSell, FixedQuantity: 3})
        }
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmRankingLatestHot, FixedQuantity: 10})
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmRankingBestSell, FixedQuantity: 10})
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount})
        return schemes
    }

    if categoryId > 0 {
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmItemCf, FixedQuantity: DefaultFixedCount})
        schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmContentBased, FixedQuantity: DefaultFixedCount})
        return
    }

    schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmItemCf, FixedQuantity: DefaultFixedCount})
    schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmContentBased, FixedQuantity: DefaultFixedCount})
    schemes = append(schemes, &AlgorithmAllocation{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount})
    return schemes
}

```



优化后的代码：

```go
/ AllocationStrategy defines the interface for allocation strategies
type AllocationStrategy interface {
	Allocate() []*AlgorithmAllocation
}

// RandomAllocationStrategy handles non-personalized allocation
type RandomAllocationStrategy struct{}

func (s *RandomAllocationStrategy) Allocate() []*AlgorithmAllocation {
	return []*AlgorithmAllocation{
		{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount},
	}
}

// LatestHotGroupStrategy handles the Latest Hot Group category allocation
type LatestHotGroupStrategy struct {
	Index int
}

func (s *LatestHotGroupStrategy) Allocate() []*AlgorithmAllocation {
	var allocations []*AlgorithmAllocation
	if s.Index < 1 {
		allocations = append(allocations, &AlgorithmAllocation{Algorithm: AlgorithmHistoricalBestSell, FixedQuantity: 3})
	}
	allocations = append(allocations,
		&AlgorithmAllocation{Algorithm: AlgorithmRankingLatestHot, FixedQuantity: 10},
		&AlgorithmAllocation{Algorithm: AlgorithmRankingBestSell, FixedQuantity: 10},
		&AlgorithmAllocation{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount},
	)
	return allocations
}

// CategoryBasedStrategy handles allocations for specific categories
type CategoryBasedStrategy struct{}

func (s *CategoryBasedStrategy) Allocate() []*AlgorithmAllocation {
	return []*AlgorithmAllocation{
		{Algorithm: AlgorithmItemCf, FixedQuantity: DefaultFixedCount},
		{Algorithm: AlgorithmContentBased, FixedQuantity: DefaultFixedCount},
	}
}

// DefaultPersonalizedStrategy handles default personalized allocations
type DefaultPersonalizedStrategy struct{}

func (s *DefaultPersonalizedStrategy) Allocate() []*AlgorithmAllocation {
	return []*AlgorithmAllocation{
		{Algorithm: AlgorithmItemCf, FixedQuantity: DefaultFixedCount},
		{Algorithm: AlgorithmContentBased, FixedQuantity: DefaultFixedCount},
		{Algorithm: AlgorithmRandom, FixedQuantity: DefaultFixedCount},
	}
}

// getAllocationStrategy selects the appropriate allocation strategy
func (r *mockDomainRecommendService) getAllocationStrategy(categoryId int64, enablePersonalizedService bool, index int) AllocationStrategy {
	if !enablePersonalizedService {
		return &RandomAllocationStrategy{}
	}

	if categoryId == RecommendCategoryLatestHotGroup {
		return &LatestHotGroupStrategy{Index: index}
	}

	if categoryId > 0 {
		return &CategoryBasedStrategy{}
	}

	return &DefaultPersonalizedStrategy{}
}

// getAlgorithmAllocationSchemes refactored function using Strategy Pattern
func (r *mockDomainRecommendService) getAlgorithmAllocationSchemesV2(categoryId int64, enablePersonalizedService bool, index int) []*AlgorithmAllocation {
	strategy := r.getAllocationStrategy(categoryId, enablePersonalizedService, index)
	return strategy.Allocate()
}

```





**其实你说优化前的代码很复杂吗？**其实没有，但有两个问题



从可读性来说：我阅读的时候，各个分配方案都在一个函数里面，我要留意他们是否有依赖关系。

从扩展性来说：一方面如果我后面新加场景，新增分配方案，这个函数里面的if会越来越多，会不好测试不好维护。另外一方面如果需要组合不同分配方案的时候，会出现重复代码。



而优化后的代码，虽然整体代码量上来了，但可读性上你可以看出来每次就是返回一个策略，逻辑清晰。其次后面需要新增策略也不会导致复杂度上升，且不同策略之间可以进行组合。



然后 go 语言还可以换一种函数式的策略模式。

## golang的特别优化
go 还可以用下面这种函数式和接口混用的策略模式。

备注：这里可以不用声明接口也可以实现，实现接口是为了更清晰，以及策略可以定义一个结构体，而不仅仅限制在函数。

```go
package main

import "fmt"

// 定义RouteStrategy为一个接口，包含CalculateTime方法
type RouteStrategy interface {
	CalculateTime(origin, destination string) int
}

// 使用函数类型作为策略
type StrategyFunc func(origin, destination string) int

// 实现RouteStrategy接口的CalculateTime方法
func (sf StrategyFunc) CalculateTime(origin, destination string) int {
	return sf(origin, destination)
}

// 实现三种策略：步行、骑行、驾车

func WalkStrategyFunc(origin, destination string) int {
	// 假设固定耗时30分钟
	return 30
}

func BikeStrategyFunc(origin, destination string) int {
	// 假设固定耗时15分钟
	return 15
}

func DriveStrategyFunc(origin, destination string) int {
	// 假设固定耗时10分钟
	return 10
}

// 路线规划器
type RoutePlanner struct {
	strategy RouteStrategy
}

// 设置策略
func (rp *RoutePlanner) SetStrategy(strategy RouteStrategy) {
	rp.strategy = strategy
}

// 估算出行时间
func (rp *RoutePlanner) EstimateTime(origin, destination string) int {
	return rp.strategy.CalculateTime(origin, destination)
}

func main() {
	planner := &RoutePlanner{}

	// 使用步行策略
	walkStrategy := StrategyFunc(WalkStrategyFunc)
	planner.SetStrategy(walkStrategy)
	fmt.Println("Walk Time:", planner.EstimateTime("Home", "School"))

	// 使用骑行策略
	bikeStrategy := StrategyFunc(BikeStrategyFunc)
	planner.SetStrategy(bikeStrategy)
	fmt.Println("Bike Time:", planner.EstimateTime("Home", "School"))

	// 使用驾车策略
	driveStrategy := StrategyFunc(DriveStrategyFunc)
	planner.SetStrategy(driveStrategy)
	fmt.Println("Drive Time:", planner.EstimateTime("Home", "Work"))
}

```



优化的逻辑是使用策略模式，接下来再稍微了解一下策略模式

# 策略模式
## 什么是策略模式？
> **策略模式**是一种行为设计模式， 它能让你定义一系列算法， 并将每种算法分别放入独立的类中， 以使算法的对象能够相互替换。
>



就像我们上面的业务场景，不同来源的用户，我们需要为他分配不同的分配方案。使用策略模式以后，每一种分配方案都有一个单独的结构体（类），各自隔离，且可以组合使用。



## 应用场景
1， **当你有许多仅在执行某些行为时略有不同的相似类时， 可使用策略模式。可以减少重复代码**

**2， 如果算法在上下文的逻辑中不是特别重要， 使用该模式能将类的业务逻辑与其算法实现细节隔离开来。**

## 实现方式
> 1. 从上下文类中找出修改频率较高的算法 （也可能是用于在运行时选择某个算法变体的复杂条件运算符）。
> 2. 声明该算法所有变体的通用策略接口。
> 3. 将算法逐一抽取到各自的类中， 它们都必须实现策略接口。
> 4. 在上下文类中添加一个成员变量用于保存对于策略对象的引用。 然后提供设置器以修改该成员变量。 上下文仅可通过策略接口同策略对象进行交互， 如有需要还可定义一个接口来让策略访问其数据。
> 5. 客户端必须将上下文类与相应策略进行关联， 使上下文可以预期的方式完成其主要工作。
>



## 策略模式的优缺点
> 说一下优点
>
> 1. 你可以在运行时切换对象内的算法。
> 2.  你可以将算法的实现和使用算法的代码隔离开来。
> 3.  你可以使用组合来代替继承。
> 4.  _开闭原则_。 你无需对上下文进行修改就能够引入新的策略。
>
> 说一下缺点：
>
> 1.  如果你的算法极少发生改变， 那么没有任何理由引入新的类和接口。 使用该模式只会让程序过于复杂。
> 2.  客户端必须知晓策略间的不同——它需要选择合适的策略。
> 3.  许多现代编程语言支持函数类型功能， 允许你在一组匿名函数中实现不同版本的算法。 这样， 你使用这些函数的方式就和使用策略对象时完全相同， 无需借助额外的类和接口来保持代码简洁。
>



这里注意一下缺点，确实对比上面我的改动，也发现了代码确实复杂了。

## 和其他设计模式之间的关系
以后补充，或者欢迎大家补充



参考文章：[https://refactoringguru.cn/design-patterns/strategy](https://refactoringguru.cn/design-patterns/strategy)



