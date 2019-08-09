## Babel工作原理

### 什么是Babel？

Babel 对于前端开发者来说应该是很熟悉了，日常开发中基本上是离不开它的。
Babel 是一个 JavaScript 编译器,它能将es2015,react等低端浏览器无法识别的语言，进行编译，使它在旧的浏览器或者环境中也能够运行。Babel 的功能很纯粹。我们传递一段源代码给 Babel，然后它返回一串新的代码给我们。就是这么简单，它不会运行我们的代码，也不会去打包我们的代码。

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

Parse 阶段可以细分为两个阶段：词法分析（Lexical Analysis, LA）和语法分析

词法分析: 将代码(字符串)分割为token流,即语法单元成的数组
语法分析: 分析token流(上面生成的数组)并生成 AST

```markdown
const tokens = tokenize('const add = (a, b) => a + b')；
console.log(tokens);

[
  { "type": "Keyword", "value": "const" },
  { "type": "Identifier", "value": "add" },
  { "type": "Punctuator", "value": "=" },
  { "type": "Paren", "value": "(" },
  { "type": "Identifier", "value": "a" },
  { "type": "Punctuator", "value": "," },
  { "type": "Identifier", "value": "b" },
  { "type": "Paren", "value": ")" },
  { "type": "ArrowFunction", "value": "=>" },
  { "type": "Identifier", "value": "a" },
  { "type": "Operator", "value": "+" },
  { "type": "Identifier", "value": "b" }
]

const ast = parser(tokens)；
console.log(ast);

{
    "type": "Program",
    "body": [
        {
            "type": "VariableDeclaration",
            "identifierName": "add",
            "init": {
                "type": "ArrowFunctionExpression",
                "params": [
                    {
                        "type": "identifier",
                        "identifierName": "a"
                    },
                    {
                        "type": "identifier",
                        "identifierName": "b"
                    }
                ],
                "body": {
                    "type": "BinaryExpression",
                    "left": {
                        "type": "identifier",
                        "identifierName": "a"
                    },
                    "operator": "+",
                    "right": {
                        "type": "identifier",
                        "identifierName": "b"
                    }
                }
            }
        }
    ]
}

```
#### 转换

在 Babel 中我们使用者最常使用的地方就是代码转换,大家常用的 Babel 插件就是定义代码转换规则而生的。
这一步做的事情就是操作 AST，可以增删改这些节点，从而转换成实际需要的 AST。

transform 过程：遍历 AST 树并应用各 transformers（plugin） 生成变换后的 AST 树
babel 中最核心的是 babel-core，它向外暴露出 babel.transform 接口。

`const add = (a, b) => a + b` =>  

`var add = function(a, b) => { return a + b }`

```markdown
{
  "type": "Program",
  "start": 0,
  "end": 28,
  "body": [
    {
      "type": "VariableDeclaration",
      "start": 0,
      "end": 27,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 6,
          "end": 27,
          "id": {
            "type": "Identifier",
            "start": 6,
            "end": 9,
            "name": "add"
          },
          "init": {
            "type": "ArrowFunctionExpression",
            "start": 12,
            "end": 27,
            "id": null,
            "expression": true,
            "generator": false,
            "params": [
              {
                "type": "Identifier",
                "start": 13,
                "end": 14,
                "name": "a"
              },
              {
                "type": "Identifier",
                "start": 16,
                "end": 17,
                "name": "b"
              }
            ],
            "body": {
              "type": "BinaryExpression",
              "start": 22,
              "end": 27,
              "left": {
                "type": "Identifier",
                "start": 22,
                "end": 23,
                "name": "a"
              },
              "operator": "+",
              "right": {
                "type": "Identifier",
                "start": 26,
                "end": 27,
                "name": "b"
              }
            }
          }
        }
      ],
      "kind": "const"
    }
  ],
  "sourceType": "module"
}
```

```markdown
{
  "type": "Program",
  "start": 0,
  "end": 43,
  "body": [
    {
      "type": "VariableDeclaration",
      "start": 0,
      "end": 41,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 4,
          "end": 41,
          "id": {
            "type": "Identifier",
            "start": 4,
            "end": 7,
            "name": "add"
          },
          "init": {
            "type": "FunctionExpression",
            "start": 10,
            "end": 41,
            "id": null,
            "expression": false,
            "generator": false,
            "params": [
              {
                "type": "Identifier",
                "start": 19,
                "end": 20,
                "name": "a"
              },
              {
                "type": "Identifier",
                "start": 22,
                "end": 23,
                "name": "b"
              }
            ],
            "body": {
              "type": "BlockStatement",
              "start": 25,
              "end": 41,
              "body": [
                {
                  "type": "ReturnStatement",
                  "start": 27,
                  "end": 39,
                  "argument": {
                    "type": "BinaryExpression",
                    "start": 34,
                    "end": 39,
                    "left": {
                      "type": "Identifier",
                      "start": 34,
                      "end": 35,
                      "name": "a"
                    },
                    "operator": "+",
                    "right": {
                      "type": "Identifier",
                      "start": 38,
                      "end": 39,
                      "name": "b"
                    }
                  }
                }
              ]
            }
          }
        }
      ],
      "kind": "var"
    }
  ],
  "sourceType": "module"
}
```

```markdown
//babel核心库，用来实现核心的转换引擎
let babel = require('babel-core');
//可以实现类型判断，生成AST节点
let types = require('babel-types');
let code = `const add = (a, b) => a + b`;//转换语句
//visitor可以对特定节点进行处理
let visitor = {
    VariableDeclaration(path) { //定义需要转换的节点,这里拦截let和const
        const node = path.node;
        ['let', 'const'].includes(node.kind) && (node.kind = 'var');
    },
    ArrowFunctionExpression(path) { //定义需要转换的节点,这里拦截箭头函数
        let { id, params, body, generator, async } = path.node;
        //箭头函数我们会简写{return a+b} 为 a+b    
        if (!t.isBlockStatement(body)) {    
          const node = t.returnStatement(body);
          body = t.blockStatement([node]);
        }
        path.replaceWith(t.functionExpression(id, params, body, generator, async));
      }
    }
}
//将code转成ast
let result = babel.transform(code, {
    plugins: [
        { visitor }
    ]
})
console.log(result.code)

```
#### 生成

利用 babel-generator 将 AST 树输出为转码后的代码字符串

