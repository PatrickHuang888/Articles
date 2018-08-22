# A Reduce/Reduce Conflict Analysis in SQL Parser Generating
## 问题
我们看`y.output`
```console
 234: reduce/reduce conflict  (red'ns 131 and 166) on $end
 234: reduce/reduce conflict  (red'ns 131 and 166) on UNION
 234: reduce/reduce conflict  (red'ns 131 and 166) on FROM
 ...
state 234
 	bool_pri:  bool_pri compare subquery.    (131)
 	simple_expression:  subquery.    (166)

```

## 分析
rule 131和166冲突,冲突在`subquery`上可以reduce到bool_pri也可以reduce到simple_expression,
根据规则reduce/reduce冲突的时候按规则顺序执行,那么`bool_pri compare subquery` reduce到
 bool_pri是对的,如果没有不能匹配131的语法则从`subquery.`reduce到simple_expression也是对的。
反之则有问题，如果先执行了`subquery.`得到simple_expression,那么`bool_pri compare simple_expression`是不成立的. 所以以上这个冲突目前不需要调整。
