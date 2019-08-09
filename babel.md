## Babel工作原理

### 什么是Babel？

Babel 对于前端开发者来说应该是很熟悉了，日常开发中基本上是离不开它的。
Babel 是一个 JavaScript 编译器,它能将es2015,react等低端浏览器无法识别的语言，进行编译，使它在旧的浏览器或者环境中也能够运行。Babel 的功能很纯粹。我们传递一段源代码给 Babel，然后它返回一串新的代码给我们。就是这么简单，它不会运行我们的代码，也不会去打包我们的代码。

查看 [https://astexplorer.net/](https://astexplorer.net/)将源码生成抽象语法树

### Babel 是如何工作的？

Babel的编译过程跟绝大多数其他语言的编译器大致同理，分为三个阶段：

- 解析: 将代码(其实就是字符串)转换成 AST( 抽象语法树)
- 变换：对抽象语法树进行变换操作（访问 AST 的节点进行变换操作生成新的 AST）
- 生成：根据变换后的抽象语法树再生成代码字符串

具体工作流程如下图所示：
![avatar](/img/babel.png)

#### 解析
babel解析的过程就是生成抽象语法树的过程，需要经过以下两个阶段：

使用 babylon 解析器对输入的源代码字符串进行解析并生成初始 AST（File.prototype.parse）
利用 babel-traverse 这个独立的包对 AST 进行遍历，并解析出整个树的 path，通过挂载的 metadataVisitor 读取对应的元信息，这一步叫 set AST 过程


#### 分词

```markdown
if (1 > 0) {
  alert("if \"1 > 0\"");
}
```

