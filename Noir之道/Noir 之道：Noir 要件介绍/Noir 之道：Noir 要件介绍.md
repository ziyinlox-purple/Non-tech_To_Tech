# Noir 之道：Noir 要件介绍

## TL;DR

System: MacOS

tool: VSCode

installation: nargo, noirup, Barretenberg

process: 

1. creating project(Nargo.toml & src)
2. generating `Prove.toml`
3. using `bb prove` to generate proof and verify

---

Noir 借鉴了 Rust，但学 Noir 不需要先学 Rust，直接上手是最快的方式。

Noir 需要一个编译工具 `nargo`，它依赖 Rust 的 cargo。别担心，你不用学 Rust，只需安装环境。具体这个怎么用，我们后面再说。

如果要升级更换 Noir 的版本，我们需要使用 `noirup` 这个工具。

这里再强调一下安装顺序：

1. 首先安装 Rust，因为我们需要用 Rust 的 `cargo` 来安装 `nargo` ，Rust 安装完毕以后就可以使用 `nargo` 了，具体如何安装 Rust 自行问 AI，我们最后可以用 `rustc --version` 来检查 Rust 安装完毕好没好；
2. 做完前置准备，我们开始装 `nargo`，
    1. 在终端输入 
        
        ```bash
        cargo install nargo
        ```
        
    2. 再输入 `nargo --version` ，如果看到版本号（如 0.15.0）说明安装好了。
3. 【补充】除了 `nargo`，我们还要再安装一个版本升级管理的工具—— `noirup` 。如果是已经安装了 Noir 要升级到最新的版本，直接输入：
    
    ```bash
    noirup --version
    ```
    
    就可以了。
    
    当然也可以指定升级到具体的版本，官方现在有 nightly 还有 beta 版本，具体 nightly 和 beta 的版本是不一样的，可以看 GPT 老师的总结。
    
    ![image.png](Noir%20%E4%B9%8B%E9%81%93%EF%BC%9ANoir%20%E8%A6%81%E4%BB%B6%E4%BB%8B%E7%BB%8D%201b6041c72ed5805fb641c76c7beb08b8/image.png)
    
    正常选 Beta 版本就可以了。
    

Ok，现在我们已经做好了前置的准备，安装好了 Noir 的编译工具 `nargo` ，还有升级管理工具 `noirup` 。

---

接下来我们实际创建项目，这个过程里面就会用到 `nargo` 的命令：

1.  `nargo new` 创建新项目
2.  `nargo check` 命令构建输入/输出文件

我现在在名为 `Noir_projects_files` 的这个文件夹中，创建了我名为 `first_project` 的最小项目。

![它告诉你这个最小的项目文件被创建成功了，位于哪里](./image%201.png)

它告诉你这个最小的项目文件被创建成功了，位于哪里

要构成最基础的 Nargo 项目，这个项目文件里应该有以下几个部分：

- src
- Prover.toml
- Nargo.toml

我们看看这个项目文件里实际有什么：

![image.png](./image%202.png)

好吧，只有两个文件，这是为啥？这是因为 `Pover.toml` 文件是用来存储证明相关信息的，所以逻辑上来说我们得有证明的相关信息，才能有这个文件产生。在项目刚被建立，默认只是一个空的 Noir 项目，没有可执行的证明逻辑，自然也就没有 `Prover.toml` 这个文件了，是不是很好理解？

那我们接下来要做什么呢？要做的就是进入 `src` 这个文件中，你会看到 [`main.nr`](http://main.nr) 的文件。

![*ps:源目录 src 保存了你的 Noir 程序的源代码。默认情况下，其中只会生成一个 [main.nr](http://main.nr/) 文件。* ](./image%203.png)

*ps:源目录 src 保存了你的 Noir 程序的源代码。默认情况下，其中只会生成一个 [main.nr](http://main.nr/) 文件。* 

VSCode 可以提前下载一个 Noir 的插件，让我们用 VSCode 打开这个文件。

![image.png](./image%204.png)

![最原始的 [main.nr](http://main.nr) 里的内容](./image%205.png)

最原始的 [main.nr](http://main.nr) 里的内容

我们来解析一下上面这些代码并分析其作用，可以分为两个部分：

- **`x: Field`**：`x` 是一个私有输入（Prover 提供的秘密值）。
- **`y: pub Field`**：`y` 是一个**公共输入**（Verifier 可以看到的值）。
- **`assert(x != y);`**：这个 `assert` 语句要求 `x` **必须不等于** `y`，否则证明就会失败。

简单来说，这个 Noir 证明电路确保了：

1.  x 和 y 不能相等
2. 只有当 x≠y 时，证明才会生成并验证成功

让我们再看下去。值得一提的是，Noir 提供了测试的功能，也就是下方的测试函数**`#[test]`**，便于随时测试 Noir 代码的正确性。

看到 **`main(x: 1, y: 2);`** ，可能第一反应就是这个 x=1, y=2 的数值是哪里来的？是在 `Prover.toml`  里的吗？其实并不是。实际上，这是在测试函数里是被写死的，其实里面具体是什么数字都无所谓啦，只要满足这个代码的逻辑关系就好了，毕竟这个  `test_main` 是一个测试函数。它不依赖 Noir 的外部输入文件，也不会用 `Prover.toml` 作为输入。这里要强调一点，在 Noir 里，测试 (`#[test]`) 和正式的 ZK 证明 (`nargo prove`) 是两回事：

- **测试 (`nargo test`)**：直接在 Rust 风格的测试函数里写入参数并调用 Noir 代码。
- **证明 (`nargo prove`)**：使用 `Prover.toml` 作为输入文件，提供 `x` 和 `y` 的值。

现在基本了解就好了，我们继续深入下去会用到 `nargo prove`。

哦对了，再提一嘴备注里的 `//main(1, 1);` ，它的用处是如果你想让上面这个测试代码的逻辑出现问题，那你就把前面的 `//` 删掉，这个测试就会默认使用 x=1, y=1 的输入，它逻辑上就会出现问题，测试就失败了。

上面说了这么多，就是想说  `#[test]` 就是为了确保 Noir 代码的**逻辑正确**，防止错误的逻辑进入生产环境。

---

**我们继续，但我接下来会简单用个例子实践——比较两个数。**

我把 [`main.nr`](http://main.nr) 中的代码部分改了，如图：

![image.png](./image%206.png)

![image.png](./image%207.png)

但是出现了问题，上面这个代码没法跑通，原因是 Noir 的 ZK 特性限制，不支持直接 `>`、`<` 翻译成 ZK 电路约束，需要用 `std::cmp` 或其他约束方法。

**经过上面这一顿操作，我在理解 Noir 的测试这部分遇到了一些问题**，具体是这样的。正如我上文所说，Noir 在默认生成的 [main.nr](http://main.nr) 文件中就提供默认的测试功能，但是这个功能据说之前是可以直接输入值然后调用运行，但新版本的 Noir 中已经不行了，需要用到 `nargo check` 和 `nargo test` 这个命令来运行测试。其中，`nargo check` 是检查语法是否通过， `nargo test` 是运行测试。

鉴于不是新手很友好，我决定换继续回到默认的初始代码，也就是下图。

![最原始的 [main.nr](http://main.nr) 里的内容](./image%205.png)

最原始的 [main.nr](http://main.nr) 里的内容

ok，我什么也不改，接着我在终端里输入 `nargo check` ，再输入 `nargo test` 

![image.png](./image%208.png)

哎呀，成功了！说明这种测试方法是可以的。这个时候如果你返回上一级，会发现多了一个 `Prover.toml` 的文件，其实到这里为止，最基础的项目所必需的文件就已经满足了，让我们来打开这个 `Prove.toml` 的文件。

![image.png](./image%209.png)

![image.png](./image%2010.png)

我们可以喂两个值，因为我们定义的逻辑关系是 `x ≠ y` ，所以我随便选择 3 和 5，事实上证明最初默认的测试用例里给的值 1 和 2 其实并不影响我们实际测试，只要逻辑上的 `≠` 满足就好了。

![image.png](./image%2011.png)

然后我们保存退出，在终端里输入命令 `nargo execute` ，我们就可以生成 `witness`。

![image.png](./image%2012.png)

woho，这时候我们看一下项目文件里，就会发现了多了一个 target 的文件，其中还有一个名为 `first_project.gz` 的项目文件。

![image.png](./image%2013.png)

Noir 官方文档里对这里的解释是：

> 运行这个命令后，会把执行过程中生成的 **witness（见证数据/电路的中间表示）** 存到 `./target/witness-name.gz` 这个文件里。
> 
> 
> 另外，如果你的 Noir 代码还没编译过，或者你刚刚修改过代码，这个命令也会**自动编译**你的 Noir 程序，并把编译后的结果存到 `./target/hello_world.json` 里。
> 
> 现在，**电路已经编译好，witness 也生成了，我们可以开始证明了！**
> 

我发现随着版本的更迭，Noir 的命令也出现了一些变化，比如以前是还有 `nagro prove` 的，现在在新版本中就没有了，现在 Noir 开始通过 nargo backend 管理后端（如 Barretenberg），证明流程可能被重新设计。反正到目前运行完 `nargo test` 命令，只生成了 `witness` ，但还没有 `proof` 。

我看了最新的 Noir 文档（截止到 2025.3.15），需要我们安装 Barretenberg 来用 bb prove。`bb prove` 是 Barretenberg 的 CLI 工具，用于根据 Noir 生成的电路和 witness 文件生成证明。

### 安装 Barretenberg 的步骤

1. 前置条件：
    1. 确保已经安装了 `nargo`，
    2. 安装依赖：macOS 终端输入：`brew install cmake libomp`
2. **安装 Barretenberg CLI 工具 (bb)**
    
    Noir 文档建议使用 bbup 脚本安装最新版本的 Barretenberg。以下是步骤：
    
    - 运行以下命令下载并安装 bb：`curl -L https://raw.githubusercontent.com/AztecProtocol/aztec-packages/master/barretenberg/bbup/install | bash`
    - 更新到特定版本（例如 v0.72.1，根据最新兼容性）：`bbup -v 0.72.1`
    - 验证安装：`bb --version`

然后我运行 `bb prove` 想生成 proof，结果出现了以下问题

![image.png](./image%2014.png)

也就是说我要先安装 jq，好的我下载一下。

![image.png](./image%2015.png)

安装完毕。

![image.png](./image%2016.png)

到这里我以为是我的 proof 还没生成，但实际上它已经生成啦！！我一开始用  `ls` 命令的时候它是没有显现的，可能比较文件比较小没显示，后来用  `ls -1h` 就可以了。

![image.png](./image%2017.png)

既然现在已经生成了 proof，那么我们就可以 verify 了。我们需要用到的命令是 `bb write_vk` 和 `bb verify` ，其中前者 从电路文件（`first_project.json`）生成验证密钥（VK），用于后续验证证明；后者验证密钥（`vk`） 和 证明文件（`proof`） 来检查证明是否有效。

![image.png](./image%2018.png)