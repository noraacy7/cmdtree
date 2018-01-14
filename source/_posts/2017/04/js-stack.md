---
title: 深入理解JavaScript错误和堆栈追踪
date: 2017-04-20T00:21:11+00:00
category: JavaScript
---

有时候人们并不关注这些细节，但这方面的知识肯定有用，尤其是当你正在编写与测试或errors相关的库。例如这个星期我们的chai中出现了一个令人惊叹的<a href="https://github.com/chaijs/chai/pull/922">Pull Request</a>，它大大改进了我们处理堆栈跟踪的方式，并在用户断言失败时提供了更多的信息。

操作堆栈记录可以让你清理无用数据，并集中精力处理重要事项。此外，当你真正弄清楚Error及其属性，你将会更有信心地利用它。

<strong>本文开头部分或许太过于简单，但当你开始处理堆栈记录时，它将变得稍微有些复杂，所以请确保你在开始这个那部分章节之前已经充分理解前面的内容</strong>。
<h2 id="-"><strong>堆栈调用如何工作</strong></h2>
在谈论errors之前我们必须明白堆栈调用如何工作。它非常简单，但对于我们将要深入的内容而言却是至关重要的。如果你已经知道这部分内容，请随时跳过本节。

<strong>每当函数被调用，它都会被推到堆栈的顶部。函数执行完毕，便会从堆栈顶部移除。</strong>

这种数据结构的有趣之处在于<strong>最后一个入栈的将会第一个从堆栈中移除</strong>，这也就是我们所熟悉的LIFO(后进，先出)特性。

这也就是说我们在函数<code>x</code>中调用函数<code>y</code>,那么对应的堆栈中的顺序为<code>x</code> <code>y</code>。

假设你有下面这样的代码：
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> c</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'c'</span><span class="pun">);</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> b</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'b'</span><span class="pun">);</span><span class="pln">
    c</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> a</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'a'</span><span class="pun">);</span><span class="pln">
    b</span><span class="pun">();</span>
<span class="pun">}</span><span class="pln">

a</span><span class="pun">();</span></code></pre>
在上面这里例子中，当执行<code>a</code>函数时，<code>a</code>便会添加到堆栈的顶部，然后当<code>b</code>函数在<code>a</code>函数中被调用，<code>b</code>也会被添加到堆栈的顶部，依次类推，在<code>b</code>中调用<code>c</code>也会发生同样的事情。

当<code>c</code>执行时，堆栈中的函数的顺序为<code>a</code> <code>b</code> <code>c</code>

<code>c</code>执行完毕后便会从栈顶移除，这时控制流重新回到了<code>b</code>中，<code>b</code>执行完毕同样也会从栈顶移除，最后控制流又回到了<code>a</code>中，最后<code>a</code>执行完毕，<code>a</code>也从堆栈中移除。

我们可以利用<code>console.trace()</code>来更好的演示这种行为，它会在控制台打印出当前堆栈中的记录。此外，通常而言你应该从上到下读取堆栈记录。想想下面的每一行代码都是在哪调用的。
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> c</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'c'</span><span class="pun">);</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">trace</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> b</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'b'</span><span class="pun">);</span><span class="pln">
    c</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> a</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'a'</span><span class="pun">);</span><span class="pln">
    b</span><span class="pun">();</span>
<span class="pun">}</span><span class="pln">

a</span><span class="pun">();</span></code></pre>
在Node REPL服务器上运行上述代码会得到如下结果：
<pre class="prettyprint"><code><span class="typ">Trace</span><span class="pln">
    at c </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">:</span><span class="lit">3</span><span class="pun">:</span><span class="lit">9</span><span class="pun">)</span><span class="pln">
    at b </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">:</span><span class="lit">3</span><span class="pun">:</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
    at a </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">:</span><span class="lit">3</span><span class="pun">:</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
    at repl</span><span class="pun">:</span><span class="lit">1</span><span class="pun">:</span><span class="lit">1</span> <span class="com">// &lt;-- For now feel free to ignore anything below this point, these are Node's internals</span><span class="pln">
    at realRunInThisContextScript </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">22</span><span class="pun">:</span><span class="lit">35</span><span class="pun">)</span><span class="pln">
    at sigintHandlersWrap </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">98</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">ContextifyScript</span><span class="pun">.</span><span class="typ">Script</span><span class="pun">.</span><span class="pln">runInThisContext </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">24</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">REPLServer</span><span class="pun">.</span><span class="pln">defaultEval </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">313</span><span class="pun">:</span><span class="lit">29</span><span class="pun">)</span><span class="pln">
    at bound </span><span class="pun">(</span><span class="pln">domain</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">280</span><span class="pun">:</span><span class="lit">14</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">REPLServer</span><span class="pun">.</span><span class="pln">runBound </span><span class="pun">[</span><span class="kwd">as</span> <span class="kwd">eval</span><span class="pun">]</span> <span class="pun">(</span><span class="pln">domain</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">293</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span></code></pre>
如你所见，当我们在<code>c</code>中打印堆栈，堆栈中的记录为<code>a</code>,<code>b</code>,<code>c</code>。

如果我们现在在<code>b</code>中并且在<code>c</code>执行完之后打印堆栈，我们将会发现<code>c</code>已经从堆栈的顶部移除，只剩下了<code>a</code>和<code>b</code>。
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> c</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'c'</span><span class="pun">);</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> b</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'b'</span><span class="pun">);</span><span class="pln">
    c</span><span class="pun">();</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">trace</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> a</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'a'</span><span class="pun">);</span><span class="pln">
    b</span><span class="pun">();</span>
<span class="pun">}</span><span class="pln">

a</span><span class="pun">();</span></code></pre>
正如你看到的那样，堆栈中已经没有<code>c</code>，因为它已经完成运行，已经被弹出去了。
<pre class="prettyprint"><code><span class="typ">Trace</span><span class="pln">
    at b </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">:</span><span class="lit">4</span><span class="pun">:</span><span class="lit">9</span><span class="pun">)</span><span class="pln">
    at a </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">:</span><span class="lit">3</span><span class="pun">:</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
    at repl</span><span class="pun">:</span><span class="lit">1</span><span class="pun">:</span><span class="lit">1</span>  <span class="com">// &lt;-- For now feel free to ignore anything below this point, these are Node's internals</span><span class="pln">
    at realRunInThisContextScript </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">22</span><span class="pun">:</span><span class="lit">35</span><span class="pun">)</span><span class="pln">
    at sigintHandlersWrap </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">98</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">ContextifyScript</span><span class="pun">.</span><span class="typ">Script</span><span class="pun">.</span><span class="pln">runInThisContext </span><span class="pun">(</span><span class="pln">vm</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">24</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">REPLServer</span><span class="pun">.</span><span class="pln">defaultEval </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">313</span><span class="pun">:</span><span class="lit">29</span><span class="pun">)</span><span class="pln">
    at bound </span><span class="pun">(</span><span class="pln">domain</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">280</span><span class="pun">:</span><span class="lit">14</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">REPLServer</span><span class="pun">.</span><span class="pln">runBound </span><span class="pun">[</span><span class="kwd">as</span> <span class="kwd">eval</span><span class="pun">]</span> <span class="pun">(</span><span class="pln">domain</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">293</span><span class="pun">:</span><span class="lit">12</span><span class="pun">)</span><span class="pln">
    at </span><span class="typ">REPLServer</span><span class="pun">.</span><span class="pln">onLine </span><span class="pun">(</span><span class="pln">repl</span><span class="pun">.</span><span class="pln">js</span><span class="pun">:</span><span class="lit">513</span><span class="pun">:</span><span class="lit">10</span><span class="pun">)</span></code></pre>
总结：调用方法，方法便会添加到堆栈顶部，执行完毕之后，它就会从堆栈中弹出。
<h2 id="-error-error-"><strong>Error对象 和 Error处理</strong></h2>
当程序发生错误时，通常都会抛出一个<code>Error</code>对象。<code>Error</code>对象也可以作为一个原型，用户可以扩展它并创建自定义错误。

<code>Error.prototype</code>对象通常有以下属性：
<ul>
     <li><code>constructor</code>- 实例原型的构造函数。</li>
     <li><code>message</code> - 错误信息</li>
     <li><code>name</code> - 错误名称</li>
</ul>
以上都是标准属性，（但）有时候每个环境都有其特定的属性，在例如Node，Firefox，Chorme，Edge，IE 10+，Opera 和 Safari 6+ 中，还有一个包含错误堆栈记录的<code>stack</code>属性。<strong>错误堆栈记录包含从（堆栈底部）它自己的构造函数到（堆栈顶部）所有的堆栈帧。</strong>

如果想了解更多关于<code>Error</code>对象的具体属性，我强烈推荐MDN上的<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/prototype">这篇文章</a>。

抛出错误必须使用<code>throw</code>关键字，你必须将可能抛出错误的代码包裹在<code>try</code>代码块内并紧跟着一个<code>catch</code>代码块来捕获抛出的错误。

正如Java中的错误处理,<code>try/catch</code>代码块后紧跟着一个<code>finally</code>代码块在JavaScript中也是同样允许的，无论<code>try</code>代码块内是否抛出异常，<code>finally</code>代码块内的代码都会执行。在完成处理之后，最佳实践是在<code>finally</code>代码块中做一些清理的事情，(因为)无论你的操作是否生效，都不会影响到它的执行。

(鉴于)上面所谈到的所有事情对大多数人来讲都是小菜一碟，那么就让我们来谈一些不为人所知的细节。

<code>try</code>代码块后面不必紧跟着<code>catch</code>,但(此种情况下)其后必须紧跟着<code>finally</code>。这意味着我们可以使用三种不同形式的<code>try</code>语句：
<ul>
     <li><code>try...catch</code></li>
     <li><code>try...finally</code></li>
     <li><code>try...catch...finally</code></li>
</ul>
Try语句可以像下面这样互相嵌套：
<pre class="prettyprint"><code><span class="kwd">try</span> <span class="pun">{</span>
    <span class="kwd">try</span> <span class="pun">{</span>
        <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">Error</span><span class="pun">(</span><span class="str">'Nested error.'</span><span class="pun">);</span> <span class="com">// The error thrown here will be caught by its own `catch` clause</span>
    <span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">nestedErr</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'Nested catch'</span><span class="pun">);</span> <span class="com">// This runs</span>
    <span class="pun">}</span>
<span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">err</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'This will not run.'</span><span class="pun">);</span>
<span class="pun">}</span></code></pre>
你甚至还可以在<code>catch</code>和<code>finally</code>代码块中嵌套<code>try</code>语句：
<pre class="prettyprint"><code><span class="kwd">try</span> <span class="pun">{</span>
    <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">Error</span><span class="pun">(</span><span class="str">'First error'</span><span class="pun">);</span>
<span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">err</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'First catch running'</span><span class="pun">);</span>
    <span class="kwd">try</span> <span class="pun">{</span>
        <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">Error</span><span class="pun">(</span><span class="str">'Second error'</span><span class="pun">);</span>
    <span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">nestedErr</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'Second catch running.'</span><span class="pun">);</span>
    <span class="pun">}</span>
<span class="pun">}</span></code></pre>
<pre class="prettyprint"><code><span class="kwd">try</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'The try block is running...'</span><span class="pun">);</span>
<span class="pun">}</span> <span class="kwd">finally</span> <span class="pun">{</span>
    <span class="kwd">try</span> <span class="pun">{</span>
        <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">Error</span><span class="pun">(</span><span class="str">'Error inside finally.'</span><span class="pun">);</span>
    <span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">err</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'Caught an error inside the finally block.'</span><span class="pun">);</span>
    <span class="pun">}</span>
<span class="pun">}</span></code></pre>
还有很重要的一点值得注意，那就是我们甚至可以大可不必抛出<code>Error</code>对象。尽管这看起来非常cool且非常自由，但实际并非如此，尤其是对开发第三方库的开发者来说，因为他们必须处理用户(使用库的开发者)的代码。由于缺乏标准，他们并不能把控用户的行为。你不能相信用户并简单的抛出一个<code>Error</code>对象，因为他们不一定会那么做而是仅仅抛出一个字符串或者数字(鬼知道用户会抛出什么)。这也使得处理必要的堆栈跟踪和其他有意义的元数据变得更加困难。

假设有以下代码：
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> runWithoutThrowing</span><span class="pun">(</span><span class="pln">func</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">try</span> <span class="pun">{</span><span class="pln">
        func</span><span class="pun">();</span>
    <span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">e</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'There was an error, but I will not throw it.'</span><span class="pun">);</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'The error\'s message was: '</span> <span class="pun">+</span><span class="pln"> e</span><span class="pun">.</span><span class="pln">message</span><span class="pun">)</span>
    <span class="pun">}</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> funcThatThrowsError</span><span class="pun">()</span> <span class="pun">{</span>
    <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">TypeError</span><span class="pun">(</span><span class="str">'I am a TypeError.'</span><span class="pun">);</span>
<span class="pun">}</span><span class="pln">

runWithoutThrowing</span><span class="pun">(</span><span class="pln">funcThatThrowsError</span><span class="pun">);</span></code></pre>
如果你的用户像上面这样传递一个抛出<code>Error</code>对象的函数给<code>runWithoutThrowing</code>函数(那就谢天谢地了)，然而总有些人偷想懒直接抛出一个<code>String</code>,那你就麻烦了：
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> runWithoutThrowing</span><span class="pun">(</span><span class="pln">func</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">try</span> <span class="pun">{</span><span class="pln">
        func</span><span class="pun">();</span>
    <span class="pun">}</span> <span class="kwd">catch</span> <span class="pun">(</span><span class="pln">e</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'There was an error, but I will not throw it.'</span><span class="pun">);</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'The error\'s message was: '</span> <span class="pun">+</span><span class="pln"> e</span><span class="pun">.</span><span class="pln">message</span><span class="pun">)</span>
    <span class="pun">}</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> funcThatThrowsString</span><span class="pun">()</span> <span class="pun">{</span>
    <span class="kwd">throw</span> <span class="str">'I am a String.'</span><span class="pun">;</span>
<span class="pun">}</span><span class="pln">

runWithoutThrowing</span><span class="pun">(</span><span class="pln">funcThatThrowsString</span><span class="pun">);</span></code></pre>
现在第二个<code>console.log</code>会打印出 the error’s message is <code>undefined</code>.这么看来也没多大的事(后果)呀，但是如果您需要确保某些属性存在于<code>Error</code>对象上，或以另一种方式（例如<a href="https://github.com/chaijs/chai/blob/a7e1200db4c144263599e5dd7a3f7d1893467160/lib/chai/core/assertions.js#L1506">Chai的<code>throws</code>断言</a> does)）处理<code>Error</code>对象的特定属性，那么你做需要更多的工作，以确保它会正常工资。

此外，当抛出的值不是<code>Error</code>对象时，你无法访问其他重要数据，例如<code>stack</code>，在某些环境中它是<code>Error</code>对象的一个属性。

Errors也可以像其他任何对象一样使用，并不一定非得要抛出他们，这也是它们为什么多次被用作回调函数的第一个参数(俗称 err first)。 在下面的<code>fs.readdir()</code>例子中就是这么用的。
<pre class="prettyprint"><code><span class="kwd">const</span><span class="pln"> fs </span><span class="pun">=</span> <span class="kwd">require</span><span class="pun">(</span><span class="str">'fs'</span><span class="pun">);</span><span class="pln">

fs</span><span class="pun">.</span><span class="pln">readdir</span><span class="pun">(</span><span class="str">'/example/i-do-not-exist'</span><span class="pun">,</span> <span class="kwd">function</span><span class="pln"> callback</span><span class="pun">(</span><span class="pln">err</span><span class="pun">,</span><span class="pln"> dirs</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="pln">err </span><span class="kwd">instanceof</span> <span class="typ">Error</span><span class="pun">)</span> <span class="pun">{</span>
        <span class="com">// `readdir` will throw an error because that directory does not exist</span>
        <span class="com">// We will now be able to use the error object passed by it in our callback function</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'Error Message: '</span> <span class="pun">+</span><span class="pln"> err</span><span class="pun">.</span><span class="pln">message</span><span class="pun">);</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'See? We can use Errors without using try statements.'</span><span class="pun">);</span>
    <span class="pun">}</span> <span class="kwd">else</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="pln">dirs</span><span class="pun">);</span>
    <span class="pun">}</span>
<span class="pun">});</span></code></pre>
最后，在rejecting promises时也可以使用<code>Error</code>对象。这使得它更容易处理promise rejections：
<pre class="prettyprint"><code><span class="kwd">new</span> <span class="typ">Promise</span><span class="pun">(</span><span class="kwd">function</span><span class="pun">(</span><span class="pln">resolve</span><span class="pun">,</span><span class="pln"> reject</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
    reject</span><span class="pun">(</span><span class="kwd">new</span> <span class="typ">Error</span><span class="pun">(</span><span class="str">'The promise was rejected.'</span><span class="pun">));</span>
<span class="pun">}).</span><span class="kwd">then</span><span class="pun">(</span><span class="kwd">function</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'I am an error.'</span><span class="pun">);</span>
<span class="pun">}).</span><span class="kwd">catch</span><span class="pun">(</span><span class="kwd">function</span><span class="pun">(</span><span class="pln">err</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="pln">err </span><span class="kwd">instanceof</span> <span class="typ">Error</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'The promise was rejected with an error.'</span><span class="pun">);</span><span class="pln">
        console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="str">'Error Message: '</span> <span class="pun">+</span><span class="pln"> err</span><span class="pun">.</span><span class="pln">message</span><span class="pun">);</span>
    <span class="pun">}</span>
<span class="pun">});</span></code></pre>
<h2 id="-"><strong>操纵堆栈跟踪</strong></h2>
上面啰嗦了那么多，压轴的重头戏来了，那就是如何操纵堆栈跟踪。

本章专门针对那些像NodeJS支<a href="https://nodejs.org/api/errors.html">Error.captureStackTrace</a>的环境。

<code>Error.captureStackTrace</code>函数接受一个<code>object</code>作为第一个参数，第二个参数是可选的，接受一个函数。capture stack trace 捕获当前堆栈跟踪，并在目标对象中创建一个<code>stack</code>属性来存储它。如果提供了第二个参数，则传递的函数将被视为调用堆栈的终点，因此堆栈跟踪将仅显示调用该函数之前发生的调用。

让我们用例子来说明这一点。首先，我们将捕获当前堆栈跟踪并将其存储在公共对象中。
<pre class="prettyprint"><code><span class="kwd">const</span><span class="pln"> myObj </span><span class="pun">=</span> <span class="pun">{};</span>

<span class="kwd">function</span><span class="pln"> c</span><span class="pun">()</span> <span class="pun">{</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> b</span><span class="pun">()</span> <span class="pun">{</span>
    <span class="com">// Here we will store the current stack trace into myObj</span>
    <span class="typ">Error</span><span class="pun">.</span><span class="pln">captureStackTrace</span><span class="pun">(</span><span class="pln">myObj</span><span class="pun">);</span><span class="pln">
    c</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> a</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    b</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="com">// First we will call these functions</span><span class="pln">
a</span><span class="pun">();</span>

<span class="com">// Now let's see what is the stack trace stored into myObj.stack</span><span class="pln">
console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="pln">myObj</span><span class="pun">.</span><span class="pln">stack</span><span class="pun">);</span>

<span class="com">// This will print the following stack to the console:</span>
<span class="com">//    at b (repl:3:7) &lt;-- Since it was called inside B, the B call is the last entry in the stack</span>
<span class="com">//    at a (repl:2:1)</span>
<span class="com">//    at repl:1:1 &lt;-- Node internals below this line</span>
<span class="com">//    at realRunInThisContextScript (vm.js:22:35)</span>
<span class="com">//    at sigintHandlersWrap (vm.js:98:12)</span>
<span class="com">//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)</span>
<span class="com">//    at REPLServer.defaultEval (repl.js:313:29)</span>
<span class="com">//    at bound (domain.js:280:14)</span>
<span class="com">//    at REPLServer.runBound [as eval] (domain.js:293:12)</span>
<span class="com">//    at REPLServer.onLine (repl.js:513:10)</span></code></pre>
不知道你注意到没，我们首先调用了<code>a</code>(<code>a</code>入栈)，然后我们<code>a</code>中又调用了<code>b</code>(<code>b</code>入栈且在<code>a</code>之上)。然后在<code>b</code>中我们捕获了当前堆栈记录并将其存储在<code>myObj</code>中。因此在控制台中才会按照<code>b</code> <code>a</code>的顺序打印堆栈。

现在让我们给<code>Error.captureStackTrace</code>传递一个函数作为第二个参数，看看会发生什么：
<pre class="prettyprint"><code><span class="kwd">const</span><span class="pln"> myObj </span><span class="pun">=</span> <span class="pun">{};</span>

<span class="kwd">function</span><span class="pln"> d</span><span class="pun">()</span> <span class="pun">{</span>
    <span class="com">// Here we will store the current stack trace into myObj</span>
    <span class="com">// This time we will hide all the frames after `b` and `b` itself</span>
    <span class="typ">Error</span><span class="pun">.</span><span class="pln">captureStackTrace</span><span class="pun">(</span><span class="pln">myObj</span><span class="pun">,</span><span class="pln"> b</span><span class="pun">);</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> c</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    d</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> b</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    c</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="kwd">function</span><span class="pln"> a</span><span class="pun">()</span> <span class="pun">{</span><span class="pln">
    b</span><span class="pun">();</span>
<span class="pun">}</span>

<span class="com">// First we will call these functions</span><span class="pln">
a</span><span class="pun">();</span>

<span class="com">// Now let's see what is the stack trace stored into myObj.stack</span><span class="pln">
console</span><span class="pun">.</span><span class="pln">log</span><span class="pun">(</span><span class="pln">myObj</span><span class="pun">.</span><span class="pln">stack</span><span class="pun">);</span>

<span class="com">// This will print the following stack to the console:</span>
<span class="com">//    at a (repl:2:1) &lt;-- As you can see here we only get frames before `b` was called</span>
<span class="com">//    at repl:1:1 &lt;-- Node internals below this line</span>
<span class="com">//    at realRunInThisContextScript (vm.js:22:35)</span>
<span class="com">//    at sigintHandlersWrap (vm.js:98:12)</span>
<span class="com">//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)</span>
<span class="com">//    at REPLServer.defaultEval (repl.js:313:29)</span>
<span class="com">//    at bound (domain.js:280:14)</span>
<span class="com">//    at REPLServer.runBound [as eval] (domain.js:293:12)</span>
<span class="com">//    at REPLServer.onLine (repl.js:513:10)</span>
<span class="com">//    at emitOne (events.js:101:20)</span></code></pre>
当把<code>b</code>传给<code>Error.captureStackTraceFunction</code>时，它隐藏了<code>b</code>本身以及它之后所有的调用帧。因此控制台仅仅打印出一个<code>a</code>。

至此你应该会问自己：“这到底有什么用？”。这非常有用，因为你可以用它来隐藏与用户无关的内部实现细节。在Chai中，我们使用它来避免向用户显示我们是如何实施检查和断言本身的不相关的细节。
<h2 id="-"><strong>操作堆栈追踪实战</strong></h2>
正如我在上一节中提到的，Chai使用堆栈操作技术使堆栈跟踪更加与我们的用户相关。下面将揭晓我们是如何做到的。

首先，让我们来看看当断言失败时抛出的<code>AssertionError</code>的构造函数：
<pre class="prettyprint"><code><span class="com">// `ssfi` stands for "start stack function". It is the reference to the</span>
<span class="com">// starting point for removing irrelevant frames from the stack trace</span>
<span class="kwd">function</span> <span class="typ">AssertionError</span> <span class="pun">(</span><span class="pln">message</span><span class="pun">,</span><span class="pln"> _props</span><span class="pun">,</span><span class="pln"> ssf</span><span class="pun">)</span> <span class="pun">{</span>
  <span class="kwd">var</span><span class="pln"> extend </span><span class="pun">=</span><span class="pln"> exclude</span><span class="pun">(</span><span class="str">'name'</span><span class="pun">,</span> <span class="str">'message'</span><span class="pun">,</span> <span class="str">'stack'</span><span class="pun">,</span> <span class="str">'constructor'</span><span class="pun">,</span> <span class="str">'toJSON'</span><span class="pun">)</span>
    <span class="pun">,</span><span class="pln"> props </span><span class="pun">=</span><span class="pln"> extend</span><span class="pun">(</span><span class="pln">_props </span><span class="pun">||</span> <span class="pun">{});</span>

  <span class="com">// Default values</span>
  <span class="kwd">this</span><span class="pun">.</span><span class="pln">message </span><span class="pun">=</span><span class="pln"> message </span><span class="pun">||</span> <span class="str">'Unspecified AssertionError'</span><span class="pun">;</span>
  <span class="kwd">this</span><span class="pun">.</span><span class="pln">showDiff </span><span class="pun">=</span> <span class="kwd">false</span><span class="pun">;</span>

  <span class="com">// Copy from properties</span>
  <span class="kwd">for</span> <span class="pun">(</span><span class="kwd">var</span><span class="pln"> key </span><span class="kwd">in</span><span class="pln"> props</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">this</span><span class="pun">[</span><span class="pln">key</span><span class="pun">]</span> <span class="pun">=</span><span class="pln"> props</span><span class="pun">[</span><span class="pln">key</span><span class="pun">];</span>
  <span class="pun">}</span>

  <span class="com">// Here is what is relevant for us:</span>
  <span class="com">// If a start stack function was provided we capture the current stack trace and pass</span>
  <span class="com">// it to the `captureStackTrace` function so we can remove frames that come after it</span><span class="pln">
  ssf </span><span class="pun">=</span><span class="pln"> ssf </span><span class="pun">||</span><span class="pln"> arguments</span><span class="pun">.</span><span class="pln">callee</span><span class="pun">;</span>
  <span class="kwd">if</span> <span class="pun">(</span><span class="pln">ssf </span><span class="pun">&amp;&amp;</span> <span class="typ">Error</span><span class="pun">.</span><span class="pln">captureStackTrace</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="typ">Error</span><span class="pun">.</span><span class="pln">captureStackTrace</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span><span class="pln"> ssf</span><span class="pun">);</span>
  <span class="pun">}</span> <span class="kwd">else</span> <span class="pun">{</span>
    <span class="com">// If no start stack function was provided we just use the original stack property</span>
    <span class="kwd">try</span> <span class="pun">{</span>
      <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">Error</span><span class="pun">();</span>
    <span class="pun">}</span> <span class="kwd">catch</span><span class="pun">(</span><span class="pln">e</span><span class="pun">)</span> <span class="pun">{</span>
      <span class="kwd">this</span><span class="pun">.</span><span class="pln">stack </span><span class="pun">=</span><span class="pln"> e</span><span class="pun">.</span><span class="pln">stack</span><span class="pun">;</span>
    <span class="pun">}</span>
  <span class="pun">}</span>
<span class="pun">}</span></code></pre>
如你所见，我们使用<code>Error.captureStackTrace</code>捕获堆栈追踪并将它存储在我们正在创建的<code>AssertError</code>实例中（如果存在的话），然后我们将一个起始堆栈函数传递给它，以便从堆栈跟踪中删除不相关的调用帧，它只显示Chai的内部实现细节，最终使堆栈变得清晰明了。

现在让我们来看看<a href="https://github.com/meeber">@meeber</a>在这个<a href="https://github.com/chaijs/chai/pull/922">令人惊叹的PR</a>中提交的代码。

在你开始看下面的代码之前，我必须告诉你<code>addChainableMethod</code>方法是干啥的。它将传递给它的链式方法添加到断言上，它也用包含断言的方法标记断言本身，并将其保存在变量<code>ssfi</code>(启动堆栈函数指示符)中。这也就意味着当前断言将会是堆栈中的最后一个调用帧，因此我们不会在堆栈中显示Chai中的任何进一步的内部方法。我没有添加整个代码，因为它做了很多事情，有点棘手，但如果你想读它，<a href="https://github.com/meeber/chai/blob/42ff3c012b8a5978e7381b17d712521299ced341/lib/chai/utils/addChainableMethod.js">点我阅读</a>。

下面的这个代码片段中，我们有一个<code>lengOf</code>断言的逻辑，它检查一个对象是否有一定的<code>length</code>。我们希望用户可以像这样来使用它：<code>expect(['foo', 'bar']).to.have.lengthOf(2)</code>。
<pre class="prettyprint"><code><span class="kwd">function</span><span class="pln"> assertLength </span><span class="pun">(</span><span class="pln">n</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="pln">msg</span><span class="pun">)</span><span class="pln"> flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'message'</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">);</span>
    <span class="kwd">var</span><span class="pln"> obj </span><span class="pun">=</span><span class="pln"> flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'object'</span><span class="pun">)</span>
        <span class="pun">,</span><span class="pln"> ssfi </span><span class="pun">=</span><span class="pln"> flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'ssfi'</span><span class="pun">);</span>

    <span class="com">// Pay close attention to this line</span>
    <span class="kwd">new</span> <span class="typ">Assertion</span><span class="pun">(</span><span class="pln">obj</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">,</span><span class="pln"> ssfi</span><span class="pun">,</span> <span class="kwd">true</span><span class="pun">).</span><span class="pln">to</span><span class="pun">.</span><span class="pln">have</span><span class="pun">.</span><span class="pln">property</span><span class="pun">(</span><span class="str">'length'</span><span class="pun">);</span>
    <span class="kwd">var</span><span class="pln"> len </span><span class="pun">=</span><span class="pln"> obj</span><span class="pun">.</span><span class="pln">length</span><span class="pun">;</span>

    <span class="com">// This line is also relevant</span>
    <span class="kwd">this</span><span class="pun">.</span><span class="kwd">assert</span><span class="pun">(</span><span class="pln">
            len </span><span class="pun">==</span><span class="pln"> n
        </span><span class="pun">,</span> <span class="str">'expected #{this} to have a length of #{exp} but got #{act}'</span>
        <span class="pun">,</span> <span class="str">'expected #{this} to not have a length of #{act}'</span>
        <span class="pun">,</span><span class="pln"> n
        </span><span class="pun">,</span><span class="pln"> len
    </span><span class="pun">);</span>
<span class="pun">}</span>

<span class="typ">Assertion</span><span class="pun">.</span><span class="pln">addChainableMethod</span><span class="pun">(</span><span class="str">'lengthOf'</span><span class="pun">,</span><span class="pln"> assertLength</span><span class="pun">,</span><span class="pln"> assertLengthChain</span><span class="pun">);</span></code></pre>
在上面的代码片段中，我突出强调了与我们现在相关的代码。让我们从调用<code>this.assert</code>开始说起。

以下是<code>this.assert</code>方法的源代码：
<pre class="prettyprint"><code><span class="typ">Assertion</span><span class="pun">.</span><span class="pln">prototype</span><span class="pun">.</span><span class="kwd">assert</span> <span class="pun">=</span> <span class="kwd">function</span> <span class="pun">(</span><span class="pln">expr</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">,</span><span class="pln"> negateMsg</span><span class="pun">,</span><span class="pln"> expected</span><span class="pun">,</span><span class="pln"> _actual</span><span class="pun">,</span><span class="pln"> showDiff</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="kwd">var</span><span class="pln"> ok </span><span class="pun">=</span><span class="pln"> util</span><span class="pun">.</span><span class="pln">test</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span><span class="pln"> arguments</span><span class="pun">);</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="kwd">false</span> <span class="pun">!==</span><span class="pln"> showDiff</span><span class="pun">)</span><span class="pln"> showDiff </span><span class="pun">=</span> <span class="kwd">true</span><span class="pun">;</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="kwd">undefined</span> <span class="pun">===</span><span class="pln"> expected </span><span class="pun">&amp;&amp;</span> <span class="kwd">undefined</span> <span class="pun">===</span><span class="pln"> _actual</span><span class="pun">)</span><span class="pln"> showDiff </span><span class="pun">=</span> <span class="kwd">false</span><span class="pun">;</span>
    <span class="kwd">if</span> <span class="pun">(</span><span class="kwd">true</span> <span class="pun">!==</span><span class="pln"> config</span><span class="pun">.</span><span class="pln">showDiff</span><span class="pun">)</span><span class="pln"> showDiff </span><span class="pun">=</span> <span class="kwd">false</span><span class="pun">;</span>

    <span class="kwd">if</span> <span class="pun">(!</span><span class="pln">ok</span><span class="pun">)</span> <span class="pun">{</span><span class="pln">
        msg </span><span class="pun">=</span><span class="pln"> util</span><span class="pun">.</span><span class="pln">getMessage</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span><span class="pln"> arguments</span><span class="pun">);</span>
        <span class="kwd">var</span><span class="pln"> actual </span><span class="pun">=</span><span class="pln"> util</span><span class="pun">.</span><span class="pln">getActual</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span><span class="pln"> arguments</span><span class="pun">);</span>

        <span class="com">// This is the relevant line for us</span>
        <span class="kwd">throw</span> <span class="kwd">new</span> <span class="typ">AssertionError</span><span class="pun">(</span><span class="pln">msg</span><span class="pun">,</span> <span class="pun">{</span><span class="pln">
                actual</span><span class="pun">:</span><span class="pln"> actual
            </span><span class="pun">,</span><span class="pln"> expected</span><span class="pun">:</span><span class="pln"> expected
            </span><span class="pun">,</span><span class="pln"> showDiff</span><span class="pun">:</span><span class="pln"> showDiff
        </span><span class="pun">},</span> <span class="pun">(</span><span class="pln">config</span><span class="pun">.</span><span class="pln">includeStack</span><span class="pun">)</span> <span class="pun">?</span> <span class="kwd">this</span><span class="pun">.</span><span class="kwd">assert</span> <span class="pun">:</span><span class="pln"> flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'ssfi'</span><span class="pun">));</span>
    <span class="pun">}</span>
<span class="pun">};</span></code></pre>
<code>assert</code>方法负责检查断言布尔表达式是否通过。如果不通过，我们则实例化一个<code>AssertionError</code>。不知道你注意到没，在实例化<code>AssertionError</code>时，我们也给它传递了一个堆栈追踪函数指示器(<code>ssfi</code>)，如果配置的<code>includeStack</code>处于开启状态，我们通过将<code>this.assert</code>本身传递给它来为用户显示整个堆栈跟踪。反之，我们则只显示<code>ssfi</code>标记中存储的内容，隐藏掉堆栈跟踪中更多的内部实现细节。

现在让我们来讨论下一行和我们相关的代码吧：
<pre class="prettyprint"><code><span class="str">`new Assertion(obj, msg, ssfi, true).to.have.property('length');`</span></code></pre>
As you can see here we are passing the content we’ve got from the <code>ssfi</code> flag when creating our nested assertion. This means that when the new assertion gets created it will use this function as the starting point for removing unuseful frames from the stack trace. By the way, this is the <code>Assertion</code> constructor: 如你所见，我们在创建嵌套断言时将从<code>ssfi</code>标记中的内容传递给了它。这意味着新创建的断言会使用那个方法作为起始调用帧，从而可以从堆栈追踪中清除没有的调用栈。顺便也看下<code>Assertion</code>的构造器吧：
<pre class="prettyprint"><code><span class="kwd">function</span> <span class="typ">Assertion</span> <span class="pun">(</span><span class="pln">obj</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">,</span><span class="pln"> ssfi</span><span class="pun">,</span><span class="pln"> lockSsfi</span><span class="pun">)</span> <span class="pun">{</span>
    <span class="com">// This is the line that matters to us</span><span class="pln">
    flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'ssfi'</span><span class="pun">,</span><span class="pln"> ssfi </span><span class="pun">||</span> <span class="typ">Assertion</span><span class="pun">);</span><span class="pln">
    flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'lockSsfi'</span><span class="pun">,</span><span class="pln"> lockSsfi</span><span class="pun">);</span><span class="pln">
    flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'object'</span><span class="pun">,</span><span class="pln"> obj</span><span class="pun">);</span><span class="pln">
    flag</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">,</span> <span class="str">'message'</span><span class="pun">,</span><span class="pln"> msg</span><span class="pun">);</span>

    <span class="kwd">return</span><span class="pln"> util</span><span class="pun">.</span><span class="pln">proxify</span><span class="pun">(</span><span class="kwd">this</span><span class="pun">);</span>
<span class="pun">}</span></code></pre>
不知道你是否还记的我先前说过的<code>addChainableMethod</code>方法，它使用自己的父级方法设置<code>ssfi</code>标志，这意味着它始终处于堆栈的底部，我们可以删除它之上的所有调用帧。

通过将<code>ssfi</code>传递给嵌套断言，它只检查我们的对象是否具有长度属性，我们就可以避免重置我们将要用作起始指标器的调用帧，然后在堆栈中可以看到以前的<code>addChainableMethod</code>。

这可能看起来有点复杂，所以让我们回顾一下我们想从栈中删除无用的调用帧时Chai中所发生的事情：
<ul>
     <li>断言失败时，我们会移除所有我们在参考帧之后保存的内部调用帧。</li>
     <li>当我们运行断言时，我们将它自己的方法作为移除堆栈中的下一个调用帧的参考</li>
     <li>如果存在嵌套的断言。我们必须依旧使用当前断言的父方法作为删除下一个调用帧的参考点，因此我们把当前的<code>ssfi</code>（起始函数指示器）传递给我们所创建的断言，以便它可以保存。</li>
</ul>
&nbsp;
<blockquote>原文地址：<a href="http://lucasfcosta.com/2017/02/17/JavaScript-Errors-and-Stack-Traces.html?utm_source=javascriptweekly&amp;utm_medium=email">http://lucasfcosta.com/2017/02/17/JavaScript-Errors-and-Stack-Traces.html?utm_source=javascriptweekly&amp;utm_medium=email</a></blockquote>