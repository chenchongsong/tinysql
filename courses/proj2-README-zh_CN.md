# Parser

## 概览

在 Parser 部分，我们将介绍 TinySQL 是如何将文本转化为 AST 的。

## Parser 简介

可以参考 [TiDB 源码阅读系列之 TiDB SQL Parser 的实现](https://pingcap.com/blog-cn/tidb-source-code-reading-5/)。

## 作业描述

完成 `JoinTable` 的实现，你可以利用 parser test 里失败的测试确定需要补充哪些语法部分。

具体做法：
1. 修改`parser/parser.y`
2. 执行`cd parser && make all`，以重新生成`parser/parser.go`

## 测试

执行`cd parser && go test -v .` 并通过测试 `TestDMLStmt`。

## 评分

通过 `TestDMLStmt` 即可满分。
