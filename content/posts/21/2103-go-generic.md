---
title: golang 泛型
date: 2021-03-10 23:33:00
tags: [golang, generic]
---

# golang 泛型

终于，"大道至简"的go，泛型终于合并到master分支上去了。那么来了，让我们来搞一下

## 首先需要从源码编译golang

省略，把CGO禁了就好编译了

## 写个泛型 MAX

``` go
package main

import "fmt"

func gMax[T interface {type int, float64}](a T, b T) T {
    if T(a) > T(b) {
        return T(a)
    }
    return T(b)
}

func main() {
    fmt.Printf("%T=%[1]v\n", gMax(1, 2))
    fmt.Printf("%T=%[1]v\n",gMax(1.0, 2.2))
    fmt.Printf("%T=%[1]v\n",gMax[int](3.0, 4.0))
}
```

运行参数

``` sh
go run -gcflags="-G=3" main.go
```

结果

``` sh
int=2
float64=2.2
int=4
```

`-G=3` 可以从 `go tool compile -help` , -G 是 accept generic code

这样就能体验go的泛型了

## 跟java泛型区别 TODO

TODO
