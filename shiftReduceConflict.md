# A Shift Reduce Conflict Resolving in SQL Parser Generating
## 问题
用goyacc编译SQL语法文件时提示冲突:
`conflicts: 1 shift/reduce`  
查看中间调试文件`y.output`找到如下信息
```console
38: shift/reduce conflict (shift 112(0), red'n 146(0)) on COLLATE
state 38
	bit_expression:  simple_expression.    (146)
	simple_expression:  simple_expression.COLLATE charset

	COLLATE  shift 112
	.  reduce 146 (src line 938)
```

## 诊断
第一行提示我们在`COLLATE`的时候遇到shift和reduce的冲突，看下面状态38的点语法表示：
存在两个状态一个是编号146的reduce转换`simple_expression.`到`bit_expression`,
另一个是`COLLATE`的shift操作，我们看状态112
```console
state 112
	simple_expression:  simple_expression COLLATE.charset

	ID  shift 176
	STRING  shift 177
	.  error

	charset  goto 175

state 175
  	simple_expression:  simple_expression COLLATE charset.    (149)

  	.  reduce 149 (src line 950)

```
其实就是形成一个包含`COLLATE`的simple_expression.  
根据goyacc的规则shift/reduce冲突时缺省的操作是shift，那么语法分析器会去看是否有`COLLATE` token，如果有就根据我们的语法
```console
  simple_expression COLLATE charset %prec UNARY
  {
    $$ = &CollateExpr{Expr: $1, Charset: $3}
  }
```
从`CollateExpr`得到simple_expression，然后执行reduce规则146;如果没有就直接执行reduce规则146直接得到`bit_expression`，总之都是reduce到bit_expression,这是我们想要的结果，从上面的分析可以看出缺省的语法冲突解决方法没有问题，暂时不需要理会这个警告

## Reference
[Yacc Theory](https://www.epaperpress.com/lexandyacc/thy.html)  
[Yacc](http://dinosaur.compilertools.net/yacc/)  
[Yacc Parser Conflict Handling](https://www2.cs.arizona.edu/classes/cs453/fall14/DOCS/conflicts.pdf)
