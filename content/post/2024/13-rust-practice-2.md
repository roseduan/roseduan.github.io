---
title: Rust 练手项目 2 - 表达式计算
date: 2024-05-02T10:51:56+08:00
categories:
    - Rust
tags:
    - Rust 项目实战
---

> 本文完整代码：[https://github.com/rosedblabs/rust-practice](https://github.com/rosedblabs/rust-practice)

表达式解析、计算是一种基本和常见的任务，例如最常见的算术表达式，计算的方法有很多，比如逆波兰表达式、LL、LR 算法等等。

这一次介绍一种最简单的、容易理解的基于运算符优先级的算法来完成这个任务。

基于运算符优先级的算法叫做 `Precedence Climbing`，它本质上是一种递归下降解析表达式的方法，通过递归地处理运算符和操作数来解析表达式，并根据运算符的优先级和结合性来确定表达式的计算顺序。

这种算法的核心思想是利用运算符的优先级进行“爬升”（Climbing），以决定表达式的结构和计算顺序。

首先我们做一些约束，由于运算符众多，我们可以支持几种最常用的：

- `+` 加
- `-` 减
- `*` 乘
- `/` 除
- `^` 幂

并且我们知道，幂运算的优先级是最高的，其次是 `*` 和 `/`，优先级最低的是 `+` 和 `-`。

所以约定其运算符的优先级分别为 `3（^）、2（* /）、1（+ -）`

```text
2 + 3 ^ 2 * 3 + 4

|---------------|   : prec 1
    |-------|       : prec 2
    |---|           : prec 3
```

约定优先级的主要作用是在计算的时候，需要根据优先级来确定计算的顺序。

确定了优先级的问题，第二个问题是结合性，运算符的结合性其实也是确定的，例如加法是左结合的，这意味着 `2 + 3 + 4` 等价于 `(2 + 3) + 4`，而幂运算是右结合的，这意味着 `2 ^ 3 ^ 4` 实际上等价于 `2 ^ (3 ^ 4)`。

最后还需要注意一个问题，那就是子表达式，也就是用括号包裹的部分，这部分实际上是需要单独进行计算的，并且比运算符的优先级更高。

其实也很容易理解，比如 `2 * (3 + 5) * 7`，尽管 `*` 的优先级比 `+` 高，但是需要先计算括号内的部分。

确定了这些需求，我们再来看如何用 Rust 代码来进行实现。

首先我们需要将表达式进行解析，也就是词法分析的阶段，将一个表达式解析为不同的 Token，下面是约定的几种 Token：

```rust
// Token 表示，数字、运算符号、括号
#[derive(Debug, Clone, Copy)]
enum Token {
    Number(i32),
    Plus,       // 加
    Minus,      // 减
    Multiply,   // 乘
    Divide,     // 除
    Power,      // 幂
    LeftParen,  // 左括号
    RightParen, // 右括号
}
```

然后定义了一个 Tokenizer 结构体，主要是利用 Peekable 接口将表达式解析为不同的 Token：

```rust
// 将一个算术表达式解析成连续的 Token
// 并通过 Iterator 返回，也可以通过 Peekable 接口获取
struct Tokenizer<'a> {
    tokens: Peekable<Chars<'a>>,
}
```

然后自定义实现了一个 Iterator，让解析后的 Token 可以通过迭代器进行返回。

```rust
impl<'a> Iterator for Tokenizer<'a> {
    type Item = Token;

    fn next(&mut self) -> Option<Self::Item> {
        // 消除前面的空格
        self.consume_whitespace();
        // 解析当前位置的 Token 类型
        match self.tokens.peek() {
            Some(c) if c.is_numeric() => self.scan_number(),
            Some(_) => self.scan_operator(),
            None => return None,
        }
    }
}
```

假如我们的表达式是 `2 + 3 ^ 2 * 3 + 4`，实际上解析后的 Token 就是：

```text
Token::Number(2)
Token::Plus
Token::Number(3)
Token::Power
Token::Number(2)
Token::Multiply
Token::Number(3)
Token::Plus
Token::Number(4)
```

拿到 Token 之后，进入到了语法分析的阶段，需要根据每个表达式的含义，以及其优先级，计算对应的结果。

首先定义一个方法，计算单个 Token 以及子表达式，这只存在两种情况，分别是 Number 这个 Token，以及带括号的子表达式。

```rust
fn compute_atom(&mut self) -> Result<i32> {
        match self.iter.peek() {
            // 如果是数字的话，直接返回
            Some(Token::Number(n)) => {
                let val = *n;
                self.iter.next();
                return Ok(val);
            }
            // 如果是左括号的话，递归计算括号内的值
            Some(Token::LeftParen) => {
                self.iter.next();
                let result = self.compute_expr(1)?;
                match self.iter.next() {
                    Some(Token::RightParen) => (),
                    _ => return Err(ExprError::Parse("Unexpected character".into())),
                }
                return Ok(result);
            }
            _ => {
                return Err(ExprError::Parse(
                    "Expecting a number or left parenthesis".into(),
                ))
            }
        }
    }
```

这里其实比较好理解，如果是 Number 直接返回，如果是子表达式，则重新调用计算表达式的方法进行计算。

然后是另一个核心的方法计算表达式：

```rust
fn compute_expr(&mut self, min_prec: i32) -> Result<i32> {
    // 计算第一个 Token
    let mut atom_lhs = self.compute_atom()?;
    
    loop {
        let cur_token = self.iter.peek();
        if cur_token.is_none() {
            break;
        }
        let token = *cur_token.unwrap();

        // 1. Token 一定是运算符
        // 2. Token 的优先级必须大于等于 min_prec
        if !token.is_operator() || token.precedence() < min_prec {
            break;
        }

        let mut next_prec = token.precedence();
        if token.assoc() == ASSOC_LEFT {
            next_prec += 1;
        }

        self.iter.next();

        // 递归计算右边的表达式
        let atom_rhs = self.compute_expr(next_prec)?;
        
        // 得到了两边的值，进行计算
        match token.compute(atom_lhs, atom_rhs) {
            Some(res) => atom_lhs = res,
            None => return Err(ExprError::Parse("Unexpected expr".into())),
        }
    }
    Ok(atom_lhs)
}
```

这个方法中核心的逻辑可以分几个步骤来理解：

一是使用了 min_prec 参数控制当前层级的优先级，如果表达式的优先级小于 min_prec 则直接跳出循环，返回当前的值。

比如 `2 * 3 + 4`，`*` 会先解析到，然后 `+` 运算符的优先级明显比 `*` 更低，会直接返回当前值 `3`。

二是如果运算符的结合性是左边的话，则下一次迭代的 min_prec 需要递增。

比如表达式是 `2 * 3 * 4`，解析到第二个 `*` 的时候，`*` 的优先级本来是 `2`，但它是左结合的，所以此时 min_prec 是 `3`，会直接跳出循环，所以实际上会先计算 `2 * 3`。

最后是得到了运算符两边的值，就可以进行计算了，这里是根据运算符的实际含义来进行的：

```rust
// 根据当前运算符进行计算
fn compute(&self, l: i32, r: i32) -> Option<i32> {
    match self {
        Token::Plus => Some(l + r),
        Token::Minus => Some(l - r),
        Token::Multiply => Some(l * r),
        Token::Divide => Some(l / r),
        Token::Power => Some(l.pow(r as u32)),
        _ => None,
    }
}
```

这就是根据运算符优先级来进行表达式计算的整体流程，这个算法看起来还是非常简洁优雅的，非常巧妙的利用优先级来解决运算的顺序和结合等问题。

完整的代码也只有 200 多行，比较适合用来练手，通过这个项目，可以学习到：

- 一个优雅、简洁的表达式计算的算法
- 解决类似写一个计算器的面试问题
- Rust 基础数据类型、枚举、结构体基本用法
- 函数、递归
- match 表达式
- 自定义 Result 错误处理
- 迭代器的常见用法 next、peekable 等
- 自定义迭代器
- Option 使用

------

最后附上项目地址：
[https://github.com/rosedblabs/rust-practice](https://github.com/rosedblabs/rust-practice)
对你有帮助的话，欢迎给个 star ⭐️ 哦！
