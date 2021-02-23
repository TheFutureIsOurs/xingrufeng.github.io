---
layout: post
title: "深入理解go map底层实现及与php map实现对比"
date: 2021-02-23
tags:
 - map解析
 - php map实现解析
 - go map实现解析
---

## 前言

之前在[openresty协程调度对比go协程调度](https://www.imflybird.cn/2021/01/07/openresty%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6%E5%AF%B9%E6%AF%94go%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6/)文章中分析了go程序启动过程(从go程序开始执行到用户的main程序执行前发生了些什么)，今天结合一个简短例子说下go map的底层实现（顺便看下在编译阶段发生了什么），同时对比下php的map实现

# go map实现

** 说明：go版本：1.15.6，平台：linux **


一段简单的代码如下：

```go

package main
import (
    "fmt"
)

func main() {
    m := make(map[string]int64, 53)
	m["a"] = 111111111
	m["b"] = 111111111
	delete(m, "a")
	fmt.Println(m["a"], m["b"])
}

```

为了定位到其底层实现，我们先看下go语言的编译过程。

## 编译

其实，go作为一个编译型语言，go程序会在编译阶段经过一系列处理后最终生成机器码：

### 解析阶段

源码地址：cmd/compile/internal/syntax


> 1.词法分析
> 
> 解析成token，我们可以通过go提供的这两个package:"go/scanner"和"go/token"模拟

```go

func TestToken(t *testing.T) {
	src := []byte(`package main
    import "fmt"

    func main() {
        m := make(map[string]int64, 53)
        m["a"] = 111111111
        fmt.Println(m["a"])
    }
    `)
	var s scanner.Scanner
	fset := token.NewFileSet()
	file := fset.AddFile("", fset.Base(), len(src))
	s.Init(file, src, nil, 0)
	for {
		pos, tok, lit := s.Scan()
		if tok == token.EOF {
			break
		}
		fmt.Printf("%s\t%s\t%q\n", fset.Position(pos), tok, lit)
	}
}

```
最终结果：

```go

1:1	package	"package"
1:9	IDENT	"main"
1:13	;	"\n"
2:5	import	"import"
2:12	STRING	"\"fmt\""
2:17	;	"\n"
4:5	func	"func"
4:10	IDENT	"main"
4:14	(	""
4:15	)	""
4:17	{	""
5:9	IDENT	"m"
5:11	:=	""
5:14	IDENT	"make"
5:18	(	""
5:19	map	"map"
5:22	[	""
5:23	IDENT	"string"
5:29	]	""
5:30	IDENT	"int64"
5:35	,	""
5:37	INT	"53"
5:39	)	""
5:40	;	"\n"
6:9	IDENT	"m"
6:10	[	""
6:11	STRING	"\"a\""
6:14	]	""
6:16	=	""
6:18	INT	"111111111"
6:27	;	"\n"
7:9	IDENT	"fmt"
7:12	.	""
7:13	IDENT	"Println"
7:20	(	""
7:21	IDENT	"m"
7:22	[	""
7:23	STRING	"\"a\""
7:26	]	""
7:27	)	""
7:28	;	"\n"
8:5	}	""
8:6	;	"\n"

```
>
> 2.语法分析
> 将扫描后生成的token转化为抽象语法树，每个go源码文件被构造为独立的语法树。同样，我们可以通过go提供的包：go/parser和go/ast来看下AST

```go

func TestParser(t *testing.T) {
	src := []byte(`package main
    import "fmt"

    func main() {
        m := make(map[string]int64, 53)
        m["a"] = 111111111
        fmt.Println(m["a"])
    }
    `)
	fset := token.NewFileSet()
	file, err := parser.ParseFile(fset, "", src, 0)
	if err != nil {
		log.Fatal(err)
	}
	ast.Print(fset, file)
}

```

输出结果：

```go

 0  *ast.File {
 1  .  Package: 1:1
 2  .  Name: *ast.Ident {
 3  .  .  NamePos: 1:9
 4  .  .  Name: "main"
 5  .  }
 6  .  Decls: []ast.Decl (len = 2) {
 7  .  .  0: *ast.GenDecl {
 8  .  .  .  TokPos: 2:5
 9  .  .  .  Tok: import
10  .  .  .  Lparen: -
11  .  .  .  Specs: []ast.Spec (len = 1) {
12  .  .  .  .  0: *ast.ImportSpec {
13  .  .  .  .  .  Path: *ast.BasicLit {
14  .  .  .  .  .  .  ValuePos: 2:12
15  .  .  .  .  .  .  Kind: STRING
16  .  .  .  .  .  .  Value: "\"fmt\""
17  .  .  .  .  .  }
18  .  .  .  .  .  EndPos: -
19  .  .  .  .  }
20  .  .  .  }
21  .  .  .  Rparen: -
22  .  .  }
23  .  .  1: *ast.FuncDecl {
24  .  .  .  Name: *ast.Ident {
25  .  .  .  .  NamePos: 4:10
26  .  .  .  .  Name: "main"
27  .  .  .  .  Obj: *ast.Object {
28  .  .  .  .  .  Kind: func
29  .  .  .  .  .  Name: "main"
30  .  .  .  .  .  Decl: *(obj @ 23)
31  .  .  .  .  }
32  .  .  .  }
33  .  .  .  Type: *ast.FuncType {
34  .  .  .  .  Func: 4:5
35  .  .  .  .  Params: *ast.FieldList {
36  .  .  .  .  .  Opening: 4:14
37  .  .  .  .  .  Closing: 4:15
38  .  .  .  .  }
39  .  .  .  }
40  .  .  .  Body: *ast.BlockStmt {
41  .  .  .  .  Lbrace: 4:17
42  .  .  .  .  List: []ast.Stmt (len = 3) {
43  .  .  .  .  .  0: *ast.AssignStmt {
44  .  .  .  .  .  .  Lhs: []ast.Expr (len = 1) {
45  .  .  .  .  .  .  .  0: *ast.Ident {
46  .  .  .  .  .  .  .  .  NamePos: 5:9
47  .  .  .  .  .  .  .  .  Name: "m"
48  .  .  .  .  .  .  .  .  Obj: *ast.Object {
49  .  .  .  .  .  .  .  .  .  Kind: var
50  .  .  .  .  .  .  .  .  .  Name: "m"
51  .  .  .  .  .  .  .  .  .  Decl: *(obj @ 43)
52  .  .  .  .  .  .  .  .  }
53  .  .  .  .  .  .  .  }
54  .  .  .  .  .  .  }
55  .  .  .  .  .  .  TokPos: 5:11
56  .  .  .  .  .  .  Tok: :=
57  .  .  .  .  .  .  Rhs: []ast.Expr (len = 1) {
58  .  .  .  .  .  .  .  0: *ast.CallExpr {
59  .  .  .  .  .  .  .  .  Fun: *ast.Ident {
60  .  .  .  .  .  .  .  .  .  NamePos: 5:14
61  .  .  .  .  .  .  .  .  .  Name: "make"
62  .  .  .  .  .  .  .  .  }
63  .  .  .  .  .  .  .  .  Lparen: 5:18
64  .  .  .  .  .  .  .  .  Args: []ast.Expr (len = 2) {
65  .  .  .  .  .  .  .  .  .  0: *ast.MapType {
66  .  .  .  .  .  .  .  .  .  .  Map: 5:19
67  .  .  .  .  .  .  .  .  .  .  Key: *ast.Ident {
68  .  .  .  .  .  .  .  .  .  .  .  NamePos: 5:23
69  .  .  .  .  .  .  .  .  .  .  .  Name: "string"
70  .  .  .  .  .  .  .  .  .  .  }
71  .  .  .  .  .  .  .  .  .  .  Value: *ast.Ident {
72  .  .  .  .  .  .  .  .  .  .  .  NamePos: 5:30
73  .  .  .  .  .  .  .  .  .  .  .  Name: "int64"
74  .  .  .  .  .  .  .  .  .  .  }
75  .  .  .  .  .  .  .  .  .  }
76  .  .  .  .  .  .  .  .  .  1: *ast.BasicLit {
77  .  .  .  .  .  .  .  .  .  .  ValuePos: 5:37
78  .  .  .  .  .  .  .  .  .  .  Kind: INT
79  .  .  .  .  .  .  .  .  .  .  Value: "53"
80  .  .  .  .  .  .  .  .  .  }
81  .  .  .  .  .  .  .  .  }
82  .  .  .  .  .  .  .  .  Ellipsis: -
83  .  .  .  .  .  .  .  .  Rparen: 5:39
84  .  .  .  .  .  .  .  }
85  .  .  .  .  .  .  }
86  .  .  .  .  .  }
87  .  .  .  .  .  1: *ast.AssignStmt {
88  .  .  .  .  .  .  Lhs: []ast.Expr (len = 1) {
89  .  .  .  .  .  .  .  0: *ast.IndexExpr {
90  .  .  .  .  .  .  .  .  X: *ast.Ident {
91  .  .  .  .  .  .  .  .  .  NamePos: 6:9
92  .  .  .  .  .  .  .  .  .  Name: "m"
93  .  .  .  .  .  .  .  .  .  Obj: *(obj @ 48)
94  .  .  .  .  .  .  .  .  }
95  .  .  .  .  .  .  .  .  Lbrack: 6:10
96  .  .  .  .  .  .  .  .  Index: *ast.BasicLit {
97  .  .  .  .  .  .  .  .  .  ValuePos: 6:11
98  .  .  .  .  .  .  .  .  .  Kind: STRING
99  .  .  .  .  .  .  .  .  .  Value: "\"a\""
100  .  .  .  .  .  .  .  .  }
101  .  .  .  .  .  .  .  .  Rbrack: 6:14
102  .  .  .  .  .  .  .  }
103  .  .  .  .  .  .  }
104  .  .  .  .  .  .  TokPos: 6:16
105  .  .  .  .  .  .  Tok: =
106  .  .  .  .  .  .  Rhs: []ast.Expr (len = 1) {
107  .  .  .  .  .  .  .  0: *ast.BasicLit {
108  .  .  .  .  .  .  .  .  ValuePos: 6:18
109  .  .  .  .  .  .  .  .  Kind: INT
110  .  .  .  .  .  .  .  .  Value: "111111111"
111  .  .  .  .  .  .  .  }
112  .  .  .  .  .  .  }
113  .  .  .  .  .  }
114  .  .  .  .  .  2: *ast.ExprStmt {
115  .  .  .  .  .  .  X: *ast.CallExpr {
116  .  .  .  .  .  .  .  Fun: *ast.SelectorExpr {
117  .  .  .  .  .  .  .  .  X: *ast.Ident {
118  .  .  .  .  .  .  .  .  .  NamePos: 7:9
119  .  .  .  .  .  .  .  .  .  Name: "fmt"
120  .  .  .  .  .  .  .  .  }
121  .  .  .  .  .  .  .  .  Sel: *ast.Ident {
122  .  .  .  .  .  .  .  .  .  NamePos: 7:13
123  .  .  .  .  .  .  .  .  .  Name: "Println"
124  .  .  .  .  .  .  .  .  }
125  .  .  .  .  .  .  .  }
126  .  .  .  .  .  .  .  Lparen: 7:20
127  .  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
128  .  .  .  .  .  .  .  .  0: *ast.IndexExpr {
129  .  .  .  .  .  .  .  .  .  X: *ast.Ident {
130  .  .  .  .  .  .  .  .  .  .  NamePos: 7:21
131  .  .  .  .  .  .  .  .  .  .  Name: "m"
132  .  .  .  .  .  .  .  .  .  .  Obj: *(obj @ 48)
133  .  .  .  .  .  .  .  .  .  }
134  .  .  .  .  .  .  .  .  .  Lbrack: 7:22
135  .  .  .  .  .  .  .  .  .  Index: *ast.BasicLit {
136  .  .  .  .  .  .  .  .  .  .  ValuePos: 7:23
137  .  .  .  .  .  .  .  .  .  .  Kind: STRING
138  .  .  .  .  .  .  .  .  .  .  Value: "\"a\""
139  .  .  .  .  .  .  .  .  .  }
140  .  .  .  .  .  .  .  .  .  Rbrack: 7:26
141  .  .  .  .  .  .  .  .  }
142  .  .  .  .  .  .  .  }
143  .  .  .  .  .  .  .  Ellipsis: -
144  .  .  .  .  .  .  .  Rparen: 7:27
145  .  .  .  .  .  .  }
146  .  .  .  .  .  }
147  .  .  .  .  }
148  .  .  .  .  Rbrace: 8:5
149  .  .  .  }
150  .  .  }
151  .  }
152  .  Scope: *ast.Scope {
153  .  .  Objects: map[string]*ast.Object (len = 1) {
154  .  .  .  "main": *(obj @ 27)
155  .  .  }
156  .  }
157  .  Imports: []*ast.ImportSpec (len = 1) {
158  .  .  0: *(obj @ 12)
159  .  }
160  .  Unresolved: []*ast.Ident (len = 4) {
161  .  .  0: *(obj @ 59)
162  .  .  1: *(obj @ 67)
163  .  .  2: *(obj @ 71)
164  .  .  3: *(obj @ 117)
165  .  }
166  }

```

### 类型检查

源码地址：cmd/compile/internal/gc

在这个阶段，会对上面生成的每个文件对应的AST进行节点遍历，对每个节点类型进行检查,去掉声明但未使用，消除dead code等

### 生成中间代码

源码地址：

cmd/compile/internal/gc（AST转换成SSA）
cmd/compile/internal/ssa（SSA多轮传递及规则）

在这个阶段，AST会被转换为SSA（静态单赋值，Static Single Assignment）形式的中间代码，同时关键字，操作符会被转换成runtime函数。

如我们今天要说的map这个关键字及其操作就会在这个阶段进行转换。

我们可以通过执行下面命令产生ssa文件：

> GOSSAFUNC=func go build go file

> 如：GOSSAFUNC=maps go build main.go

** 说明 **

笔者当前版本为1.15.6，如果对main方法执行ssa生成会报panic，


[这个issue](https://github.com/golang/go/issues/40919)会在1.16版本进行修复

生成ssa文件内容如下，选择源码某行，可以看到对应的各阶段过程：

![](https://www.imflybird.cn/static/img/2020/GO/map/ssa.png)


### 最终生成机器码


## 定位

通过编译过程，我们看到map的底层实现是在编译阶段将map关键字及其操作进行了运行时转换。

而我们想要定位到这段代码map的相关实现，可以有多种方式：

> 1.借助go tool工具生成反汇编代码：go build -gcflags "-S" main.go或go tool compile -S main.go
> 
> 2.gdb或dlv调试看反汇编代码
> 
> 3.看中间代码（SSA生成）


由此我们可以定位到，map的底层代码的实现。


数据结构：

```go

// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

```

源码中的解释也比较清晰，在此不再赘述。

不过需要指出的是bmap作为真正存储map数据的地方，为什么只有tophash字段，其key和value呢？其实我们知道map的key和value有多种类型，go作为强类型语言，在定义后其类型就确定了，所以bmap的key和value字段类型就可以在编译阶段确定，避免了穷举，减少代码复杂度。

bmap中key和value类型通过[func bmap(t *types.Type) *types.Type](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/cmd/compile/internal/gc/reflect.go#L82:6)确定。

bmap结构为：

```go

type bmap struct {
	topbits  [8]uint      // key的hash高8位
	keys     [8]keytype	  // key
	elems    [8]elemtype  // value
	overflow uintptr	  // 溢出后下一个bmap
}

```

## go map实现

### 总览

先上总结，大家在学习的时候根据总结来看源码，更容易理解。

> 1.编译器会根据key类型，位数来映射不同的运行时函数，但他们实现大体是一致的。如访问运行时函数：mapaccess，mapaccess1_fast32，mapaccess2_fast64，mapaccess1_faststr等，所以大家看到不同的文章中有对不同函数的解析。
> 
> 2.go使用bmap数据结构来存储key和value
> 
> 3.查找时先定位key属于哪个bucket，再查看位于bmap的哪个位置
> 
> 4.go的map使用开放地址法来解决hash冲突，并放入bmap的overflow字段指向的bmap

> 



回到最开始的代码，来分别看下初始化，赋值，删除，扩容过程

代码比较多，我只讲一些我认为值得提一下或者大家看起来模糊的部分，函数我都会给出链接，大家可以跳进去看源码。

### 初始化

make(map[k]v, hint)或map[k]v{}

初始化函数家族:

[func makemap_small() *hmap](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L292:6) 在 hint <= 8 或不提供hint时编译器会使用这个函数用来初始化

[func makemap64(t *maptype, hint int64, h *hmap) *hmap](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L282:6)

[func makemap(t *maptype, hint int, h *hmap) *hmap](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L303:6)

其实除了上面外，还有可能在汇编层面看不到makemap相关函数，其实编译器还会根据逃逸分析结果，来确定map初始化，[相关函数位置在这里](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/cmd/compile/internal/gc/walk.go#L1206:7)


我们主要看makemap的实现。


> 1.[h.hash0 = fastrand()](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L313:12)  //初始化hash0

使用随机数对hash0进行初始化。看到了吗，其使用了xorshift64进行的随机数生成，我们之前介绍的[分布式唯一ID生成](https://www.imflybird.cn/2020/12/09/%E4%BB%8EMongoID%E8%AE%A8%E8%AE%BA%E5%88%86%E5%B8%83%E5%BC%8F%E5%94%AF%E4%B8%80ID%E7%94%9F%E6%88%90%E6%96%B9%E6%A1%88/)文章中最后的实现也是使用这种高效的随机数生成算法。

而其使用的mp.fastrand，其fastrandseed正是我们在[openresty协程调度对比go协程调度](https://www.imflybird.cn/2021/01/07/openresty%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6%E5%AF%B9%E6%AF%94go%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6/)文章中提到的runtime.args初始化阶段由startupRandomData字段生成的

> 2.[使用func overLoadFactor(count int, B uint8) bool 函数初始化h.B](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L318:6)

B从0开始，不断增加，直到找到B使得 负载因子 * 2^B >= count，其中负载因子为6.5。对于我们上面这段代码，count就是53.得到的B就是4.

> 3.[初始化h.buckets,h.extra.nextOverflow](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L328:29)

至此，初始化完成，如下图所示：
![](https://www.imflybird.cn/static/img/2020/GO/map/init.png)

### 赋值

赋值家族函数：mapassign*。同样，我们结合本例，看下赋值过程。

在本例中，赋值函数为：[func mapassign_faststr(t *maptype, h *hmap, s string) unsafe.Pointer](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L202:6)

1.again标签内：
> [bucket := hash & bucketMask(h.B)](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L224:2) // hash值的低2^h.B-1位来定位到这个key属于哪个bucket
> 
> [b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L228:2) // 这个bucket内的bmap
> 
> [top := tophash(hash)](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L229:2) // hash的高8位为tophash

2.bucketloop标签内：

```go
for {
	for i := uintptr(0); i < bucketCnt; i++ {
		if b.tophash[i] != top {
			if isEmpty(b.tophash[i]) && insertb == nil {
				insertb = b
				inserti = i
			}
			if b.tophash[i] == emptyRest {
				break bucketloop
			}
			continue
		}
		k := (*stringStruct)(add(unsafe.Pointer(b), dataOffset+i*2*sys.PtrSize))
		if k.len != key.len {
			continue
		}
		if k.str != key.str && !memequal(k.str, key.str, uintptr(key.len)) {
			continue
		}
		// already have a mapping for key. Update it.
		inserti = i
		insertb = b
		goto done
	}
	ovf := b.overflow(t)
	if ovf == nil {
		break
	}
	b = ovf
}

```

> [双重for循环寻找合适插入点](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L236:2)

* 遍历顺序：从当前bmap开始，从前往后遍历，如果找不到，会从overflow的bmap中继续查找
* 我们考虑两种情况：
* 1.刚初始化完毕：此时每个bmap的tophash均为emptyRest(即0值)，所以找到了当前的插入点直接跳出bucketloop标签。
* 2.被删除过,则tophash值为emptyOne(即1值)，如当前bmap的tophash为|1|1|1|76|0|0|0|0|，则会优先插入到第一个值为1的位置。此时还要继续遍历看是否有和计算出的top值相等的tophash值没，如果有，且key值相等，则更新插入点。
* 可以看出插入赋值是一个紧凑的过程（有空位时优先往插入前面的）

```go

if insertb == nil {
	// all current buckets are full, allocate a new one.
	insertb = h.newoverflow(t, b)
	inserti = 0 // not necessary, but avoids needlessly spilling inserti
}

```
如果没找到插入点，则会创建溢出的bmap（拉链法）：

* 1.会优先使用之前预分配的bucket（h.extra.nextOverflow）
* 如果当前h.extra.nextOverflow不是预分配的最后一位，则把h.extra.nextOverflow往后移一位（之前介绍数据结构时提到，nextOverflow就是下一个待分配的bmap）
* 如果是最后一位，则把其overflow置为nil（初始化时我们看到最后一个bucket的overflow指向了第一个bucket）,同时h.extra.nextOverflow无可用bucket，置为nil
* 2.如果无可用的nextOverflow，则新建一个

对于m[k]=v，我们看到赋值函数并没有接收v，而是返回了v的内存指针

其实，是编译器生成的汇编指令来完成值的存储的，我们用gdb来看下反汇编代码：

```code shell

   0x00000000004999e0 <+160>:	callq  0x4117c0 <runtime.mapassign_faststr>
   0x00000000004999e5 <+165>:	mov    0x20(%rsp),%rax
   0x00000000004999ea <+170>:	mov    %rax,0x60(%rsp)
   0x00000000004999ef <+175>:	test   %al,(%rax)
   0x00000000004999f1 <+177>:	movq   $0x69f6bc7,(%rax) // 寄存器间接寻址，rax存储的是v的内存地址，把 111111111放入这个内存地址
   0x00000000004999f8 <+184>:	lea    0xe841(%rip),%rax        # 0x4a8240

```

当然在赋值过程中会出现需要扩容的情况，我们后面说

### 访问

访问家族函数：mapaccess*。同样，我们结合本例，看下访问过程。

本例中访问函数为：[func mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L12:6)

上面讲了赋值过程的查找，访问也是一样的过程。访问函数中区分了只有一个bucket和多bucket情况，我们看下多bucket情况。

> 1.同样需要先计算key的hash值
> 
> 2.使用hash值的低2^B-1位定位出key所在bucket
> 
> 3.使用hash值的高8位来匹配tophash
> 
> 4.依次在当前bucket所在的bmap中寻找，没有的话在overflow中寻找，循环寻找，直到结束

### 删除

删除家族函数：mapdelete*。

本例中删除函数为：[func mapdelete_faststr(t *maptype, h *hmap, ky string)](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L297:6)

> 1.删除前同样需要先查找到该key的位置，查找过程同上面的访问过程。
> 
> 2.b.tophash[i] = emptyOne // 删除仅需把tophash值设为1
> 
> 3.每删除一个key，需要查看当前key位置的下一位tophash值是否是emptyRest(即是否是0值)，如果是0值的话，需要把下一位前的emptyOne变更为emptyRest，直到遇到非emptyOne为止。

```go

for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// 从当前bucket的第一个bmap查找，直到当前bmap的前一个停止
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}

```

我们通过一个示例来看下这个过程：

![](https://www.imflybird.cn/static/img/2020/GO/map/delete.png)

### 扩容

由编译器确定选用哪个函数进行增量式扩容： growWork*。

在赋值阶段，可能会触发扩容，触发条件为：

> 1.本次元素如果插入，负载将达到临界点：插入的元素数量 > 负载因子*2^B；此时容量需扩大一倍
> 2.bmap的overflow过多，overflow数量 >= 2^(B&15)的数量。此时容量不变，目的仅是使排除删除造成的空洞，减少overflow，使数据结构更紧凑

我们看下过程：

> 1.扩容开始，使用[func hashGrow(t *maptype, h *hmap)](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L1017:6)进行扩容准备：判断是否需要容量翻倍，并设置h.oldbuckets为h.buckets，[新建buckets和nextOverflow](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map.go#L1027:30)
> 
> 2.增量搬迁：搬迁家族函数为evacuate*,在赋值和删除时，定位到key所属的bucket，则对该bucket内的bmap及其overflow的bmap进行搬迁。
> 
> 对于我们这个例子中，如果不停增加元素直至扩容，则搬迁函数为 [func evacuate_faststr(t *maptype, h *hmap, oldbucket uintptr)](https://sourcegraph.com/github.com/golang/go@release-branch.go1.15/-/blob/src/runtime/map_faststr.go#L393:6) 
> 
> 3.直至搬迁完毕。

搬迁使用的数据结构为:

```go

type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

```

我们可用把上面这个看作一个箱子。

我们看下搬迁过程：

> 1.如果扩容是等量扩容，则使用一个箱子搬迁；如果是双倍扩容，则使用两个箱子进行搬迁。
> 
> 2.从旧bucket中bmap第一个开始，顺序搬迁直至overflow也搬迁完毕停止。
> 
> 3.如果是双倍扩容，则会分为高和低两部分（两部分容量相等），并使用两个exacDst：y和x分别对应高位和低位。计算落到高位还是低位。
> 
> 4.搬迁过程中tophash值不变。这样在赋值和删除时，仍通过高八位定位bmap中的位置。
> 

双倍扩容后，如何确定key所属的bucket呢，我们看到双倍扩容会使用两个exacDst进行分流，我们看到确定是使用x还是y时，计算方式为：

```go

hash := t.hasher(k, uintptr(h.hash0))
if hash&newbit != 0 {
	useY = 1
}

```
而定位bucket计算方式为：

```go
bucket := hash & bucketMask(h.B)
```
其中newbit为原来的2^B，如果得到useY=1，则说明高位为1，最终的bucket为2^h.B（原来的B）+原来的bucket。

举个例子：

扩容前：h.B为4（即100），原来的bucket为：hash&11，假设为2，即10

扩容后：8(1000),现在的bucket为：hash&111

useY=1，则说明hash值后三位为1??，现在的bucket=110,即
2^h.B（原来的B）+原来的bucket=4+2=6。


等量扩容如下图：

![](https://www.imflybird.cn/static/img/2020/GO/map/remap1.png)

双倍扩容如下图：

![](https://www.imflybird.cn/static/img/2020/GO/map/remap2.png)


# PHP的map实现

** php版本:7.4.15 **

其实在php中，在语言层面，准确来说叫数组，而数组底层实现了map的功能。我们知道，map的key定位是需要hash函数来实现O(1)查询的，是无序的，而php中是如何兼顾数组的有序及hash函数的映射呢？

我们看其数据结构，定义在[Zend/zend_types.h文件里](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_types.h#L248:28)：

```php

typedef struct _Bucket {
	zval              val;
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;

typedef struct _zend_array HashTable;

struct _zend_array {
	zend_refcounted_h gc; // 引用计数,gc使用
	union { // 辅助信息
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    flags,
				zend_uchar    _unused,
				zend_uchar    nIteratorsCount,
				zend_uchar    _unused2)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableMask; // 哈希值计算掩码，等于nTableSize的负值(nTableMask = -nTableSize)
	Bucket           *arData; // 指向数组的第一个Bucket
	uint32_t          nNumUsed; // 已使用的Buckets数量
	uint32_t          nNumOfElements; // 有效元素数量
	uint32_t          nTableSize; // 数组总容量,2^n
	uint32_t          nInternalPointer; // 内部遍历使用指针
	zend_long         nNextFreeElement; // 下一个可用数字索引
	dtor_func_t       pDestructor; //析构函数
};

```

在php中，分为两部分：

> 1.数组元素表，每个数组内都是一个Buckets。容量为2^n，插入元素时顺序插入，新插入的元素索引idx为[ht->nNumUsed++](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_hash.c#L775:12),nNumUsed是递增的，表示已使用的Buckets数量。
> 
> 
> 2.索引表，其和数组元素容量等长。存储每个key在数组元素中的索引idx。
> 如果插入时出现了hash冲突，通过拉链法解决hash冲突，新插入的这个元素的val.u2.next为该key所在索引表内存储的索引值。


索引计算方法：nIndex = h | ht->nTableMask;

其中h为key经过times 33算法后的值，对应计算函数为：[zend_inline_hash_func(const char *str, size_t len)](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_string.h#L364:38)

如：一个容量为4，有两个元素的数组结构图总览如下图：

![](https://www.imflybird.cn/static/img/2020/php/map/map.png)

## 添加

对应函数：[zend_hash_index_add_or_update()](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_hash.c#L1043:30)

[zend_hash_str_add_or_update()](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_hash.c#L892:30)


数组元素表顺序插入，更新索引表对应位置数据。值得注意的是，php中并没有对hash冲突进行特殊处理，而是每个key插入数组中的Bucket时，都让其val.u2.next为当前索引表中的值，然后再更新索引表中的值。这样如果没有冲突产生，则val.u2.next值就为-1。无需特殊处理。

如下图，新增一个元素，且出现了hash冲突：

![](https://www.imflybird.cn/static/img/2020/php/map/hash-clock.png)


## 查找

先在索引表中拿到数组元素的idx，然后拿到数据，如果key不等，在其val.u2.next中进行比较，直至结束。

## 删除

先找到该元素，然后将该位置数据val.type置为IS_UNDEF，并不移动数据。

## 扩容

在添加的过程中，如果已使用的buckets数量到达元素数组总容量，则触发扩容，对应函数为[zend_hash_do_resize()](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_hash.c#L1146:27)。

> 1.如果ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)，则不更改容量，仅重新hash
> 2.如果ht->nTableSize < HT_MAX_SIZE，则会进行双倍扩容。此时会申请双倍容量内存，并把之前的数据全部拷贝到新地址，释放旧内存，然后重新hash

重新hash时，如果存在空洞（删除导致元素数组空洞），则需要从前往后，把删除的元素往迁移，把空洞给填满（使数据更紧凑）。对应函数为[zend_hash_rehash(HashTable *ht)](https://sourcegraph.com/github.com/php/php-src@PHP-7.4.15/-/blob/Zend/zend_hash.c#L1171:28)

# 对比

可以看出，
> 1.php使用索引表和元素数组两部分来实现map和数组的功能；而go使用元素数组部分，每个数组内最多可装8个数据，使用负载因子控制扩容，两者的实现各有千秋。
> 
> 2.两者都使用拉链法来处理hash冲突。
> 
> 3.在插入时，如果出现过空洞（删除导致），go更倾向往前插入来使得结构更紧凑，而php是顺序插入，在扩容阶段来处理空洞。
> 
> 4.在扩容时，go使用增量扩容，而php是全量。
> 


## 参考：

[go源码1.15](https://github.com/golang/go/tree/dev.boringcrypto.go1.15)

[golang-notes/map.md](https://github.com/cch123/golang-notes/blob/master/map.md)

[php源码7.4.15](https://github.com/php/php-src/blob/PHP-7.4.15)
