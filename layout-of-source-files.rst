********************************
Solidity源代码文件结构
********************************

Source files can contain an arbitrary number of contract definitions, include directives
and pragma directives.

源代码文件可以包含任意数量的合约定义, 包含指令和代码指令.

.. index:: ! pragma, version

.. _version_pragma:

Version Pragma 使用 Pragma 来标记版本号
========================================================

Source files can (and should) be annotated with a so-called version pragma to reject
being compiled with future compiler versions that might introduce incompatible
changes. We try to keep such changes to an absolute minimum and especially
introduce changes in a way that changes in semantics will also require changes
in the syntax, but this is of course not always possible. Because of that, it is always
a good idea to read through the changelog at least for releases that contain
breaking changes, those releases will always have versions of the form
``0.x.0`` or ``x.0.0``.

源代码首行应该包含版本号, 以防止未来版本编译器引入不兼容的更新. 我们将尽量控制不兼容性,
尤其是在更改语义同时更改语法的情况, 但无法保证未来一直兼容. 所以尽量关注大版本号的变动.
例如 ``0.x.0`` 或者 ``x.0.0``.

The version pragma is used as follows,

版本号定义语法如下,

::

  pragma solidity ^0.4.0;

Such a source file will not compile with a compiler earlier than version 0.4.0
and it will also not work on a compiler starting from version 0.5.0 (this
second condition is added by using ``^``). The idea behind this is that
there will be no breaking changes until version ``0.5.0``, so we can always
be sure that our code will compile the way we intended it to. We do not fix
the exact version of the compiler, so that bugfix releases are still possible.

使用此版本标记的代码, 将不会兼容 0.4.0 以前的编译器, 也不兼容 0.5.0 版本以后的编译器(^符号限制).
这个规则默认在 ``0.5.0`` 版本以前不会做出极度的更改, 所有我们可以保证我们代码以我们的期望运行.
我们并不指定确切的版本号, 以便更新小版本后仍可以运行.

It is possible to specify much more complex rules for the compiler version,
the expression follows those used by `npm <https://docs.npmjs.com/misc/semver>`_.

可以定义更复杂的版本号规则, 表达式规则遵循 `npm <https://docs.npmjs.com/misc/semver>`_.

.. index:: source file, ! import

.. _import:

Importing other Source Files 引入其他代码文件
========================================================

Syntax and Semantics 语法和语义
-----------------------------------

Solidity supports import statements that are very similar to those available in JavaScript
(from ES6 on), although Solidity does not know the concept of a "default export".

Solidity支持引入语句, 这与js-es6非常类似, 尽管Solidity并没有 "default export" 的概念.

At a global level, you can use import statements of the following form:

在全局变量等级, 你可以使用 import 语句, 例如:

::

  import "filename";

This statement imports all global symbols from "filename" (and symbols imported there) into the
current global scope (different than in ES6 but backwards-compatible for Solidity).

这个语句引入filename文件中所有全局的(以及从其他文件引入到filename中的)符号到当前全局环境(这与 js-es6 有些许不同).

::

  import * as symbolName from "filename";

...creates a new global symbol ``symbolName`` whose members are all the global symbols from ``"filename"``.

这样的引入方式给 ``"filename"`` 中引入的全局变量外部包装了一个命名空间, 称为 ``symbolName`` .

::

  import {symbol1 as alias, symbol2} from "filename";

...creates new global symbols ``alias`` and ``symbol2`` which reference ``symbol1`` and ``symbol2`` from ``"filename"``, respectively.

这与的引入方式将 ``"filename"`` 中的 ``symbol1`` 和 ``symbol2`` 引入, 但是 symbol1 要使用别名 ``alias`` .

Another syntax is not part of ES6, but probably convenient:

另一种语法并不是ES6的, 但是更方便 ( Python? )

::

  import "filename" as symbolName;

which is equivalent to ``import * as symbolName from "filename";``.

与 ``import * as symbolName from "filename";`` 完全相同.

Paths
-----

In the above, ``filename`` is always treated as a path with ``/`` as directory separator,
``.`` as the current and ``..`` as the parent directory.  When ``.`` or ``..`` is followed by a character except ``/``,
it is not considered as the current or the parent directory.
All path names are treated as absolute paths unless they start with the current ``.`` or the parent directory ``..``.

上面的例子中, 文件名中包含的 ``/`` 都被当做文件夹处理, ``.`` 当做当前文件夹, ``..`` 当成上一级文件夹.
当 ``.`` 或 ``..`` 后面不是 ``/`` 符号, 那么牛不当做文件夹处理, 而作为文件名的一部分, 例如 ``.env`` 是一个文件.
所有文件路径除非以 ``.`` 或 ``..`` 开头, 都以绝对路径解析.

To import a file ``x`` from the same directory as the current file, use ``import "./x" as x;``.
If you use ``import "x" as x;`` instead, a different file could be referenced
(in a global "include directory").

从相同文件夹导入 ``x`` 文件, 必须使用 ``import "./x" as x;`` 形式. 如果使用 ``import "x" as x;`` 则可能会引入完全错误的文件.

It depends on the compiler (see below) how to actually resolve the paths.
In general, the directory hierarchy does not need to strictly map onto your local
filesystem, it can also map to resources discovered via e.g. ipfs, http or git.

编译器具体如何解析路径要依照情况而定, 一般除了支持本地文件路径外, 还指出资源发现, 基于 ipfs, http 或 git 等方式.

Use in Actual Compilers 使用编译器
----------------------------------------------

When the compiler is invoked, it is not only possible to specify how to
discover the first element of a path, but it is possible to specify path prefix
remappings so that e.g. ``github.com/ethereum/dapp-bin/library`` is remapped to
``/usr/local/dapp-bin/library`` and the compiler will read the files from there.
If multiple remappings can be applied, the one with the longest key is tried first. This
allows for a "fallback-remapping" with e.g. ``""`` maps to
``"/usr/local/include/solidity"``. Furthermore, these remappings can
depend on the context, which allows you to configure packages to
import e.g. different versions of a library of the same name.

**solc**:

For solc (the commandline compiler), these remappings are provided as
``context:prefix=target`` arguments, where both the ``context:`` and the
``=target`` parts are optional (where target defaults to prefix in that
case). All remapping values that are regular files are compiled (including
their dependencies). This mechanism is completely backwards-compatible (as long
as no filename contains = or :) and thus not a breaking change. All imports
in files in or below the directory ``context`` that import a file that
starts with ``prefix`` are redirected by replacing ``prefix`` by ``target``.

So as an example, if you clone
``github.com/ethereum/dapp-bin/`` locally to ``/usr/local/dapp-bin``, you can use
the following in your source file:

::

  import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;

and then run the compiler as

.. code-block:: bash

  solc github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ source.sol

As a more complex example, suppose you rely on some module that uses a
very old version of dapp-bin. That old version of dapp-bin is checked
out at ``/usr/local/dapp-bin_old``, then you can use

.. code-block:: bash

  solc module1:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin/ \
       module2:github.com/ethereum/dapp-bin/=/usr/local/dapp-bin_old/ \
       source.sol

so that all imports in ``module2`` point to the old version but imports
in ``module1`` get the new version.

Note that solc only allows you to include files from certain directories:
They have to be in the directory (or subdirectory) of one of the explicitly
specified source files or in the directory (or subdirectory) of a remapping
target. If you want to allow direct absolute includes, just add the
remapping ``=/``.

If there are multiple remappings that lead to a valid file, the remapping
with the longest common prefix is chosen.

**Remix**:

`Remix <https://remix.ethereum.org/>`_
provides an automatic remapping for github and will also automatically retrieve
the file over the network:
You can import the iterable mapping by e.g.
``import "github.com/ethereum/dapp-bin/library/iterable_mapping.sol" as it_mapping;``.

Other source code providers may be added in the future.


.. index:: ! comment, natspec

Comments 注释
=================

Single-line comments (``//``) and multi-line comments (``/*...*/``) are possible.

单行注释 ``//`` 与 多行注释 ``/*..*/``.

::

  // This is a single-line comment.

  // 单行注释

  /*
  This is a
  multi-line comment.
  */

  /*
  多行
  注释
  */


Additionally, there is another type of comment called a natspec comment,
for which the documentation is not yet written. They are written with a
triple slash (``///``) or a double asterisk block(``/** ... */``) and
they should be used directly above function declarations or statements.
You can use `Doxygen <https://en.wikipedia.org/wiki/Doxygen>`_-style tags inside these comments to document
functions, annotate conditions for formal verification, and provide a
**confirmation text** which is shown to users when they attempt to invoke a
function.

此外有另一种注释, 单行 ``///`` , 多行 ``/**..*/`` . 必须直接在函数定义或者语句上方使用.
这一的注释可以使用 `Doxygen <https://en.wikipedia.org/wiki/Doxygen>`_ 语法来书写标签,
之后可以自动生成文档.

In the following example we document the title of the contract, the explanation
for the two input parameters and two returned values.

下面我们给一个合约书写了注释, 该方法有两个输入参数和两个返回值.

::

    pragma solidity ^0.4.0;

    /** @title 标题, Shape calculator. */
    contract shapeCalculator {
        /** @dev 计算矩形面积和周长 Calculates a rectangle's surface and perimeter.
          * @param w 矩形宽 Width of the rectangle.
          * @param h 矩形高 Height of the rectangle.
          * @return s 面积 The calculated surface.
          * @return p 周长 The calculated perimeter.
          */
        function rectangle(uint w, uint h) returns (uint s, uint p) {
            s = w * h;
            p = 2 * (w + h);
        }
    }
