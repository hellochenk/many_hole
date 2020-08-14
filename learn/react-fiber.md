# [ react core 带佬の原文](https://github.com/acdlite/react-fiber-architecture/blob/master/README.md)

## What is a fiber?

<details>
  <summary>
    我们要讨论 react fiber 架构的核心，Fibers 是一个非常低等级的抽象-比开发者通常思考的还要低。如果你发现自己在理解它的努力中受挫，别沮丧。坚持下去最终会有意义.（当你最终搞定它时，请提议如何改进它）
  </summary>

We're about to discuss the heart of React Fiber's architecture. Fibers are a much lower-level abstraction than application developers typically think about. If you find yourself frustrated in your attempts to understand it, don't feel discouraged. Keep trying and it will eventually make sense. (When you do finally get it, please suggest how to improve this section.)
</details>

Here we go!

<details>
  <summary>
    我们已经确认fiber的主要目标是让react可以拥有调度的优点，可以明确我们要做到
    <ul>
      <li>暂停work并过会再返回</li>
      <li>为不同的work分配优先级</li>
      <li>重用已经完成的work</li>
      <li>取消那些不在需要的work</li>
    </ul>
  </summary>
    We've established that a primary goal of Fiber is to enable React to take advantage of scheduling. Specifically, we need to be able to

  - pause work and come back to it later.
  - assign priority to different types of work.
  - reuse previously completed work.
  - abort work if it's no longer needed.
</details>

<details>
  <summary>
    为了做到这一点，首先我们需要一种将work分解成单元的方法。在某种意义上，这就是fiber。<b>fiber表示一个work单元</b>。
  </summary>
  
  In order to do any of this, we first need a way to break work down into units. In one sense, that's what a fiber is. A fiber represents a **unit of work**.
</details>

<details>
  <summary>
    为了更进一步，让我们回到作为数据函数的React组件的概念，通常表示为
    <br/>
    v = f(d)
  </summary>

  To go further, let's go back to the conception of React components as functions of data, commonly expressed as
  ```javascript
    v = f(d)
  ```
</details>

<details>
  <summary>
    因此，呈现React应用程序类似于调用一个函数，该函数的主体包含对其他函数的调用，等等。这个类比对我们理解fiber很有用。
  </summary>
  
  It follows that rendering a React app is akin to calling a function whose body contains calls to other functions, and so on. This analogy is useful when thinking about fibers.
</details>

<details>
  <summary>
    计算机跟踪程序执行的典型方式是使用调用堆栈。当一个函数被执行时，一个新的堆栈帧被添加到堆栈中。该堆栈框架表示由该函数执行的工作。
  </summary>

  The way computers typically track a program's execution is using the call stack. When a function is executed, a new stack frame is added to the stack. That stack frame represents the work that is performed by that function.
</details>

<details>
  <summary>
    在处理ui时，问题是如果一次执行了太多的工作，可能会导致动画拖放帧，看起来卡顿。更重要的是，如果它被更近期的更新所取代，其中一些工作可能是不必要的。这就是UI组件和函数之间的比较的失败之处，因为组件比一般函数有更具体的关注点。
  </summary>

  When dealing with UIs, the problem is that if too much work is executed all at once, it can cause animations to drop frames and look choppy. What's more, some of that work may be unnecessary if it's superseded by a more recent update. This is where the comparison between UI components and function breaks down, because components have more specific concerns than functions in general.
</details>

<details>
  <summary>
    较新的浏览器(和React Native)实现了帮助解决这个问题的api: requestIdleCallback调度一个低优先级函数在空闲期间被调用，而requestAnimationFrame调度一个高优先级函数在下一个动画帧中被调用。问题是，为了使用这些api，您需要一种将呈现工作分解为增量单元的方法。如果您只依赖于调用堆栈，它将继续工作，直到堆栈为空。
  </summary>

  Newer browsers (and React Native) implement APIs that help address this exact problem: requestIdleCallback schedules a low priority function to be called during an idle period, and requestAnimationFrame schedules a high priority function to be called on the next animation frame. The problem is that, in order to use those APIs, you need a way to break rendering work into incremental units. If you rely only on the call stack, it will keep doing work until the stack is empty.
</details>

<details>
  <summary>
    如果我们可以定制调用堆栈的行为来优化ui的呈现，这不是很好吗?如果我们可以随意中断调用堆栈并手动操作堆栈帧，这不是很好吗?
  </summary>

  Wouldn't it be great if we could customize the behavior of the call stack to optimize for rendering UIs? Wouldn't it be great if we could interrupt the call stack at will and manipulate stack frames manually?
</details>

<details>
  <summary>
    这就是React Fiber。Fiber是堆栈的重新实现，专门用于react组件。您可以将单个fiber看作<b>虚拟堆栈帧</b>。
  </summary>

  That's the purpose of React Fiber. Fiber is reimplementation of the stack, specialized for React components. You can think of a single fiber as a **virtual stack frame**.
</details>

<details>
  <summary>
    重新实现堆栈的好处是，您可以将<a link='https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/'>堆栈帧保存在内存中</a>，并在任何需要的时候执行它们。这对于完成我们的计划目标是至关重要的。
  </summary>

  The advantage of reimplementing the stack is that you can [keep stack frames in memory](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/) and execute them however (and *whenever*) you want. This is crucial for accomplishing the goals we have for scheduling.
</details>

<details>
  <summary>
    除了调度之外，手动处理堆栈帧可以解锁并发和错误边界等潜在特性。我们将在以后的章节中讨论这些主题。
  </summary>

  Aside from scheduling, manually dealing with stack frames unlocks the potential for features such as concurrency and error boundaries. We will cover these topics in future sections.
</details>

In the next section, we'll look more at the structure of a fiber.

<details>
  <summary>
    具体地说，fiber是一个JavaScript对象，它包含关于组件、输入和输出的信息。
  </summary>
  In concrete terms, a fiber is a JavaScript object that contains information about a component, its input, and its output.
</details>

<details>
  <summary>
    fiber对应于堆栈帧，但它也对应于组件的实例。
  </summary>

  A fiber corresponds to a stack frame, but it also corresponds to an instance of a component.
</details>

<details>
  <summary>
    以下是一些属于fiber的重要领域。(这个列表并不详尽。)
  </summary>

  Here are some of the important fields that belong to a fiber. (This list is not exhaustive.)
</details>

<details>
  <summary>
    <b>type</b>和<b>key</b>
  </summary>

  **type** and **key**
</details>

<details>
  <summary>
    fiber的type和key与React的作用是一样的。(实际上，当从一个元素创建一个fiber时，这两个字段会被直接复制。)
  </summary>

  The type and key of a fiber serve the same purpose as they do for React elements. (In fact, when a fiber is created from an element, these two fields are copied over directly.)
</details>

<details>
  <summary>
    fiber的类型描述了它所对应的组分。对于复合组件，类型是函数或类组件本身。对于主组件(div、span等)，类型是一个字符串。
  </summary>

  The type of a fiber describes the component that it corresponds to. For composite components, the type is the function or class component itself. For host components (div, span, etc.), the type is a string.
</details>

<details>
  <summary>
    从概念上讲，类型是堆栈帧跟踪其执行的函数(如v = f(d))。
  </summary>

  Conceptually, the type is the function (as in v = f(d)) whose execution is being tracked by the stack frame.
</details>

<details>
  <summary>
    随着类型，关键是在调和期间使用，以确定fiber是否可以重复使用。
  </summary>

  Along with the type, the key is used during reconciliation to determine whether the fiber can be reused.
</details>

<details>
  <summary>
  <b>child</b> and <b>sibling</b>
  </summary>

  child and sibling
</details>

<details>
  <summary>
    这些字段指向其他fiber，描述fiber的递归树结构。
  </summary>

  These fields point to other fibers, describing the recursive tree structure of a fiber.
</details>

<details>
  <summary>
    子fiber对应于组件的render方法返回的值。在下面的例子中
  </summary>

  The child fiber corresponds to the value returned by a component's render method. So in the following example
</details>

```javascript 
function Parent() {
  return <Child />
}
```


<details>
  <summary>
    父fiber的子fiber对应子fiber。
  </summary>

  The child fiber of Parent corresponds to Child.
</details>

<details>
  <summary>
    sibling字段用于渲染返回多个子元素的情况(这是Fiber的一个新特性):
  </summary>

  The sibling field accounts for the case where render returns multiple children (a new feature in Fiber!):
</details>

```javascript 
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

<details>
  <summary>
    子fibers形成一个单链表，其头是第一个子fibers。在这个例子中，Parent的子结点是Child1和Child1的兄弟结点Child2。
  </summary>

  The child fibers form a singly-linked list whose head is the first child. So in this example, the child of Parent is Child1 and the sibling of Child1 is Child2.
</details>

<details>
  <summary>
    回到我们的函数类比，你可以把子fiber看作尾部的函数。
  </summary>
  
  Going back to our function analogy, you can think of a child fiber as a tail-called function.
</details>

#### `return`

<details>
  <summary>
    返回fiber是程序在处理当前fiber后应该返回的fiber。它在概念上与堆栈帧的返回地址相同。它也可以被认为是母fiber
  </summary>

  The return fiber is the fiber to which the program should return after processing the current one. It is conceptually the same as the return address of a stack frame. It can also be thought of as the parent fiber.
</details>

<details>
  <summary>
    如果一个fiber有多个子fiber，每个子fiber的返回fiber是父fiber(又有子fiber)。因此，在上一节的示例中，child1和child2返回的fiber是父fiber。
  </summary>

  If a fiber has multiple child fibers, each child fiber's return fiber is the parent. So in our example in the previous section, the return fiber of Child1 and Child2 is Parent.
</details>

#### `pendingProps` and `memoizedProps`

<details>
  <summary>
    从概念上讲，props是函数的参数。fiber的pendingProps在执行的开始设置，memoizedProps在执行的结束设置。
  </summary>

  Conceptually, props are the arguments of a function. A fiber's pendingProps are set at the beginning of its execution, and memoizedProps are set at the end.
</details>

<details>
  <summary>
    当传入的pendingProps等于memoizedProps时，它表示可以重用fiber以前的输出，从而避免不必要的工作。
  </summary>

  When the incoming pendingProps are equal to memoizedProps, it signals that the fiber's previous output can be reused, preventing unnecessary work.
</details>

#### `pendingWorkPriority`

<details>
  <summary>
    一个代表fiber在一个work中优先级的数字。ReactPriorityLevel模块列出了不同的优先级级别以及它们所代表的内容。
  </summary>

  A number indicating the priority of the work represented by the fiber. The ReactPriorityLevel module lists the different priority levels and what they represent.
</details>

<details>
  <summary>
    除了NoWork为0之外，较大的数字表示较低的优先级。例如，您可以使用以下函数来检查fiber的优先级是否至少达到给定的优先级
  </summary>
  
  With the exception of NoWork, which is 0, a larger number indicates a lower priority. For example, you could use the following function to check if a fiber's priority is at least as high as the given level:
</details>

```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```
_This function is for illustration only; it's not actually part of the React Fiber codebase._

<details>
  <summary>
    调度器使用优先级字段搜索要执行的下一个工作单元。这个算法将在以后的章节中讨论。
  </summary>

  The scheduler uses the priority field to search for the next unit of work to perform. This algorithm will be discussed in a future section.
</details>

#### `alternate`

flush
To flush a fiber is to render its output onto the screen.

<details>
  <summary>
    flush <br/>
    To flush a fiber就是把它的输出输出到屏幕上。
  </summary>
  
  flush <br/>
  To flush a fiber is to render its output onto the screen.
</details>

#### `work-in-progress`

<details>
  <summary>
    一个尚未完成的fiber; 从概念上说，还没有返回的堆栈帧(virtual stack)。
  </summary>
  
  A fiber that has not yet completed; conceptually, a stack frame which has not yet returned.
</details>

<details>
  <summary>
    在任何时候，一个组件实例最多有两个对应的fibers:当前已刷新的fibers和正在进行的fibers。
  </summary>

  At any time, a component instance has at most two fibers that correspond to it: the current, flushed fiber, and the work-in-progress fiber.
</details>

<details>
  <summary>
    当前fiber的替换是正在进行的工作，正在进行的工作的替换是当前fiber。
  </summary>

  The alternate of the current fiber is the work-in-progress, and the alternate of the work-in-progress is the current fiber.
</details>

<details>
  <summary>
    fiber的替代品是通过一个名为`cloneFiber`的函数惰性地创建的。与总是创建新对象不同，`cloneFiber`将尝试重用fiber的备用对象(如果存在的话)，从而最小化分配。
  </summary>
  
  A fiber's alternate is created lazily using a function called `cloneFiber`. Rather than always creating a new object, `cloneFiber` will attempt to reuse the fiber's alternate if it exists, minimizing allocations.
</details>

<details>
  <summary>
    您应该把备用字段看作实现细节，但是它在代码库中经常出现，因此在这里讨论它是很有价值的。
  </summary>

  You should think of the alternate field as an implementation detail, but it pops up often enough in the codebase that it's valuable to discuss it here.
</details>

#### `output`

_host component_
<details>
  <summary>
    响应应用程序的叶节点。它们是特定于呈现环境的(例如，在浏览器应用程序中，它们是' div '、' span '等)。在JSX中，它们使用小写标记名称表示。
  </summary>
  
  The leaf nodes of a React application. They are specific to the rendering environment (e.g., in a browser app, they are `div`, `span`, etc.). In JSX, they are denoted using lowercase tag names.
</details>

<details>
  <summary>
    从概念上讲，fiber的输出是函数的返回值。
  </summary>

  Conceptually, the output of a fiber is the return value of a function.
</details>

<details>
  <summary>
    每个fiber最终都有输出，但输出仅由主组件在叶节点上创建。然后输出被向上传输到树中。
  </summary>

  Every fiber eventually has output, but output is created only at the leaf nodes by host components. The output is then transferred up the tree.
</details>

<details>
  <summary>
    输出是最终提供给渲染器的内容，以便它能够刷新对渲染环境的更改。渲染器负责定义如何创建和更新输出。
  </summary>

  The output is what is eventually given to the renderer so that it can flush the changes to the rendering environment. It's the renderer's responsibility to define how the output is created and updated.
</details>

## Future sections

<details>
  <summary>
    目前就这些了，但是这个文档还远没有完成。以后的部分将描述更新生命周期中使用的算法。涉及的主题包括:
    <ul>
      <li>调度器如何找到下一个要执行的工作单元。</li>
      <li>如何通过fiber树跟踪和传播优先级。</li>
      <li>调度器如何知道何时暂停和恢复工作。</li>
      <li>如何刷新工作并将其标记为完成。</li>
      <li>副作用(如生命周期方法)是如何工作的。</li>
      <li>协同程序是什么以及如何使用它来实现诸如上下文和布局之类的特性</li>
    </ul>
  </summary>

  That's all there is for now, but this document is nowhere near complete. Future sections will describe the algorithms used throughout the lifecycle of an update. Topics to cover include:

- how the scheduler finds the next unit of work to perform.
- how priority is tracked and propagated through the fiber tree.
- how the scheduler knows when to pause and resume work.
- how work is flushed and marked as complete.
- how side-effects (such as lifecycle methods) work.
- what a coroutine is and how it can be used to implement features like context and layout.
</details>



