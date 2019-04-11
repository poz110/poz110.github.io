## 浅谈AST(抽象语法树)

### 什么是AST？

AST是Abstract Syntax Tree（抽象语法树）的缩写。
传说中的程序员三大浪漫是编译原理、图形学、操作系统，不把AST玩转，显得逼格不够，而本文目标就是为你揭示AST在现代化JavaScript项目中的应用。

按照语法规则书写的代码，是用来让开发者可阅读、可理解的。对编译器等工具来讲，它可以理解的就是抽象语法树了
```markdown
var a = 42
function addA(d){
  return a + d;
}
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### AST是如何产生的？

生成抽象语法树需要经过两个阶段：
- 分词（tokenize）：将整个代码字符串分割成 语法单元 数组
- 语义分析(parse)：在分词结果的基础之上分析 语法单元之间的关系

#### 分词
首先解释一下什么是语法单元：语法单元是被解析语法当中具备实际意义的最小单元，通俗点说就是类似于自然语言中的词语。
那么回到代码的解析当中，JS代码有哪些语法单元呢？大致有以下这些（其他语言也许类似但通常都有区别）：

- 空白：JS中连续的空格、换行、缩进等这些如果不在字符串里，就没有任何实际逻辑意义，所以把连续的空白符直接组合在一起作为一个语法单元。
- 注释：行注释或块注释，虽然对于人类来说有意义，但是对于计算机来说知道这是个“注释”就行了，并不关心内容，所以直接作为一个不可再拆的语法单元
- 字符串：对于机器而言，字符串的内容只是会参与计算或展示，里面再细分的内容也是没必要分析的
- 数字：JS语言里就有16、10、8进制以及科学表达法等数字表达语法，数字也是个具备含义的最小单元
- 标识符：没有被引号扩起来的连续字符，可包含字母、_、$、及数字（数字不能作为开头）。标识符可能代表一个变量，或者true、false这种内置常量、也可能是if、return、function这种关键字，是哪种语义，分词阶段并不在乎，只要正确切分就好了。
- 运算符：+、-、*、/、>、<等等
- 括号：(...)可能表示运算优先级、也可能表示函数调用，分词阶段并不关注是哪种语义，只把“(”或“)”当做一种基本语法单元
- 还有其他：如中括号、大括号、分号、冒号、点等等不再一一列举
### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
