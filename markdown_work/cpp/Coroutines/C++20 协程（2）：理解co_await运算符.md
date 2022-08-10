# C++20 协程（2）：理解co_await运算符

在上一篇文章，我描述了函数和协程的高层差异，但没有涉及任何[C++协程TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf)里的语法和语义。

C++协程TS添加的关键设施是暂停协程的能力，允许协程被恢复执行。实现该功能的设施是`co_await`运算符。

理解`co_await`的工作方式可以让我们更明白协程的行为，以及协程如何被暂停和恢复。在这篇文章中，我将解释`co_await`运算符的机制并引入**Awaitable**和**Awaiter**概念（concepts）。

在开始之前，我想先给出一个协程TS的概述，以提供一些基础知识。

## 协程TS给了我们什么？

- 3个新的关键字：`co_await`、`co_yield`和`co_return`
- 几个新的类型（在`std::experimental`命名空间中）：
  - `coroutine_handle<P>`
  - `coroutine_traits<Ts...>`
  - `suspend_always`
  - `suspend_never`
- 一个通用的机制，库的开发者可以用它和协程交互，并定制他们的行为。
- 一个语言设施，它使得编写异步代码更简单。

C++协程TS提供的设施可以看作是一个用于协程的*低级汇编语言*。这些设施很难以安全的方式直接使用，它更倾向于给库的开发者，让他们可以编写出应用程序开发者可以安全使用的高级抽象。

## 编译器 <-> 库 的交互

有趣的的，协程TS没有定义协程的语义。它没有定义如何产生返回给调用者的值。它没有定义传递给`co_return`语句的返回值要做什么，以及如何处理传递出协程的异常。它没有定义协程应该在哪个线程上恢复。

作为替换，它为库代码规范了一个通用的机制，通过实现符合特定接口的类型，以定制化协程的执行。编译器因此能生成代码，这些代码调用库提供的类的实例方法。该方式类似于定制化范围for循环的行为（即定义`begin()`/`end()`方法和`iterator`类型）。

协程TS没有规定任何语义这一事实，让协程成为了一个强大的工具。它允许库的开发者定义不同类型的协程，出于各种目的。

例如，你可以定义一个异步的产生单个值的协程；或者定义一个惰性产生值序列的协程；或者定义一个协程，它能简化获取`optional<T>`值的控制流（如果遇到`nullopt`则提前退出）。

协程TS定义了两种类型的接口：**Promise**接口和**Awaitable**接口。

**Promise**接口规定了一些和协程自身行为相关的方法。库的开发者可以定制：当协程被调用时的行为，当协程返回时的行为（正常返回或未处理的异常），以及定制协程内任何`co_await`和`co_yield`表达式的行为。

**Awaitable**接口规定了一些控制`co_await`表达式语义的方法。当一个值被`co_await`时，这部分代码将被翻译为一系列awaitable对象的方法，这使得可以规定：是否要暂停当前协程，在暂停协程后是否要执行一些逻辑，在协程恢复执行后是否要产生`co_await`表达式的结果。

我将在未来的文章中覆盖**Promise**接口的细节，但是现在让我们先看一下**Awaitable**接口。

## Awaiters和Awaitables：解释 `operator co_await`

`co_await`运算符是一个新的单目运算符，它可以应用到一个值。例如：`co_await someValue`。

`co_await`运算符只能被用在协程的上下文内。这像是个废话，因为按照定义，任何包含`co_await`运算符的函数都将视为协程编译。

一个支持`co_await`运算符的类型被称为**Awaitable**类型。

注意，`co_await`运算符能否应用于一个类型，取决于`co_await`表达式出现的上下文。一个协程的promise类型可以修改协程内`co_await`表达式的含义，通过promise类型的`await_transform`方法（将在之后描述）。

一个**Awaiter**类型是一个实现了3个特殊方法的类型，这3个方法作为`co_await`表达式的一部分被调用：`await_ready`、`await_suspend`和`await_resume`。

注意，我无耻的借用了来自C#中的术语‘Awaiter’，查看[这篇文章](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern)以获取更多关于C# awaiters的细节。

注意，一个类型可以同时是**Awaitable**和**Awaiter**类型。

当看见`co_await <expr>`表达式时，编译器实际上有很多不同的方式去翻译它，取决于涉及到的类型。

### 获得Awaiter

编译器做的第一件事是生成代码，以获得用于await值的Awaiter对象。获取一个awaiter对象有很多方法，这些方法罗列在 N4680 section 5.3.8(3).

让我们假设promise对象具有类型`P`，并且该`promise`是一个左指引用。

如果该promise类型`P`有一个名为`await_transform`的成员，那么`<expr>`首先会被传递到`promise.await_transform(<expr>)`以获取**Awaitable**值：`awaitable`。

然后，如果该**Awaitable**对象，`awaitable`，有一个合适的`operator co_await()`重载，那么它将被调用以获得`Awaiter`对象。否则，`awaitable`对象本身被用作awaiter对象。

如果我们将该规则编码成函数`get_awaitable()`和`get_awaiter()`，那么他们看上去像这样：

~~~cpp
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
~~~

### 等待（Awaiting）Awaiter

假定我们将转化`<expr>`结果到**Awaitable**对象的逻辑封装成上述函数，那么`co_await <expr>`的语义可以大致转换成：

~~~cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::experimental::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(p));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
~~~

`await_suspend()`的`void`返回类型的版本将无条件转移执行权回协程的调用者/恢复者，而`bool`返回类型的版本允许awaiter对象条件地立即恢复协程执行而不返回给调用者/恢复者。

当awaiter启动一个异步操作，而该异步操作有时能同步完成时，`await_suspend()`的`bool`返回类型的版本会很有用。在能同步完成的情况下，`await_suspend()`方法可以返回`false`来表明协程应该被立即恢复并继续执行。

在`<suspend-coroutine>`处，编译器会生成一些代码，这些代码保存协程的状态以在之后恢复执行。这会存储`<resume-point>`的位置和寄存器的值到协程帧。

在`<suspend-coroutine>`操作完成后，协程进入暂停状态。协程进入暂停状态后，第一个能观测到的点是在`await_suspend()`内。

`await_suspend()`方法负责调度协程，以便在未来恢复/摧毁协程。注意，从`await_suspend()`返回`false`被视为立即在当前线程上恢复协程的执行。

`await_ready()`方法的目的是允许你避免`<suspend-coroutine>`操作的开销，因为在某些情况下，某个操作将同步地（直接）完成而无需暂停协程。

在`<return-to-caller-or-resumer>`处，执行权被转移回调用者或恢复者，探出栈帧但保持协程帧。

当（或者如果）协程被恢复，执行权将在`<resume-point>`处恢复。即，在`await_resume()`方法之前，`await_resume()`方法用来获取操作的结果。

`await_resume()`方法的返回值是`co_await`表达式的结果。`await_resume()`方法也可以抛出一个异常，此时异常会传递出`co_await`表达式。

注意，如果一个异常传递出了`await_suspend()`，那么协程会被自动恢复（无需调用`await_resume()`），并且该异常会传递出`co_await`表达式。

## 协程句柄

可能你已经注意到了`coroutine_handle<P>`类型。

该类型表示一个非占有（无所有权）的协程帧句柄，可用于恢复协程执行或摧毁协程帧。它也可以用于访问协程的promise对象。

`coroutine_handle`类型有如下接口（已简化）：

~~~cpp
namespace std::experimental
{
  template<typename Promise>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void>
  {
    bool done() const;

    void resume();
    void destroy();

    void* address() const;
    static coroutine_handle from_address(void* address);
  };

  template<typename Promise>
  struct coroutine_handle : coroutine_handle<void>
  {
    Promise& promise() const;
    static coroutine_handle from_promise(Promise& promise);

    static coroutine_handle from_address(void* address);
  };
}
~~~

当实现**Awaitable**类型时，你将使用的关键方法是`.resume()`，它应该在操作完成或想恢复协程执行时调用。调用`.resume()`方法将在`<resume-point>`处重新激活协程。当协程下一次抵达`<return-to-caller-or-resumer>`处时，`.resume()`返回。

`.destroy()`方法摧毁协程帧，调用所有作用域内变量的析构函数，并释放协程帧使用的内存。你通常不需要（甚至应该避免）调用`.destroy()`，除非你是库的开发者。通常，协程帧被一些RAII类型拥有，该类型在调用协程时返回。因此调用`.destroy()`而不管RAII对象可能导致双析构（double-destruction）问题。

`.promise()`方法返回协程promise对象的引用。然而，和`.destroy()`一样，这通常用于库的开发者。你应该将协程的promise对象视为协程的内部实现细节。对于大多数**Normally Awaitable**类型，你应该用`coroutine_handle<void>`代替`coroutine_handle<Promise>`，作为`await_suspend()`方法的参数类型。

`coroutine_handle<P>::from_promise(P& promise)`函数从协程的promise对象构造一个协程具柄。注意，你必须确保类型`P`完全和协程帧使用的primise类型相同；当协程帧的promise类型是`Derived`时，尝试构造一个`coroutine_handle<Base>`将产生未定义行为。

`.address()` / `from_address()`函数转换一个协程具柄到/从一个`void*`指针。它主要用于传递一个‘上下文’参数到C风格的API，所以你可能发现它在某些情况下会有用。然而，在大多数情况下，我发现传递附加信息到回调函数的上下文参数上很有必要，所以通常存储`coroutine_handle`在一个结构体内，然后传递一个指向该结构体的指针到上下文参数中，而不是直接使用`.address()`的返回值。

## 编写无需同步机制的异步代码

`co_await`运算符的一个强大特性是，我们可以在协程被暂停之后，恢复之前，执行一些代码。

这使得Awaiter对象可以在协程暂停后发起一个异步操作，同时传递该协程的`coroutine_handle`到该操作，当操作完成时（可能在其他线程），可以安全的恢复协程而无需任何额外的同步机制。

例如，当协程已经暂停后，可以在`await_suspend()`内启动一个异步读操作，当该操作完成时，由于协程已经暂停，所以我们可以恢复协程而无需通过任何同步机制，来协调启动该操作的线程和完成该操作的线程。

~~~cpp
Time     Thread 1                           Thread 2
  |      --------                           --------
  |      ....                               Call OS - Wait for I/O event
  |      Call await_ready()                    |
  |      <supend-point>                        |
  |      Call await_suspend(handle)            |
  |        Store handle in operation           |
  V        Start AsyncFileRead ---+            V
                                  +----->   <AsyncFileRead Completion Event>
                                            Load coroutine_handle from operation
                                            Call handle.resume()
                                              <resume-point>
                                              Call to await_resume()
                                              execution continues....
           Call to AsyncFileRead returns
         Call to await_suspend() returns
         <return-to-caller/resumer>
~~~

当利用该方式的优势时，一个需要*非常小心*的事情是，只要你将协程具柄传递给其他线程，那么另一个线程可能在`await_suspend()`返回之前恢复协程（在另一个线程上），因此可能与`await_suspend()`方法中剩下的代码并发执行。

当协程恢复执行时，它要做的第一件事情是，调用`await_resume()`获得结果，然后通常会立即销毁**Awaiter**对象（即，`await_suspend()`的`this`指针）。在`await_suspend()`返回之前，协程会潜在的运行、销毁awaiter和promise对象。

所以在`await_suspend()`方法内，一旦协程可能在其他线程中并发恢复，你需要避免访问`this`指针或协程的`.primise()`对象，因为他们可能已经被销毁了。

### 对比有栈协程（Stackful Coroutines）

协程TS提供的无栈协程能够在协程暂停后执行一段代码，我想用这种能力和已有的有栈协程（例如Win32 fibers 和 boost::context）做一个快速的对比。

对于许多有栈协程框架，协程的暂停操作与另一个协程的恢复结合形成了‘上下文切换’操作，这导致在暂停协程之后，转移执行权到另一个协程之前，通常没有机会去执行额外逻辑。

这意味着，如果我们想在有栈协程顶部实现一个类似的异步文件读操作，我们必须在暂停该协程*之前*启动该异步操作。但是该操作可能在协程被暂停之前完成（在另一个线程上）。该操作在另一个线程上完成，和协程暂停之间存在潜在的竞争，因此需要一些线程同步机制去决定最后的赢家。

要解决这个问题，可以使用一个trampoline上下文（trampoline context），它能在初始上下文被暂停后，代表初始上下文启动该操作。然后这需要额外的基础设施和额外的上下文切换，同时它引入的额外开销会大于直接实用同步机制的开销。

## 避免内存分配

异步操作通常需要存储每个操作的状态，这些状态跟踪该操作的进展。这些状态通常需要在操作执行期间存在，并且不能在该操作完成前释放。

例如，调用异步Win32 I/O函数需要分配和传递一个`OVERLAPPED`结构体。调用者确保该指针在操作完成前持续有效。

对于传统的基于回调函数的API来说，这些状态通常需要在堆上分配，以确保它们具有合适的生命周期。如果你执行了许多操作，你可能需要为每个操作分配和释放该状态。如果有性能问题的话，那么一个自定义的allocator（例如从池上分配这些状态）会很有用。

然而，当我们使用协程时，我们可以避免在堆上分配这些状态，因为协程帧内的局部变量将在协程暂停期间保持有效。

为了在**Awaiter**对象中实现每个操作的状态，我们可以高效的从协程帧上“借用”内存来存储这些状态（在`co_await`表达式执行期间）。一旦操作完成，协程恢复执行、**Awaiter**对象被摧毁，同时释放协程帧内的内存给其他局部变量使用。

本质上，协程帧可能仍然在堆上分配。然而，一旦分配，一个协程帧可以用来执行许多次异步操作，而这仅需要一次堆分配。

仔细想想，协程帧像是一种高性能的内存allocator。编译器可以在编译时计算出需要的总内存大小，而没有额外开销！试试用一个自定义的allocator来打败它吧🤓

## 一个例子：实现一个单线程同步原语

现在我们已经讨论了大量`co_await`运算符的机制，我想展示如果用这些知识实现一个基本的awaitable同步原语：一个异步manual-reset事件。

该事件的需求时：它需要被多个并发执行的协程等待，当这些写成因为等待而暂停执行时，直到某个线程调用`.set()`方法，所有等待着的协程都将被恢复。如果某个线程调用了`.set()`，那么协程应该继续执行而无需暂停。

理想地，我们应该使它`noexcept`，无需堆分配，无需锁。

它的使用方式看上去像这样：

~~~cpp
T value;
async_manual_reset_event event;

// A single call to produce a value
void producer()
{
  value = some_long_running_computation();

  // Publish the value by setting the event.
  event.set();
}

// Supports multiple concurrent consumers
task<> consumer()
{
  // Wait until the event is signalled by call to event.set()
  // in the producer() function.
  co_await event;

  // Now it's safe to consume 'value'
  // This is guaranteed to 'happen after' assignment to 'value'
  std::cout << value << std::endl;
}
~~~

该事件可能的状态有：被‘设置’和‘不被设置’（'not set' and 'set'）。

当它处于'not set'状态时，有一个等待着的协程的链表（可能为空），这些协程等待状态变为`set`。

当它处于'set'状态时，不存在任何等待着的协程，同时`co_await`该事件的协程会继续执行而不被暂停。

该状态实际上可以表示为一个`std::atomic<void*>`。

- 一个特殊的指针值用于'set'状态。此时我们将使用该事件的`this`指针，因为它不会和链表中的元素相同（即，不同的对象地址值不同）。
- 否则该事件处于'not set'状态。此时的值是一个等待着的协程的链表的表头指针。

我们可以在避免在堆上分配链表的节点，而是直接将节点存储在一个‘awaiter’对象内。

事件类的接口看上去像是这样：

~~~cpp
class async_manual_reset_event
{
public:

  async_manual_reset_event(bool initiallySet = false) noexcept;

  // No copying/moving
  async_manual_reset_event(const async_manual_reset_event&) = delete;
  async_manual_reset_event(async_manual_reset_event&&) = delete;
  async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
  async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

  bool is_set() const noexcept;

  struct awaiter;
  awaiter operator co_await() const noexcept;

  void set() noexcept;
  void reset() noexcept;

private:

  friend struct awaiter;

  // - 'this' => set state
  // - otherwise => not set, head of linked list of awaiter*.
  mutable std::atomic<void*> m_state;

};
~~~

这是一个相当直接和简单的接口。需要注意的是，`operator co_await()`方法返回了一个未定义的类型`awaiter`。

现在让我们定义`awaiter`类型。

### 定义Awaiter

由于`async_manual_reset_event`对象将被等待（awaiting），所以需要一个该事件的引用并通过构造函数初始化。

它也需要作为链表的节点，所以它需要维护一个指向下一个`awaiter`对象的指针。

它还需要存储协程的`coroutine_handle`，以便在事件变成‘set’时恢复协程的执行。我们不关心协程的promise类型，我们只需要一个`coroutine_handle<>`（`coroutine_handle<void>`的简写）。

最后，他需要实现**Awaiter**接口，因此它需要3个特殊的方法：`await_ready`、`await_suspend`和`await_resume`。因为我们不需要从`co_await`表达式返回值，所以`await_resume`会返回`void`。

现在`awaiter`类的借口看上去像这样：

~~~cpp
struct async_manual_reset_event::awaiter
{
  awaiter(const async_manual_reset_event& event) noexcept
  : m_event(event)
  {}

  bool await_ready() const noexcept;
  bool await_suspend(std::experimental::coroutine_handle<> awaitingCoroutine) noexcept;
  void await_resume() noexcept {}

private:

  const async_manual_reset_event& m_event;
  std::experimental::coroutine_handle<> m_awaitingCoroutine;
  awaiter* m_next;
};
~~~

当`co_await`一个事件时，如果事件已经是`set`状态，则无需暂停协程。因此当事件处于`set`状态时，`await_ready()`返回`true`。

~~~cpp
bool async_manual_reset_event::awaiter::await_ready() const noexcept
{
  return m_event.is_set();
}
~~~

接下来让我们看一下`await_suspend()方法。这通常是一个让awaitable类型变得神秘的地方。

首先我们我们需要将协程具柄暂存在`m_awaitingCoroutine`成员中，以便该事件可以在之后调用`.resume()`恢复它。

然后我们需要将awaiter原子的插入链表。如果成功插入，则返回`true`表示我们不希望立即恢复协程，否则如果我们发现该事件被并发的更改为了`set`状态，则返回`false`表示协程应该被立即恢复。

~~~cpp
bool async_manual_reset_event::awaiter::await_suspend(
  std::experimental::coroutine_handle<> awaitingCoroutine) noexcept
{
  // Special m_state value that indicates the event is in the 'set' state.
  const void* const setState = &m_event;

  // Remember the handle of the awaiting coroutine.
  m_awaitingCoroutine = awaitingCoroutine;

  // Try to atomically push this awaiter onto the front of the list.
  void* oldValue = m_event.m_state.load(std::memory_order_acquire);
  do
  {
    // Resume immediately if already in 'set' state.
    if (oldValue == setState) return false; 

    // Update linked list to point at current head.
    m_next = static_cast<awaiter*>(oldValue);

    // Finally, try to swap the old list head, inserting this awaiter
    // as the new list head.
  } while (!m_event.m_state.compare_exchange_weak(
             oldValue,
             this,
             std::memory_order_release,
             std::memory_order_acquire));

  // Successfully enqueued. Remain suspended.
  return true;
}
~~~

注意，当加载旧状态时我们使用了‘acquire’内存序，以便当我们读取到特殊的‘set’值时，写可见性发生在调用‘set()’之前。

当compare-exchange成功时，我们需要‘release语义’，以便后续对‘set()’的调用将看到我们对m_awaitingCoroutine的写入以及之前对协程状态的写入。

### 补充剩余代码

现在我们已经定义了`awaiter`类型，让我们继续实现`async_manual_reset_event`的方法。

首先是构造函数。它需要初始成`not set`或`set`状态。

~~~cpp
async_manual_reset_event::async_manual_reset_event(
  bool initiallySet) noexcept
: m_state(initiallySet ? this : nullptr)
{}
~~~

`is_set()`方法非常直接：

~~~cpp
bool async_manual_reset_event::is_set() const noexcept
{
  return m_state.load(std::memory_order_acquire) == this;
}
~~~

下面是`reset()`方法。如果处于‘set’状态，我们将它转移为‘not set’状态，否则保持不变。

~~~cpp
void async_manual_reset_event::reset() noexcept
{
  void* oldValue = this;
  m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
}
~~~

对于`set()`方法，我们通过用`this`与当前状态交换，将该事件转移到‘set’状态，然后检查旧值。如果存在等待着的协程，那么在该方法返回前依次恢复这些协程。

~~~cpp
void async_manual_reset_event::set() noexcept
{
  // Needs to be 'release' so that subsequent 'co_await' has
  // visibility of our prior writes.
  // Needs to be 'acquire' so that we have visibility of prior
  // writes by awaiting coroutines.
  void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
  if (oldValue != this)
  {
    // Wasn't already in 'set' state.
    // Treat old value as head of a linked-list of waiters
    // which we have now acquired and need to resume.
    auto* waiters = static_cast<awaiter*>(oldValue);
    while (waiters != nullptr)
    {
      // Read m_next before resuming the coroutine as resuming
      // the coroutine will likely destroy the awaiter object.
      auto* next = waiters->m_next;
      waiters->m_awaitingCoroutine.resume();
      waiters = next;
    }
  }
}
~~~

最后，我们需要实现`operator co_await()`方法。

~~~cpp
async_manual_reset_event::awaiter
async_manual_reset_event::operator co_await() const noexcept
{
  return awaiter{ *this };
}
~~~

一个awaitable的异步manual-reset事件是无锁的、无需内存分配的和`noexcept`的。

如果你想获取这些代码或想知道MSVC和Clang是如何编译它们的，请查看[godbolt](https://godbolt.org/g/Ad47tH)

你可以在[cppcoro](https://github.com/lewissbaker/cppcoro)中找到这个类的实现，以及大量有用的awaitable类型例如（`async_mutex`和`async_auto_reset_event`）。

## 结语

这边文章解释了`operator co_await`是如何实现的，并定义了**Awaitable**和**Awaiter**概念。

这篇文章还讲解了如何实现一个可等待的异步线程同步原语，它避免了堆分配。

我希望这篇文章能帮你揭开`co_await`运算符的神秘面纱。

在下一篇文章中，我将讨论**Promise**概念，以及一个协程类型（coroutine-type）的作者应该如何自定义协程的行为。

## 参考

- 翻译自 <https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await>
