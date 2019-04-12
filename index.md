## 浅谈AST(抽象语法树)

### 什么是AST？

AST是Abstract Syntax Tree（抽象语法树）的缩写。
在计算机科学中，抽象语法树（AST）或语法树是用编程语言编写的源代码的抽象语法结构的树表示。树的每个节点表示在源代码中出现的构造。语法是“抽象的”，因为它不代表真实语法中出现的每个细节，而只是结构，内容相关的细节。

按照语法规则书写的代码，是用来让开发者可阅读、可理解的。对编译器等工具来讲，它可以理解的就是抽象语法树了。
```markdown
var a = 18
function sum(b){
  return a + b;
}
```

查看 [https://astexplorer.net/](https://astexplorer.net/)将源码生成抽象语法树

你会留意到 AST 的每一层都拥有相同的结构：
```markdown
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
{
  type: "Identifier",
  name: ...
}
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

这样的每一层结构也被叫做 节点（Node）。 一个 AST 可以由单一的节点或是成百上千个节点构成。 它们组合在一起可以描述用于静态分析的程序语法。

每一个节点都有如下所示的接口（Interface）：
```markdown
interface Node {
  type: string;
}
```

字符串形式的 type 字段表示节点的类型（如： "FunctionDeclaration"，"Identifier"，或 "BinaryExpression"）。 每一种类型的节点定义了一些附加属性用来进一步描述该节点类型。

查看 [AST对象文档](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API#Node_objects)

### AST是如何产生的？

Babel的编译过程跟绝大多数其他语言的编译器大致同理，分为三个阶段：

- 解析：将代码字符串解析成抽象语法树
- 变换：对抽象语法树进行变换操作
- 再建：根据变换后的抽象语法树再生成代码字符串

babel解析的过程就是生成抽象语法树的过程，需要经过以下两个阶段：

1.分词（tokenize）：将整个代码字符串分割成 语法单元 数组
2.语义分析(parse)：在分词结果的基础之上分析 语法单元之间的关系

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

分词的过过程从逻辑来讲并不难解释，但是这是个精细活，要考虑清楚所有的情况。还是以一个代码为例：

```markdown
if (1 > 0) {
  alert("if \"1 > 0\"");
}
```
我们希望得到的分词是：

```markdown
'if'     ' '       '('    '1'      ' '    '>'    ' '    ')'    ' '    '{'
'\n  '   'alert'   '('    '"if \"1 > 0\""'    ')'    ';'    '\n'   '}'
```
注意其中"if \"1 > 0\""是作为一个语法单元存在，没有再查分成if、1、>、0这样，而且其中的转译符会阻止字符串早结束。

这拆分过程其实没啥可取巧的，就是简单粗暴地一个字符一个字符地遍历，然后分情况讨论，整个实现方法就是顺序遍历和大量的条件判断。我用一个简单的实现来解释，在关键的地方注释，我们只考虑上面那段代码里存在的语法单元类型。

```markdown
function tokenizeCode (code) {
  const tokens = [];    // 结果数组
  for (let i = 0; i < code.length; i++) {
    // 从0开始，一个字符一个字符地读取
    let currentChar = code.charAt(i);

    if (currentChar === ';') {
      // 对于这种只有一个字符的语法单元，直接加到结果当中
      tokens.push({
        type: 'sep',
        value: ';',
      });
      // 该字符已经得到解析，不需要做后续判断，直接开始下一个
      continue;
    }
    
    if (currentChar === '(' || currentChar === ')') {
      // 与 ; 类似只是语法单元类型不同
      tokens.push({
        type: 'parens',
        value: currentChar,
      });
      continue;
    }

    if (currentChar === '}' || currentChar === '{') {
      // 与 ; 类似只是语法单元类型不同
      tokens.push({
        type: 'brace',
        value: currentChar,
      });
      continue;
    }

    if (currentChar === '>' || currentChar === '<') {
      // 与 ; 类似只是语法单元类型不同
      tokens.push({
        type: 'operator',
        value: currentChar,
      });
      continue;
    }

    if (currentChar === '"' || currentChar === '\'') {
      // 引号表示一个字符传的开始
      const token = {
        type: 'string',
        value: currentChar,       // 记录这个语法单元目前的内容
      };
      tokens.push(token);

      const closer = currentChar;
      let escaped = false;        // 表示下一个字符是不是被转译的

      // 进行嵌套循环遍历，寻找字符串结尾
      for (i++; i < code.length; i++) {
        currentChar = code.charAt(i);
        // 先将当前遍历到的字符无条件加到字符串的内容当中
        token.value += currentChar;
        if (escaped) {
          // 如果当前转译状态是true，就将改为false，然后就不特殊处理这个字符
          escaped = false;
        } else if (currentChar === '\\') {
          // 如果当前字符是 \ ，将转译状态设为true，下一个字符不会被特殊处理
          escaped = true;
        } else if (currentChar === closer) {
          break;
        }
      }
      continue;
    }
    
    if (/[0-9]/.test(currentChar)) {
      // 数字是以0到9的字符开始的
      const token = {
        type: 'number',
        value: currentChar,
      };
      tokens.push(token);

      for (i++; i < code.length; i++) {
        currentChar = code.charAt(i);
        if (/[0-9\.]/.test(currentChar)) {
          // 如果遍历到的字符还是数字的一部分（0到9或小数点）
          // 这里暂不考虑会出现多个小数点以及其他进制的情况
          token.value += currentChar;
        } else {
          // 遇到不是数字的字符就退出，需要把 i 往回调，
          // 因为当前的字符并不属于数字的一部分，需要做后续解析
          i--;
          break;
        }
      }
      continue;
    }

    if (/[a-zA-Z\$\_]/.test(currentChar)) {
      // 标识符是以字母、$、_开始的
      const token = {
        type: 'identifier',
        value: currentChar,
      };
      tokens.push(token);

      // 与数字同理
      for (i++; i < code.length; i++) {
        currentChar = code.charAt(i);
        if (/[a-zA-Z0-9\$\_]/.test(currentChar)) {
          token.value += currentChar;
        } else {
          i--;
          break;
        }
      }
      continue;
    }
    
    if (/\s/.test(currentChar)) {
      // 连续的空白字符组合到一起
      const token = {
        type: 'whitespace',
        value: currentChar,
      };
      tokens.push(token);

      // 与数字同理
      for (i++; i < code.length; i++) {
        currentChar = code.charAt(i);
        if (/\s]/.test(currentChar)) {
          token.value += currentChar;
        } else {
          i--;
          break;
        }
      }
      continue;
    }

    // 还可以有更多的判断来解析其他类型的语法单元

    // 遇到其他情况就抛出异常表示无法理解遇到的字符
    throw new Error('Unexpected ' + currentChar);
  }
  return tokens;
}

const tokens = tokenizeCode(`
if (1 > 0) {
  alert("if 1 > 0");
}
`);

```

执行结果如下：
```markdown
[
 { type: "whitespace", value: "\n" },
 { type: "identifier", value: "if" },
 { type: "whitespace", value: " " },
 { type: "parens", value: "(" },
 { type: "number", value: "1" },
 { type: "whitespace", value: " " },
 { type: "operator", value: ">" },
 { type: "whitespace", value: " " },
 { type: "number", value: "0" },
 { type: "parens", value: ")" },
 { type: "whitespace", value: " " },
 { type: "brace", value: "{" },
 { type: "whitespace", value: "\n " },
 { type: "identifier", value: "alert" },
 { type: "parens", value: "(" },
 { type: "string", value: "\"if 1 > 0\"" },
 { type: "parens", value: ")" },
 { type: "sep", value: ";" },
 { type: "whitespace", value: "\n" },
 { type: "brace", value: "}" },
 { type: "whitespace", value: "\n" },
]
```
经过这一步的分词，这个数组就比摊开的字符串更方便进行下一步处理了。

#### 语义分析

语义分析就是把词汇进行立体的组合，确定有多重意义的词语最终是什么意思、多个词语之间有什么关系以及又应该再哪里断句等。

在编程语言解释当中，这就是要最终生成语法树的步骤了。不像自然语言，像“从句”这种结构往往最多只有一层，编程语言的各种从属关系更加复杂。

在编程语言的解析中有两个很相似但是又有区别的重要概念：

- 语句：语句是一个具备边界的代码区域，相邻的两个语句之间从语法上来讲互不干扰，调换顺序虽然可能会影响执行结果，但不会产生语法错误
比如return true、var a = 10、if (...) {...}
- 表达式：最终有个结果的一小段代码，它的特点是可以原样嵌入到另一个表达式
比如myVar、1+1、str.replace('a', 'b')、i < 10 && i > 0等
很多情况下一个语句可能只包含一个表达式，比如console.log('hi');。estree标准当中，这种语句节点称作ExpressionStatement。

语义分析的过程又是个遍历语法单元的过程，不过相比较而言更复杂，因为分词过程中，每个语法单元都是独立平铺的，而语法分析中，语句和表达式会以树状的结构互相包含。针对这种情况我们可以用栈，也可以用递归来实现。

以下是语义分析的伪代码：
```markdown
function parse (tokens) {
  let i = -1;     // 用于标识当前遍历位置
  let curToken;   // 用于记录当前符号

  // 读取下一个语句
  function nextStatement () {
    // 暂存当前的i，如果无法找到符合条件的情况会需要回到这里
    stash();
    
    // 读取下一个符号
    nextToken();

    if (curToken.type === 'identifier' && curToken.value === 'if') {
      // 解析 if 语句
      const statement = {
        type: 'IfStatement',
      };
      // if 后面必须紧跟着 (
      nextToken();
      if (curToken.type !== 'parens' || curToken.value !== '(') {
        throw new Error('Expected ( after if');
      }

      // 后续的一个表达式是 if 的判断条件
      statement.test = nextExpression();

      // 判断条件之后必须是 )
      nextToken();
      if (curToken.type !== 'parens' || curToken.value !== ')') {
        throw new Error('Expected ) after if test expression');
      }

      // 下一个语句是 if 成立时执行的语句
      statement.consequent = nextStatement();

      // 如果下一个符号是 else 就说明还存在 if 不成立时的逻辑
      if (curToken === 'identifier' && curToken.value === 'else') {
        statement.alternative = nextStatement();
      } else {
        statement.alternative = null;
      }
      commit();
      return statement;
    }

    if (curToken.type === 'brace' && curToken.value === '{') {
      // 以 { 开头表示是个代码块，我们暂不考虑JSON语法的存在
      const statement = {
        type: 'BlockStatement',
        body: [],
      };
      while (i < tokens.length) {
        // 检查下一个符号是不是 }
        stash();
        nextToken();
        if (curToken.type === 'brace' && curToken.value === '}') {
          // } 表示代码块的结尾
          commit();
          break;
        }
        // 还原到原来的位置，并将解析的下一个语句加到body
        rewind();
        statement.body.push(nextStatement());
      }
      // 代码块语句解析完毕，返回结果
      commit();
      return statement;
    }
    
    // 没有找到特别的语句标志，回到语句开头
    rewind();

    // 尝试解析单表达式语句
    const statement = {
      type: 'ExpressionStatement',
      expression: nextExpression(),
    };
    if (statement.expression) {
      nextToken();
      if (curToken.type !== 'EOF' && curToken.type !== 'sep') {
        throw new Error('Missing ; at end of expression');
      }
      return statement;
    }
  }

  // 读取下一个表达式
  function nextExpression () {
    nextToken();

    if (curToken.type === 'identifier') {
      const identifier = {
        type: 'Identifier',
        name: curToken.value,
      };
      stash();
      nextToken();
      if (curToken.type === 'parens' && curToken.value === '(') {
        // 如果一个标识符后面紧跟着 ( ，说明是个函数调用表达式
        const expr = {
          type: 'CallExpression',
          caller: identifier,
          arguments: [],
        };

        stash();
        nextToken();
        if (curToken.type === 'parens' && curToken.value === ')') {
          // 如果下一个符合直接就是 ) ，说明没有参数
          commit();
        } else {
          // 读取函数调用参数
          rewind();
          while (i < tokens.length) {
            // 将下一个表达式加到arguments当中
            expr.arguments.push(nextExpression());
            nextToken();
            // 遇到 ) 结束
            if (curToken.type === 'parens' && curToken.value === ')') {
              break;
            }
            // 参数间必须以 , 相间隔
            if (curToken.type !== 'comma' && curToken.value !== ',') {
              throw new Error('Expected , between arguments');
            }
          }
        }
        commit();
        return expr;
      }
      rewind();
      return identifier;
    }

    if (curToken.type === 'number' || curToken.type === 'string') {
      // 数字或字符串，说明此处是个常量表达式
      const literal = {
        type: 'Literal',
        value: eval(curToken.value),
      };
      // 但如果下一个符号是运算符，那么这就是个双元运算表达式
      // 此处暂不考虑多个运算衔接，或者有变量存在
      stash();
      nextToken();
      if (curToken.type === 'operator') {
        commit();
        return {
          type: 'BinaryExpression',
          left: literal,
          right: nextExpression(),
        };
      }
      rewind();
      return literal;
    }

    if (curToken.type !== 'EOF') {
      throw new Error('Unexpected token ' + curToken.value);
    }
  }

  // 往后移动读取指针，自动跳过空白
  function nextToken () {
    do {
      i++;
      curToken = tokens[i] || { type: 'EOF' };
    } while (curToken.type === 'whitespace');
  }

  // 位置暂存栈，用于支持很多时候需要返回到某个之前的位置
  const stashStack = [];

  function stash (cb) {
    // 暂存当前位置
    stashStack.push(i);
  }

  function rewind () {
    // 解析失败，回到上一个暂存的位置
    i = stashStack.pop();
    curToken = tokens[i];
  }

  function commit () {
    // 解析成功，不需要再返回
    stashStack.pop();
  }
  
  const ast = {
    type: 'Program',
    body: [],
  };

  // 逐条解析顶层语句
  while (i < tokens.length) {
    const statement = nextStatement();
    if (!statement) {
      break;
    }
    ast.body.push(statement);
  }
  return ast;
}

const ast = parse([
 { type: "whitespace", value: "\n" },
 { type: "identifier", value: "if" },
 { type: "whitespace", value: " " },
 { type: "parens", value: "(" },
 { type: "number", value: "1" },
 { type: "whitespace", value: " " },
 { type: "operator", value: ">" },
 { type: "whitespace", value: " " },
 { type: "number", value: "0" },
 { type: "parens", value: ")" },
 { type: "whitespace", value: " " },
 { type: "brace", value: "{" },
 { type: "whitespace", value: "\n " },
 { type: "identifier", value: "alert" },
 { type: "parens", value: "(" },
 { type: "string", value: "\"if 1 > 0\"" },
 { type: "parens", value: ")" },
 { type: "sep", value: ";" },
 { type: "whitespace", value: "\n" },
 { type: "brace", value: "}" },
 { type: "whitespace", value: "\n" },
]);
```

以上就是语义解析的部分主要思路。注意现在的nextExpression已经颇为复杂，但实际实现要比现在这里展示的要更复杂很多，因为这里根本没有考虑单元运算符、运算优先级等等。
