---
title: 计算的本质1 - 从自动机到正则表达式 Java实现
date: 2020-04-04 00:37:21
tags:
 - DFA
 - NFA
 - 正则表达式
 - 读书笔记
 - Java
categories:
 - 读书笔记
---

现代计算机的强大能力伴随着同样的复杂性，我们很难了解所有的细节，此时计算机的简化模型就显得很有用了，可以帮助我们建立完整的认识。

有限自动机（Finite Automaton）是一台计算机的极简模型，没有持久化的存储，只有极少的RAM，最简单输入输出模式，我们从FA开始，看下它能做到什么。

这篇文章是《计算的本质》第3章的读书笔记，通过Java一步一步实现DFA、NFA，最后实现一个简单的正则表达式引擎。

## DFA（确定性有限自动机）

![fa1](http://media.kosho.tech/blog/20200420/fa1.png)

这是一个非常简单的DFA，起始状态为1，接受状态为2，状态之间的箭头代表规则：

- 状态 1 并且读入字符 • a 时，切换到状态 2。
- 处于状态 2 并且读入字符 • a 时，切换到状态 1。

这个自动机接受奇数数量的a字符串，“a”，“aaa”等，拒绝其他的字符串输入。并且提供一个“是 / 否”的输出，以表明这个序列是否已经被接受，它已经可以被称之为称为简单计算机了。

|          | 真实计算机                | 有限自动机                   |
| -------- | ------------------------- | ---------------------------- |
| 持久存储 | 硬盘                      | 无                           |
| 临时存储 | RAM                       | 当前状态                     |
| 输入     | 键盘、鼠标、网络等        | 字符流                       |
| 输出     | 显示设备、音响、网络等    | 当前状态是否为一个接受状态   |
| 处理器   | 能执行任何程序的 CPU 核心 | 根据输入改变状态的硬编码规则 |

### 确定性

上面说的这台自动机的状态始终是确定的，如果满足下面两个条件，就可以称之为DFA：

- 不存在这样的状态：它的下一次转换状态因为有彼此冲突的规则而有二义性（这意味着一个状态对于同样的输入，不能有多个规则）
- 没有遗漏：不存在这样的状态：它的下一次转换状态因为缺失规则而未知（这意味着每个状态都必须针对每个可能的输入字符有至少一个规则）

### 代码模拟

DFA很容易用代码对其进行模拟，然后就可以进行一些交互测试了。不得不说ruby的确是非常优雅的语言，用java描述稍显冗长。

FA规则：

```java
public class FaRule {
    private Object state;
    private Character character;
    private Object nextState;

    public FaRule(Object state, Character character, Object nextState) {
        this.state = state;
        this.nextState = nextState;
        this.character = character;
    }

    public boolean appliesTo(Object state, Character character) {
        return this.state.equals(state) && equals(this.character, character);
    }

    public Object follow() {
        return nextState;
    }

    public Character getCharacter() {
        return character;
    }

    @Override
    public String toString() {
        return "FaRule{" + state + "--" + character + "-->" + nextState + '}';
    }

    private boolean equals(Character a, Character b) {
        return (a == null && b == null) || (a != null && a.equals(b));
    }
}
```

规则表：

```java
public class DfaRuleBook {
    private List<FaRule> rules;

    public DfaRuleBook(List<FaRule> rules) {
        this.rules = rules;
    }

    public Object nextState(Object state, char character) {
        return ruleFor(state, character)
                .map(FaRule::follow)
                .orElse(null);
    }

    public Optional<FaRule> ruleFor(Object state, char character) {
        return rules.stream()
                .filter(faRule -> faRule.appliesTo(state, character))
                .findFirst();
    }
}
```

DFA：

```java
public class Dfa {
    private Object currentState;
    private Object[] acceptStates;
    private DfaRuleBook ruleBook;

    public Dfa(Object currentState, Object[] acceptStates, DfaRuleBook ruleBook) {
        this.currentState = currentState;
        this.acceptStates = acceptStates;
        this.ruleBook = ruleBook;
    }

    public boolean accepting() {
        for (Object acceptState : acceptStates) {
            if (acceptState.equals(currentState)) {
                return true;
            }
        }

        return false;
    }

    public void readCharacter(char character) {
        currentState = ruleBook.nextState(currentState, character);
    }

    public void readString(String str) {
        for (int i = 0; i < str.length(); i++) {
            readCharacter(str.charAt(i));
        }
    }
}
```

我们只需要这三个类就可以模拟一个DFA了，再增加一个DfaDesign类用来简化操作：

```java
public class DfaDesign {
    private Object startState;
    private Object[] acceptStates;
    private DfaRuleBook ruleBook;

    public DfaDesign(Object startState, Object[] acceptStates, DfaRuleBook ruleBook) {
        this.startState = startState;
        this.acceptStates = acceptStates;
        this.ruleBook = ruleBook;
    }

    public Dfa toDfa() {
        return new Dfa(startState, acceptStates, ruleBook);
    }

    public boolean accepts(String str) {
        Dfa dfa = toDfa();
        dfa.readString(str);
        return dfa.accepting();
    }
}
```

好了，现在我们来模拟这样一个DFA，接受"ab"，"baba"，"baaab"这样的字符串，拒绝"a"，"baa"等形式的字符串：

![fa2](http://media.kosho.tech/blog/20200420/fa2.png)

```java
List<FaRule> rules = new ArrayList<>();
rules.add(new FaRule(1, 'a', 2));
rules.add(new FaRule(1, 'b', 1));
rules.add(new FaRule(2, 'a', 2));
rules.add(new FaRule(2, 'b', 3));
rules.add(new FaRule(3, 'a', 3));
rules.add(new FaRule(3, 'b', 3));
DfaRuleBook ruleBook = new DfaRuleBook(rules);
DfaDesign dfaDesign = new DfaDesign(1, new Object[]{3}, ruleBook);
Assert.assertTrue(dfaDesign.accepts("baaab"));
```

现在看起来还没什么用，只能模拟一些非常简单的东西，后面我们可以看到更实际的应用。

## NFA（非确定性有限自动机）

DFA理解和实现都很简单，如果我们去掉确定性约束，让自动机拥有对立的规则和多条可能路径，会怎么样呢？

### 非确定性

如果我们想设计一台DFA，接受a和b组成，倒数第三个字符是a的字符串。会发现比较麻烦，无法预先知道什么时候能读到倒数第三个字符。

如果我们放松确定性的限制，并且允许规则手册对于一个状态和输入包含多条规则（或者根本没有规则），那么就可以设计一台能完成任务的机器，即一台NFA：

![nfa1](http://media.kosho.tech/blog/20200420/nfa1.png)

如果存在某条路径能让 NFA 按照它的某些规则执行并终止于一个接受状态，那它就能接受这个字符串。

### 代码模拟

NFA规则表：

```java
public class NfaRuleBook {
    private List<FaRule> rules;

    public NfaRuleBook(List<FaRule> rules) {
        this.rules = rules;
    }

    public Set<Object> nextStates(Collection<Object> states, Character character) {
        return states.stream()
                .map(state -> followRulesFor(state, character))
                .flatMap(Collection::stream)
                .collect(Collectors.toSet());
    }

    public Set<Object> followRulesFor(Object state, Character character) {
        return rulesFor(state, character).stream().map(FaRule::follow).collect(Collectors.toSet());
    }

    public Set<FaRule> rulesFor(Object state, Character character) {
        return rules.stream()
                .filter(faRule -> faRule.appliesTo(state, character))
                .collect(Collectors.toSet());
    }

    public Set<Object> followFreeMoves(Collection<Object> states) {
        Set<Object> moreStates = nextStates(states, null);
        if (states.containsAll(moreStates)) {
            return new HashSet<>(states);
        } else {
            states.addAll(moreStates);
            return followFreeMoves(states);
        }
    }

    public List<Character> alphabet() {
        return rules.stream()
                .map(FaRule::getCharacter)
                .distinct()
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }

    public List<FaRule> getRules() {
        return rules;
    }
}
```

NFA：

```java
public class Nfa {
    private Set<Object> currentStates;
    private Set<Object> acceptStates;
    private NfaRuleBook ruleBook;

    public Nfa(Set<Object> currentStates, Set<Object> acceptStates, NfaRuleBook ruleBook) {
        this.currentStates = currentStates;
        this.acceptStates = acceptStates;
        this.ruleBook = ruleBook;
    }

    public boolean accepting() {
        for (Object currentState : getCurrentStates()) {
            if (acceptStates.contains(currentState)) {
                return true;
            }
        }

        return false;
    }

    public void readCharacter(Character character) {
        currentStates = ruleBook.nextStates(getCurrentStates(), character);
    }

    public void readString(String str) {
        for (int i = 0; i < str.length(); i++) {
            readCharacter(str.charAt(i));
        }
    }

    public Set<Object> getCurrentStates() {
        return ruleBook.followFreeMoves(currentStates);
    }
}
```

NfaDesign：

```java
public class NfaDesign {
    private Object startState;
    private Set<Object> acceptStates;
    private NfaRuleBook ruleBook;

    public NfaDesign(Object startState, Set<Object> acceptStates, NfaRuleBook ruleBook) {
        this.startState = startState;
        this.acceptStates = acceptStates;
        this.ruleBook = ruleBook;
    }

    public Nfa toNfa() {
        return new Nfa(Sets.create(startState), acceptStates, ruleBook);
    }

    public Nfa toNfa(Set<Object> currentStates) {
        return new Nfa(currentStates, acceptStates, ruleBook);
    }

    public boolean accepts(String str) {
        Nfa nfa = toNfa();
        nfa.readString(str);
        return nfa.accepting();
    }

    public Object getStartState() {
        return startState;
    }

    public Set<Object> getAcceptStates() {
        return acceptStates;
    }

    public NfaRuleBook getRuleBook() {
        return ruleBook;
    }
}
```

至此，已经可以模拟一台带有自由移动的NFA了，它可以在没有输入的情况下，在不同的状态之间做出转移。

现在我们设计一台可以接受2或3倍数长度的a字符串的NFA，其中的虚线箭头表示自由移动：

![nfa2](http://media.kosho.tech/blog/20200420/nfa2.png)

现在用代码模拟试试：

```java
List<FaRule> rules = new ArrayList<>();
rules.add(new FaRule(1, null, 2));
rules.add(new FaRule(1, null, 4));
rules.add(new FaRule(2, 'a', 3));
rules.add(new FaRule(3, 'a', 2));
rules.add(new FaRule(4, 'a', 5));
rules.add(new FaRule(5, 'a', 6));
rules.add(new FaRule(6, 'a', 4));
NfaRuleBook ruleBook = new NfaRuleBook(rules);
NfaDesign nfaDesign = new NfaDesign(1, Sets.create(2, 4), ruleBook);
Assert.assertFalse(nfaDesign.accepts("aaaaa"));
Assert.assertTrue(nfaDesign.accepts("aaaaaa"));
```

## 正则表达式

上面说了这么多，那它到底有什么用呢？一个比较典型的应用就是正则表达式，基本上所有的语言都实现了对正则表达式的支持，大部分也都采用了NFA来实现。

试一试用Java实现一个简易的正则表达式引擎，支持这样几个功能：

1. 空字符串
2. 'a'，'b'这样的字符
3. |或运算符
4. *重复运算符
5. 把正则连接起来，如a和b连接得到ab

这里没有实现?和+这些功能，其实ab?等同于ab|a，而ab+等同于abb*，其他计数重复（如 a{2,5} ）和字符组（如 [abc] ）等方便的特性也是这样，可以用已有的特性实现。至于捕获组、反向引用，那就是高级特性了。

我们为每类正则表达式定义一个类，用它们来表示语法树，同时，可以将它们转换成对应的NFA。

```java
// 正则基类
public abstract class Pattern {
    public abstract int precedence();

    public abstract NfaDesign toNfaDesign();

    public String bracket(int outerPrecedence) {
        if (precedence() < outerPrecedence) {
            return "(" + toString() + ")";
        }

        return toString();
    }

    public String inspect() {
        return "/" + this + "/";
    }

    public boolean matches(String str) {
        return toNfaDesign().accepts(str);
    }
}

// 空字符串
public class Empty extends Pattern {
    @Override
    public int precedence() {
        return 3;
    }

    @Override
    public NfaDesign toNfaDesign() {
        Integer start = IdGen.gen();
        NfaRuleBook ruleBook = new NfaRuleBook(Collections.emptyList());
        return new NfaDesign(start, Sets.create(start), ruleBook);
    }

    @Override
    public String toString() {
        return "";
    }
}

// 字符
public class Literal extends Pattern {
    private Character character;

    public Literal(Character character) {
        this.character = character;
    }

    @Override
    public int precedence() {
        return 3;
    }

    @Override
    public NfaDesign toNfaDesign() {
        Integer start = IdGen.gen();
        Integer accept = IdGen.gen();
        FaRule rule = new FaRule(start, character, accept);
        NfaRuleBook ruleBook = new NfaRuleBook(Collections.singletonList(rule));
        return new NfaDesign(start, Sets.create(accept), ruleBook);
    }

    @Override
    public String toString() {
        return String.valueOf(character);
    }
}

// 连接
public class Concat extends Pattern {
    private Pattern first;
    private Pattern second;

    public Concat(Pattern first, Pattern second) {
        this.first = first;
        this.second = second;
    }

    @Override
    public String toString() {
        return first.bracket(precedence()) + second.bracket(precedence());
    }

    @Override
    public int precedence() {
        return 1;
    }

    @Override
    public NfaDesign toNfaDesign() {
        NfaDesign firstNfaDesign = first.toNfaDesign();
        NfaDesign secondNfaDesign = second.toNfaDesign();
        Object startState = firstNfaDesign.getStartState();
        Set<Object> acceptStates = secondNfaDesign.getAcceptStates();

        List<FaRule> rules = new ArrayList<>();
        rules.addAll(firstNfaDesign.getRuleBook().getRules());
        rules.addAll(secondNfaDesign.getRuleBook().getRules());
        List<FaRule> extraRules = firstNfaDesign.getAcceptStates().stream()
                .map(state -> new FaRule(state, null, secondNfaDesign.getStartState()))
                .collect(Collectors.toList());
        rules.addAll(extraRules);

        NfaRuleBook ruleBook = new NfaRuleBook(rules);

        return new NfaDesign(startState, acceptStates, ruleBook);
    }
}

// 选择
public class Choose extends Pattern {
    private Pattern first;
    private Pattern second;

    public Choose(Pattern first, Pattern second) {
        this.first = first;
        this.second = second;
    }

    @Override
    public String toString() {
        return first.bracket(precedence()) + "|" + second.bracket(precedence());
    }

    @Override
    public int precedence() {
        return 0;
    }

    @Override
    public NfaDesign toNfaDesign() {
        NfaDesign firstNfaDesign = first.toNfaDesign();
        NfaDesign secondNfaDesign = second.toNfaDesign();
        Integer startState = IdGen.gen();
        Set<Object> acceptStates = new HashSet<>();
        acceptStates.addAll(firstNfaDesign.getAcceptStates());
        acceptStates.addAll(secondNfaDesign.getAcceptStates());

        List<FaRule> rules = new ArrayList<>();
        rules.addAll(firstNfaDesign.getRuleBook().getRules());
        rules.addAll(secondNfaDesign.getRuleBook().getRules());
        // extra rule
        rules.add(new FaRule(startState, null, firstNfaDesign.getStartState()));
        rules.add(new FaRule(startState, null, secondNfaDesign.getStartState()));

        NfaRuleBook ruleBook = new NfaRuleBook(rules);
        return new NfaDesign(startState, acceptStates, ruleBook);
    }
}

// 重复
public class Repeat extends Pattern {
    private Pattern pattern;

    public Repeat(Pattern pattern) {
        this.pattern = pattern;
    }

    @Override
    public String toString() {
        return pattern.bracket(precedence()) + "*";
    }

    @Override
    public int precedence() {
        return 2;
    }

    @Override
    public NfaDesign toNfaDesign() {
        NfaDesign patternNfaDesign = pattern.toNfaDesign();
        Integer startState = IdGen.gen();
        Set<Object> acceptStates = new HashSet<>();
        acceptStates.add(startState);
        acceptStates.addAll(patternNfaDesign.getAcceptStates());

        ArrayList<FaRule> rules = new ArrayList<>(patternNfaDesign.getRuleBook().getRules());

        patternNfaDesign.getAcceptStates().stream()
                .map(acceptState -> new FaRule(acceptState, null, patternNfaDesign.getStartState()))
                .forEach(rules::add);
        rules.add(new FaRule(startState, null, patternNfaDesign.getStartState()));

        NfaRuleBook ruleBook = new NfaRuleBook(rules);
        return new NfaDesign(startState, acceptStates, ruleBook);
    }
}
```

precedence是为了toString的时候正确展示括号，核心的操作是toNfaDesign，把正则转换为NFA。

空字符串和字符转换NFA非常简单，不必多说。对于连接，|，*这三种运算，如何构造NFA呢：

### 连接操作转换NFA

现在能把单个字符的正则表达式 a和 b 转换成 NFA，那怎么把 ab 转成一个 NFA 呢？
对于 ab ，我们可以把两个 NFA 按顺序连接到一起，用自由移动把它们联结在一起，并且保留第二个 NFA 的接受状态：

![nfa3](http://media.kosho.tech/blog/20200420/nfa3.png)

### 选择操作转换NFA

在最简单的情况下，正则表达式 a 和 b 的 NFA 能结合起来构造成正则表达式 a|b 的 NFA，方法是增加一个新的起始状态并使用自由移动把它与两台原始机器之前的起始状态连接起来：

![nfa4](http://media.kosho.tech/blog/20200420/nfa4.png)

### 重复操作转换NFA

如何把与一个字符串匹配的 NFA，转换成能匹配同一个字符串重复零次或者更多次的 NFA 呢？我们为 a* 构造一个 NFA，其开头是一个 a 对应的NFA，然后做两个补充：

1. 从它的接受状态到开始状态增加一个自由移动，这样它就可以与多于一个 • 'a' 匹配了。
2. 增加一个可自由移动到旧的开始状态的新状态，并且使其作为接受状态，这样它就可以匹配空字符串了。
![nfa5](http://media.kosho.tech/blog/20200420/nfa5.png)

再来测试一下，已经可以构造复杂一些的模式来匹配字符串了，把正则转换成NFA来执行。

```java
Pattern repeat = new Repeat(
        new Choose(
                new Concat(
                        new Literal('a'),
                        new Literal('b')),
                new Literal('a')));
Assert.assertEquals(repeat.inspect(), "/(ab|a)*/");
Assert.assertTrue(repeat.matches("ababa"));
```

### 正则语法树解析

虽然我们已经构建了一个正则实现，但是还缺一个语法解析器，如果我们能直接写(ab|a)*然后转换成AST（抽象语法树），而不是上面这一堆的new Repeat xxx手动构造语法树，那就方便多了。

书里用了ruby的一个叫Treetop的库来做这个事，Java的话可以用antlr或者javacc，他们都是类似yacc的语法解析器，可以很方便的实现自己的DSL，我们项目里的类Sql的DSL就是用antlr实现的。

不过这个正则语法比较简单，可以手写递归下降语法分析。

Token定义：

```java
public class Token {
    public static final Token REPEAT = new Token(TokenType.REPEAT, null);

    public static final Token CHOOSE = new Token(TokenType.CHOOSE, null);

    public static final Token LPAR = new Token(TokenType.LPAR, null);

    public static final Token RPAR = new Token(TokenType.RPAR, null);

    public enum TokenType {
        CHAR, LPAR, RPAR, CHOOSE, REPEAT
    }

    public TokenType tokenType;

    public Object value;

    public Token(TokenType tokenType, Object value) {
        this.tokenType = tokenType;
        this.value = value;
    }
}
```

词法解析器：

```java
public class Tokenizer {
    private String regex;

    private Token cur = null;

    private int index = 0;

    public Tokenizer(String regex) {
        this.regex = regex;
        if (regex != null && regex.length() > 0) {
            cur = toToken(regex.charAt(0));
        }
    }

    public Token getToken() {
        return cur;
    }

    public void consumeToken() {
        index++;
        if (index < regex.length()) {
            cur = toToken(regex.charAt(index));
        } else if (index == regex.length()) {
            cur = null;
        } else {
            throw new NoSuchElementException();
        }
    }

    private Token toToken(char ch) {
        switch (ch) {
            case '*':
                return Token.REPEAT;

            case '|':
                return Token.CHOOSE;

            case '(':
                return Token.LPAR;

            case ')':
                return Token.RPAR;

            default:
                return new Token(Token.TokenType.CHAR, ch);
        }
    }
}
```

语法解析器：

```java
public class RegexParser {
    private final Tokenizer tokenizer;

    public RegexParser(String regex) {
        this.tokenizer = new Tokenizer(regex);
    }

    public Pattern parse() {
        return choose();
    }

    private Pattern choose() {
        Pattern left = concatOrEmpty();
        Token cur = tokenizer.getToken();
        if (cur != null && cur.tokenType == CHOOSE) {
            tokenizer.consumeToken();
            return new Choose(left, choose());
        }

        return left;
    }

    private Pattern concatOrEmpty() {
        Token cur = tokenizer.getToken();
        if (cur == null) {
            return new Empty();
        }

        Pattern concat = concat();
        if (concat == null) {
            return new Empty();
        }

        return concat;
    }

    private Pattern concat() {
        Pattern left = repeat();
        if (left == null) {
            return null;
        }

        Token cur = tokenizer.getToken();
        if (cur == null) {
            return left;
        }

        Pattern concat = concat();
        if (concat != null) {
            return new Concat(left, concat);
        }

        return left;
    }

    private Pattern repeat() {
        Pattern left = brackets();
        Token cur = tokenizer.getToken();
        if (cur != null && cur.tokenType == REPEAT) {
            tokenizer.consumeToken();
            return new Repeat(left);
        }
        return left;
    }

    private Pattern brackets() {
        Token cur = tokenizer.getToken();
        if (cur.tokenType == LPAR) {
            tokenizer.consumeToken();
            Pattern choose = choose();
            match(tokenizer.getToken(), RPAR);
            return choose;
        }

        return literal();
    }

    private Pattern literal() {
        Token cur = tokenizer.getToken();
        if (cur.tokenType == CHAR) {
            tokenizer.consumeToken();
            return new Literal((Character) cur.value);
        }

        return null;
    }

    private void match(Token token, Token.TokenType tokenType) {
        if (token.tokenType != tokenType) {
            throw new IllegalStateException();
        }

        tokenizer.consumeToken();
    }
}
```

运算符的优先级从上到下越来越高，| 运算符的绑定最宽松，因此 choose 规则在最前面。另外还要考虑带有括号的情况。

最后给Pattern类加上编译函数

```java
public static Pattern compile(String regex) {
    return new RegexParser(regex).parse();
}
```

现在测试一下这个正则解析器：

```java
Pattern pattern = Pattern.compile("(a(|b))*");
Assert.assertEquals("/(a(|b))*/", pattern.inspect());
Assert.assertTrue(pattern.matches("ababa"));
Assert.assertFalse(pattern.matches("ababac"));
```

完全没毛病。

## 等价性

非确定性和自由移动让设计有限状态机执行特定的工作更容易，可以很方便的把正则转换为NFA。但是NFA做了什么DFA做不了的事吗？其实没有，他们完全等价。

DFA是从一个当前状态移动到另一个，而NFA是从一个当前可能状态的集合移动到另一个可能状态的集合。尽管一个 NFA 的规则手册可以是非确定性的，但是对于一个给定的输入从当前状态出发移动到哪些状态，这个决定总是完全确定性的。

所以完全可以用一台DFA来模拟NFA：

```java
public class NfaSimulation {
    private NfaDesign nfaDesign;

    public NfaSimulation(NfaDesign nfaDesign) {
        this.nfaDesign = nfaDesign;
    }

    public Set<Object> nextState(Set<Object> state, char character) {
        Nfa nfa = nfaDesign.toNfa(state);
        nfa.readCharacter(character);
        return nfa.getCurrentStates();
    }

    public List<FaRule> rulesFor(Set<Object> state) {
        return nfaDesign.getRuleBook().alphabet().stream()
                .map(ch -> new FaRule(state, ch, nextState(state, ch))).collect(Collectors.toList());
    }

    public Pair<Set<Set<Object>>, List<FaRule>> discoverStatesAndRules(Set<Set<Object>> states) {
        List<FaRule> rules = states.stream()
                .map(this::rulesFor)
                .flatMap(Collection::stream)
                .collect(Collectors.toList());
        Set<Object> moreStates = rules.stream().map(FaRule::follow).collect(Collectors.toSet());
        if (states.containsAll(moreStates)) {
            return new Pair<>(states, rules);
        } else {
            Set<Set<Object>> newStates = new HashSet<>(states);
            moreStates.forEach(state -> newStates.add((Set<Object>) state));
            return discoverStatesAndRules(newStates);
        }
    }

    public DfaDesign toDfaDesign() {
        Set<Object> startStates = nfaDesign.toNfa().getCurrentStates();
        Pair<Set<Set<Object>>, List<FaRule>> statesAndRules = discoverStatesAndRules(Sets.create(startStates));
        Set<Set<Object>> states = statesAndRules.getLeft();
        List<FaRule> rules = statesAndRules.getRight();
        Set<Set<Object>> acceptStates = states.stream().filter(state -> nfaDesign.toNfa(state).accepting()).collect(Collectors.toSet());
        return new DfaDesign(startStates, acceptStates.toArray(), new DfaRuleBook(rules));
    }
}
```

再次测试：

```java
Pattern pattern = Pattern.compile("a(b|c)*d");
Assert.assertTrue(pattern.matches("abcbccbccd"));
Assert.assertFalse(pattern.matches("abcbccca"));

NfaDesign nfaDesign = pattern.toNfaDesign();
DfaDesign dfaDesign = new NfaSimulation(nfaDesign).toDfaDesign();
Assert.assertTrue(dfaDesign.accepts("abcbccbccd"));
Assert.assertFalse(dfaDesign.accepts("abcbcccae"));
```

从性能上来说，DFA显然要比NFA高不少。DFA的时间复杂度是O(n)，NFA则是O(nm^2)，n是字符串长度，m是节点数。

那为什么不用DFA实现正则呢？有这么两个原因：

1. NFA可以保留正则表达式的结构，实现捕获组等特性需要。
2. DFA构造，最小化时间复杂度高，占用空间大（极限情况是NFA子集数量的节点数）。

Java原生Pattern的实现，是直接将输入字符串编译成NFA结构，匹配时用回溯的方式进行深度优先遍历（书里是广度优先遍历，每一步计算出集合所有元素）



完整代码放到了我的github上：

https://github.com/koshox/understanding-computation-java