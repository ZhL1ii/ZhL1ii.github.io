---
title: "Go: 闭包（一）"
pubDatetime: 2026-09-09T00:00:00+08:00
description: "以两道简单的 LeetCode 题认识闭包。"
tags:
  - Go
  - Grammar
  - LeetCode
---

## 看两道 LeetCode

二叉树的中序遍历：

```go
func inorderTraversal(root *TreeNode) []int {
	res := []int{}

	var inorder func(node *TreeNode)

	inorder = func(node *TreeNode) {
		if node == nil {
			return
		}

		inorder(node.Left)
		res = append(res, node.Val)
		inorder(node.Right)
	}

	inorder(root)

	return res
}
```

二叉树的直径：

```go
func diameterOfBinaryTree(root *TreeNode) int {
	maxDiameter := 0

	var height func(*TreeNode) int

	height = func(node *TreeNode) int {
		if node == nil {
			return 0
		}

		left := height(node.Left)
		right := height(node.Right)

		maxDiameter = max(maxDiameter, left+right)

		return max(left, right) + 1
	}

	height(root)

	return maxDiameter
}
```

里面涉及到了 Golang 中非常重要的语法：

先声明一个变量：

```go
var inorder func(node *TreeNode)
```

然后再往里面塞一个函数：

```go
inorder = func(node *TreeNode) {
	...
}
```

这个函数里面可以递归调用自己：

```go
inorder(node.Left)
```

同时，它还能直接访问外面的变量：`res`

或者 `maxDiameter`

这背后主要涉及几个概念：函数类型、函数值、匿名函数、闭包、递归闭包

------

## 一、函数类型

我们平时最熟悉的函数写法是：

```go
func add(a int, b int) int {
	return a + b
}
```

在 Go 中，函数其实也是一种值。

既然是值，就意味着它可以：

- 赋值给变量
- 作为参数传递
- 作为返回值返回
- 被保存在结构体字段中

比如：

```go
var f func(int, int) int
```

这里声明了一个变量 `f`，`f` 的类型是：

```go
func(int, int) int
```

意思是：接收两个 int，返回一个 int

于是我们可以给它赋值：

```go
f = func(a int, b int) int {
	return a + b
}
```

然后像普通函数一样调用：

```go
result := f(1, 2)
```

所以：

```go
var inorder func(node *TreeNode)
```

只是：声明一个叫 inorder 的变量，这个变量的类型是 func(*TreeNode) 也就是它保存一个接收 *TreeNode、没有返回值的函数。

------

## 二、`func(...) { ... }` 

接下来再看下面的代码：

```go
inorder = func(node *TreeNode) {
	...
}
```

右边这一坨：

```go
func(node *TreeNode) {
	...
}
```

在 Go 中叫作**函数字面量 (function literal)**，它表示一个匿名函数。这个表达式求值后会产生一个函数值，然后这个函数值被赋给变量 `inorder`。

例如：

```go
f := func() {
	fmt.Println("hello")
}

f()
```

也可以直接调用：

```go
func() {
	fmt.Println("hello")
}()
```

最后这一对 `()` 表示立即调用这个匿名函数。

------

## 三、匿名函数和闭包

```go
f := func(a int, b int) int {
	return a + b
}
```

这个函数只使用自己的参数 `a`、`b`，没有捕获任何外层变量。严格按照 Go 语言规范，函数字面量**属于闭包**。

不过我们平时讨论“闭包”时，通常更关注下面这种**捕获外部变量**的情况。

```go
sum := 0

add := func(x int) {
	sum += x
}
```

这里匿名函数内部使用了一个并不属于自己的变量：`sum`，`sum` 是在外层函数中定义的，但内部函数依然能够读取甚至修改它，这就是闭包最核心的地方。可以粗略理解为：`闭包 = 函数 + 它所引用的外部环境`

------

## 四、中序遍历

重新看中序遍历：

```go
func inorderTraversal(root *TreeNode) []int {
	res := []int{}

	var inorder func(node *TreeNode)

	inorder = func(node *TreeNode) {
		if node == nil {
			return
		}

		inorder(node.Left)
		res = append(res, node.Val)
		inorder(node.Right)
	}

	inorder(root)

	return res
}
```

这里：

```go
res := []int{}
```

属于 `inorderTraversal` 函数，但是内部函数 `inorder` 直接使用了它：

```go
res = append(res, node.Val)
```

于是 `inorder` 就形成了一个闭包。

它捕获了：`res` 这个变量。

于是递归过程中，不管调用多少次：

```go
inorder(node.Left)
inorder(node.Right)
```

它们都在操作同一个 `res`，这也是为什么我们不需要写成：

```go
func inorder(node *TreeNode, res *[]int)
```

如果不用闭包，确实可以显式传递结果：

```go
func inorder(node *TreeNode, res *[]int) {
	if node == nil {
		return
	}

	inorder(node.Left, res)
	*res = append(*res, node.Val)
	inorder(node.Right, res)
}
```

这种写法没有问题，但是闭包让我们把 `res` 从递归函数的参数列表中拿掉了。于是递归函数本身只关心当前节点 node

这也是闭包在算法题里很好用的一个原因：它可以帮助我们保存“辅助状态”。

------

## 五、二叉树直径

再看直径问题：

```go
func diameterOfBinaryTree(root *TreeNode) int {
	maxDiameter := 0

	var height func(*TreeNode) int

	height = func(node *TreeNode) int {
		if node == nil {
			return 0
		}

		left := height(node.Left)
		right := height(node.Right)

		maxDiameter = max(maxDiameter, left+right)

		return max(left, right) + 1
	}

	height(root)

	return maxDiameter
}
```

这里很有意思，因为整个算法其实同时存在两条数据流，第一条是树高。

每一个节点需要从左右子树拿到高度：

```go
left := height(node.Left)
right := height(node.Right)
```

然后向父节点返回：

```go
return max(left, right) + 1
```

这是递归函数正常的返回值，但与此同时，我们还想统计整棵树的最大直径。

对于当前节点，`left + right` 就是经过当前节点的最长路径长度。

于是：

```go
maxDiameter = max(maxDiameter, left+right)
```

在遍历过程中不断更新全局的最优结果。

------

## 六、不用闭包

不用闭包也完全可以写，例如可以多写一个函数：

```go
func height(node *TreeNode, maxDiameter *int) int {
	if node == nil {
		return 0
	}

	left := height(node.Left, maxDiameter)
	right := height(node.Right, maxDiameter)

	*maxDiameter = max(*maxDiameter, left+right)

	return max(left, right) + 1
}
```

然后外层：

```go
func diameterOfBinaryTree(root *TreeNode) int {
	maxDiameter := 0

	height(root, &maxDiameter)

	return maxDiameter
}
```

两种方式逻辑完全一样，区别在于闭包把：`maxDiameter` 变成了函数隐式可访问的上下文。

因此：`height(node.Left)`

看起来更加纯粹，递归函数参数只包含与递归结构真正相关的 `node`，而 `maxDiameter` 被作为辅助状态放在外层函数中，这就是闭包在算法题里的一个很实用的点。

------

## 七、递归闭包

还有一个非常值得注意的语法点，为什么用 `var` 定义函数：

```go
var height func(*TreeNode) int

height = func(node *TreeNode) int {
	left := height(node.Left)
	...
}
```

而不是直接用 `:=` ？

```go
height := func(node *TreeNode) int {
	left := height(node.Left)
	...
}
```

原因在于变量的作用域，在：

```go
height := func(...) {
	...
}
```

右边表达式正在创建函数值的时候，`height` 这个变量本身还没有完成声明，但是函数体里又需要调用 `height`，于是就会产生问题报错。所以递归闭包常见的写法是：

```go
var height func(*TreeNode) int
```

先声明变量，这时 `height` 已经存在，只不过它目前还是函数类型的零值 `nil`，接下来再给它赋值：

```go
height = func(node *TreeNode) int {
	...
	height(node.Left)
	...
}
```

当这个函数真正开始执行的时候，`height` 已经指向这个函数自身，于是就可以递归调用了。

------

## 八、函数携带状态

如果只从语法角度看，闭包可能只是：`外层变量 + 匿名函数`，但真正理解以后，会发现一个很重要的思想，函数不一定只是行为，它可以携带一部分状态。

例如：

```go
func makeCounter() func() int {
	count := 0

	return func() int {
		count++
		return count
	}
}
```

使用：

```go
counter := makeCounter()

fmt.Println(counter())
fmt.Println(counter())
fmt.Println(counter())
```

会得到输出：

```text
1
2
3
```

虽然 `makeCounter` 已经执行结束了，但是返回出去的函数仍然能够继续访问 count，也就是说，`count` 的生命周期并没有简单地随着 `makeCounter` 返回而结束，只要闭包还需要它，这个变量就必须继续存在。

这也是闭包名字中“闭”的含义之一：函数和它依赖的外部环境绑定到了一起。
