# 第十章 并发（和并行）

我希望通过本章为您提供所需的所有信息和工具，以便您在 Rust 程序中有效利用并发功能，在您的库中实现并发支持，以及正确使用 Rust 的并发原语。我不会直接教您如何实现并发数据结构或编写高性能并发应用程序。相反，我的目标是让您充分理解底层机制，以便您能够自行运用它们来完成任何您需要的任务。

&nbsp;&nbsp;&nbsp;&nbsp;并发有三种形式：单线程并发（例如我们在第 8 章中讨论过的 async/await）、单核多线程并发，以及实现真正并行的多核并发。每种类型都允许程序中的并发任务以不同的方式交错执行。如果考虑到操作系统调度和抢占的细节，还会有更多子类型，但我们不会对此进行深入探讨。

&nbsp;&nbsp;&nbsp;&nbsp;在类型层面，Rust 仅代表并发的一个方面：多线程。一个类型要么可以被多个线程安全地使用，要么不安全。即使你的程序有多个线程（因此是并发的），但只有一个核心（因此不是并行的），Rust 也必须假设如果有多个线程，就可能存在并行。我们将要讨论的大多数类型和技术都同样适用，无论两个线程是否真正并行执行，因此为了保持语言的简洁，在本章中，我将使用“并发”一词的非正式含义，即“几乎同时运行的事物”。当区别很重要时，我会特别指出。

&nbsp;&nbsp;&nbsp;&nbsp; Rust基于类型的安全多线程方法的特别之处在于，它并非编译器的功能，而是一个库功能，开发者可以扩展它来开发复杂的并发合约。由于线程安全在类型系统中通过 `Send` 和 `Sync` 的实现和边界来表达，这些边界会一直传播到应用程序代码，因此整个程序的线程安全仅通过类型检查即可完成。

&nbsp;&nbsp;&nbsp;&nbsp; Rust 编程语言已经涵盖了并发的大部分基础知识，包括 `Send` 和 `Sync` `trait`、`Arc` 和 `Mutex` 以及 `channels`。因此，我不会在这里重复太多内容，除非在某些主题中需要特别重复。我们将探讨并发的难点，以及一些旨在解决这些难题的常见并发模式。在深入探讨如何使用原子操作实现底层并发操作之前，我们还将探讨并发和异步是如何相互作用的（以及它们是如何不相互作用的）。最后，我将在本章的结尾提供一些建议，帮助您在编写并发代码时保持理智。

## 并发问题

在深入探讨并发编程的良好模式以及 Rust 并发机制的细节之前，我们有必要花些时间理解一下，为什么并发编程如此具有挑战性。也就是说，为什么我们需要为并发代码制定特殊的模式和机制？

### 正确性

并发的主要难点在于协调对多个线程共享的资源的访问，尤其是写访问。如果许多线程只想读取某个资源，那么这通常很容易：你可以将其放入 `Arc` 中，或者将其放置在一个可以使用 `&'static` 的地方，这样就搞定了。但是，一旦任何线程想要写入，就会出现各种各样的问题，通常以数据争用的形式出现。简而言之，当一个线程更新共享状态，而另一个线程也在访问该状态（无论是读取还是更新）时，就会发生数据争用。如果没有额外的保护措施，第二个线程可能会读取部分覆盖的状态，破坏第一个线程写入的部分内容，或者根本看不到第一个线程的写入！一般来说，所有数据争用都被视为未定义行为。

&nbsp;&nbsp;&nbsp;&nbsp;数据竞争属于一类更广泛的问题，这类问题主要（但并非唯一）发生在并发环境中：竞争条件。竞争条件是指，根据系统中其他事件的相对时间，一系列指令可能产生多种结果。这些事件可以是线程执行特定代码段、计时器启动、网络数据包传入，或任何其他随时间变化的事件。与数据竞争不同，竞争条件本身并不坏，也不被视为未定义行为。然而，当发生特别特殊的竞争时，它们会成为错误的滋生地，正如您将在本章中看到的那样。

### 性能

开发人员通常会在程序中引入并发，希望提升性能。或者更准确地说，他们希望并发能够利用更多硬件资源，使他们能够每秒执行更多操作。这可以在单核上实现，即让一个线程运行，另一个线程等待；也可以在多核上实现，即让线程同时在每个内核上执行工作，而这些工作原本只能在一个内核上串行执行。大多数开发人员在谈论并发时指的是后一种性能提升，并发通常与可扩展性相关。可扩展性在这里的意思是“程序的性能会随着内核数量的增加而扩展”，这意味着如果为程序提供更多内核，其性能就会提高。

&nbsp;&nbsp;&nbsp;&nbsp;虽然实现这样的加速是可能的，但实际操作起来却比想象中要难。可扩展性的最终目标是线性可扩展性，即核心数量翻倍，程序单位时间内完成的工作量也会翻倍。线性可扩展性也常被称为完美可扩展性。然而，实际上，很少有并发程序能够实现这样的加速。亚线性扩展更为常见，即吞吐量随着核心数量的增加而几乎呈线性增长，但增加更多核心的收益却递减。有些程序甚至会经历负扩展，即让程序访问更多核心反而会降低吞吐量，这通常是因为许多线程都在争用某些共享资源。

&nbsp;&nbsp;&nbsp;&nbsp;想象一下一群人试图戳破气泡膜上的所有气泡——一开始增加人手会有帮助，但到了一定时候，收益就会递减，因为拥挤会让任何一个人的工作都更加困难。如果参与其中的人效率特别低，你们的团队最终可能会站着讨论谁该下一个戳破，结果一个气泡也戳不破！这种本应并行执行的任务之间的干扰被称为争用，也是良好扩展的一大障碍。争用可能以多种方式出现，但主要原因是互斥、共享资源耗尽和错误共享。

#### 互斥

如果同一时刻只允许一个并发任务执行一段特定代码，我们称该代码段的执行是互斥的——如果一个线程执行了该代码段，则其他线程不能同时执行。互斥锁（mutex）就是一个典型的例子，它明确地强制同一时刻只有一个线程可以进入程序代码的特定关键代码段。然而，互斥也可以隐式地发生。例如，如果您启动一个线程来管理共享资源，并通过 mpsc 通道向其发送作业，则该线程实际上实现了互斥，因为同一时刻只有一个这样的作业可以执行。

&nbsp;&nbsp;&nbsp;&nbsp;当调用操作系统或库调用时，如果这些调用在内部强制对临界区进行单线程访问，也可能发生互斥。例如，多年来，标准内存分配器要求某些分配操作必须互斥，这使得内存分配操作在原本高度并行的程序中引发了严重的争用。同样，许多看似独立的操作系统操作（例如在同一目录中创建两个名称不同的文件）最终可能不得不在内核中顺序执行。

> 可扩展的并发分配是 `jemalloc` 内存分配器存在的理由！

&nbsp;&nbsp;&nbsp;&nbsp; 互斥是并行加速最明显的障碍，因为根据定义，它会强制程序的某些部分串行执行。即使你让程序的其余部分与核心数量完美匹配，你所能实现的总加速比也会受到互斥串行部分长度的限制。务必注意互斥部分，并尽量将其限制在绝对必要的范围内。

> 对于理论型的人来说，可以使用阿姆达尔定律来计算由于代码段相互排斥而导致的可实现加速的限制

#### 共享资源耗尽

不幸的是，即使你的任务实现了完美的并发性，这些任务需要与之交互的环境本身也可能并非完美可扩展。内核每秒在给定 TCP 套接字上处理的发送次数有限，内存总线一次只能执行有限的读取操作，而你的 GPU 的并发能力也有限。这个问题没有解决办法。在实践中，环境通常是完美可扩展性失效的地方，而修复这种情况往往需要大量的重新设计（甚至需要新的硬件！），因此我们在本章中不会过多讨论这个话题。只需记住，可扩展性很少是你能够“实现”的，而更多是你努力追求的。

#### 伪共享

伪共享是指两个本不该相互竞争的操作发生竞争，从而阻碍了高效的同时执行。这种情况通常是因为两个操作恰好在某些共享资源上相交，尽管它们使用了该资源中不相关的部分。

&nbsp;&nbsp;&nbsp;&nbsp; 最简单的例子是锁过度共享，即一个锁守护着某个复合状态，两个原本独立的操作都需要获取该锁来更新各自的状态部分。这意味着操作必须串行执行，而不是并行执行。在某些情况下，可以将单个锁拆分成两个，每个不相交的部分各一个，这样操作就可以并行进行。然而，像这样拆分锁并不总是那么简单——状态可能共享一个锁，因为某个第三个操作需要锁定该状态的所有部分。通常情况下，你仍然可以拆分锁，但必须注意不同线程获取拆分锁的顺序，以避免两个操作尝试以不同的顺序获取锁时可能发生的死锁（如果你感兴趣，可以查阅“哲学家就餐问题”）。或者，对于某些问题，你或许可以通过使用底层算法的无锁版本来完全避开临界区，尽管这些算法也很难做到正确。归根结底，伪共享是一个很难解决的问题，而且没有一个万能的解决方案——但识别问题是一个好的开始。

&nbsp;&nbsp;&nbsp;&nbsp; 伪共享的一个更微妙的例子发生在 CPU 层面，正如我们在第二章简要讨论过的。CPU 内部以缓存行（内存中较长的连续字节序列）而不是单个字节的方式对内存进行操作，以分摊内存访问的成本。例如，在大多数 Intel 处理器上，缓存行大小为 64 字节。这意味着每次内存操作实际上都会读取或写入 64 字节的倍数。当两个核心想要更新恰好位于同一缓存行上的两个不同字节的值时，就会发生伪共享；即使这些更新在逻辑上是不相交的，也必须按顺序执行。

&nbsp;&nbsp;&nbsp;&nbsp; 这看起来似乎太底层，无关紧要，但在实践中，这种错误共享可能会大幅降低应用程序的并行加速。想象一下，你分配了一个整数数组来指示每个线程完成了多少个操作，但这些整数都位于同一个缓存行中——现在，所有原本并行的线程在执行每个操作时都会争用这一个缓存行。如果操作相对较快，那么你的大部分执行时间最终可能会花在争用这些计数器上！

&nbsp;&nbsp;&nbsp;&nbsp; 避免错误缓存行共享的技巧是填充值，使其大小与缓存行大小一致。这样，两个相邻的值总是位于不同的缓存行上。当然，这也会增大数据结构的大小，因此仅在基准测试表明存在问题时才使用此方法。

> 可扩展性的成本
>
> 您应该注意并发性中一个与正交性略有不同之处，那就是首先引入并发性的成本。编译器非常擅长优化单线程代码——毕竟它们已经做了很长时间了——而且单线程代码通常比并发代码更少地使用昂贵的保护措施（例如锁、通道或原子指令）。总的来说，无论核心数量多少，并发性的各种成本都可能导致并行程序比单线程程序更慢！这就是为什么在优化和并行化之前和之后进行测量非常重要：结果可能会让您大吃一惊。
> 如果你对这个话题感兴趣，我强烈建议你阅读 Frank McSherry 于 2015 年发表的论文《可扩展性！但代价是什么？》（https://www.frankmcsherry.org/assets/COST.pdf），其中揭示了一些“代价高昂的扩展”的极其恶劣的例子。

## 并发模型

Rust 有三种模式可以为您的程序添加并发功能，这些模式您经常会遇到：共享内存并发、工作池和 `Actor`。要详细讲解每一种添加并发功能的方式，恐怕需要写一本书，所以这里我将只关注这三种模式。

### 共享内存

共享内存并发的概念非常简单：线程通过操作它们之间共享的内存区域进行协作。这可能表现为由互斥锁保护的状态，或存储在哈希映射中，并支持多个线程的并发访问。这些线程可能对不相交的数据块执行相同的任务，例如，多个线程对一个向量的不相交子范围执行某个函数；或者，它们可能执行需要共享状态的不同任务，例如在数据库中，一个线程处理用户对表的查询，而另一个线程在后台优化用于存储该表的数据结构。

&nbsp;&nbsp;&nbsp;&nbsp;使用共享内存并发时，数据结构的选择至关重要，尤其是在相关线程需要紧密协作的情况下。常规互斥锁可能会阻止扩展到极少数核心之外；读写锁可能会允许更多并发读取，但写入速度会降低；而分片读写锁可能会允许完全可扩展的读取，但写入操作会造成极大的破坏性。同样，一些并发哈希映射旨在实现良好的整体性能，而另一些则专门针对写入操作较少的并发读取。一般而言，在共享内存并发中，您需要使用专为尽可能贴近目标用例而设计的数据结构，以便您可以利用优化措施，在应用程序不关心的性能方面与它关心的性能方面之间做出权衡。

&nbsp;&nbsp;&nbsp;&nbsp;共享内存并发非常适合线程需要以非交换的方式联合更新某些共享状态的用例。也就是说，如果一个线程需要使用某个函数 `f` 更新状态 `s`，而另一个线程需要使用某个函数 `g` 更新状态，并且 f(g(s)) != g(f(s))，那么共享内存并发很可能是必要的。如果不是这种情况，其他两种模式可能更适合，因为它们往往能带来更简单、更高效的设计。

> 一些问题已知有算法可以在不使用锁的情况下提供并发共享内存操作。随着核心数量的增长，这些无锁算法可能比基于锁的算法具有更好的扩展性，尽管由于其复杂性，它们的单核性能通常也较低。性能问题始终重要，因此请先进行基准测试，然后再寻找替代方案。

### 工作池

在工作池模型中，许多相同的线程从共享的作业队列接收作业，然后完全独立地执行这些作业。例如，Web 服务器通常有一个工作池来处理传入的连接，异步代码的多线程运行时倾向于使用工作池来集体执行应用程序的所有未来任务（或者更准确地说，是其顶级任务）。

&nbsp;&nbsp;&nbsp;&nbsp;共享内存并发和工作池之间的界限通常很模糊，因为工作池倾向于使用共享内存并发来协调如何从队列中获取作业以及如何将未完成的作业返回队列。例如，假设您正在使用数据并行库 `Rayon` 对向量的每个元素并行执行某项函数。`Rayon` 在后台启动一个工作池，将向量拆分为子范围，然后将子范围分发给池中的线程。当池中的线程完成一个范围的操作时，`Rayon` 会安排它开始处理下一个未处理的子范围。该向量在所有工作线程之间共享，并且线程通过一个支持工作窃取的共享内存队列式数据结构进行协调。

&nbsp;&nbsp;&nbsp;&nbsp;工作窃取是大多数工作池的关键特性。其基本原理是，如果一个线程提前完成其工作，并且没有其他未分配的工作，则该线程可以窃取已分配给其他工作线程但尚未启动的工作。并非所有工作的完成时间都相同，因此即使每个工作线程分配到的工作数量相同，某些工作线程最终也可能比其他工作线程更快地完成其工作。与其等待那些提取了运行时间较长的作业的线程完成，那些提前完成的线程应该帮助那些落后的线程，以便整个操作能够更快地完成。

&nbsp;&nbsp;&nbsp;&nbsp;实现一个支持这种工作窃取的数据结构是一项艰巨的任务，并且不会因为线程不断尝试互相窃取工作而产生巨大的开销，但这一特性对于高性能工作池至关重要。如果你发现自己需要一个工作池，最好的选择通常是使用一个已经完成大量工作的，或者至少重用现有数据结构的数据结构，而不是自己从头编写一个。

&nbsp;&nbsp;&nbsp;&nbsp;当每个线程执行的工作相同，但执行的数据不同时，工作池是一个不错的选择。在 `Rayon` 并行映射操作中，每个线程执行相同的映射计算；它们只是在底层数据的不同子集上执行。在多线程异步运行时中，每个线程只需调用 `Future::poll`；它们只是在不同的 `Future` 上调用它。如果您开始需要区分线程池中的线程，那么不同的设计可能更合适。

> 连接池
>
> 连接池是一种共享内存结构，它保存一组已建立的连接，并将它们分配给需要连接的线程。它是管理外部服务连接的库中常见的设计模式。如果某个线程需要连接但没有可用的连接，则会建立新的连接，或者强制该线程阻塞。当某个线程用完一个连接后，它会将该连接返回到连接池中，以便其他可能正在等待的线程可以使用它。
>
> &nbsp;&nbsp;&nbsp;&nbsp;通常，连接池最难的任务是管理连接的生命周期。连接可以以最后一个使用该连接的线程所处的状态返回到池中。因此，连接池必须确保与连接相关的任何状态（无论是在客户端还是在服务器上）都已重置，以便当其他线程随后使用该连接时，该线程可以像获得了一个新的专用连接一样运行。

### Actors

`Actor` 并发模型在很多方面与工作池模型截然相反。工作池有许多相同的线程共享一个作业队列，而 `Actor` 模型则拥有许多独立的作业队列，每个作业“主题”对应一个作业队列。每个作业队列都会将数据馈送到特定的 `Actor`，该 `Actor` 负责处理与应用程序状态子集相关的所有作业。该状态可能是数据库连接、文件、指标收集数据结构，或者任何其他您能想到的可能需要多个线程访问的结构。无论它是什么，一个 `Actor` 都拥有该状态，如果某个任务想要与该状态交互，它需要向拥有该状态的 `Actor` 发送一条消息，概述其希望执行的操作。当拥有该状态的 `Actor` 收到该消息时，它会执行指示的操作，并向发出请求的任务返回操作结果（如果相关）。由于 `Actor` 对其内部资源拥有独占访问权，因此除了消息传递所需的机制外，不需要任何锁或其他同步机制。

&nbsp;&nbsp;&nbsp;&nbsp;`Actor` 模式的一个关键点是所有 `Actor` 之间都相互通信。例如，如果一个负责日志记录的 `Actor` 需要写入文件和数据库表，它可能会向负责每个操作的 `Actor` 发送消息，要求它们执行相应的操作，然后继续处理下一个日志事件。这样，`Actor` 模型更像是一张网，而不是车轮上的辐条——用户对 Web 服务器的请求可能最初是向负责该连接的 `Actor` 发出的单个请求，但在用户请求得到满足之前，可能会向系统更深层次的 `Actor` 传递数十、数百甚至数千条消息。

&nbsp;&nbsp;&nbsp;&nbsp;`Actor` 模型并不要求每个 `Actor` 都拥有独立的线程。相反，大多数 `Actor` 系统建议应该存在大量的 `Actor`，因此每个 `Actor` 应该映射到一个任务而不是一个线程。毕竟，`Actor` 仅在执行时才需要对其包装资源的独占访问权，而并不关心它们是否位于自己的线程中。事实上，`Actor` 模型经常与工作池模型结合使用——例如，使用多线程异步运行时 `Tokio` 的应用程序可以为每个 `Actor` 生成一个异步任务，然后 `Tokio` 会将每个 `Actor` 的执行作为其工作池中的作业。因此，给定 `Actor` 的执行可能会随着 `Actor` 的放弃和恢复而在工作池中从一个线程移动到另一个线程，但每次 `Actor` 执行时，它都保持对其包装资源的独占访问权。

&nbsp;&nbsp;&nbsp;&nbsp;`Actor` 并发模型非常适合于拥有众多资源且这些资源可以相对独立运行，并且每个资源内部几乎没有或根本没有并发机会的情况。例如，操作系统可能为每个硬件设备分配一个 `Actor`，Web 服务器可能为每个后端数据库连接分配一个 `Actor`。如果您只需要少数 `Actor`，或者 `Actor` 之间的工作严重不均衡，或者某些 `Actor` 规模过大，那么 `Actor` 模型的效果就不太好。在所有这些情况下，您的应用程序最终可能会因为系统中某个 `Actor` 的执行速度而成为瓶颈。而且，由于每个 `Actor` 都希望独占访问其所属的那一小部分资源，因此您无法轻松地并行化那个瓶颈 `Actor` 的执行。

## 异步和并行

正如我们在第八章中讨论的那样，Rust 中的异步特性实现了并发，而无需并行——我们可以使用 `selects` 和 `joins` 之类的结构，让单个线程轮询多个 `Future`，并在其中一个、部分或全部完成时继续执行。由于不涉及并行，因此 `Future` 的并发性从根本上来说并不要求这些 `Future` 必须发送。即使创建一个 `Future` 作为额外的顶级任务运行，也从根本上来说不需要发送，因为单个执行线程可以同时管理多个 `Future` 的轮询。

&nbsp;&nbsp;&nbsp;&nbsp;然而，在大多数情况下，应用程序既需要并发性，也需要并行性。例如，如果一个 Web 应用程序为每个传入连接构建一个 `Future`，因此同时存在多个活动连接，它很可能希望异步执行器能够充分利用主机上的多个核心。但这并非自然而然就能实现：您的代码必须明确地告诉执行器哪些 `Future` 可以并行运行，哪些不能。

&nbsp;&nbsp;&nbsp;&nbsp;具体来说，必须向执行器提供两条信息，以便它知道可以将 `Future` 中的工作分散到线程工作池中。第一，所讨论的 `Future` 是否`Send`——如果不是，则执行器不允许将 `Future` 发送给其他线程进行处理，并且无法实现并行；只有构造此类 `Future` 的线程可以轮询它们。

&nbsp;&nbsp;&nbsp;&nbsp;第二部分信息是如何将 `Future` 拆分成可以独立运行的任务。这与第八章中关于任务与 `Future` 的讨论相关：如果一个巨型 `Future` 包含多个 `Future` 实例，而这些实例本身又对应着可以并行运行的任务，那么执行器仍然必须在顶层 `Future` 上调用 `poll`，并且必须在单个线程中执行此操作，因为 `poll` 需要 `&mut self`。因此，为了实现 `Future` 的并行性，你必须显式地生成你想要并行运行的 `Future`。此外，由于第一个要求，你用于执行此操作的执行器函数将要求传入的 `Future` 为 `Send`。

> 异步同步原语
>
> 大多数用于阻塞代码的同步原语（例如 `std::sync`）也都有异步版本。通道、互斥锁、读写锁、屏障以及各种其他类似结构都有异步版本。我们需要这些，因为正如第八章所讨论的，在 `Future` 内部进行阻塞会阻碍执行器可能需要执行的其他工作，因此不建议这样做。
>
> &nbsp;&nbsp;&nbsp;&nbsp;然而，这些原语的异步版本通常比同步版本慢，因为需要额外的机制来执行必要的唤醒。因此，即使在异步环境中，只要使用同步原语不会阻塞执行器，您也可能需要使用同步原语。例如，虽然获取互斥锁通常可能会阻塞很长时间，但对于某些特定的互斥锁来说，情况可能并非如此，因为这些互斥锁可能很少被获取，而且只会在短时间内被阻塞。在这种情况下，短暂地阻塞直到互斥锁再次可用可能实际上不会造成任何问题。您需要确保在持有 `MutexGuard` 时永远不会放弃或执行其他长时间运行的操作，但除此之外，您不应该遇到问题。
>
> &nbsp;&nbsp;&nbsp;&nbsp;不过，与此类优化一样，请务必先进行测量，并且仅在同步原语能够显著提升性能的情况下才选择它。如果不能，那么在异步环境中使用同步原语所带来的额外麻烦可能并不值得。

## 低级并发

标准库提供了 `std::sync::atomic` 模块，它提供对底层 CPU 原语的访问，这些原语是通道和互斥量等高级结构构建的基础。这些原语以原子类型（名称以 `Atomic` 开头）的形式出现，例如 `AtomicUsize`、`AtomicI32`、`AtomicBool`、`AtomicPtr` 等等；`Ordering` 类型；以及两个名为 `fence` 和 `compilation_fence` 的函数。我们将在接下来的几节中逐一介绍这些原语。

&nbsp;&nbsp;&nbsp;&nbsp;这些类型是用于构建任何需要在线程间通信的代码的块。互斥锁、通道、屏障、并发哈希表、无锁堆栈以及所有其他同步结构最终都依赖于这几个原语来完成它们的工作。它们本身也适用于线程间的轻量级协作，当像互斥锁这样的重量级同步过于繁琐时——例如，增加共享计数器或将共享布尔值设置为 `true`。

&nbsp;&nbsp;&nbsp;&nbsp;原子类型的特殊之处在于，它们定义了当多个线程尝试并发访问它们时会发生什么的语义。这些类型都支持（大部分）相同的 API：`load`、`store`、`fetch_*` 和 `compare_exchange`。在本节的其余部分，我们将了解它们的作用、如何正确使用它们以及它们的用途。但首先，我们必须讨论一下低级内存操作和内存排序。

#### 内存操作

非正式地，我们通常将访问变量称为“读取”或“写入”内存。实际上，在使用变量的代码和访问内存硬件的实际 CPU 指令之间存在许多机制。为了理解并发内存访问的行为方式，理解这些机制（至少在高层次上）非常重要。

&nbsp;&nbsp;&nbsp;&nbsp;编译器决定在程序读取变量值或为其赋值时发出哪些指令。它可以对代码执行各种转换和优化，并可能最终重新排序程序语句、消除其认为多余的操作，或者使用 CPU 寄存器而不是实际内存来存储中间计算结果。编译器对这些转换有很多限制，但最终只有一小部分变量访问最终会变成内存访问指令。

&nbsp;&nbsp;&nbsp;&nbsp;在 CPU 层面，内存指令主要有两种形式：加载和存储。加载指令将字节从内存中的某个位置加载到 CPU 寄存器中，而存储指令将字节从 CPU 寄存器存储到内存中的某个位置。加载和存储指令每次操作的内存块较小：在现代 CPU 上通常为 8 个字节或更少。如果变量访问跨度超过单个加载或存储指令所能访问的字节数，编译器会根据需要自动将其转换为多个加载或存储指令。CPU 在执行程序指令的方式上也有一定的灵活性，以便更好地利用硬件并提高程序性能。例如，现代 CPU 通常并行执行指令，甚至乱序执行，因为它们彼此之间没有依赖关系。每个 CPU 和计算机的 DRAM 之间还有多层缓存，这意味着加载给定内存位置的指令不一定能看到该内存位置的最新存储（以挂钟时间计算）。

&nbsp;&nbsp;&nbsp;&nbsp;在大多数代码中，编译器和 CPU 只能以不影响最终程序语义的方式转换代码，因此这些转换对程序员来说是不可见的。然而，在并行执行的环境中，这些转换可能会对应用程序行为产生重大影响。因此，CPU 通常提供多种不同的加载和存储指令变体，每种变体对于 CPU 如何重新排序以及如何与其他 CPU 上的并行操作交错执行都有不同的保证。同样，编译器（或者更确切地说，编译器编译的语言）提供了不同的注解，您可以使用它们来强制执行某些内存访问子集的特定执行约束。在 Rust 中，这些注解以原子类型及其方法的形式出现，我们将在本节的剩余部分对它们进行分析。

#### 原子类型

Rust 的原子类型之所以如此命名，是因为它们可以被原子地访问——也就是说，原子类型变量的值会被一次性写入，并且永远不会使用多次存储来写入，从而保证了该变量的加载不会观察到只有组成该值的部分字节发生了变化，而其他字节（目前）尚未发生变化。通过与非原子类型进行对比，这一点更容易理解。例如，将新值重新赋给 `(i64, i64)` 类型的元组通常需要两条 CPU 存储指令，每个 8 字节值一条。如果一个线程同时执行这两个存储操作，另一个线程（如果我们暂时忽略借位检查器）可能会在第一次存储之后、第二次存储之前读取元组的值，从而导致元组值的视图不一致。它最终会读取第一个元素的新值和第二个元素的旧值，而这个值实际上从未被任何线程存储过。

&nbsp;&nbsp;&nbsp;&nbsp; CPU 只能原子访问特定大小的值，因此只有少数原子类型，所有这些类型都位于 `atomic` 模块中。每种原子类型都属于 CPU 支持原子访问的大小之一，并有多种变体，例如值是否带符号，以及区分原子 `usize` 和指针（其大小与 usize 相同）。此外，原子类型具有显式的方法来加载和存储它们所持有的值，以及一些更复杂的方法（我们稍后会讨论），以便程序员编写的代码与生成的 CPU 指令之间的映射更加清晰。例如，`AtomicI32::load` 执行一次带符号的 32 位值的加载，而 `AtomicPtr::store` 执行一次指针大小（在 64 位平台上为 64 位）值的存储。

#### 内存排序

原子类型的大多数方法都接受一个 `Ordering` 类型的参数，该类型参数规定了原子操作所遵循的内存排序限制。在不同线程之间，编译器和 CPU 只能以与该原子值上每个原子操作所请求的内存排序兼容的交错方式对原子值的加载和存储进行排序。在接下来的几节中，我们将通过一些示例来说明为什么控制排序对于从编译器和 CPU 获得预期的语义至关重要且必要。

&nbsp;&nbsp;&nbsp;&nbsp;内存排序常常被认为是违反直觉的，因为我们人类喜欢从上到下阅读程序，并想象它们逐行执行——但代码到达硬件时的实际执行方式并非如此。内存访问可以重新排序，甚至完全省略，并且一个线程上的写入操作可能不会立即对其他线程可见，即使后续按程序顺序进行的写入操作已经被观察到。

&nbsp;&nbsp;&nbsp;&nbsp;可以这样想：每个内存位置都会看到来自不同线程的一系列修改，并且不同内存位置的修改序列是独立的。如果两个线程 T1 和 T2 都写入内存位置 M，那么即使用户用秒表测量 T1 首先执行，如果两个线程的执行之间没有任何其他约束，T2 对 M 的写入仍然可能看起来是首先发生的。本质上，计算机在确定给定内存位置的值时不会考虑挂钟时间——重要的是程序员对有效执行的执行约束。例如，如果 T1 写入 M，然后生成线程 T2，然后 T2 写入 M，则计算机必须识别出 T1 的写入是首先发生的，因为 T2 的存在依赖于 T1。

&nbsp;&nbsp;&nbsp;&nbsp;如果这难以理解，别担心——内存排序可能令人费解，语言规范往往会使用非常精确但不太直观的措辞来描述它。我们可以构建一个更容易理解（即使稍微简化）的思维模型，重点关注底层硬件架构。简单来说，你的计算机内存结构是一个树状的存储层次结构，其中叶子是 CPU 寄存器，根是物理内存芯片上的存储空间，通常称为主内存。两者之间有几层缓存，层次结构的不同层级可以驻留在不同的硬件上。当一个线程执行向内存位置的存储操作时，实际上发生的是，CPU 发起一个写入请求，请求指向给定 CPU 寄存器中的值，然后该请求必须沿着内存层次结构向上传递到主内存。当一个线程执行加载操作时，请求会沿着层次结构向上流动，直到到达具有可用值的层级，然后从那里返回。问题就在这里：写入操作并非在所有缓存都可见，直到所有写入内存位置的缓存都更新完毕，但其他 CPU 可以同时针对同一内存位置执行指令，于是奇怪的事情就发生了。因此，内存排序是一种请求精确语义的方法，用于指示当多个 CPU 访问特定内存位置执行特定操作时会发生什么。

&nbsp;&nbsp;&nbsp;&nbsp;考虑到这一点，让我们来看看 Ordering 类型，它是我们作为程序员可以规定哪些并发执行有效的额外约束的主要机制。

排序被定义为一个枚举，其变体如清单 10-1 所示。

```rust
enum Ordering {
 Relaxed,
 Release,
 Acquire,
 AcqRel,
 SeqCst
}

// 清单 10-1：排序的定义
```

&nbsp;&nbsp;&nbsp;&nbsp;这些都对从源代码到执行语义的映射施加了不同的限制，我们将在本节的其余部分依次探讨每一个。

##### Relaxed Ordering

宽松排序本质上不保证对值的并发访问，除了访问是原子的之外。具体来说，宽松排序不保证跨不同线程的内存访问的相对顺序。这是最弱的内存排序形式。清单 10-2 展示了一个简单的程序，其中两个线程使用 `Ordering::Relaxed` 访问两个原子变量。

```rust
static X: AtomicBool = AtomicBool::new(false);
static Y: AtomicBool = AtomicBool::new(false);
let t1 = spawn(|| {
 /*1*/ let r1 = Y.load(Ordering::Relaxed);
 /*2*/ X.store(r1, Ordering::Relaxed);
});
let t2 = spawn(|| {
 /*3*/ let r2 = X.load(Ordering::Relaxed);
 /*4*/ Y.store(true, Ordering::Relaxed)
});

// 示例 10-2：使用 Ordering::Relaxed 的两个竞争线程
```

&nbsp;&nbsp;&nbsp;&nbsp;看看作为 `t2` 生成的线程，你可能会认为 `r2` 永远不会为真，因为在同一个线程读取 `X` 之后，将 `Y` 赋值为真之前，所有值都是假。然而，在`Relaxed Ordering`下，这种结果是完全可能的。原因是 CPU 可以重新排序相关的加载和存储操作。让我们来详细了解一下这里究竟发生了什么，使得 `r2 = true`。

&nbsp;&nbsp;&nbsp;&nbsp;首先，CPU 注意到 *4* 不必在 *3* 之后发生，因为 *4* 不使用 `3` 的任何输出或副作用。也就是说，*4* 对 *3* 没有执行依赖。因此，CPU 决定重新排序它们，理由是*挥挥手*，这样可以提高程序运行速度。因此，CPU 继续执行，首先执行 *4*，并将 `Y` 设置为 `true`，即使 *3* 尚未运行。然后，操作系统将 `t2` 置于睡眠状态，线程 `t1` 执行几条指令，或者 `t1` 在另一个核心上执行。在 `t1` 中，编译器确实必须先运行 *1*，然后再运行 *2*，因为 *2* 依赖于 *1* 中读取的值。因此，*t1* 将 `true` 从 `Y`（由 *4* 写入）读取到 `r1`，然后将其写回 `X`。最后，`t2` 执行 *3*，它读取 `X` 并得到 `true`，正如 *2* 写入的那样。

&nbsp;&nbsp;&nbsp;&nbsp;`Relaxed Ordering`允许这种执行，因为它不会对并发执行施加任何额外的限制。也就是说，在`Relaxed Ordering`下，编译器只需确保任何给定线程的执行依赖关系得到满足（就像不涉及原子操作一样）；它无需对并发操作的交错做出任何承诺。对于单线程执行，允许对 `3` 和 `4` 进行重新排序，因此在宽松排序下也允许这样做。

&nbsp;&nbsp;&nbsp;&nbsp;在某些情况下，这种重新排序是可以接受的。例如，如果你有一个计数器，它只是跟踪指标，那么它相对于其他指令的具体执行时间并不重要，`Ordering::Relaxed` 就可以了。在其他情况下，这可能会带来灾难性的后果：比如，如果你的程序使用 `r2` 来判断安全保护措施是否已经设置，最终错误地认为它们已经设置好了。

&nbsp;&nbsp;&nbsp;&nbsp;在编写未大量使用原子操作的代码时，通常不会注意到这种重新排序——CPU 必须保证编写的代码与每个线程实际执行的代码之间没有可观察到的差异，因此一切看起来都像按照您编写的顺序运行。这被称为尊重程序顺序或求值顺序；这两个术语是同义词。

##### Acquire/Release Ordering

在内存排序层次结构的上一级，我们有`Ordering::Acquire`、`Ordering::Release` 和 `Ordering::AcqRel`（获取加释放）。从高层次上讲，这些操作在一个线程中的存储操作和另一个线程中的加载操作之间建立了执行依赖关系，并限制了操作相对于该加载和存储操作的重新排序方式。至关重要的是，这些依赖关系不仅建立了单个值的存储和加载之间的关系，还对相关线程中的其他加载和存储操作施加了排序约束。这是因为每次执行都必须遵循程序顺序；如果线程 B 中的加载操作依赖于线程 A 中的某个存储操作（A 中的存储操作必须在 B 中的加载操作之前执行），那么在该加载操作之后在 B 中进行的任何读写操作也必须在 A 中的存储操作之后进行。

> `Acquire` 内存排序仅适用于加载操作，`Release` 仅适用于存储操作，而 `AcqRel` 仅适用于同时包含加载和存储的操作（例如 `fetch_add`）。

具体来说，这些内存排序对执行施加了以下限制：

1. 加载和存储操作不能向前移动到使用 `Ordering::Release` 的存储操作之后。
2. 加载和存储操作不能向后移动到使用 `Ordering::Acquire` 的加载操作之前,
3. 变量的 `Ordering::Acquire` 加载必须能够看到在 `Ordering::Release` 存储（该存储操作存储了加载操作所加载的内容）之前发生的所有存储操作。

&nbsp;&nbsp;&nbsp;&nbsp;为了了解这些内存顺序如何改变情况，清单 10-3 再次展示了清单 10-2，但将内存顺序替换为获取和释放。

```rust
static X: AtomicBool = AtomicBool::new(false);
static Y: AtomicBool = AtomicBool::new(false);
let t1 = spawn(|| {
 let r1 = Y.load(Ordering::Acquire);
 X.store(r1, Ordering::Release);
});
let t2 = spawn(|| {
 /*1*/ let r2 = X.load(Ordering::Acquire);
 /*2*/ Y.store(true, Ordering::Release)
});

// Listing 10-3: Listing 10-2 with Acquire/Release memory ordering
```

&nbsp;&nbsp;&nbsp;&nbsp;这些额外的限制意味着 `t2` 不再可能看到 `r2 = true`。要了解原因，请考虑示例 10-2 中导致奇怪结果的主要原因：*1* 和 *2* 的重新排序。第一个限制，即对使用 `Ordering::Release` 的存储的限制，规定我们不能将 *1* 移动到 *2* 以下，所以一切都很好！

&nbsp;&nbsp;&nbsp;&nbsp;但这些规则的作用远不止于这个简单的例子。例如，假设你实现了一个互斥锁。你想确保线程在持有锁期间运行的任何加载和存储操作都只在它实际持有锁时执行，并且对之后获取锁的任何线程都可见。这正是 `Release` 和 `Acquire` 能够实现的功能。通过执行 `Release` 存储来释放锁，执行 `Acquire` 加载来获取锁，你可以保证临界区中的加载和存储操作永远不会被移动到实际获取锁之前或之后！

> 在某些 CPU 架构上，例如 x86，获取/释放顺序由硬件保证，因此使用 `Ordering::Release` 和 `Ordering::Acquire` 相对于 `Ordering::Relaxed` 不会产生额外开销。在其他架构上则并非如此，如果您将原子操作切换到 `Relaxed` 模式，使其能够容忍较弱的内存顺序保证，您的程序可能会看到速度提升。

##### Sequentially Consistent Ordering (顺序一致排序)

顺序一致排序 (`Ordering::SeqCst`) 是我们目前能够访问的最强内存排序。它的确切保证有些难以确定，但大致来说，它不仅要求每个线程看到的结果与`Acquire/Release`一致，还要求所有线程看到的顺序彼此相同。通过与获取和释放的行为进行对比，可以最好地看出这一点。具体来说，`Acquire/Release`排序并不能保证如果两个线程 A 和 B 原子地加载由另外两个线程 X 和 Y 写入的值，A 和 B 会看到 X 相对于 Y 写入时间的一致模式。这相当抽象，因此请考虑清单 10-4 中的示例，该示例展示了`Acquire/Release`排序可能产生意外结果的情况。之后，我们将看到顺序一致排序如何避免这种特定的意外结果。

```rust
static X: AtomicBool = AtomicBool::new(false);
static Y: AtomicBool = AtomicBool::new(false);
static Z: AtomicI32 = AtomicI32::new(0);
let t1 = spawn(|| {
 X.store(true, Ordering::Release);
});
let t2 = spawn(|| {
 Y.store(true, Ordering::Release);
});
let t3 = spawn(|| {
 while (!X.load(Ordering::Acquire)) {}
 /*1*/ if (Y.load(Ordering::Acquire)) {
 Z.fetch_add(1, Ordering::Relaxed); }
});
let t4 = spawn(|| {
 while (!Y.load(Ordering::Acquire)) {}
 /*2*/ if (X.load(Ordering::Acquire)) {
 Z.fetch_add(1, Ordering::Relaxed); }
});


// 示例 10-4：获取/释放顺序的奇怪结果
```
&nbsp;&nbsp;&nbsp;&nbsp;两个线程 `t1` 和 `t2` 分别将 `X` 和 `Y` 设置为 true。线程 `t3`等待 X 变为 `true`；一旦 `X` 为 `true`，它会检查 `Y` 是否为 `true`，如果是，则将 `Z` 加 1。线程 `t4` 则等待 `Y `变为 `true`，然后检查 `X` 是否为 `true`，如果是，则将 `Z` 加 1。此时的问题是：所有线程终止后，Z 可能的值是什么？在我告诉你答案之前，请根据上一节中关于释放和获取顺序的定义，尝试自己解答一下。

&nbsp;&nbsp;&nbsp;&nbsp;首先，让我们回顾一下 `Z` 递增的条件。线程 `t3` 如果在观察到 `X` 为`true`之后，又发现 `Y` 为`true`，则递增 `Z`，而这种情况只有在 `t2` 在 `t3` 运行 *1* 之前运行的情况下才会发生。相反，线程 `t4` 如果在观察到 Y 为`true`之后，又发现 `X` 为真，则递增 `Z`，而这种情况只有在 `t1` 在 `t4` 运行 *2* 之前运行的情况下才会发生。为了简化解释，我们暂时假设每个线程一旦运行就会运行完成。

&nbsp;&nbsp;&nbsp;&nbsp;因此，逻辑上，如果线程按 1、2、3、4 的顺序运行，则 `Z` 可以递增两次——`X` 和 `Y` 都设置为 `true`，然后 `t3` 和 `t4` 运行并发现它们递增 Z 的条件都得到满足。类似地，如果线程按 1、3、2、4 的顺序运行，则 `Z` 可以简单地递增一次。这满足了 `t4` 递增 `Z` 的条件，但不满足 `t3` 的条件。然而，让 `Z` 为 0 似乎是不可能的：如果我们想阻止 `t3` 递增 `Z`，`t2` 必须在 `t3` 之后运行。由于 `t3` 只在 `t1` 之后运行，这意味着 `t2` 在 `t1` 之后运行。然而，`t4` 直到 `t2` 运行完毕后才会运行，因此 `t1` 必须在 `t4` 运行时运行并将 `X` 设置为 `true`，因此 `t4` 才会递增 `Z`。

&nbsp;&nbsp;&nbsp;&nbsp;我们无法使 `Z` 为 0，主要源于人类倾向于线性解释；这件事发生了，然后这件事发生了，然后这件事又发生了。计算机不受同样的限制，也不需要将所有事件限制在一个全局顺序中。“Release”和“Aquire”的规则中没有任何内容规定 `t3` 必须遵循与 `t4` 相同的 `t1` 和 `t2` 的执行顺序。就计算机而言，让 `t3` 观察到 `t1` 先执行，同时让 `t4` 观察到 `t2` 先执行，这都是可以的。考虑到这一点，如果 `t3` 在观察到 `X` 为`true`之后观察到 `Y` 为`false`（这意味着 `t2` 在 `t1` 之后运行），而在同一次执行中，`t4` 在观察到 `Y` 为真之后观察到 `X` 为假（这意味着 `t2` 在 `t1` 之前运行），那么这种执行是完全合理的，即使这对于我们普通人来说似乎有些不可思议。

&nbsp;&nbsp;&nbsp;&nbsp; 正如我们之前讨论过的，`Acquire/Release` 仅要求 `Ordering::Acquire` 加载变量时，必须能够看到在 `Ordering::Release` 存储加载内容之前发生的所有存储。在刚才讨论的排序中，计算机确实遵循了这一属性：`t3` 看到 `X == true`，并且确实看到了在将 `X` 设置为 `true` 之前 `t1` 执行的所有存储——没有存储。它也看到 `Y == false`，这是主线程在程序启动时存储的，因此没有任何相关的存储需要关注。同样，`t4` 看到 `Y = true`，也看到了在将 `Y` 设置为 `true` 之前 `t2` 执行的所有存储——同样，没有存储。它也看到 `X == false`，这是主线程存储的，并且没有先前的存储。没有违反任何规则，但不知何故，它看起来就是不对。

&nbsp;&nbsp;&nbsp;&nbsp;我们的直觉预期是，我们可以将线程按某种全局顺序排列，以便理解每个线程的观察和操作，但本例中的`Acquire/Release`顺序并非如此。为了更接近这种直觉预期，我们需要顺序一致性。顺序一致性要求参与原子操作的所有线程进行协调，以确保每个线程观察到的内容与某个单一的、通用的执行顺序相对应（或至少看起来与某个单一的、通用的执行顺序相对应）。这使得推理更容易，但也使其成本高昂。

&nbsp;&nbsp;&nbsp;&nbsp;标有 `Ordering::SeqCst` 的原子加载和存储操作会指示编译器采取任何额外的预防措施（例如使用特殊的 CPU 指令），以保证这些加载和存储操作的顺序一致性。
围绕这一点的具体形式相当复杂，但顺序一致性本质上确保了，如果您查看所有线程中所有相关的 `SeqCst` 操作，则可以按某种顺序排列线程执行，以便加载和存储的值都能匹配。

&nbsp;&nbsp;&nbsp;&nbsp;如果我们将清单 10-4 中的所有内存排序参数替换为 `SeqCst`，那么在所有线程退出后，`Z` 不可能像我们最初预期的那样为 0。在顺序一致性下，必须能够说
`t1` 肯定在 `t2` 之前运行，或者 `t2` 肯定在 `t1` 之前运行，因此 `t3` 和 `t4` 的执行顺序不同是不允许的，因此 `Z` 不可能为 0。













































