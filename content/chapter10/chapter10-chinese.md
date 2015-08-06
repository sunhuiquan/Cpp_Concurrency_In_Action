#第10章 高级线程管理

**本章主要内容**

- 并发相关的错误<br>
- 定位错误和代码审查<br>
- 设计多线程测试用例<br>
- 多线程代码的性能<br>

到目前为止，我们已经关注过如何书写并发代码——那些可以使用的工具，应该如何使用。不过，在软件开发中还有很重要的一部分我们没有提及：测试与调试。如果你希望阅读完本章后，就能很轻松的去调试并发代码，那么你将会失望。测试和调试并发代码是很困难的。除了对一些重要问题的思考，这里我也会展示一些技巧让测试和调试变得更容易一些。

测试和调试就像一个硬币的两面——测试就是为了找到代码中可能存在的错误，并且需要调试并修复错误。如果你自己发现了某个错误，而不是用户发现了这个错误，那就很幸运，会将这个错误的破坏力降低好几个数量级。

在我们了解测试和调试之前，需要了解一下并发代码可能会出现哪些问题。

我们先来看一下这些问题。

##10.1 与并发相关的错误类型

你可以在并发代码中发现各式各样的错误；不会集中于某个方面。不过，一些错误与使用并发直接相关，本章重点关注这样的错误。通常，与并发相关的错误通常有两大类：

- 不必要阻塞

- 条件竞争

这两大类的颗粒度很大，让我们将其分成颗粒度较小的问题。

首先，就让我们来看一下不必要的阻塞。

###10.1.1 不必要阻塞

“不必要的阻塞”是什么意思？一个线程被阻塞的时候是不能处理任何任务的，因为它在等待其他“条件”的达成。通常这些“条件”就是一个互斥量，一个条件变量，或一个future，也可能是一个I/O操作。多线程带代码的天生特性，不过这也不是在任何时候都可取的——会衍生“不必要阻塞”的问题。这就引起了下一个问题：为什么阻塞是不需要的？通常，是因为其他线程在等待该阻塞线程上的某些操作，这样那些线程就会被阻塞。在这个主题可以分成以下几个问题：

- 死锁——如你在第3章所见，在死锁的情况下，两个线程会互相等待。当线程产生死锁，应该完成的任务就会持续搁置。举个例子来说，一些线程是负责对用户界面操作的线程，在死锁的情况下，用户界面就会无响应。在另一些例子中，界面接口会保持响应，不过有些任务就无法完成，比如：查询无结果返回，或文档未打印。

- 活锁——与死锁的情况类似。不同的地方在于，这里线程不是在阻塞等待着，而是在循环中持续被检查，例如：自旋锁。在一些比较严重的情况下，其表现和死锁一样(应用不会做任何处理，停止响应)，不过CPU的使用率还是居高不下，这是因为线程还在循环中被检查，而不是被阻塞等待。在一些不太严重的情况下，因为使用随机调度，活锁问题是有解的。不过，在被“活”锁的时，将会在任务中产生一个巨大延迟，同时伴随着很高的CPU使用率。

- I/O阻塞或外部输入——当线程被外部输入所阻塞，线程也就不能做其他事情了(即使，等待输入的情况永远不会发生)。因此，被外部输入所阻塞，就会让人不太高兴，因为可能有其他线程正在等待这个线程完成其任务。

简单的介绍了一下“不必要阻塞”的组成。

那么，条件竞争呢？

###10.1.2 条件竞争

条件竞争在多线程代码中很常见——很多条件竞争表现为死锁与活锁。并非所有条件竞争都是恶性的——对独立线程相关操作的调度，决定了条件竞争发生的时间。很多条件竞争是良性的，比如：哪一个线程去处理任务队列中的下一个任务。不过，很多并发错误的引起也是因为条件竞争。特别是，条件竞争经常会产生以下几种类型的错误：

- 数据竞争——因为未同步访问一块共享内存位置，将会导致代码产生未定义行为。我在第5章已经介绍了数据竞争，也了解了C++的内存模型。数据竞争通常发生在，错误的使用原子操作来同步线程的时候；或没被互斥量所保护的共享数据位置。

- 破坏不变量——主要表现为悬空指针(因为其他线程已经将要访问的数据删除了)，随机存储错误(因为局部更行，导致线程读取了不一样的数据)，以及双重释放(比如：当两个线程对同一个队列同时执行pop操作，想要删除同一个关联数据)，等等。不变量被破坏可以看作为“基于数值”的问题。当独立线程需要以一定顺序执行某些操作时，错误的同步会导致条件竞争，比如：顺序被破坏。

- 生命周期问题——虽然这类问题也能归结为破坏了不变量，不过这里将其作为一个单独的类别给出。这里的问题是，线程会访问不存在的数据，这可能是因为数据被删除或销毁了，或者转移到其他对象中去了。生命周期问题，通常是在一个线程引用了局部变量，在线程还没有完成前，局部变量的“死期”就已经到了，不过这个问题并不止存在这种情况下。当你手动迪欧用join()，为了等待线程完成工作，你需要保证当有异常抛出的时候，join()还会等待其他未完成工作的线程。这是线程中基本异常安全的应用。

恶性条件竞争就如同一个杀手。死锁和活锁会表现为，应用挂起和反应迟钝，或超长时间完成任务。当一个线程产生死锁或活锁，可以用调试器附着到该线程上进行调试。条件竞争，破坏不变量和生命周期问题，表现都是代码可见的(比如，随机崩溃或错误输出)——可能重写了系统部分的内存会用方式(不会改太多)。其中，可能是因为执行时间的问题，问题可能无法定位到具体的位置。这是共享内存系统的诅咒——需要通过线程尝试限制可以访问的数据，并且还要确定同步被正确的使用，在这个应用用任何线程都可以复写被其他线程访问的数据。

现在我们已经了解了这两大类中都有哪些具体问题了。

下面就让我们来看下，如何在你的代码中定位和修复这些问题。

##10.2 定位并发错误的技术

在之前的章节我们了解了与并发相关的错误类型，以及这些问题是如何在你的代码中体现出来的。这些信息可以帮助我们来判断，在我们的代码中，是否存在有隐藏的错误。

可能最简单直接的就是直接看代码。虽然这可能看起来比较明显，但是要彻底的修复问题，却是很难的。读刚写完的代码，要比读已经存在的代码简单的多。同理，当在审查别人写好的代码是，给出一个通读结果是很容易的，比如：与你自己的代码标准作对比，以及高亮标出显而易见的问题。为什么要花时间来仔细梳理代码？想想之前提到的并发问题——也要考虑非并发问题。(也可以在很久以后做这件事。不过，最后bug依旧存在)我们可以在审阅代码的时候，考虑一些具体的事情，并发现问题。

即使已经很对代码进行了很详细的审阅，依旧会错过一些bug，并且你需要在一些情况下确定代码的确做了对应的工作。因此，在测试多线程代码时，我们会继续介绍一些代码审阅的技巧。

###10.2.1 代码审阅——发现潜在的错误

之前提到，当审阅多线程代码时，重点要检查与并发相关的错误。如果可能，可以让同事/同伴来审阅。因为这不是他们写的代码，他们将会考虑这段代码是怎么工作的，这就可能会覆盖到一些你没有想到的情况，从而找出一些潜在的错误。审阅人员需要有时间去做审阅——并非在休闲时间简单的扫一眼。大多数并发问题需要的不仅仅是一次快速浏览——通常需要在找到问题上花费很多时间。

如果你让你的同时来审阅代码，他肯定对你的代码不是很熟悉。因此，他/她会从不同的角度来看你的代码，然后指出你没有注意的事情。如果你的同时都没有空，你可以叫你的朋友，或传到网络上，让网友审阅(注意，别传一些机密代码上去)。实在没有人审阅你的代码，不要着急——你还可以做很多事情。对于初学者，可以将代码放置一段时间——先去做应用的另外的部分，或是阅读一本书籍，亦或出去溜达溜达。当有了一定的休息之后，当再集中注意力做某些事情，你的潜意识会考虑很多问题了。同样，当你做完其他事情，回头再看这段代码可能就会不熟悉了——你可能会从另一个角度来看你自己所写的代码。

另一种让其他人来检查你的代码就是：自己做。就是向别人详细的介绍你所写的功能。其可能并不是一个真正的人——可能要像一只玩具熊或一只橡皮鸡来进行解释，并且我个人觉得写一些比较详细的笔记是非常有益的。如你所释，考虑每一行过后，会有什么事情发生，有哪些数据被访问了，等等。问自己关于代码的问题，并且向自己解释这些问题。我觉得这是种非常有效的技巧——通过自问自答，对每个问题认真考虑，这些问题往往都会揭示一些问题。这些问题也会有益于任何形式的代码审阅。

**审阅多线程代码需要考虑的问题**

在审阅代码的时候考虑和代码相关的问题，有利于找出代码中的问题。这些问题审阅者需要在代码中找到相应的回答，或找到相关的错误。我认为下面这些问题必须要问(当然，不是一个综合性的列表)。你也可以找一些其他问题来帮助你找到代码的问题。这里，列一下我的清单：

- 并发访问时，那些数据需要保护？

- 如何确定访问数据收到了保护？

- 是否会有多个线程同时访问这段代码？

- 这个线程获取了哪个互斥量？

- 其他线程可能获取哪些互斥量？

- 两个线程间的操作是否有依赖关系？如何满足这种要求？

- 这个线程加载的数据还是合法数据吗？数据是否被其他线程修改过？

- 当假设其他线程可以对数据进行修改，这将意味着什么？并且，怎么确保这样的事情不会发生？

最后一个问题，我最喜欢，因为它让我着实的去考虑线程之间的关系。通过假设一个bug和一行代码相关联，你就可以扮演侦探来追踪一下bug出现的原因。为了让你自己确定代码里面没有bug，需要考虑代码运行的各种情况。在数据被多个互斥量所保护的时候，这种方式尤其有用，比如：使用线程安全队列(第6章)，可以对队头和队尾使用独立的互斥量:就是为了确保，在持有一个互斥量的时候，访问是安全的，这里必须确保持有其他互斥量的线程不能同时访问同一元素。这里也需要特别关注，对公共数据的显式处理，或使用一个指针或引用的方式让其他代码来获取数据。

倒数第二个问题也很重要，因为这是很容易产生错误的地方：当先释放，然后获取一个互斥量时，必须假设其他线程可能会修改共享数据。虽然这很明显，但是当互斥锁不是立即可见——可能因为其内部有一个对象——就会不知不觉的掉入陷阱中。在第6章，已经了解到这种情况是怎么引起条件竞争，以及是如何给细粒度线程安全数据结构带来bug的。不过，对于非线程安全栈将top()和pop()操作分开是有意义的，多线程会并发的访问这个栈，问题会马上发生，因为在两个操作的调用间，内部互斥锁已经被释放，并且另一个线程对栈进行了修改。如在第6章所见，解决方案就是讲两个操作合并，这样就能用同一个所来对操作的执行进行保护，因此，就消除了条件竞争的问题。

OK，你已经审阅过你的代码了(或者让别人看过)。现在，你确信代码没有问题。需要用味觉来证明，你现在吃的东西——怎么测试才能确认或否认你的代码没有bug呢？

###10.2.2 通过测试定位并发相关的错误

在写单线程应用时，如果时间充足，测试起来相对简单。原则上，可以设置各种可能的输入(或设置成感兴趣的情况)，然后执行应用。如果应用行为和输出正确，那么就能判断其能对给定输入集给出正确的答案。检查错误状态(比如：处理磁盘满载错误)就会比处理输入测试复杂的多，不过原理是一样的——设置初始条件，然后让程序执行。

测试多线程代码的难度就要比单线程大好几个数量级，因为不确定是都对线程进行精确调度，且会以很多形式进行运行。因此，即使使用测试单线程的输入数据，如果条件变量潜藏在代码中，那么代码的结果可能会时对时错。只是因为条件变量可能会在有些时候，等待其他事情，从而导致结果错误或正确。

因为与并发相关的bug相当难判断，所以需要在设计并发代码时格外谨慎。在设计的时候，每小段代码都需要进行测试，以保证没有问题，这样才能在测试出现问题的时候，剔除并发相关的bug——例如，对队列的推送和弹出，分别进行并发的测试，就要好于，直接使用队列，测试其中全部功能。这种想法就能帮你在设计代码的时候，考虑什么样的代码可以用来测试正在设计的这个结构——在本章后续章节中看到与设计测试代码相关的内容。

测试的目的就是为了消除与并发相关的问题。如果在单线程测试的时候，遇到了问题，那这个问题就是普通的bug，而非并发相关的bug。当问题发生在未测试区域(*in the wild*)，也就是没有在测试范围之内，像这样的情况就要特别注意。bug出现在应用的多线程部分，并不意味着其是一个多线程相关的bug。当使用线程池管理某一级并发的时候，通常这里会有一个可配置的参数，可以用来指定工作线程的数量。当手动管理线程时，就需要将代码改成单线程的方式进行测试。不管哪种方式，将多线程简化为单线程后，就能将与多线程相关的bug排除掉。反过来说，当问题在单芯系统中消失(即使还是以多线程方式)，不过问题在多芯系统或多核系统中出现，就能确定你被多线程相关的bug坑了，可能是条件变量的问题，还有可能是同步或内存序的问题(太坑了！)。

测试并发代码的代码很多，不过测试过的代码结构就没那么多了；对结构的测试也很重要，就像对环境的测试一样。如果你依旧将测试并发队列当做一个测试例，你就需要考虑这些情况：

- 使用单线程调用push()或pop()，来确定在一般情况下队列工作正常

- 在其他线程调用pop()时，使用另一线程在空队列上调用push()

- 在空队列上，以多线程的方式调用push()

- 在满载队列上，以多线程的方式调用push()

- 在空队列上，以多线程的方式调用pop()

- 在满载队列上，以多线程的方式调用pop() 

- 在非满载队列上(任务数量小于线程数量)，以多线程的方式调用pop()

- 当一线程在空队列上调用pop()的同时，以多线程的方式调用push()

- 当一线程在满载队列上调用pop()的同时，以多线程的方式调用push()

- 当多线程在空队列上调用pop()的同时，以多线程方式调用push()

- 当多线程在满载队列上调用pop()的同时，以多线程方式调用push()

这是我所能想到的场景，可能还有更多，之后你需要考虑测试环境的因素：

- “多线程”是有多少个线程(3个，4个，还是1024个？)

- 系统中是否有足够的处理器，能让每个线程运行在属于自己的处理器上

- 测试需要运行在哪种处理器架构上

- 在测试中如何对“同时”进行合理的安排

这些因素的考虑会具体到一些特殊情况。四个因素都需要考虑，第一个和最后一个会影响测试结构本身(在10.2.5节中会介绍)，另外两个就和实际的物理测试环境相关了。与使用线程数量相关的测试代码，需要独立测试，不过可通过很多结构化测试，获得最合适的调度方式。在了解这些技巧前，先来了解一下如何让你的应用更容易进行测试。

###10.2.3 可测试性设计

测试多线程代码很困难，所以你需要将其变得简单一些。很重要的一件事就是，在设计代码时，考虑其的可测试性。可测试的单线程代码设计已经说烂了，而且其中许多建议，在现在依旧适用。通常，如果代码满足一下几点，就很容易进行测试：

- 每个函数和类的关系都很清楚。

- 函数短小精悍。

- 测试用例可以完全控制被测试代码周边的环境。

- 执行特定操作的代码应该集中测试，而非分布式测试。

- 需要在完成编写后，考虑如何进行测试。

以上这些在多线程代码中依旧适用。实际上，我会认为对多线程代码的可测试性的关注，要比单线程的更多，因为多线程的情况更加复杂。最后一个因素尤为重要：即使你不会在写完代码后，去写测试用例，这点也是一个很好的建议，能让你在写代码之前，想想应该怎么去测试它——用什么作为输入，什么情况看起来会让结果变得糟糕，以及如何激发代码中潜在的问题，等等。

并发代码测试的一种最好的方式：去并发。如果代码在线程间的通讯路径上出现问，那么就可以让一个已通讯的单线程进行执行，这样会大大减小问题的难度。在对数据进行访问的应用进行测试时，就可以使用单线程的方式进行。这样在线程通讯和对特定数据块进行访问时，就只有一个线程，就达到了更容易测试的目的。

例如，当应用设计为一个多线程状态机时，可以将其分为若干块。将每个逻辑状态分开，就能保证对于每个可能的输入事件，转换和其他操作的结果是正确的；这就是使用了单线程测试的技巧，测试用例提供的输入事件将来自于其他线程。之后，核心状态机和消息路的代码，就能保证时间能以正确的顺序，正确的传递给可单独测试的线程上，不过对于多并发线程，需要为测试专门设计简单的逻辑状态。

或者，如果能将代码分割成多个块(比如：读共享数据/变换数据/更新共享数据)，就能使用单线程来测试变换数据的部分。麻烦的多线程测试问题，转换成单线程测试读和更新共享数据，这样就简单了许多。

有一件事需要小心，就是库会用其内部变量存储状态，当多线程使用同一库中的函数，这个状态就会被共享。这的确是一个问题，不过这个问题不会马上出现在访问共享数据的代码中。不过，随着你对这个库的熟悉，就会清楚这样的情况会在什么时候出现。之后，可以适当的加一些保护和同步，或使用B计划——能让多线程安全并发访问的功能。

将并发代码设计的有更好的测试性，要比以代码分块的方式处理并发相关的问题好很多。当然，还要注意对非线程安全库的调用。在10.2.1节中那些问题，也需要在审阅自己代码的时候，格外注意。虽然这些问题和测试与可测试性没有直接的关系，但带上“测试帽子”时候，就要考虑这些问题了，并且还要考虑如何测试已写好的代码，这就会影响设计方向的选择，也就会让测试做的更加容易一些。

我们已经了解了如何能让测试变得更加简单，以及将代码分成一些“并发”块(比如，线程安全容器，或事件逻辑状态机)以“单线程”的形式(可能还通过并发块，和其他线程进行互动)进行测试。

下面就让我们了解一下测试多线程代码的技术。

###10.2.4 多线程测试技术

想通过一些技巧写一些较短的代码，来对函数进行测试。比如，如何处理调度序列上的bug？

这里的确有几个方法能进行测试，让我们从蛮力测试(或称压力测试)开始。

**蛮力测试**

当代码有问题的时候，就要求蛮力测试一定能看到这个错误。这通常意味着代码要运行很多遍，可能会有很多线程在同一时间运行。要是有bug出现，只能是被特殊安排线程的时候；代码运行次数的增加，就意味着bug出现的次数会增多。当有几次代码测试通过，你可能会对代码的正确性有一些信心。如果连续运行10次，并且每次都通过，你就会更有信心。如果你运行十亿次，并且每次都通过了，那么你就会认为这段代码没有问题了。

自信的来源是每次测试的结果。如果你的测试粒度很细，就像测试之前的线程安全队列，那么蛮力测试会让你对这段代码持有高度的自信。另一方面，当测试对象体积较大的时候，调度序列将会很长，即使运行了十亿次测试用例，也不让你对这段代码产生什么信心。

蛮力测试的确定就是，可能会误导你。如果写出来的测试用例就为了不让有问题的情况发生，那么怎么运行，测试都不会失败，可能会因环境的原因，出现几次失败的情况。最糟糕的情况就是，问题不会出现在你的测试系统中，因为在某些特殊的系统中，这段代码就会出现问题。除非你代码运行在与测试机系统相同的系统中，不过特殊的硬件和操作系统的因素结合起来，可能就会让运行环境与测试环境有所不同，问题可能就会随之出现了。

这里有一个经典的案例，就是在单处理器系统上测试多线程应用。因为每个线程都在同一个处理器上运行，任何事情都是串行的，并且还有很多条件竞争和乒乓缓存问题，这些问题可能在真正的多处理器系统中，根本不会出现。这并非唯一的变数；不同处理器架构提供不同的的同步和内存序机制。比如，在x86和x86-64架构上，原子加载操作通常是相同的，无论是使用memory_order_relaxed，还是memory_order_seq_cst(详见5.3.3节)。这就意味着在x86架构上使用松散内存序没有问题，但在有更精细的内存序指令集的架构(比如：SPARC)下，这样使用可能会产生错误。

如果你希望你的应用能跨平台使用，就要在相关的平台上进行测试。这就是我把处理器架构也列在测试需要考虑的清单中的原因(详见10.2.2)。

要想避免误导的产生，关键点在于成功的蛮力测试。这就需要进行仔细考虑和设计，这就不仅仅是选择相关单元测试就可以了，还要遵守测试系统设计准则，以及选定测试环境。需要保证代码分支被尽可能的测试到，且尽可能多的测试线程间的互相作用。不仅仅是这些，你还需要知道哪部分被测试覆盖到，哪些没有覆盖。

虽然，蛮力测试能够给你一些信心，不过其不保证能找到所有的问题。如果有时间将下面的技术应用到你的代码或软件中，就能保证所有的问题都能被找到。

我称其为——组合仿真测试。

**组合仿真测试**

这个名字比较口语化，我需要解释一下这个测试是什么意思。这个测试就是使用一种特殊的软件，用来模拟代码运行的真实情况。你应该知道这种软件，能让你在一台物理机上运行多个虚拟环境，或系统环境，而硬件环境则由监控软件来完成。除了环境是模拟的以外，模拟软件会记录对数据序列访问，上锁，以及对每个线程的原子操作。然后使用C++内存模型的规则，重复的运行，从而识别条件竞争和死锁。

虽然这种组合测试可以保证所有与系统相关的问题都会被找到，不过过于零碎的程序将会在这种测试中耗费太长时间，因为组合数目和执行的操作数量将会随线程的增多，呈指数增长态势。这个测试最好留给需要细粒度测试的代码段，而非整个应用。另一个缺点就是，你的代码对操作的处理，往往会依赖与模拟软件的可用性。

所以，测试需要在正常情况下，运行很多次，不过这样可能会错过问题；并且，也可以在一些特殊情况下，运行多次，不过这样更像是为了验证某些问题。还有其他的测试选项吗？

第三个选项就是使用一个库，在运行测试的时候，检查代码中的问题。

**使用专用库对代码进行测试**

虽然这个选择不会像组合仿真的方式，提供彻底的检查，不过可以通过特别实现的库(使用同步原语)来发现一些问题，比如：互斥量，锁和条件变量。例如，在访问某块公共数据的时候，就需要将指定的互斥量上锁。在数据被访问后，发现一些互斥量已经上锁，就需要确定相关的互斥量是否被访问线程锁住；如果没有，测试库将报告这个错误。当需要测试库对某块代码进行检查时，可以对对应的共享数据进行标记。

当不止一个互斥量同时被一个线程持有，测试库也会对锁的序列进行记录。如果有其他线程以不同的顺序进行上锁，即使在运行的时候，测试用例没有发生死锁，这测试库都会将这个行为记录为“有潜在死锁”可能。

当测试多线程代码的时候，另一种库可能会用到，以线程原语实现的库，比如：互斥量和条件变量；当多线程代码在等待，或是被条件变量通过notify_one()提醒的某个线程，测试作者可以通过线程，获取到锁。这就可以让你来安排一些特殊的情况，以验证代码是否会在这些特定的环境下产生期望的结果。

C++标准库实现中，某些测试工具已经存在于标准库中，没有实现的测试工具，可以基于标准库进行实现。

了解完各种运行测试代码的方式，将让我们来了解一下，如何以你想要的调度方式来构建代码。

###10.2.5 构建多线程测试代码

在10.2.2节中提过，需要找一种合适的调度方式来处理测试中“同时”的部分。现在就是解决这个问题的时间了。

重点在于，你需要安排一系列线程，在特定时间，同时去执行指定的代码段。最简单的情况：两个线程的情况，就很容易扩展到多个线程。

第一步，你需要知道每个测试的不同之处：

- 环境布置代码，必须首先执行

- 线程特定设置代码，需要在每个线程上执行

- 线程上执行的代码，需要有并发性

- 在并发执行结束后，后续代码需要对代码的状态进行断言检查

这几条后面再解释，先让我们考虑一下10.2.2节中的一个特殊的情况：一个线程在空队列上调用push()，同时让其他线程调用pop()。

通常，布置环境的代码比较简单：创建队列即可。线程在执行pop()的时候，没有线程特定设置代码。线程设置代码是在执行push()操作的线程上进行的，其依赖与队列的接口和对象的存储类型。如果存储的对象需要很大的开销才能构建，或必须在堆上分配的对象，那么你最好在线程特定设置代码中进行构建或分配，这样就不会影响到测试结果。另一方面，如果队列中只存简单的int类型对象，在构建int对象时就不会有太多额外的开销。实际上，已测试代码相对简单——一个线程调用push()，另一个线程调用pop()——那么，“完成后”的代码到底是什么样子呢？

在这个例子中，pop()具体做的事情，会直接影响“完成后”代码。如果有数据块，那么返回的肯定就是数据了，那么push()操作就成功的向队列中推送了一块数据，并在在数据返回后，队列依旧是空的。如果pop()没有返回数据块，也就是队列为空的情况下，操作也能执行，这样就需要两个方向的测试：要不就是pop()返回push()推送到队列中的数据块，之后队列依旧为空；要不pop()会示意队列中没有元素，但同时push()向队列推送了一个数据块。这两种情况都是真实存在的；你需要避免的情况是：pop()示意队列中没有数据的同时，队列还是空的，或pop()返回数据块的同时，队列中还有数据块。为了简化测试，可以假设pop()是可以阻塞的。那么最终代码中，需要用断言判断一下弹出的数据与推入的数据，并且还要判断队列为空。

现在，已经了解了各个代码块，这时就需要保证所有事情要在计划中进行。一种方式是使用一组`std::promise` 来表示一切准备就绪。每个线程使用一个promise来表示是否准备好，然后从一个第三方`std::promise`等待(复制)一个`std::shared_future`;主线曾会等待每个线程上的promise设置后，才按下“开始”键。这就能保证每个线程能够同时开始，并且在准备代码执行完成后，并发代码就可以开始执行了；任何的线程特定设置都需要在设置线程的promise前完成。最终，主线程会等待所有线程完成，并且检查其最终状态。还需要格外关系异常，并且保证不会将线程未准备好的情况下，就按下“开始”键，这样的话未准备好的线程就不会运行。

下面的代码，就构建了这样的测试。

清单10.1 对一个队列并发调用push()和pop()的测试用例
```c++
void test_concurrent_push_and_pop_on_empty_queue()
{
  threadsafe_queue<int> q;  // 1
  
  std::promise<void> go,push_ready,pop_ready;  // 2
  std::shared_future<void> ready(go.get_future());  // 3
  
  std::future<void> push_done;  // 4
  std::future<int> pop_done;
 
  try
  {
    push_done=std::async(std::launch::async,  // 5
                         [&q,ready,&push_ready]()
                         {
                           push_ready.set_value();
                           ready.wait();
                           q.push(42);
                         }
      );
    pop_done=std::async(std::launch::async,  // 6
                        [&q,ready,&pop_ready]()
                        {
                          pop_ready.set_value();
                          ready.wait();
                          return q.pop();  // 7
                        }
      );
    push_ready.get_future().wait();  // 8
    pop_ready.get_future().wait();
    go.set_value();  // 9

    push_done.get();  // 10
    assert(pop_done.get()==42);  // 11
    assert(q.empty());
  }
  catch(...)
  {
    go.set_value();  // 12
    throw;
  }
}
```

首先，在环境设置代码中创建了空队列①。然后，为准备状态创建promise对象②，并且为go信号获取一个`std::shared_future`对象③。再后，创建了future用来表示线程是否结束④。这些都需要放在try块外面，之后，在设置go信号时抛出异常，就不需要等待其他城市线程完成任务了(这将会产生死锁——如果测试代码产生死锁，测试代码就是不理想的代码)。

在try块中，可以启动线程⑤⑥——使用`std::launch::async`来保证每个任务在自己的线程上完成。注意，使用`std::async`会让你任务更容易成为线程安全的任务，这里不用普通`std::thread`，是因为其析构函数会对future进行线程汇入。lambda函数会捕捉指定的任务(会在队列中引用)，并且为promise准备相关的信号，同时对从go中获取的ready做一份拷贝。

如之前所说，每个任务集都有自己的ready信号，并且会在执行测试代码前，等待所有的ready信号。而，主线程不同——等待所有线程的信号前⑧，提示所有线程可以开始进行测试了⑨。

最终，异步调用等待线程完成后⑩⑪，主线程会从中获取future，再调用get()成员函数获取结果，最后对结果进行检查。注意，这里pop操作通过future返回检索值⑦，所以这里能获取最终的结果⑪。

当有异常抛出，需要通过对go信号的设置来避免悬空指针的产生，然后再重新抛出异常⑫。future与之后声明的任务相对应④，所以future将会被首先销毁，如果future都没有就绪，析构函数将会等待相关任务完成后执行操作。

虽然，这看起来就像是使用测试模板，对两个调用进行测试。使用类似的东西是必要的，这样会便于测试的进行。例如，实际启动一个线程就是一个很耗时的过程，所以如果没有线程在等待go信号时，推送线程可能会在弹出线程开始之前，就已经完成了；这样就失去了测试的作用。以这种方式使用future，就是为了保证线程都在运行，并且阻塞在同一个future上。future去阻塞，将会让所有线程运行起来。当你熟悉了这个结构，其就能相对简单的以同样的模式创建新的测试用例。为了测试两个以上的线程，可以很容易的对这种模式进行扩展。

目前，我们已经了解了多线程代码的正确性。虽然这是最最重要的问题，但是其不是我们做测试的唯一原因：多线程性能的测试同样重要，下面就让我们来了解一下性能测试。
 
###10.2.6 测试多线程代码性能

选择以并发的方式开发应用，就是为了能够使用日益增长的处理器数量；通过处理器数量的增加，来提升应用的执行效率。因此，确定性能是否有真正的提高就很重要了(就像你做的其他优化一样)。

并发效率中有个特别的问题，就是可扩展性——你希望代码能很快的运行24次，或在24芯的机器上对数据进行24(或更多)次处理，或其他等价情况。你不会希望，你的代码运行两次的数据和在双芯机器上执行一样快的同时，在24芯的机器上会很慢。如你在8.4.2节中所见，当有很重要的代码以单线程方式运行时，就会限制性能的提高。因此，在做测试之前，回顾一下代码的设计结构是很有必要的；这样你能判断，你的代码在24芯的机器上时，性能会不会提高24倍，或是因为有串行部分的存在，最大的加速比只有3。

如你在之前章节所见，在对数据访问的时候，处理器之间会有竞争，这就会对性能有很大的影响。需要合理的权衡性能和处理器的数量，处理器数量太少，会等待很久；处理器过多，又会因为竞争的原因，等待很久。

因此，在对应的系统上，通过不同的配置检查多线程的性能就很有必要，这样你可以得到一张性能伸缩图。最起码，(如果条件允许)你应该在一个单处理器的系统上和一个有着多处理核芯的系统上进行测试

##10.3 小结

本章中我们了解了各种与并发相关的bug，这些bug我们可能会在写代码的时候遇到，从死锁和活锁，再到数据竞争和其他恶性条件竞争。我们也是用了一些技术来定位bug。同样，也讲了在做代码审阅的时候需做哪些思考，写可测试代码的指导意见，以及如何为并发代码构造测试用例。最终，我们还了解了一些实用的工具，这些工具会对测试有帮助。