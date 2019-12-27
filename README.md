# [Writing High-Performance Swift Code](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst)

- [Enabling Optimizations](#enabling-optimizations)
- [Whole Module Optimizations](#whole-module-optimizations)
- [Reducing Dynamic Dispatch](#reducing-dynamic-dispatch)
	- [Dynamic Dispatch](#dynamic-dispatch)
	- Advice: Use 'final' when you know the declaration does not need to be overridden
	- Advice: Use 'private' and 'fileprivate' when declaration does not need to be accessed outside of file
- [Using Container Types Efficiently](#using-container-types-efficiently)
	- Advice: Use value types in Array
	- Advice: Use ContiguousArray with reference types when NSArray bridging is unnecessary
	- Advice: Use inplace mutation instead of object-reassignment
- [Unchecked operations](#unchecked-operations)
	- Advice: Use unchecked integer arithmetic when you can prove that overflow cannot occur
- [Generics](#generics)
	- Advice: Put generic declarations in the same module where they are used
- [The cost of large Swift values](#the-cost-of-large-swift-values)
	- Advice: Use copy-on-write semantics for large values
- [Unsafe code](#unsafe-code)
	- Advice: Use unmanaged references to avoid reference counting overhead
- [Protocols](#protocols)
	- Advice: Mark protocols that are only satisfied by classes as class-protocols
- [Unsupported Optimization Attributes](#unsupported-optimization-attributes)
- [Footnotes](#footnotes)

> The following document is a gathering of various tips and tricks for writing high-performance Swift code. The intended audience of this document is compiler and standard library developers.

下面这篇文档收集了一系列编写高性能 Swift 代码的要诀和技巧。文档的目标读者是编译器和标准库开发人员。

> Some of the tips in this document can help improve the quality of your Swift program and make your code less error prone and more readable. Explicitly marking final-classes and class-protocols are two obvious examples. However some of the tips described in this document are unprincipled, twisted and come to solve a specific temporary limitation of the compiler or the language. Many of the recommendations in this document come with trade offs for things like program runtime, binary size, code readability, etc.

文档中的一些技巧可以帮助提升您的 Swift 程序质量，使您的代码不容易出错且可读性更好。显式地标记最终类和类协议是两个显而易见的例子。 然而文档中还有一些技巧是不符合规矩的，扭曲的，仅仅解决一些比编译器或语言的特殊的临时性需求。文档中的很多建议来自于多方面的权衡，例如：运行时、字节大小、代码可读性等等。

## Enabling Optimizations
## 启用优化

> The first thing one should always do is to enable optimization. Swift provides four different optimization levels:

第一个应该做的事情就是启用优化。Swift 提供了三种不同的优化级别：

> - -Onone: This is meant for normal development. It performs minimal optimizations and preserves all debug info.
> - -O: This is meant for most production code. The compiler performs aggressive optimizations that can drastically change the type and amount of emitted code. Debug information will be emitted but will be lossy.
> - -Ounchecked: This is a special optimization mode meant for specific libraries or applications where one is willing to trade safety for performance. The compiler will remove all overflow checks as well as some implicit type checks. This is not intended to be used in general since it may result in undetected memory safety issues and integer overflows. Only use this if you have carefully reviewed that your code is safe with respect to integer overflow and type casts.
> - -Osize: This is a special optimization mode where the compiler prioritizes code size over performance.

- -Onone: 这意味着正常的开发。它执行最小优化和保存所有调试信息。
- -O: 这意味着对于大多数生产代码。编译器执行积极地优化，可以大大改变提交代码的类型和数量。调试信息将被省略但还是会有损害的。
- -Ounchecked: 这是一个特殊的优化模式，它意味着特定的库或应用程序，这是以安全性来交换的。编译器将删除所有溢出检查以及一些隐式类型检查。这不是在通常情况下使用的，因为它可能会导致内存安全问题和整数溢出。如果你仔细审查你的代码，那么对整数溢出和类型转换来说是安全的。
- -Osize: 这是一个特殊的优化模式，牺牲性能以换取更小的代码大小。 

> In the Xcode UI, one can modify the current optimization level as follows:

在 Xcode UI 中，可以修改的当前优化级别如下： 

...

## Whole Module Optimizations
## 整个组件优化

> By default Swift compiles each file individually. This allows Xcode to compile multiple files in parallel very quickly. However, compiling each file separately prevents certain compiler optimizations. Swift can also compile the entire program as if it were one file and optimize the program as if it were a single compilation unit. This mode is enabled using the swiftc command line flag -whole-module-optimization. Programs that are compiled in this mode will most likely take longer to compile, but may run faster.

默认情况下 Swift 单独编译每个文件。这使得 Xcode 可以非常快速的并行编译多个文件。然而，分开编译每个文件可以预防某些编译器优化。Swift 也可以犹如它是一个文件一样编译整个程序，犹如就好像它是一个单一的编译单元一样优化这个程序。这个模式可以使用命令行 flag-whole-module-optimization 来激活。在这种模式下编译的程序将最最有可能需要更长时间来编译，单可以运行得更快。

> This mode can be enabled using the Xcode build setting 'Whole Module Optimization'.

这个模式可以通过 XCode 构建设置中的“Whole Module Optimization”来激活。

## Reducing Dynamic Dispatch
## 降低动态调度

> Swift by default is a very dynamic language like Objective-C. Unlike Objective-C, Swift gives the programmer the ability to improve runtime performance when necessary by removing or reducing this dynamism. This section goes through several examples of language constructs that can be used to perform such an operation.

Swift 在默认情况下是一个类似 Objective-C 的非常动态的语言。与 Objective-C 不同的是，Swift 给了程序员通过消除和减少这种特性来提供运行时性能的能力。本节提供几个可被用于这样的操作的语言结构的例子。

### Dynamic Dispatch
### 动态调度

> Classes use dynamic dispatch for methods and property accesses by default. Thus in the following code snippet, a.aProperty, a.doSomething() and a.doSomethingElse() will all be invoked via dynamic dispatch:

类使用动态调度的方法和默认的属性访问。因此在下面的代码片段中，a.aProperty、a.doSomething() 和 a.doSomethingElse() 都将通过动态调度来调用：

````swift
class A {
  var aProperty: [Int]
  func doSomething() { ... }
  dynamic doSomethingElse() { ... }
}

class B: A {
  override var aProperty {
    get { ... }
    set { ... }
  }

  override func doSomething() { ... }
}

func usingAnA(_ a: A) {
  a.doSomething()
  a.aProperty = ...
}
````

> In Swift, dynamic dispatch defaults to indirect invocation through a vtable [1]. If one attaches the dynamic keyword to the declaration, Swift will emit calls via Objective-C message send instead. In both cases this is slower than a direct function call because it prevents many compiler optimizations [2] in addition to the overhead of performing the indirect call itself. In performance critical code, one often will want to restrict this dynamic behavior.

在 Swift 中，动态调度默认通过一个 vtable[1]（虚函数表）间接调用。如果使用一个 dynamic 关键字来声明，Swift 将会通过 Objective-C 消息转发来调用。这两种情况中，这种情况会比直接的函数调用较慢，因为它防止了对间接呼叫本身之外程序开销的许多编译器优化[2]。在性能关键的代码中，人们常常会想限制这种动态行为。

### Advice: Use 'final' when you know the declaration does not need to be overridden
### 建议：当你知道声明不需要被重写时使用 final

> The final keyword is a restriction on a declaration of a class, a method, or a property such that the declaration cannot be overridden. This implies that the compiler can emit direct function calls instead of indirect calls. For instance in the following C.array1 and D.array1 will be accessed directly [3]. In contrast, D.array2 will be called via a vtable:

final 关键字是一个类、一个方法、或一个属性声明中的一个限制，使得这样的声明不得被重写。这意味着编译器可以呼叫直接的函数调用代替间接调用。例如下面的 C.array1 和 D.array1 将会被直接[3]访问。与之相反，D.array2 将通过一个虚函数表访问：

````swift
final class C {
  // No declarations in class 'C' can be overridden.
  var array1: [Int]
  func doSomething() { ... }
}

class D {
  final var array1: [Int] // 'array1' cannot be overridden by a computed property.
  var array2: [Int]      // 'array2' *can* be overridden by a computed property.
}

func usingC(_ c: C) {
   c.array1[i] = ... // Can directly access C.array without going through dynamic dispatch.
   c.doSomething() = ... // Can directly call C.doSomething without going through virtual dispatch.
}

func usingD(_ d: D) {
   d.array1[i] = ... // Can directly access D.array1 without going through dynamic dispatch.
   d.array2[i] = ... // Will access D.array2 through dynamic dispatch.
}
````

### Advice: Use 'private' and 'fileprivate' when declaration does not need to be accessed outside of file
### 建议：当声明的东西不需要被文件外部被访问到的时候，就用 private

> Applying the private or fileprivate keywords to a declaration restricts the visibility of the declaration to the file in which it is declared. This allows the compiler to be able to ascertain all other potentially overriding declarations. Thus the absence of any such declarations enables the compiler to infer the final keyword automatically and remove indirect calls for methods and field accesses accordingly. For instance in the following, e.doSomething() and f.myPrivateVar, will be able to be accessed directly assuming E, F do not have any overriding declarations in the same file:

将 private 关键词用在一个声明上，会限制对其进行了声明的文件的可见性。这会让编辑器有能力甄别出所有其它潜在的覆盖声明。如此，由于没有了任何这样的声明，使得编译器可以自动地推断出 final 关键词，并据此去掉对方面的间接调用和属性的访问。例如在如下的 e.doSomething()  和 f.myPrivateVar 中，就将可以被直接访问，假定在同一个文件中，E, F 并没有任何覆盖的声明：

````swift
private class E {
  func doSomething() { ... }
}

class F {
  fileprivate var myPrivateVar: Int
}

func usingE(_ e: E) {
  e.doSomething() // There is no sub class in the file that declares this class.
                  // The compiler can remove virtual calls to doSomething()
                  // and directly call E's doSomething method.
}

func usingF(_ f: F) -> Int {
  return f.myPrivateVar
}
````

## Using Container Types Efficiently
## 高效的使用容器类型

> An important feature provided by the Swift standard library are the generic containers Array and Dictionary. This section will explain how to use these types in a performant manner.

通用的容器 Array 和 Dictionary 是有 Swift 标准库提供的一个重要的功能特性。本节将介绍如何用一种高性能的方式使用这些类型。

### Advice: Use value types in Array
### 建议：在数组中使用值类型

> In Swift, types can be divided into two different categories: value types (structs, enums, tuples) and reference types (classes). A key distinction is that value types cannot be included inside an NSArray. Thus when using value types, the optimizer can remove most of the overhead in Array that is necessary to handle the possibility of the array being backed an NSArray.

在 Swift 中，类型可以分为不同的两类：值类型（结构体，枚举，元组）和引用类型（类）。一个关键的区分是 NSArray 不能含有值类型。因此当使用值类型时，优化器就不需要去处理对 NSArray 的支持，从而可以在数组上省去大部分消耗。

> Additionally, in contrast to reference types, value types only need reference counting if they contain, recursively, a reference type. By using value types without reference types, one can avoid additional retain, release traffic inside Array.

此外，相比引用类型，如果值类型递归地含有引用类型，那么值类型仅仅需要引用计数器。而如果使用没有引用类型的值类型，就可以避免额外的开销，从而释放数组内的流量。

````swift
// Don't use a class here.
struct PhonebookEntry {
  var name: String
  var number: [Int]
}

var a: [PhonebookEntry]
````
> Keep in mind that there is a trade-off between using large value types and using reference types. In certain cases, the overhead of copying and moving around large value types will outweigh the cost of removing the bridging and retain/release overhead.

记住要在使用大值类型和使用引用类型之间做好权衡。在某些情况下，拷贝和移动大值类型数据的消耗要大于移除桥接和持有/释放的消耗。

### Advice: Use ContiguousArray with reference types when NSArray bridging is unnecessary
### 建议：当 NSArray 桥接不必要时，使用 ContiguousArray 存储引用类型

> If you need an array of reference types and the array does not need to be bridged to NSArray, use ContiguousArray instead of Array:

如果你需要一个引用类型的数组，而且数组不需要桥接到 NSArray 时，使用 ContiguousArray 替代 Array：

````swift
class C { ... }
var a: ContiguousArray<C> = [C(...), C(...), ..., C(...)]
````

### Advice: Use inplace mutation instead of object-reassignment
### 建议：使用适当的改变而不是对象分配

> All standard library containers in Swift are value types that use COW (copy-on-write) [^4] to perform copies instead of explicit copies. In many cases this allows the compiler to elide unnecessary copies by retaining the container instead of performing a deep copy. This is done by only copying the underlying container if the reference count of the container is greater than 1 and the container is mutated. For instance in the following, no copying will occur when d is assigned to c, but when d undergoes structural mutation by appending 2, d will be copied and then 2 will be appended to d:

在 Swift 中所有的标准库容器都使用 COW(copy-on-write) 执行拷贝代替即时拷贝。在很多情况下，这可以让编译器通过持有容器而不是深度拷贝，从而省掉不必要的拷贝。如果容器的引用计数大于 1 并容器时被改变时，就会拷贝底层容器。例如：在下面这种情况：当 d 被分配给 c 时不拷贝，但是当 d 经历了结构性的改变追加 2，那么 d 将会被拷贝，然后 2 被追加到 b：

````swift
var c: [Int] = [ ... ]
var d = c        // No copy will occur here.
d.append(2)      // A copy *does* occur here.
````

> Sometimes COW can introduce additional unexpected copies if the user is not careful. An example of this is attempting to perform mutation via object-reassignment in functions. In Swift, all parameters are passed in at +1, i.e. the parameters are retained before a callsite, and then are released at the end of the callee. This means that if one writes a function like the following:

如果用户不小心时，有时 COW 会引起额外的拷贝。例如，在函数中，试图通过对象分配执行修改。在 Swift 中，所有的参数传递时都会被拷贝一份，例如，参数在调用点之前持有一份，然后在调用的函数结束时释放。也就是说，像下面这样的函数：

````swift
func append_one(_ a: [Int]) -> [Int] {
  a.append(1)
  return a
}

var a = [1, 2, 3]
a = append_one(a)
````

> a may be copied [5] despite the version of a without one appended to it has no uses after append_one due to the assignment. This can be avoided through the usage of inout parameters:

尽管由于分配，a 的版本没有任何改变 ，在 append_one 后也没有使用 ，  但 a 也许会被拷贝。这可以通过使用 inout 参数来避免这个问题：

````swift
func append_one_in_place(a: inout [Int]) {
  a.append(1)
}

var a = [1, 2, 3]
append_one_in_place(&a)
````

## Unchecked operations
## 未检查操作

> Swift eliminates integer overflow bugs by checking for overflow when performing normal arithmetic. These checks are not appropriate in high performance code where one knows that no memory safety issues can result.

Swift 通过在执行普通计算时检查溢出的方法解决了整数溢出的 bug。这些检查在已确定没有内存安全问题会发生的高效的代码中，是不合适的。

### Advice: Use unchecked integer arithmetic when you can prove that overflow cannot occur
### 建议：当你确切的知道不会发生溢出时使用未检查整型计算

> In performance-critical code you can elide overflow checks if you know it is safe.

在对性能要求高的代码中，如果你知道你的代码是安全的，那么你可以忽略溢出检查。

````swift
a: [Int]
b: [Int]
c: [Int]

// Precondition: for all a[i], b[i]: a[i] + b[i] does not overflow!
for i in 0 ... n {
  c[i] = a[i] &+ b[i]
}
````

## Generics
## 泛型

> Swift provides a very powerful abstraction mechanism through the use of generic types. The Swift compiler emits one block of concrete code that can perform MySwiftFunc<T> for any T. The generated code takes a table of function pointers and a box containing T as additional parameters. Any differences in behavior between MySwiftFunc<Int> and MySwiftFunc<String> are accounted for by passing a different table of function pointers and the size abstraction provided by the box. An example of generics:
  
Swift 通过泛型类型的使用，提供了一个非常强大的抽象机制 。Swift 编译器发出一个可以对任何 T 执行 MySwiftFunc<T> 的具体的代码块。生成的代码需要一个函数指针表和一个包含 T 的盒子作为额外的参数。MySwiftFunc<Int> 和 MySwiftFunc<String> 之间的不同的行为通过传递不同的函数指针表和通过盒子提供的抽象大小来说明。一个泛型的例子：

````swift
class MySwiftFunc<T> { ... }

MySwiftFunc<Int> X    // Will emit code that works with Int...
MySwiftFunc<String> Y // ... as well as String.
````

> When optimizations are enabled, the Swift compiler looks at each invocation of such code and attempts to ascertain the concrete (i.e. non-generic type) used in the invocation. If the generic function's definition is visible to the optimizer and the concrete type is known, the Swift compiler will emit a version of the generic function specialized to the specific type. This process, called specialization, enables the removal of the overhead associated with generics. Some more examples of generics:

当优化器启用时，Swift 编译器寻找这段代码的调用，并试着确认在调用中具体使用的类型（例如：非泛型类型）。如果泛型函数的定义对优化器来说是可见的，并知道具体类型，Swift 编译器将生成一个有特殊类型的特殊泛型函数。那么调用这个特殊函数的这个过程就可以避免关联泛型的消耗。一些泛型的例子：

````swift
class MyStack<T> {
  func push(_ element: T) { ... }
  func pop() -> T { ... }
}

func myAlgorithm(_ a: [T], length: Int) { ... }

// The compiler can specialize code of MyStack<Int>
var stackOfInts: MyStack<Int>
// Use stack of ints.
for i in ... {
  stack.push(...)
  stack.pop(...)
}

var arrayOfInts: [Int]
// The compiler can emit a specialized version of 'myAlgorithm' targeted for
// [Int]' types.
myAlgorithm(arrayOfInts, arrayOfInts.length)
````

### Advice: Put generic declarations in the same module where they are used
### 建议：将泛型的声明放在使用它的文件中

> The optimizer can only perform specialization if the definition of the generic declaration is visible in the current Module. This can only occur if the declaration is in the same file as the invocation of the generic, unless the -whole-module-optimization flag is used. NOTE The standard library is a special case. Definitions in the standard library are visible in all modules and available for specialization.

只有在泛型声明在当前模块可见的情况下优化器才能执行特殊化。这只有在使用泛型的代码和声明泛型的代码在同一个文件中才能发生。注意标准库是一个例外。在标准库中声明的泛型对所有模块可见并可以进行特殊化。

## The cost of large Swift values
## 大的值对象的开销

> In Swift, values keep a unique copy of their data. There are several advantages to using value-types, like ensuring that values have independent state. When we copy values (the effect of assignment, initialization, and argument passing) the program will create a new copy of the value. For some large values these copies could be time consuming and hurt the performance of the program.

在 swift 语言中，值类型保存它们数据独有的一份拷贝。使用值类型有很多优点，比如值类型具有独立的状态。当我们拷贝值类型时（相当于复制，初始化参数传递等操作），程序会创建值类型的一个拷贝。对于大的值类型，这种拷贝时很耗费时间的，可能会影响到程序的性能。

> Consider the example below that defines a tree using "value" nodes. The tree nodes contain other nodes using a protocol. In computer graphics scenes are often composed from different entities and transformations that can be represented as values, so this example is somewhat realistic.

让我们看一下下面这段代码。这段代码使用值类型的节点定义了一个树，树的节点包含了协议类型的其他节点，计算机图形场景经常由可以使用值类型表示的实体以及形态变化，因此这个例子很有实践意义。

````swift
protocol P {}
struct Node: P {
  var left, right: P?
}

struct Tree {
  var node: P?
  init() { ... }
}
````

> When a tree is copied (passed as an argument, initialized or assigned) the whole tree needs to be copied. In the case of our tree this is an expensive operation that requires many calls to malloc/free and a significant reference counting overhead.

当树进行拷贝时（参数传递，初始化或者赋值）整个树都需要被复制.这是一项花销很大的操作，需要很多的 malloc/free 调用以及以及大量的引用计数操作。

> However, we don't really care if the value is copied in memory as long as the semantics of the value remains.

然而，我们并不关系值是否被拷贝，只要在这些值还在内存中存在就可以。

### Advice: Use copy-on-write semantics for large values
### 对大的值类型使用 COW（copy-on-write，写时复制和数组有点类似）

> To eliminate the cost of copying large values adopt copy-on-write behavior. The easiest way to implement copy-on-write is to compose existing copy-on-write data structures, such as Array. Swift arrays are values, but the content of the array is not copied around every time the array is passed as an argument because it features copy-on-write traits.

减少复制大的值类型数据开销的办法时采用写时复制行为（当对象改变时才进行实际的复制工作）。最简单的实现写时复制的方案时使用已经存在的写时复制的数据结构，比如数组。Swift 的数据是值类型，但是当数组作为参数被传递时并不每次都进行复制，因为它具有写时复制的特性。

> In our Tree example we eliminate the cost of copying the content of the tree by wrapping it in an array. This simple change has a major impact on the performance of our tree data structure, and the cost of passing the array as an argument drops from being O(n), depending on the size of the tree to O(1).

在我们的 Tree 的例子中我们通过将 tree 的内容包装成一个数组来减少复制的代价。这个简单的改变对我们 tree 数据结构的性能影响时巨大的，作为参数传递数组的代价从 O(n) 变为 O(1)。

````swift
struct Tree: P {
  var node: [P?]
  init() {
    node = [thing]
  }
}
````

> There are two obvious disadvantages of using Array for COW semantics. The first problem is that Array exposes methods like "append" and "count" that don't make any sense in the context of a value wrapper. These methods can make the use of the reference wrapper awkward. It is possible to work around this problem by creating a wrapper struct that will hide the unused APIs and the optimizer will remove this overhead, but this wrapper will not solve the second problem. The Second problem is that Array has code for ensuring program safety and interaction with Objective-C. Swift checks if indexed accesses fall within the array bounds and when storing a value if the array storage needs to be extended. These runtime checks can slow things down.

但是使用数组实现 COW 机制有两个明显的不足，第一个问题是数组暴露的诸如 append 以及 count 之类的方法在值包装的上下文中没有任何作用，这些方法使得引用类型的封装变得棘手。也许我们可以通过创建一个封装的结构体并隐藏这些不用的 API 来解决这个问题，但是却无法解决第二个问题。第二个问题就是数组内部存在保证程序安全性的代码以及和 OC 交互的代码。Swift 要检查给出的下表是否搂在数组的边界内，当保存值的时候需要检查是否需要扩充存储空间。这些运行时检查会降低速度。

> An alternative to using Array is to implement a dedicated copy-on-write data structure to replace Array as the value wrapper. The example below shows how to construct such a data structure:

一个替代的方案是实现一个专门的使用 COW 机制的数据结构代替采用数组作为值的封装。构建这样一个数据结构的示例如下所示：

````swift
final class Ref<T> {
  var val: T
  init(_ v: T) {val = v}
}

struct Box<T> {
    var ref: Ref<T>
    init(_ x: T) { ref = Ref(x) }

    var value: T {
        get { return ref.val }
        set {
          if (!isKnownUniquelyReferenced(&ref)) {
            ref = Ref(newValue)
            return
          }
          ref.val = newValue
        }
    }
}
````

> The type Box can replace the array in the code sample above.

类型 Box 可以代替上个例子中的数组。

## Unsafe code
## 不安全的代码

> Swift classes are always reference counted. The Swift compiler inserts code that increments the reference count every time the object is accessed. For example, consider the problem of scanning a linked list that's implemented using classes. Scanning the list is done by moving a reference from one node to the next: elem = elem.next. Every time we move the reference Swift will increment the reference count of the next object and decrement the reference count of the previous object. These reference count operations are expensive and unavoidable when using Swift classes.

Swift 语言的类都是采用引用计数进行内存管理的。Swift 编译器会在每次对象被访问的时候插入增加引用计数的代码。例如，考虑一个遍历使用类实现的一个链表的例子。遍历链表是通过移动引用到链表的下一个节点来完成的：elem = elem.next，每次移动这个引用，Swift 都要增加 next 对象的引用计数并减少前一个对象的引用计数，这种引用计数代价昂贵但是只要使用 swift 类就无法避免。

````swift
final class Node {
 var next: Node?
 var data: Int
 ...
}
````

### advice: Use unmanaged references to avoid reference counting overhead
### 建议：使用未托管的引用避免引用计数的负荷

> Note, Unmanaged<T>._withUnsafeGuaranteedRef is not a public API and will go away in the future. Therefore, don't use it in code that you can not change in the future.
  
注意：Unmanaged<T>._withUnsafeGuaranteedRef 是私有的 API，可能会在将来被移除，请不要在不能修改的代码中使用。

> In performance-critical code you can choose to use unmanaged references. The Unmanaged<T> structure allows developers to disable automatic reference counting for a specific reference.

在效率至上的代码中你可以选择使用未托管的引用。Unmanaged<T>结构体允许开发者对特别的引用关闭引用计数

> When you do this, you need to make sure that there exists another reference to instance held by the Unmanaged struct instance for the duration of the use of Unmanaged (see Unmanaged.swift for more details) that keeps the instance alive.

````swift
// The call to ``withExtendedLifetime(Head)`` makes sure that the lifetime of
// Head is guaranteed to extend over the region of code that uses Unmanaged
// references. Because there exists a reference to Head for the duration
// of the scope and we don't modify the list of ``Node``s there also exist a
// reference through the chain of ``Head.next``, ``Head.next.next``, ...
// instances.

withExtendedLifetime(Head) {

  // Create an Unmanaged reference.
  var Ref: Unmanaged<Node> = Unmanaged.passUnretained(Head)

  // Use the unmanaged reference in a call/variable access. The use of
  // _withUnsafeGuaranteedRef allows the compiler to remove the ultimate
  // retain/release across the call/access.

  while let Next = Ref._withUnsafeGuaranteedRef { $0.next } {
    ...
    Ref = Unmanaged.passUnretained(Next)
  }
}
````

## Protocols
## 协议

### Advice: Mark protocols that are only satisfied by classes as class-protocols
### 建议：将只有类实现的协议标记为类协议

> Swift can limit protocols adoption to classes only. One advantage of marking protocols as class-only is that the compiler can optimize the program based on the knowledge that only classes satisfy a protocol. For example, the ARC memory management system can easily retain (increase the reference count of an object) if it knows that it is dealing with a class. Without this knowledge the compiler has to assume that a struct may satisfy the protocol and it needs to be prepared to retain or release non-trivial structures, which can be expensive.

Swift 可以指定协议只能由类实现。标记协议只能由类实现的一个好处是编译器可以基于这一点对程序进行优化。例如，ARC 内存管理系统能够容易的持有（增加该对象的引用计数）如果它知道它正在处理一个类对象。如果编译器不知道这一点，它就必须假设结构体也可以实现协议，那么它就必须准备好持有或者释放不同的数据结构，而这代价将会十分昂贵。

> If it makes sense to limit the adoption of protocols to classes then mark protocols as class-only protocols to get better runtime performance.

如果限制只能由类实现某协议那么就标记该协议为类协议以获得更好的性能。

````swift
protocol Pingable: AnyObject { func ping() -> Int }
````

## Unsupported Optimization Attributes
## 不支持的优化选项

> Some underscored type attributes function as optimizer directives. Developers are welcome to experiment with these attributes and send back bug reports and other feedback, including meta bug reports on the following incomplete documentation: :ref:`UnsupportedOptimizationAttributes`. These attributes are not supported language features. They have not been reviewed by Swift Evolution and are likely to change between compiler releases.

这里指的是一些带下划线的属性名字优化器指令。欢迎开发人员尝试这些属性并发送错误报告和其他反馈，包括有关以下不完整的元错误报告：ref：`UnsupportedOptimizationAttributes`。Swift Evolution 尚未对这些指令进行审查，并且它们可能会在编译器版本之间进行更改。

## Footnotes
## 脚注

[1]	A virtual method table or 'vtable' is a type specific table referenced by instances that contains the addresses of the type's methods. Dynamic dispatch proceeds by first looking up the table from the object and then looking up the method in the table.
虚拟方法表或者 vtable 是被一个实例引用的一种包含类型方法地址的类型约束表。进行动态分发时，首先从对象中查找这张表然后查找表中的方法

[2]	This is due to the compiler not knowing the exact function being called.
这是因为编译器并不知道那个具体的方法要被调用

[3]	i.e. a direct load of a class's field or a direct call to a function.
例如，直接加载一个类的字段或者直接调用一个方法

[4]	An optimization technique in which a copy will be made if and only if a modification happens to the original copy, otherwise a pointer will be given.
解释 COW 是什么

[5]	In certain cases the optimizer is able to via inlining and ARC optimization remove the retain, release causing no copy to occur.
在特定情况下优化器能够通过内联和 ARC 优化技术移除 retain，release 因为没有引起复制
