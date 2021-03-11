---
title: golang 泛型预览
date: 2021-03-10 23:33:00
tags: [golang, generic]
---

# golang 泛型预览

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
    fmt.Printf("%T=%[1]v\n", gMax(1.0, 2.2))
    fmt.Printf("%T=%[1]v\n", gMax[int](3.0, 4.0))
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

`-G=3` 可以从 `go tool compile -help` , `-G` 是 `accept generic code`

这样就能体验go的泛型了

## 跟java泛型区别 TODO

把多余的输出干掉，方便看汇编

``` go
func main() {
    gMax(1, 2)
    gMax(1.0, 2.2)
}
```

来，编译成汇编

``` sh
go run -gcflags="-G=3 -S" main.go
```

结果

``` sh
# command-line-arguments
"".main STEXT size=103 args=0x0 locals=0x20 funcid=0x0
	0x0000 00000 (/home/goroot/bin/main.go:10)	TEXT	"".main(SB), ABIInternal, $32-0
	0x0000 00000 (/home/goroot/bin/main.go:10)	MOVQ	(TLS), R14
	0x0009 00009 (/home/goroot/bin/main.go:10)	CMPQ	SP, 16(R14)
	0x000d 00013 (/home/goroot/bin/main.go:10)	PCDATA	$0, $-2
	0x000d 00013 (/home/goroot/bin/main.go:10)	JLS	93
	0x000f 00015 (/home/goroot/bin/main.go:10)	PCDATA	$0, $-1
	0x000f 00015 (/home/goroot/bin/main.go:10)	SUBQ	$32, SP
	0x0013 00019 (/home/goroot/bin/main.go:10)	MOVQ	BP, 24(SP)
	0x0018 00024 (/home/goroot/bin/main.go:10)	LEAQ	24(SP), BP
	0x001d 00029 (/home/goroot/bin/main.go:10)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (/home/goroot/bin/main.go:10)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (/home/goroot/bin/main.go:11)	MOVQ	$1, (SP)
	0x0025 00037 (/home/goroot/bin/main.go:11)	MOVQ	$2, 8(SP)
	0x002e 00046 (/home/goroot/bin/main.go:11)	PCDATA	$1, $0
    ## CALL
	0x002e 00046 (/home/goroot/bin/main.go:11)	CALL	"".gMax[int](SB)
	0x0033 00051 (/home/goroot/bin/main.go:12)	MOVSD	$f64.3ff0000000000000(SB), X0
	0x003b 00059 (/home/goroot/bin/main.go:12)	MOVSD	X0, (SP)
	0x0040 00064 (/home/goroot/bin/main.go:12)	MOVSD	$f64.400199999999999a(SB), X0
	0x0048 00072 (/home/goroot/bin/main.go:12)	MOVSD	X0, 8(SP)
    ## CALL
	0x004e 00078 (/home/goroot/bin/main.go:12)	CALL	"".gMax[float64](SB)
	0x0053 00083 (/home/goroot/bin/main.go:13)	MOVQ	24(SP), BP
	0x0058 00088 (/home/goroot/bin/main.go:13)	ADDQ	$32, SP
	0x005c 00092 (/home/goroot/bin/main.go:13)	RET
	0x005d 00093 (/home/goroot/bin/main.go:13)	NOP
	0x005d 00093 (/home/goroot/bin/main.go:10)	PCDATA	$1, $-1
	0x005d 00093 (/home/goroot/bin/main.go:10)	PCDATA	$0, $-2
	0x005d 00093 (/home/goroot/bin/main.go:10)	NOP
	0x0060 00096 (/home/goroot/bin/main.go:10)	CALL	runtime.morestack_noctxt(SB)
	0x0065 00101 (/home/goroot/bin/main.go:10)	PCDATA	$0, $-1
	0x0065 00101 (/home/goroot/bin/main.go:10)	JMP	0
	0x0000 64 4c 8b 34 25 00 00 00 00 49 3b 66 10 76 4e 48  dL.4%....I;f.vNH
	0x0010 83 ec 20 48 89 6c 24 18 48 8d 6c 24 18 48 c7 04  .. H.l$.H.l$.H..
	0x0020 24 01 00 00 00 48 c7 44 24 08 02 00 00 00 e8 00  $....H.D$.......
	0x0030 00 00 00 f2 0f 10 05 00 00 00 00 f2 0f 11 04 24  ...............$
	0x0040 f2 0f 10 05 00 00 00 00 f2 0f 11 44 24 08 e8 00  ...........D$...
	0x0050 00 00 00 48 8b 6c 24 18 48 83 c4 20 c3 0f 1f 00  ...H.l$.H.. ....
	0x0060 e8 00 00 00 00 eb 99                             .......
	rel 5+4 t=16 TLS+0
	rel 47+4 t=7 "".gMax[int]+0
	rel 55+4 t=15 $f64.3ff0000000000000+0
	rel 68+4 t=15 $f64.400199999999999a+0
	rel 79+4 t=7 "".gMax[float64]+0
	rel 97+4 t=7 runtime.morestack_noctxt+0
"".gMax[int] STEXT nosplit size=27 args=0x18 locals=0x0 funcid=0x0
    ## 声明
	0x0000 00000 (/home/goroot/bin/main.go:3)	TEXT	"".gMax[int](SB), NOSPLIT|ABIInternal, $0-24
	0x0000 00000 (/home/goroot/bin/main.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/home/goroot/bin/main.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/home/goroot/bin/main.go:4)	MOVQ	"".b+16(SP), AX
	0x0005 00005 (/home/goroot/bin/main.go:4)	MOVQ	"".a+8(SP), CX
	0x000a 00010 (/home/goroot/bin/main.go:4)	CMPQ	AX, CX
	0x000d 00013 (/home/goroot/bin/main.go:4)	JGE	21
	0x000f 00015 (/home/goroot/bin/main.go:5)	MOVQ	CX, "".~r2+24(SP)
	0x0014 00020 (/home/goroot/bin/main.go:5)	RET
	0x0015 00021 (/home/goroot/bin/main.go:7)	MOVQ	AX, "".~r2+24(SP)
	0x001a 00026 (/home/goroot/bin/main.go:7)	RET
	0x0000 48 8b 44 24 10 48 8b 4c 24 08 48 39 c8 7d 06 48  H.D$.H.L$.H9.}.H
	0x0010 89 4c 24 18 c3 48 89 44 24 18 c3                 .L$..H.D$..
"".gMax[float64] STEXT nosplit size=33 args=0x18 locals=0x0 funcid=0x0
    ## 声明
	0x0000 00000 (/home/goroot/bin/main.go:3)	TEXT	"".gMax[float64](SB), NOSPLIT|ABIInternal, $0-24
	0x0000 00000 (/home/goroot/bin/main.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/home/goroot/bin/main.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/home/goroot/bin/main.go:4)	MOVSD	"".a+8(SP), X0
	0x0006 00006 (/home/goroot/bin/main.go:4)	MOVSD	"".b+16(SP), X1
	0x000c 00012 (/home/goroot/bin/main.go:4)	UCOMISD	X1, X0
	0x0010 00016 (/home/goroot/bin/main.go:4)	JLS	25
	0x0012 00018 (/home/goroot/bin/main.go:5)	MOVSD	X0, "".~r2+24(SP)
	0x0018 00024 (/home/goroot/bin/main.go:5)	RET
	0x0019 00025 (/home/goroot/bin/main.go:7)	MOVSD	X1, "".~r2+24(SP)
	0x001f 00031 (/home/goroot/bin/main.go:7)	NOP
	0x0020 00032 (/home/goroot/bin/main.go:7)	RET
	0x0000 f2 0f 10 44 24 08 f2 0f 10 4c 24 10 66 0f 2e c1  ...D$....L$.f...
	0x0010 76 07 f2 0f 11 44 24 18 c3 f2 0f 11 4c 24 18 90  v....D$.....L$..
	0x0020 c3                                               .
```

emm，跟 Java 类型擦除不一样，go 似乎是通过代码生成来实现泛型的，运行效率可以保证了，编译产物增大也是可以接受吧。