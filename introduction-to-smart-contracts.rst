###############################
智能合约概述
###############################

.. _simple-smart-contract:

***********************
一个简单的智能合约
***********************

Let us begin with the most basic example. It is fine if you do not understand everything
right now, we will go into more detail later.

我们开始以一个简单的例子入手了解智能合约. 现在看不懂没关系, 我们将一步步带你深入.

Storage 存储
=======

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData;

        function set(uint x) public {
            storedData = x;
        }

        function get() public constant returns (uint) {
            return storedData;
        }
    }

The first line simply tells that the source code is written for
Solidity version 0.4.0 or anything newer that does not break functionality
(up to, but not including, version 0.5.0). This is to ensure that the
contract does not suddenly behave differently with a new compiler version.
The keyword ``pragma`` is called that way because, in general,
pragmas are instructions for the compiler about how to treat the
source code (e.g. `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`_).

第一行表明代码是基于 Solidity 0.4.0 或者更高版本(小于0.5.0). 这是为了保证程序不会在新的编译器下运行异常.
关键词 ``pragma`` 用于标记版本, 是因为一般情况 pragmas 是给编译器如何编译的标记.
(例如 `pragma once <https://en.wikipedia.org/wiki/Pragma_once>`_).

A contract in the sense of Solidity is a collection of code (its *functions*) and
data (its *state*) that resides at a specific address on the Ethereum
blockchain. The line ``uint storedData;`` declares a state variable called ``storedData`` of
type ``uint`` (unsigned integer of 256 bits). You can think of it as a single slot
in a database that can be queried and altered by calling functions of the
code that manages the database. In the case of Ethereum, this is always the owning
contract. And in this case, the functions ``set`` and ``get`` can be used to modify
or retrieve the value of the variable.

在 Solidity 看来, 一个合约就是一些代码(*function*)和数据状态(*state*)的集合, 并且以指定地址存储在 Ethereum 区块链中.
``uint storedData;`` 声明一个名为storedData的状态(state)变量, 是 ``uint`` (无符号整型,256位)的. 可以将其理解为一个
数据库中的单个slot存储, 可以通过代码中定义的查询修改来管理该数据. In the case of Ethereum, this is always the owning
contract. 这种情况下 ``set`` 和 ``get`` 方法将被用于更改或恢复变量的值.

To access a state variable, you do not need the prefix ``this.`` as is common in
other languages.

想要获取变量的状态, 无需像其他语言一样使用 ``this.`` .

This contract does not do much yet (due to the infrastructure
built by Ethereum) apart from allowing anyone to store a single number that is accessible by
anyone in the world without a (feasible) way to prevent you from publishing
this number. Of course, anyone could just call ``set`` again with a different value
and overwrite your number, but the number will still be stored in the history
of the blockchain. Later, we will see how you can impose access restrictions
so that only you can alter the number.

(翻译不太好...)这个合约现在还很简单, 所有人都可以通过 ``set`` 设置数据覆盖原有值, 但是历史记录仍存在. 稍后
我们将展示一下如何提高权限管理, 以便只有你可以更改该数据.

.. note::
    All identifiers (contract names, function names and variable names) are restricted to
    the ASCII character set. It is possible to store UTF-8 encoded data in string variables.

    所有标识符(合约名,方法名,变量名)都是ASCII受限的. 可以通过string存储 UTF-8 数据.

.. warning::
    Be careful with using Unicode text as similarly looking (or even identical) characters can
    have different code points and as such will be encoded as a different byte array.

    注意使用 Unicode 文本编码的字符或者文字, 尽管外表看似相同或者相等, 也会引起代码编译成完全不同的二进制数组.

.. index:: ! subcurrency

自定义货币举例
===============

The following contract will implement the simplest form of a
cryptocurrency. It is possible to generate coins out of thin air, but
only the person that created the contract will be able to do that (it is trivial
to implement a different issuance scheme).
Furthermore, anyone can send coins to each other without any need for
registering with username and password - all you need is an Ethereum keypair.

下面的合约将实现最简单的加密货币. 可以凭空生成货币, 但是只有合约的创建者有该权限.
货币发行细节这里做极简化. 此外, 任何人都可以无需账号密码的向其他账号转账, 只需Ethereum私钥.

::

    pragma solidity ^0.4.0;

    contract Coin {
        // The keyword "public" makes those variables
        // readable from outside.
        // "public" 标记可以让变量从外部访问
        address public minter;
        mapping (address => uint) public balances;

        // Events allow light clients to react on
        // changes efficiently.
        // 事件允许轻应用对更改做出快速反应
        event Sent(address from, address to, uint amount);

        // This is the constructor whose code is
        // run only when the contract is created.
        // 构造函数, 部署时调用一次.
        function Coin() public {
            minter = msg.sender;
        }

        // 挖矿
        function mint(address receiver, uint amount) public {
            if (msg.sender != minter) return;
            balances[receiver] += amount;
        }

        // 转账
        function send(address receiver, uint amount) public {
            if (balances[msg.sender] < amount) return;
            balances[msg.sender] -= amount;
            balances[receiver] += amount;
            Sent(msg.sender, receiver, amount);
        }
    }

This contract introduces some new concepts, let us go through them one by one.

这个合约引入了一些新概念. 让我们一一看来.

The line ``address public minter;`` declares a state variable of type address that is publicly accessible.

``address public minter;`` 声明了address类型的状态变量, public允许外部访问.

The ``address`` type is a 160-bit value that does not allow any arithmetic operations. It is suitable for
storing addresses of contracts or keypairs belonging to external persons.

``address`` 类型是一个 160-位 值, 并且不允许进行任何算数运算. 用于存储合约地址或者外部用户的公/私钥.

The keyword ``public`` automatically generates a function that allows you to access the current value
of the state variable from outside of the contract. Without this keyword, other contracts have no way
to access the variable. The code of the function generated by the compiler is roughly equivalent
to the following::

    function minter() returns (address) { return minter; }

``public`` 关键字自动生成一个允许在外部访问该值的的方法. 不设置的话, 外部将无法访问该变量.
编译自动生成的代码类似如下的内容::

    function minter() returns (address) { return minter; }

Of course, adding a function exactly like that will not work
because we would have a function and a state variable with the same name,
but hopefully, you get the idea - the compiler figures that out for you.

当然, 手动添加这样的方法是不行的, 因为这样我们就有了相同名称的状态变量和方法.
这只是希望你知道, 编译器替你完成了这部分功能.

.. index:: mapping

The next line, ``mapping (address => uint) public balances;`` also
creates a public state variable, but it is a more complex datatype.
The type maps addresses to unsigned integers.
Mappings can be seen as `hash tables <https://en.wikipedia.org/wiki/Hash_table>`_ which are
virtually initialized such that every possible key exists and is mapped to a
value whose byte-representation is all zeros.

``mapping (address => uint) public balances;`` 同样创建一个 public 的变量, 但是更复杂.
数据类型将 address 类型和 uint 类型做了一个映射.
映射的文档参考 `哈希表 <https://en.wikipedia.org/wiki/Hash_table>`_, 该数据类型每个键对应的值默认都是0.

This analogy does not go too far, though, as it is neither possible
to obtain a list of all keys of a mapping, nor a list of all values.
So either keep in mind (or better, keep a list or use a more advanced
data type) what you added to the mapping or use it in a context where
this is not needed, like this one. The :ref:`getter function<getter-functions>`
created by the ``public`` keyword is a bit more complex in this case.
It roughly looks like the following::

    function balances(address _account) public view returns (uint) {
        return balances[_account];
    }

这里的map映射, 无法获取key列表数组, 也无法获取value列表数组. 所以一般需要单独维护key列表,
或者像本例子一样在一个不需要获取列表的情境下使用. 通过 ``public`` 自动创建的
:ref:`getter 方法<getter-functions>` 相对上面的更为复杂一点, 大概类似于::

    function balances(address _account) public view returns (uint) {
        return balances[_account];
    }

As you see, you can use this function to easily query the balance of a
single account.

可见, 你可以使用这个方法方便的查询任何账户的余额.

.. index:: event

The line ``event Sent(address from, address to, uint amount);`` declares
a so-called "event" which is fired in the last line of the function
``send``. User interfaces (as well as server applications of course) can
listen for those events being fired on the blockchain without much
cost. As soon as it is fired, the listener will also receive the
arguments ``from``, ``to`` and ``amount``, which makes it easy to track
transactions. In order to listen for this event, you would use ::

    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

``event Sent(address from, address to, uint amount);`` 声明了一个所谓的事件,
将在 ``send`` 方法调用后执行. 用户接口 (和服务器应用一样) 可以监听这些区块链上的事件,
不会耗费太多资源. 当事件被触发的时候, 监听者也将收到参数 ``from``, ``to`` 和 ``amount``,
这使得交易更容易被追踪. 监听该事件需要如下代码::

    Coin.Sent().watch({}, '', function(error, result) {
        if (!error) {
            console.log("Coin transfer: " + result.args.amount +
                " coins were sent from " + result.args.from +
                " to " + result.args.to + ".");
            console.log("Balances now:\n" +
                "Sender: " + Coin.balances.call(result.args.from) +
                "Receiver: " + Coin.balances.call(result.args.to));
        }
    })

Note how the automatically generated function ``balances`` is called from
the user interface.

注意: 自动生成的 ``balances`` 方法如何在用户api中调用.

.. index:: coin

The special function ``Coin`` is the
constructor which is run during creation of the contract and
cannot be called afterwards. It permanently stores the address of the person creating the
contract: ``msg`` (together with ``tx`` and ``block``) is a magic global variable that
contains some properties which allow access to the blockchain. ``msg.sender`` is
always the address where the current (external) function call came from.

特殊方法 ``Coin`` 是构造方法, 在合约创建的时候调用一次, 以后就无法再调用了. 它永久的储存了合约创建者
的地址, ``msg`` (还有 ``tx`` 和 ``block``) 是一些很有用的全局变量, 用于访问区块链的属性.
``msg.sender`` 就是当前/外部方法调用者的地址.

Finally, the functions that will actually end up with the contract and can be called
by users and contracts alike are ``mint`` and ``send``.
If ``mint`` is called by anyone except the account that created the contract,
nothing will happen. On the other hand, ``send`` can be used by anyone (who already
has some of these coins) to send coins to anyone else. Note that if you use
this contract to send coins to an address, you will not see anything when you
look at that address on a blockchain explorer, because the fact that you sent
coins and the changed balances are only stored in the data storage of this
particular coin contract. By the use of events it is relatively easy to create
a "blockchain explorer" that tracks transactions and balances of your new coin.

最后, 剩余的两个方法 ``mint`` 和 ``send`` 是用户可以直接访问的.
如果 ``mint`` 被除了合约创建者之外的其他用户访问, 将不会执行. 另一方面, ``send`` 方法可以
被任何用户(拥有本币的用户)调用, 用于转账给他人. 值得注意的是, 使用此合约给他人转账, 查看区块链
上对应账号的信息, 将不会看到任何改变, 因为转账的实质仅仅存储于该特殊的货币合约之中. 可以使用
前面提到的 event 事件来创建自己独有的区块链浏览器, 用于跟踪记录交易和余额.

.. _blockchain-basics:

*****************
区块链基础
*****************

Blockchains as a concept are not too hard to understand for programmers. The reason is that
most of the complications (mining, `hashing <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_, `elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_, `peer-to-peer networks <https://en.wikipedia.org/wiki/Peer-to-peer>`_, etc.)
are just there to provide a certain set of features and promises. Once you accept these
features as given, you do not have to worry about the underlying technology - or do you have
to know how Amazon's AWS works internally in order to use it?

区块链这个概念对应程序员来说并不是很难理解, 原因是大部分复杂的内容(挖矿,
`哈希 <https://en.wikipedia.org/wiki/Cryptographic_hash_function>`_,
`elliptic-curve cryptography <https://en.wikipedia.org/wiki/Elliptic_curve_cryptography>`_,
`P2P网络 <https://en.wikipedia.org/wiki/Peer-to-peer>`_, 等等.) 都只是用于提供一系列功能和目标.
一旦你接受了提供的这些特性, 就无需再担心底层技术了. 就像你无须知道亚马逊aws底层如何实现的一样.

.. index:: transaction

Transactions 交易
================

A blockchain is a globally shared, transactional database.
This means that everyone can read entries in the database just by participating in the network.
If you want to change something in the database, you have to create a so-called transaction
which has to be accepted by all others.
The word transaction implies that the change you want to make (assume you want to change
two values at the same time) is either not done at all or completely applied. Furthermore,
while your transaction is applied to the database, no other transaction can alter it.

区块链是极度分布式的交易存储数据库. 这意味着每个人都可以通过联网直接进入数据库.
这意味着想要修改数据库的内容, 需要创建一个被所有人认可的交易. 交易, 意味着一同修改两个账户数据,
或者同时成功, 或者同时失败. 此外, 由于交易已经写入数据库, 其他交易无法修改这个交易.

As an example, imagine a table that lists the balances of all accounts in an
electronic currency. If a transfer from one account to another is requested,
the transactional nature of the database ensures that if the amount is
subtracted from one account, it is always added to the other account. If due
to whatever reason, adding the amount to the target account is not possible,
the source account is also not modified.

举例说, 假设一个电子账户表包含所有账户的余额. 如果一个账户给另一个转账, 数据库交易的特性就会保证
如果数据从一个账号减掉, 就一定会加入到另一个账号. 如果因为任何原因, 使得加入另一个账号失败,
那么原账号也不会修改.

Furthermore, a transaction is always cryptographically signed by the sender (creator).
This makes it straightforward to guard access to specific modifications of the
database. In the example of the electronic currency, a simple check ensures that
only the person holding the keys to the account can transfer money from it.

此外, 交易将使用发起者账号密钥保持加密, 这使得维护数据库指定数据的编辑权限变得简单. 在例子中
的电子账户, 一个简单的验证即可保证只有拥有密钥的账号才能转出资金.

.. index:: ! block

Blocks 区块
==========

One major obstacle to overcome is what, in Bitcoin terms, is called a "double-spend attack":
What happens if two transactions exist in the network that both want to empty an account,
a so-called conflict?

在比特币项目中, 区块主要的目的就是解决一个重复花费问题, 如何解决存在两个交易都想清空账号这样的冲突呢?

The abstract answer to this is that you do not have to care. An order of the transactions
will be selected for you, the transactions will be bundled into what is called a "block"
and then they will be executed and distributed among all participating nodes.
If two transactions contradict each other, the one that ends up being second will
be rejected and not become part of the block.

内部的实现细节实际上你并不需要关心. 交易的顺序会为你选择执行哪一个交易. 交易会打包进区块, 之后会在所有参与运算的
节点中分布式的执行运算. 如果两个交易相互冲突, 后者将被拒绝并且抛弃, 不会进入最终区块.

These blocks form a linear sequence in time and that is where the word "blockchain"
derives from. Blocks are added to the chain in rather regular intervals - for
Ethereum this is roughly every 17 seconds.

区块以时间顺序排序, 组成链式结构, 成为区块链. 区块加入的时间间隔相对固定, Ethereum中大约为17秒.

As part of the "order selection mechanism" (which is called "mining") it may happen that
blocks are reverted from time to time, but only at the "tip" of the chain. The more
blocks that are added on top, the less likely it is. So it might be that your transactions
are reverted and even removed from the blockchain, but the longer you wait, the less
likely it will be.

作为顺序选择机制的一部分(挖矿), 数据从一个时间点回滚到另一个更早的时间点是可能的, 但是只有 "尖端" 会出现
这样的情况, 后续的区块越多, 这种可能越小. 所以交易被回滚或者取消的可能性是有的, 但是实际越久可能性越低.


.. _the-ethereum-virtual-machine:

.. index:: !evm, ! ethereum virtual machine

****************************
以太坊虚拟机
****************************

Overview 概览
========

The Ethereum Virtual Machine or EVM is the runtime environment
for smart contracts in Ethereum. It is not only sandboxed but
actually completely isolated, which means that code running
inside the EVM has no access to network, filesystem or other processes.
Smart contracts even have limited access to other smart contracts.

Ethereum虚拟机(EVM)是 Ethereum中智能合约的运行环境. 它不仅仅是沙盒状态的, 并且完全
独立, 意味着EVM中代码运行没有权利访问网络, 文件系统或者其他程序. 智能合约甚至访问其他
合约的权限也是受到限制的.

.. index:: ! account, address, storage, balance

Accounts 账号
========

There are two kinds of accounts in Ethereum which share the same
address space: **External accounts** that are controlled by
public-private key pairs (i.e. humans) and **contract accounts** which are
controlled by the code stored together with the account.

Ethereum 账号有两种, 但是共享相同的数据空间. 使用公私钥密钥对控制的**外部账户**,
还有被账户中所有智能合约代码控制的**合约账户**.

The address of an external account is determined from
the public key while the address of a contract is
determined at the time the contract is created
(it is derived from the creator address and the number
of transactions sent from that address, the so-called "nonce").

外部账户地址取决于公钥, 而合约账户地址取决于合约创建时间(受外部账户地址和合约所在区块(nonce)共同驱动).

Regardless of whether or not the account stores code, the two types are
treated equally by the EVM.

不管账户是否存储了代码, 两种账户都被 EVM 同等的对待.

Every account has a persistent key-value store mapping 256-bit words to 256-bit
words called **storage**.

每个账户有一个持久化的键值对, 存储 256位-256位 的哈希数据, 成为存储.

Furthermore, every account has a **balance** in
Ether (in "Wei" to be exact) which can be modified by sending transactions that
include Ether.

此外, 每个账户有一个 **余额**, 以 Ether(准确的说是以Wei)为单位, 通过发生含有Ether的交易可以更改余额.

.. index:: ! transaction

Transactions 交易
================

A transaction is a message that is sent from one account to another
account (which might be the same or the special zero-account, see below).
It can include binary data (its payload) and Ether.

交易是一个账户发给另一个账户的消息(可能是相同账户或者空账户, 详情继续阅读). 可以包含二进制数据(称为payload)和Ether.

If the target account contains code, that code is executed and
the payload is provided as input data.

如果目标账号含有代码, 那么数据将作为输入数据传入代码.

If the target account is the zero-account (the account with the
address ``0``), the transaction creates a **new contract**.
As already mentioned, the address of that contract is not
the zero address but an address derived from the sender and
its number of transactions sent (the "nonce"). The payload
of such a contract creation transaction is taken to be
EVM bytecode and executed. The output of this execution is
permanently stored as the code of the contract.
This means that in order to create a contract, you do not
send the actual code of the contract, but in fact code that
returns that code.

如果目标是一个空账户(地址为0), 交易创建一个 **新的合约**. 就像刚刚提到的, 该新合约的地址并非0,
而是使用发起者信息和交易所在区块nonce自动生成. 这样的创建新合约的交易的二进制数据, 将传入到EVM中
作为二进制代码存储执行. 输出内容将永久的作为合约代码存储起来. 这意味着创建合约时并非将合约代码直接
发送过去, 而是发送一个返回该合约的函数的代码.

.. index:: ! gas, ! gas price

Gas
===

Upon creation, each transaction is charged with a certain amount of **gas**,
whose purpose is to limit the amount of work that is needed to execute
the transaction and to pay for this execution. While the EVM executes the
transaction, the gas is gradually depleted according to specific rules.

每个交易运行的时候, 都会收取一部分费用, 称为 **gas**, 目的是限制需要执行的交易的工作量, 并且为工作量付费.
EVM执行交易时, gas将根据相应规则逐步消耗.

The **gas price** is a value set by the creator of the transaction, who
has to pay ``gas_price * gas`` up front from the sending account.
If some gas is left after the execution, it is refunded in the same way.

**gas 价格**是一个交易发起人创建交易时设置的值, 发起人将支付 ``gas价格 * gas消耗量`` 数量的费用.
如果交易后有 gas 剩余, 将退还给交易发起人.

If the gas is used up at any point (i.e. it is negative),
an out-of-gas exception is triggered, which reverts all modifications
made to the state in the current call frame.

如果 gas 在执行结束前用光了(变为负数), 将会触发 out-of-gas 错误, 回滚本次执行对状态变量的所有修改.

.. index:: ! storage, ! memory, ! stack

Storage, Memory and the Stack 存储, 内存 和 堆栈
=============================

Each account has a persistent memory area which is called **storage**.
Storage is a key-value store that maps 256-bit words to 256-bit words.
It is not possible to enumerate storage from within a contract
and it is comparatively costly to read and even more so, to modify
storage. A contract can neither read nor write to any storage apart
from its own.

每个账户都有一个持久化内存区域, 称为 **storage存储区**. 存储区是一个256位-256位的哈希键值对.
存储区是不可枚举的, 读写起来相当的耗费资源. 合约智能读取属于自己账户的存储区.

The second memory area is called **memory**, of which a contract obtains
a freshly cleared instance for each message call. Memory is linear and can be
addressed at byte level, but reads are limited to a width of 256 bits, while writes
can be either 8 bits or 256 bits wide. Memory is expanded by a word (256-bit), when
accessing (either reading or writing) a previously untouched memory word (ie. any offset
within a word). At the time of expansion, the cost in gas must be paid. Memory is more
costly the larger it grows (it scales quadratically).

第二块内存区域称为 **memory内存区**, 用于合约在处理每条消息时获取最新的消息实例. 内存区是线性结构,
并且允许以字节级别访问, 但是每次最大读取不允许超过 256位, 写入则可以以 8位/256位进行. 内存区当大小
以 256位进行扩容, when accessing (either reading or writing) a previously
untouched memory word (ie. any offset within a word). 扩容的时候是需要收取 gas 的, 内存区
的gas价格随着内存占用的增大而加大(1斤1块钱, 2斤3块钱, 3斤8块钱..., 第1斤1块钱, 第2斤2块钱, 第3斤5块钱...)

The EVM is not a register machine but a stack machine, so all
computations are performed on an area called the **stack**. It has a maximum size of
1024 elements and contains words of 256 bits. Access to the stack is
limited to the top end in the following way:
It is possible to copy one of
the topmost 16 elements to the top of the stack or swap the
topmost element with one of the 16 elements below it.
All other operations take the topmost two (or one, or more, depending on
the operation) elements from the stack and push the result onto the stack.
Of course it is possible to move stack elements to storage or memory,
but it is not possible to just access arbitrary elements deeper in the stack
without first removing the top of the stack.

EVM 不是基于寄存器的而是基于堆栈的. 所以计算都是基于 **stack堆栈区** 进行的. 最大上限为 1024 个元素 and contains words of 256 bits.
堆栈仅限于访问顶端, 访问方式为:
允许将顶部16个元素之一拷贝到堆栈区最顶端, 或者将堆栈区最顶端的元素与其下面的16个元素之一进行换位.
其余的操作运算都是取堆栈区最顶端的2个(或者1个,或者多个,这取决于进行什么运算)元素, 运算后将其结果放回堆栈最顶端.
当然, 可以将堆栈区元素移动到存储区或者内存区, 但是决不允许不移除顶端元素的情况下, 直接访问堆栈区中更深层的元素.

.. index:: ! instruction

Instruction Set
===============

The instruction set of the EVM is kept minimal in order to avoid
incorrect implementations which could cause consensus problems.
All instructions operate on the basic data type, 256-bit words.
The usual arithmetic, bit, logical and comparison operations are present.
Conditional and unconditional jumps are possible. Furthermore,
contracts can access relevant properties of the current block
like its number and timestamp.

上面只是简单的介绍EVM. 全部都是基于基础数据类型 256位 进行的.
常用的算数运算, bit, 逻辑和比较运算也支持, 同样支持条件跳转等. 此外, 合约也可以
访问当前区块的相关属性, 例如 number 和时间戳.

.. index:: ! message call, function;call

Message Calls
=============

Contracts can call other contracts or send Ether to non-contract
accounts by the means of message calls. Message calls are similar
to transactions, in that they have a source, a target, data payload,
Ether, gas and return data. In fact, every transaction consists of
a top-level message call which in turn can create further message calls.

合约可以调用其他合约, 或者通过消息传递的方式向非合约账户转账. 消息传递与交易类似,
因为都是有来源, 目标, 传递的数据, Ether, gas 和返回值的. 实际上, 每个交易是由高级
的消息传递组成的, 该消息结果反过来可以创建新的消息传递.

A contract can decide how much of its remaining **gas** should be sent
with the inner message call and how much it wants to retain.
If an out-of-gas exception happens in the inner call (or any
other exception), this will be signalled by an error value put onto the stack.
In this case, only the gas sent together with the call is used up.
In Solidity, the calling contract causes a manual exception by default in
such situations, so that exceptions "bubble up" the call stack.

合约可以设置内部消息调用消耗多少 gas, 保留多少gas. 如果 out-of-gas 错误(或者其他任意类型错误)
在内部调用的时候被触发, 将以error错误的形式结果存到堆栈中.
这种情况下, 只有发送调用的gas会被消耗, 运行过程中的gas不会消耗.
Solidity中, 这种情况下, 调用合约默认触发一个手动的错误, 以便错误返回调用栈.

As already said, the called contract (which can be the same as the caller)
will receive a freshly cleared instance of memory and has access to the
call payload - which will be provided in a separate area called the **calldata**.
After it has finished execution, it can return data which will be stored at
a location in the caller's memory preallocated by the caller.

被调用的合约(可以和调用合约相同)将受到一个全新的内存区实例, 有权访问传递的数据,
该数据存储在单独的区域, 名为 **calldata**. 在执行之后, 将返回数据存储在调用者
(在调用者内存区)预先分配好的内存地址中.

Calls are **limited** to a depth of 1024, which means that for more complex
operations, loops should be preferred over recursive calls.

调用深度最大限制为1024, 这意味着对于复杂的操作, 循环优于递归调用.

.. index:: delegatecall, callcode, library

Delegatecall / Callcode and Libraries
=====================================

There exists a special variant of a message call, named **delegatecall**
which is identical to a message call apart from the fact that
the code at the target address is executed in the context of the calling
contract and ``msg.sender`` and ``msg.value`` do not change their values.

存在一个特殊的消息调用方法, 称为 **delegatecall** 方法.
除了目标地址指定的代码将使用调用来源合约的上下文, ``msg.sender`` 和 ``msg.value`` 将保持原有值之外, 与普通方法没什么不同.

This means that a contract can dynamically load code from a different
address at runtime. Storage, current address and balance still
refer to the calling contract, only the code is taken from the called address.

这意味着合约可以动态从不同地址调用代码执行. 存储区, 当前地址, 余额等均使用调用来源的合约, 只有代码来自于被调用地址.

This makes it possible to implement the "library" feature in Solidity:
Reusable library code that can be applied to a contract's storage, e.g. in
order to  implement a complex data structure.

这就使得 "库" 功能得以实现. 可复用库代码, 例如在需要实现复杂数据结构的时候, 可以应用到合约的存储区.

.. index:: log

Logs 日志
=========

It is possible to store data in a specially indexed data structure
that maps all the way up to the block level. This feature called **logs**
is used by Solidity in order to implement **events**.
Contracts cannot access log data after it has been created, but they
can be efficiently accessed from outside the blockchain.
Since some part of the log data is stored in `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_, it is
possible to search for this data in an efficient and cryptographically
secure way, so network peers that do not download the whole blockchain
("light clients") can still find these logs.

支持以特殊的检索数据结构存储数据, 以便将数据追溯到区块级别. 这个功能称为 **logs日志**
在Solidity中用来实现 **events事件** 功能. 在log创建后, 合约是无法访问其中数据的,
但是可以方便的在区块链外部进行访问. 因为一部分 log 数据存储在 `bloom filters <https://en.wikipedia.org/wiki/Bloom_filter>`_ 中,
使得检索这些数据非常方便和安全, 也方便了极简版客户端(无需下载全部数据的端), 也可以查询这些内容.

.. index:: contract creation

Create 构建
===========

Contracts can even create other contracts using a special opcode (i.e.
they do not simply call the zero address). The only difference between
these **create calls** and normal message calls is that the payload data is
executed and the result stored as code and the caller / creator
receives the address of the new contract on the stack.

合约中可以通过特殊的 opcode 来创建其他合约(而不是直接与0地址交易). 这种创建合约的方式与直接通过消息
创建的不同是传递的数据将被执行, 然后结果以代码形式存储起来, 父合约调用者/合约创建者将获得新合约创建后存储于堆栈区的地址.

.. index:: selfdestruct

Self-destruct 自毁
==================

The only possibility that code is removed from the blockchain is
when a contract at that address performs the ``selfdestruct`` operation.
The remaining Ether stored at that address is sent to a designated
target and then the storage and code is removed from the state.

唯一能将合约代码从区块链中移除的方法是调用(合约的?还是该地址的?) ``selfdestruct`` 方法.
账户中剩余的 Ether 将转移到设计好的目标中, 最后存储区和代码区从当前状态中移除.

.. warning:: Even if a contract's code does not contain a call to ``selfdestruct``,
  it can still perform that operation using ``delegatecall`` or ``callcode``.
  尽管合约代码并不包含 ``selfdestruct`` 的调用, 也可以使用 ``delegatecall`` 或 ``callcode`` 执行相同功能.

.. note:: The pruning of old contracts may or may not be implemented by Ethereum
  clients. Additionally, archive nodes could choose to keep the contract storage
  and code indefinitely.
  老版本合约可能没有实现 Ethereum 客户端. 此外不能确定存档节点选择保留合约的存储区和代码.

.. note:: Currently **external accounts** cannot be removed from the state.
  当前外部账户不允许宠状态中删除.
