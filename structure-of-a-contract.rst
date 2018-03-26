.. index:: contract, state variable, function, event, struct, enum, function;modifier

.. _contract_structure:

***********************
合约的结构
***********************

Contracts in Solidity are similar to classes in object-oriented languages.
Each contract can contain declarations of :ref:`structure-state-variables`, :ref:`structure-functions`,
:ref:`structure-function-modifiers`, :ref:`structure-events`, :ref:`structure-struct-types` and :ref:`structure-enum-types`.
Furthermore, contracts can inherit from other contracts.

Solidity合约与基于对象的语言定义类似, 每个合约可以包含 :ref:`structure-state-variables` 状态变量, :ref:`structure-functions` 函数,
:ref:`structure-function-modifiers` 修饰符, :ref:`structure-events` 事件, :ref:`structure-struct-types` 结构体类型以及 :ref:`structure-enum-types` 枚举.
此外, 合约还可以继承其他合约.

.. _structure-state-variables:

State Variables 状态变量
==============================

State variables are values which are permanently stored in contract storage.
状态变量是永久存储于合约之中的值.

::

    pragma solidity ^0.4.0;

    contract SimpleStorage {
        uint storedData; // State variable 状态变量
        // ...
    }

See the :ref:`types` section for valid state variable types and
:ref:`visibility-and-getters` for possible choices for
visibility.

查看 :ref:`types` 类型部分, 看看支持哪些变量类型.
查看 :ref:`visibility-and-getters` 部分看看如何访问变量.

.. _structure-functions:

Functions 函数
===================

Functions are the executable units of code within a contract.

函数是合约中可运行的计算单元.

::

    pragma solidity ^0.4.0;

    contract SimpleAuction {
        function bid() public payable { // Function
            // ...
        }
    }

:ref:`function-calls` can happen internally or externally
and have different levels of visibility (:ref:`visibility-and-getters`)
towards other contracts.

:ref:`function-calls` 函数可以在内部/外部调用, 函数面向其他合约有不同等级的访问权限(:ref:`visibility-and-getters`)

.. _structure-function-modifiers:

Function Modifiers 函数修饰符
====================================

Function modifiers can be used to amend the semantics of functions in a declarative way
(see :ref:`modifiers` in contracts section).

修饰符通过声明的方式修改函数语义. (参考 :ref:`modifiers` 修饰符一章).

::

    pragma solidity ^0.4.11;

    contract Purchase {
        address public seller;

        modifier onlySeller() { // Modifier
            require(msg.sender == seller);
            _;
        }

        function abort() public onlySeller { // Modifier usage
            // ...
        }
    }

.. _structure-events:

Events 事件
==================

Events are convenience interfaces with the EVM logging facilities.

事件是实现EVM的日志功能的一个方便的接口.

::

    pragma solidity ^0.4.0;

    contract SimpleAuction {
        event HighestBidIncreased(address bidder, uint amount); // Event

        function bid() public payable {
            // ...
            HighestBidIncreased(msg.sender, msg.value); // Triggering event
        }
    }

See :ref:`events` in contracts section for information on how events are declared
and can be used from within a dapp.

参考 :ref:`events` 章节了解详细内容, 关于事件如何定义, 如何在dapp中使用.

.. _structure-struct-types:

Struct Types 结构体类型
=======================================

Structs are custom defined types that can group several variables (see
:ref:`structs` in types section).

结构体是自定义数据类型, 将不同类型的变量组装成一个组. 参考 :ref:`structs` 部分了解详细内容.

::

    pragma solidity ^0.4.0;

    contract Ballot {
        struct Voter { // Struct
            uint weight;
            bool voted;
            address delegate;
            uint vote;
        }
    }

.. _structure-enum-types:

Enum Types 枚举类型
==============================

Enums can be used to create custom types with a finite set of values (see
:ref:`enums` in types section).

枚举用于创建自定义类型, 基于值的有限集合, 参考 :ref:`enums` 部分.

::

    pragma solidity ^0.4.0;

    contract Purchase {
        enum State { Created, Locked, Inactive } // Enum
    }
