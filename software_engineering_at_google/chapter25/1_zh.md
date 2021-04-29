# 第25章 计算即服务

作者：Onufry Wojtaszczyk

编辑：Lisa Carey

我并不试图了解计算机。我试着去理解程序。
Barbara Liskov

在完成了编写代码的艰苦工作后，你需要一些硬件来运行它。因此，你要去购买或租用这些硬件。这在本质上就是计算即服务（CaaS），其中 "计算 "是实际运行你的程序所需的计算能力的简称。

本章是关于这个简单的概念--只要给我硬件来运行我的东西[1]--如何映射成一个系统，随着你的组织的发展和成长而生存和扩展。这一章有点长，因为这个话题很复杂，分为四个部分。

* "驯服计算环境 "涵盖了谷歌是如何得出这个问题的解决方案的，并解释了CaaS的一些关键概念。

* "为托管计算编写软件 "展示了托管计算解决方案是如何影响工程师编写软件的。我们认为，"牛，而不是宠物"/灵活的调度模式是谷歌在过去15年成功的根本，也是软件工程师工具箱中的一个重要工具。

* "CaaS随时间和规模的变化 "深入探讨了谷歌学到的一些经验，即随着组织的成长和演变，关于计算架构的各种选择是如何发挥的。

* 最后，"选择计算服务 "主要是献给那些将决定在其组织中使用何种计算服务的工程师。

## 驯服计算环境

谷歌的内部Borg系统[2]是今天许多CaaS架构（如Kubernetes或Mesos）的先驱。为了更好地理解这种服务的特定方面如何满足一个不断增长和发展的组织的需要，我们将追溯Borg的演变以及谷歌工程师为驯服计算环境所做的努力。

### 劳作的自动化

想象一下，在世纪之交的时候，你是一个大学的学生。如果你想部署一些新的、漂亮的代码，你会把代码SFTP到大学计算机实验室的一台机器上，SSH进入机器，编译并运行代码。这是一个简单而诱人的解决方案，但随着时间的推移和规模的扩大，它遇到了相当多的问题。然而，由于这大致是许多项目开始时的情况，多个组织最终采用的流程在某种程度上是这个系统的精简演变，至少对于某些任务来说是这样的--机器的数量增加了（所以你SFTP和SSH进入其中许多机器），但基础技术仍然存在。例如，在2002年，谷歌最资深的工程师之一杰夫-迪安（Jeff Dean）就运行一个自动数据处理任务作为发布过程的一部分写了以下内容。

  [运行该任务]是一个后勤、耗时的恶梦。目前，它需要获得一个50多台机器的列表，在这50多台机器中的每一台上启动一个进程，并在这50多台机器中的每一台上监控其进度。如果其中一台机器死亡，没有支持自动将计算迁移到另一台机器上，监测工作的进展是以临时的方式进行的[......]此外，由于进程可以相互干扰，有一个复杂的、由人类实施的 "注册 "文件来节制机器的使用，这导致了不太理想的调度，以及对稀缺的机器资源的更多争论

这是谷歌努力驯服计算环境的早期导火索，这很好地解释了天真的解决方案如何在更大的规模下变得不可维护。

### 简单的自动化

有一些简单的事情，一个组织可以做，以减轻一些痛苦。将二进制文件部署到50多台机器中的每一台，并在那里启动，这个过程很容易通过一个shell脚本来实现自动化，然后--如果这是一个可重复使用的解决方案--通过一个更强大的代码，用一种更容易维护的语言来执行并行部署（特别是由于 "50多台 "可能会随着时间而增加）。

更有趣的是，对每台机器的监控也可以自动化。最初，负责这个过程的人希望知道（并能够进行干预），如果其中一个副本出了问题。这意味着从进程中输出一些监控指标（如 "进程是活的 "和 "处理的文件数"）--让它写到一个共享存储中，或调用一个监控服务，在那里他们可以一目了然地看到反常情况。目前该领域的开源解决方案是，例如，在Graphana或Prometheus等监控工具中设置一个仪表盘。

如果检测到异常，通常的缓解策略是通过SSH进入机器，杀死进程（如果它还活着），然后重新启动它。这很繁琐，可能很容易出错（要确保你连接到正确的机器，并确保杀死正确的进程），而且可以自动化。

* 与其手动监测故障，不如在机器上使用一个代理，检测异常情况（比如 "该进程在过去5分钟内没有报告它是活的 "或 "该进程在过去10分钟内没有处理任何文件"），并在检测到异常情况时杀死该进程。

* 与其在死亡后登录到机器上再次启动进程，不如将整个执行过程包裹在一个 `“while true; do run && break; done”`的shell脚本中。

在云计算世界中，相当于设置一个自动修复策略（在虚拟机或容器无法通过健康检查后，杀死并重新创建）。

这些相对简单的改进解决了前面描述的Jeff Dean问题的一部分，但不是全部；人类实施的节流，以及移动到一个新机器，需要更多的解决方案。

### 自动调度

下一步自然是将机器分配自动化。这需要第一个真正的 "服务"，最终将发展成为 "计算即服务"。也就是说，要实现自动化调度，我们需要一个中央服务，知道它可用的机器的完整列表，并能按需挑选一些未被占用的机器，自动将你的二进制文件部署到这些机器上。这样就不需要手工维护的 "注册 "文件了，而是把维护机器列表的工作交给计算机。这个系统很容易让人联想到早期的时间共享架构。

这个想法的一个自然延伸是将这种调度与对机器故障的反应结合起来。通过扫描机器日志中提示健康状况不佳的表达方式（例如，大量磁盘读取错误），我们可以识别出坏掉的机器，（向人类）提示需要修理这些机器，并避免在此期间在这些机器上安排任何工作。将消除劳动的范围进一步扩大，自动化可以在人类参与之前先尝试一些修复方法，比如重启机器，希望错误的东西消失，或者运行一个自动磁盘扫描。

杰夫说的最后一个抱怨是，如果运行的机器坏了，需要人把计算迁移到另一台机器上。这里的解决方案很简单：因为我们已经有了调度自动化和检测机器故障的能力，我们可以简单地让调度员分配一个新的机器，并在这个新机器上重新开始工作，放弃旧的机器。这样做的信号可能来自于机器自省守护程序或对单个进程的监控。

所有这些改进都系统地处理了组织规模的增长。当机群是一台机器时，SFTP和SSH是完美的解决方案，但在数百或数千台机器的规模下，需要自动化来接管。我们开始引用的这句话来自于2002年 "全球工作队列 "的设计文件，这是一个早期的CaaS内部解决方案，用于谷歌的一些工作负载。

### 容器化和多租户

到目前为止，我们隐含地假设了机器和运行在机器上的程序之间是一对一的映射。就计算资源（RAM、CPU）的消耗而言，这在很多方面都是非常低效的。

* 很可能有许多不同类型的作业（具有不同的资源需求），而不是机器类型（具有不同的资源可用性），所以许多作业需要使用相同的机器类型（需要为其中最大的机器提供资源）。

* 机器的部署需要很长的时间，而程序的资源需求则随着时间的推移而增长。如果获得新的、更大的机器需要花费你的组织几个月的时间，你还需要使它们足够大，以适应在配置新机器所需时间内资源需求的预期增长，这就导致了浪费，因为新机器没有被充分利用起来。[3]

* 即使新机器到了，你还有旧机器（扔掉它们很可能是浪费），所以你必须管理一个不适应你需求的异质机群。

自然的解决方案是为每个程序指定其资源需求（在CPU、内存、磁盘空间方面），然后要求调度器将程序的副本打包到可用的机器池中。

#### 我的邻居的狗在我的内存中吠叫

如果每个人都能很好地发挥，上述的解决方案就能完美地工作。然而，如果我在配置中指定我的数据处理管道的每个副本将消耗一个CPU和200MB的内存，然后由于一个错误，或有机增长，它开始消耗更多，它被安排到的机器将耗尽资源。在CPU的情况下，这将导致相邻的服务作业出现延迟；在RAM的情况下，它将导致内核的内存外杀，或者由于磁盘交换而出现可怕的延迟。[4]

同一台计算机上的两个程序也会在其他方面发生严重的互动。许多程序都希望它们的依赖关系能以某种特定的版本安装在一台机器上--而这些可能与其他程序的版本要求相冲突。一个程序可能希望某些系统范围内的资源（想想/tmp）可以供自己单独使用。安全是一个问题--一个程序可能正在处理敏感数据，需要确保同一台机器上的其他程序不能访问它。

因此，多租户计算服务必须提供一定程度的隔离，保证一个进程能够安全地进行而不被机器的其他租户干扰。

隔离的一个经典解决方案是使用虚拟机（VM）。然而，这些虚拟机在资源使用（它们需要资源在里面运行一个完整的操作系统）和启动时间（同样，它们需要启动一个完整的操作系统）方面有很大的开销[5]。这使得它们成为一个不那么完美的解决方案，用于批量作业的容器化，而这些作业的资源占用少，运行时间短。这导致谷歌在2003年设计Borg的工程师们寻找不同的解决方案，最终产生了容器--一种基于cgroups（谷歌工程师在2007年将其纳入Linux内核）和chroot jails、bind mounts和/或union/overlay文件系统进行文件系统隔离的轻型机制。开源容器的实现包括Docker和LMCTFY。

随着时间的推移和组织的发展，越来越多的潜在隔离故障被发现。举个具体的例子，2011年，从事Borg工作的工程师发现，进程ID空间（默认设置为32,000个PID）的耗尽正在成为一种隔离故障，不得不引入对单个副本可产生的进程/线程总数的限制。我们将在本章后面详细介绍这个例子。

#### 权限化和自动扩展

2006年的Borg根据工程师在配置中提供的参数安排工作，如副本数量和资源需求。

从远处看这个问题，要求人类确定资源需求数字的想法有些缺陷：这些数字不是人类每天都在互动的。因此，随着时间的推移，这些配置参数本身就成为低效率的来源。工程师需要花时间在最初的服务启动时确定这些参数，而随着你的组织积累越来越多的服务，确定这些参数的成本也在增加。此外，随着时间的推移，程序的发展（可能会增长），但配置参数并没有跟上。这最终导致了故障的发生--事实证明，随着时间的推移，新版本的资源需求吞噬了留给意外高峰或故障的松弛，而当这种高峰或故障实际发生时，剩余的松弛被证明是不够的。

自然的解决方案是自动设置这些参数。不幸的是，这被证明是出乎意料的棘手。作为一个例子，谷歌最近才达到一个点，即整个博格机队一半以上的资源使用量是由自动化合理化决定的。也就是说，尽管这只是一半的使用量，但它是配置中较大的一部分，这意味着大多数工程师不需要关心他们容器大小的繁琐和容易出错的负担。我们认为这是对 "简单的事情应该是容易的，复杂的事情应该是可能的 "这一理念的成功应用--仅仅因为Borg工作负载的某些部分过于复杂，无法通过调整大小来妥善管理，并不意味着在处理简单情况时没有巨大的价值。

### 总结

随着你的组织的成长和你的产品变得更受欢迎，你将在所有这些轴上成长。

* 需要管理的不同应用程序的数量

* 需要运行的应用程序的副本数量

* 最大的应用程序的规模

为了有效地管理规模，需要自动化，使你能够解决所有这些增长轴。随着时间的推移，你应该期待自动化本身变得更多，既要处理新类型的要求（例如，GPU和TPU的调度是博格在过去10年里发生的一个主要变化），又要处理规模的增加。那些在较小规模下可以手工操作的行动，将需要自动化，以避免组织在负载下的崩溃。

一个例子--谷歌仍在摸索的过渡--就是对我们的数据中心进行自动化管理。十年前，每个数据中心是一个独立的实体。我们手动管理它们。启用一个数据中心是一个复杂的手工过程，需要专门的技能，需要几周的时间（从所有机器准备好的那一刻开始），而且本身就有风险。然而，谷歌管理的数据中心数量的增长意味着我们转向了一种模式，即开启数据中心是一个不需要人工干预的自动化过程。

## 编写管理计算的软件

从手工管理的机器列表到自动调度和权利调整的世界，使谷歌的机群管理变得更加容易，但这也使我们编写和思考软件的方式发生了深刻的变化。

### 失败的架构

想象一下，一个工程师要处理一批100万份文件并验证其正确性。如果处理一个文件需要一秒钟，那么整个工作将需要一台机器大约12天--这可能太长了。所以，我们把工作分散到200台机器上，这样就把运行时间减少到了更容易管理的100分钟。

正如在 "自动调度 "中所讨论的，在博格世界中，调度员可以单方面杀死200个工人中的一个，并把它移到不同的机器上。[6] "把它移到不同的机器上 "这部分意味着你的工人的新实例可以自动被印出来，不需要人去SSH进入机器，调整一些环境变量或安装软件包。

从 "工程师必须手动监控100个任务中的每一个，并在出现问题时对其进行处理 "到 "如果其中一个任务出现问题，系统会被设计成由其他任务来承担，而自动调度器会将其杀死并在新的机器上重新确定"，这一转变在许多年后通过 "宠物与牛 "的比喻来描述。[7]

如果你的服务器是一只宠物，当它坏了的时候，会有一个人来看它（通常是在恐慌中），了解出了什么问题，并希望能把它护理好，恢复健康。它很难被取代。如果你的服务器是牛，你给它们命名为replica001到replica100，如果有一个出现故障，自动化会把它移走，并在其位置上配置一个新的。牛 "的显著特点是，很容易为有关工作的新实例盖章--它不需要手动设置，可以完全自动完成。这使得前面描述的自我修复特性成为可能--在发生故障的情况下，自动化可以接管，用一个新的、健康的工作取代不健康的工作，而不需要人工干预。请注意，虽然原来的比喻说的是服务器（虚拟机），但同样适用于容器：如果你能在没有人工干预的情况下从镜像中印出一个新版本的容器，你的自动化就能在需要时自动修复你的服务。

如果你的服务器是宠物，你的维护负担将随着你的机群规模线性增长，甚至超线性增长，而这是任何组织都不应该轻易接受的负担。另一方面，如果你的服务器是牛，你的系统将能够在故障后恢复到一个稳定的状态，你将不需要花周末的时间来护理宠物服务器或容器恢复健康。

不过，让你的虚拟机或容器是牛，并不足以保证你的系统在面对故障时表现良好。有200台机器，其中一个副本被博格杀死是很有可能发生的，可能不止一次，而且每次都会使整个持续时间延长50分钟（或者不管损失多少处理时间）。为了优雅地处理这个问题，处理的架构需要改变：我们不是静态地分配工作，而是把整个100万份文件分成，比如说1000块，每块1000份文件。每当一个工人完成了一个特定的块，它就报告结果，并拿起另一个。这意味着，在工人完成某块工作后，但在报告之前死亡的情况下，我们最多损失一个工作块。幸运的是，这非常符合当时谷歌的标准数据处理架构：在计算开始时，工作并没有平均分配给一组工作者；而是在整个处理过程中动态分配，以考虑到工作者的失败。

同样，对于服务于用户流量的系统来说，你最好希望容器的重新调度不会导致错误被提供给用户。Borg调度器在计划因维护原因而重新调度一个容器时，会向容器发出信号，提前通知它的意图。容器可以通过拒绝新的请求做出反应，同时仍有时间完成它正在进行的请求。这反过来要求负载均衡器系统理解 "我不能接受新请求 "的反应（并将流量重定向到其他副本）。

总结：把你的容器或服务器当成牛，意味着你的服务可以自动恢复到健康状态，但需要额外的努力，以确保它在经历适度的故障率时能够顺利运行。

### 批量与服务

全局工作队列（我们在本章第一节中描述过）解决了谷歌工程师所说的 "批处理作业 "的问题--这些程序被期望完成一些特定的任务（如数据处理），并运行到完成。批量作业的典型例子是日志分析或机器学习模型学习。批量作业与 "服务作业 "形成鲜明对比--这些程序预计将无限期地运行并为传入的请求提供服务，典型的例子是为预先建立的索引中的实际用户搜索查询提供服务的作业。

这两类作业（通常）有不同的特点，[8] 特别是。

* 批量作业主要关心的是处理的吞吐量。服务工作关心的是服务单个请求的延迟。

* 批量作业是短命的（几分钟，或最多几个小时）。服务工作通常是长期存在的（默认情况下，只有在新版本发布时才会重新启动）。

* 因为它们是长期存在的，所以服务工作更有可能有较长的启动时间。