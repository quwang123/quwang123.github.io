---
title: 记录一个 golang 切片引发的 bug
description: 记录一个 golang 切片引发的 bug
date: 2024-10-11
tags:
  - 后端开发
  - golang
---



# 一个有一点点奇怪的bug
**背景：**

同事反馈，我们有一个兴趣列表的选择页面偶尔会返回重复的选项。同时描述了他的操作过程

1，第一次打开兴趣列表，所有兴趣正常展示，同时选择自己感兴趣的标签。

2，第二次打开兴趣列表，展示的兴趣列表中有重复的兴趣。



正常的选择页面如下，异常的则是这些标签重复出现，比如出现了两个「跨境与电商」

![]("../images/后端开发/08-1.png")



**研发处理过程：**

1，首先尝试复现bug，发现不能复现。

2，怀疑同事发生 bug 的时候是内部做了调整，导致无法复现，所以后端介入查看这个接口的配置文件是否调整？查看这里是否有内部同事在后台调整了这里的标签？发现均无异常

3，此时怀疑方向从后端调整为客户端，根据日志找到同事使用的老版本 APP，再次进行复现，发现还是无法复现。

4，此时就有点迷茫了，因为考虑到这个偶发+影响不是特别大+已经花了不少时间分析，所以先搁置，等后面有类似反馈再跟进。



处理到这里，其实我们肯定这是一个 bug，但是碍于不能复现，且 review 代码没能直接发现问题，所以只能先放一下（其他 bug 还等着修=-=）





第二次复现：

同事又复现了，同时接到反馈第一时间我也尝试复现，发现确实可以复现。同时立马抓包，发现是后端返回的数据有问题。



已知信息

+ 这个 bug 是后端接口返回数据的问题。
+ 这个 bug 是偶发的，有时候会触发，有时候不能触发。
+ 每次重复的兴趣标签不一样。



# 开始重点 review 后端代码
大家可以看一下这个代码哪里有问题

```plain
package main

import (
	"fmt"
	"github.com/patrickmn/go-cache"
	"math/rand"
	"testing"
	"time"
)

var (
	c = cache.New(5*time.Minute, 10*time.Minute)
)

type Tag struct {
	ID   int
	Name string
}

func getTagListFromMysql() []Tag {
	return []Tag{
		{ID: 1, Name: "心理"},
		{ID: 2, Name: "副业赚钱"},
		{ID: 3, Name: "行业交流"},
	}
}

func GetTagList() []Tag {
	cacheKey := "myStructArrayKey"
	cacheTagList, found := c.Get(cacheKey)
	if found {
		tagList, ok := cacheTagList.([]Tag)
		if !ok {
			panic("Expected cached struct array to be of type []Tag")
		}
		return tagList
	}

	tagList := getTagListFromMysql()
	c.Set(cacheKey, tagList, cache.DefaultExpiration)
	return tagList
}

func TestCacheStructArray(t *testing.T) {
	tagList := GetTagList()
	rand.Shuffle(len(tagList), func(i, j int) {
		tagList[i], tagList[j] = tagList[j], tagList[i]
	})
	fmt.Println(tagList)
}

```





这里核心问题就是：**切片传递的是引用。当我们使用内存缓存时，相当于所有客户端获取到的这个数据都是同一份数据。**



放到我们这个业务场景里面，就是当这个接口访问数量大的时候，可能发生其中一个协程在对这个数组做 shuffle，而另外一个协程正在遍历这个切片！！！



这个时候就会发现遍历的出来重复数据（因为 shuffle 那里改动了你正在遍历的切片）



# 最后复盘一下
第一个其实是对基建不够熟悉。其实按道理第一次我分析的时候就可以通过微服务的 access 日志肯定是后端的问题。（但之前一直以为我们不会记录返回详情）



第二个是在自己看代码的时候头绪的时候，应该早点用小黄鸭调试法（第二天就是用小黄鸭调试法发现问题）



第三个是还是自己最近分析 bug 少了，忘记了一些golang的特性，比如面试一直背的切片是引用，底层是公用数组...



