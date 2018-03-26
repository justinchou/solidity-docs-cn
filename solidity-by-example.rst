#######################
根据例子学习Solidity
#######################

.. index:: voting, ballot

.. _voting:

******
投票
******

The following contract is quite complex, but showcases
a lot of Solidity's features. It implements a voting
contract. Of course, the main problems of electronic
voting is how to assign voting rights to the correct
persons and how to prevent manipulation. We will not
solve all problems here, but at least we will show
how delegated voting can be done so that vote counting
is **automatic and completely transparent** at the
same time.

下面的合约并不复杂, 但是引入了Solidity的很多特性. 实现了投票合约.
当然, 电子选票主要功能就是给人设置正确的选举权, 以及如何防止篡改.
我们不处理所有的问题, 但至少我们会实现委托选举, 以便同时实现自动唱票和统计透明.

The idea is to create one contract per ballot,
providing a short name for each option.
Then the creator of the contract who serves as
chairperson will give the right to vote to each
address individually.

实现的思路就是每个票创建一个单独的合约, 为每个选项提供一个简称.
然后合约创建者作为公正者, 给每个地址单独的授予投票权.

The persons behind the addresses can then choose
to either vote themselves or to delegate their
vote to a person they trust.

地址所有者就可以选择是自己投票, 还是将投票权代理给信任的人让其替自己投票.

At the end of the voting time, ``winningProposal()``
will return the proposal with the largest number
of votes.

最后, ``winningProposal()`` 将返回最多得票数的候选人.

::

    pragma solidity ^0.4.16;

    /// @title Voting with delegation. 支持代理的选举系统
    contract Ballot {
        // This declares a new complex type which will
        // be used for variables later.
        // It will represent a single voter.
        // 声明一个复杂的数据类型, 用于后面定义变量, 代表一个投票人.
        struct Voter {
            uint weight; // weight is accumulated by delegation, 权重基于代理来积累
            bool voted;  // if true, that person already voted, 如果为真表示已经投过票
            address delegate; // person delegated to, 将自己的投票权代理给了谁
            uint vote;   // index of the voted proposal, 投票给谁了
        }

        // This is a type for a single proposal.
        // 这是候选人的数据类型
        struct Proposal {
            bytes32 name;   // short name (up to 32 bytes), 简称, 最长32个字符
            uint voteCount; // number of accumulated votes, 获得的总票数
        }

        // 创建者地址, 需要在构造函数中进行赋值
        address public chairperson;

        // This declares a state variable that
        // stores a `Voter` struct for each possible address.
        // 声明一个状态变量, 用于存储所有的投票者
        mapping(address => Voter) public voters;

        // A dynamically-sized array of `Proposal` structs.
        // 一个变长数组, 存储候选人列表
        Proposal[] public proposals;

        /// Create a new ballot to choose one of `proposalNames`.
        /// 根据候选人名单列表, 创建一个选举
        function Ballot(bytes32[] proposalNames) public {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;

            // For each of the provided proposal names,
            // create a new proposal object and add it
            // to the end of the array.
            // 循环候选人名字列表, 创建对应的候选人对象, 加入候选人对象列表
            for (uint i = 0; i < proposalNames.length; i++) {
                // `Proposal({...})` creates a temporary
                // Proposal object and `proposals.push(...)`
                // appends it to the end of `proposals`.
                // 使用 `Proposal({...})` 根据结构体创建对象, `proposals.push(...)` 方法向数组增加元素.
                proposals.push(Proposal({
                    name: proposalNames[i],
                    voteCount: 0
                }));
            }
        }

        // Give `voter` the right to vote on this ballot.
        // May only be called by `chairperson`.
        // 给选民赋权, 只允许公正者进行操作
        function giveRightToVote(address voter) public {
            // If the argument of `require` evaluates to `false`,
            // it terminates and reverts all changes to
            // the state and to Ether balances. It is often
            // a good idea to use this if functions are
            // called incorrectly. But watch out, this
            // will currently also consume all provided gas
            // (this is planned to change in the future).
            // require 的参数为false, 程序终止并且回滚更改. 一般在判断程序是否出错时使用.
            // 但值得注意的是, 这将消耗掉所有提供的 gas(这一点将在新版本做出改进).
            require((msg.sender == chairperson) && !voters[voter].voted && (voters[voter].weight == 0));
            voters[voter].weight = 1;
        }

        /// Delegate your vote to the voter `to`.
        /// 将投票权代理给其他人.
        function delegate(address to) public {
            // assigns reference
            // 获取当前用户的引用
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);

            // Self-delegation is not allowed.
            // 不能将全力代理给自己
            require(to != msg.sender);

            // Forward the delegation as long as
            // `to` also delegated.
            // In general, such loops are very dangerous,
            // because if they run too long, they might
            // need more gas than is available in a block.
            // In this case, the delegation will not be executed,
            // but in other situations, such loops might
            // cause a contract to get "stuck" completely.
            // 如果被授予代理权的人已经将全力代理出去, 将代理权授予给上一级代理人.
            // 一般情况, 这样的循环很危险, 因为如果运行时间太久, 将耗尽区块中可用的 gas, 那么代理将不被执行,
            // 另一种情况就是这样的循环将造成合约的假死现象.
            while (voters[to].delegate != address(0)) {
                to = voters[to].delegate;

                // We found a loop in the delegation, not allowed.
                // 发现循环代理, 不允许
                require(to != msg.sender);
            }

            // Since `sender` is a reference, this
            // modifies `voters[msg.sender].voted`
            // 由于 `sender` 是一个引用, 这将直接更改 `voters[msg.sender].voted` 的值
            sender.voted = true;
            sender.delegate = to;
            Voter storage delegate = voters[to];
            if (delegate.voted) {
                // If the delegate already voted,
                // directly add to the number of votes
                // 如果代理人已经投过票了, 直接给曾经选举的候选人加票
                proposals[delegate.vote].voteCount += sender.weight;
            } else {
                // If the delegate did not vote yet,
                // add to her weight.
                // 如果代理人没有投过票, 增加代理人的投票权重.
                delegate.weight += sender.weight;
            }
        }

        /// Give your vote (including votes delegated to you)
        /// to proposal `proposals[proposal].name`.
        /// 将选票(包含他人代理的选票)投给 `proposals[proposal].name`
        function vote(uint proposal) public {
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);
            sender.voted = true;
            sender.vote = proposal;

            // If `proposal` is out of the range of the array,
            // this will throw automatically and revert all
            // changes.
            // 如果候选人标号超出数组长度, 将自动回滚所有操作.
            proposals[proposal].voteCount += sender.weight;
        }

        /// @dev Computes the winning proposal taking all
        /// previous votes into account.
        /// @dev 计算最终赢家将所有选票列入账户.
        function winningProposal() public view returns (uint winningProposal)
        {
            uint winningVoteCount = 0;
            for (uint p = 0; p < proposals.length; p++) {
                if (proposals[p].voteCount > winningVoteCount) {
                    winningVoteCount = proposals[p].voteCount;
                    winningProposal = p;
                }
            }
        }

        // Calls winningProposal() function to get the index
        // of the winner contained in the proposals array and then
        // returns the name of the winner
        // 调用 winningProposal() 方法获取最终赢家的序号, 返回赢家名字
        function winnerName() public view returns (bytes32 winnerName)
        {
            winnerName = proposals[winningProposal()].name;
        }
    }

Possible Improvements 可能的改进
==================================

Currently, many transactions are needed to assign the rights
to vote to all participants. Can you think of a better way?

当前, 有许多交易, 需要给大量参与者一个个授权, 有更好的办法么?

.. index:: auction;blind, auction;open, blind auction, open auction

****************
秘密竞价（盲拍）
****************

In this section, we will show how easy it is to create a
completely blind auction contract on Ethereum.
We will start with an open auction where everyone
can see the bids that are made and then extend this
contract into a blind auction where it is not
possible to see the actual bid until the bidding
period ends.

本节我们将展示如何基于 Ethereum 轻松的创建一个完整的盲拍合约. 我们将开始将创建一个开放的竞价, 每个人都可以看到报价,
然后再一步步改成盲拍, 成交前隐藏真实的竞价.

.. _simple_auction:

Simple Open Auction 简单的公开竞价拍卖
=======================================

The general idea of the following simple auction contract
is that everyone can send their bids during
a bidding period. The bids already include sending
money / ether in order to bind the bidders to their
bid. If the highest bid is raised, the previously
highest bidder gets her money back.
After the end of the bidding period, the
contract has to be called manually for the
beneficiary to receive his money - contracts cannot
activate themselves.

一般的思路实现简单的竞价合约是每个人可以在竞价周期内出价. 出价数据已经包含了金额, 和出价人.
如果出现最高价格, 前一个最高出价者的资金就会返回到账户中. 在竞价结束后,
需要手动调用合约方法让受益人收到资金, 合约无法自动执行.

::

    pragma solidity ^0.4.11;

    contract SimpleAuction {
        // Parameters of the auction. Times are either
        // absolute unix timestamps (seconds since 1970-01-01)
        // or time periods in seconds.
        // 拍卖参数, 时间以时间戳或者倒计时剩余时间秒数计
        address public beneficiary;
        uint public auctionEnd;

        // Current state of the auction.
        // 当前拍卖状态, 最高出价者 & 最高价格
        address public highestBidder;
        uint public highestBid;

        // Allowed withdrawals of previous bids
        // 前面失去最高价的人的竞价历史
        mapping(address => uint) pendingReturns;

        // Set to true at the end, disallows any change
        // 结束标记, 不再允许修改
        bool ended;

        // Events that will be fired on changes.
        // 修改时触发的事件
        event HighestBidIncreased(address bidder, uint amount);
        event AuctionEnded(address winner, uint amount);

        // The following is a so-called natspec comment,
        // recognizable by the three slashes.
        // It will be shown when the user is asked to
        // confirm a transaction.

        /// Create a simple auction with `_biddingTime`
        /// seconds bidding time on behalf of the
        /// beneficiary address `_beneficiary`.
        /// 初始化一个简单的拍卖, 参数 `_biddingTime` 表示持续时长, `_beneficiary` 表示最终受益人.
        function SimpleAuction(
            uint _biddingTime,
            address _beneficiary
        ) public {
            beneficiary = _beneficiary;
            auctionEnd = now + _biddingTime;
        }

        /// Bid on the auction with the value sent
        /// together with this transaction.
        /// The value will only be refunded if the
        /// auction is not won.
        /// 竞价时将竞价款随消息一起发送. 价款只在竞价失败时退还.
        function bid() public payable {
            // No arguments are necessary, all
            // information is already part of
            // the transaction. The keyword payable
            // is required for the function to
            // be able to receive Ether.
            // 无需参数, 所有信息都包含在了交易之中. 关键字payable用于声明方法支持接收 Ether.

            // Revert the call if the bidding
            // period is over.
            // 如果竞拍已经结束, 那么回滚
            require(now <= auctionEnd);

            // If the bid is not higher, send the
            // money back.
            // 如果出价已经非最高价, 退款
            require(msg.value > highestBid);

            if (highestBidder != 0) {
                // Sending back the money by simply using
                // highestBidder.send(highestBid) is a security risk
                // because it could execute an untrusted contract.
                // It is always safer to let the recipients
                // withdraw their money themselves.
                // 直接将款项退回到 highestBidder.send(highestBid) 是有安全风险的,
                // 因为可能触发一个不受信的合约. 让收款人自己提款更安全.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBidder = msg.sender;
            highestBid = msg.value;
            HighestBidIncreased(msg.sender, msg.value);
        }

        /// Withdraw a bid that was overbid.
        /// 当竞价被超越时, 收回出价款
        function withdraw() public returns (bool) {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `send` returns.
                // 将数据置空很重要, 防止此方法在获取结果前被多次调用.
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)) {
                    // No need to call throw here, just reset the amount owing
                    // 不成功无需出发错误, 只需要将数额回滚即可
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        /// 竞拍结束, 将最高价金额发送给受益人
        function auctionEnd() public {
            // It is a good guideline to structure functions that interact
            // with other contracts (i.e. they call functions or send Ether)
            // into three phases:
            // 1. checking conditions
            // 2. performing actions (potentially changing conditions)
            // 3. interacting with other contracts
            // If these phases are mixed up, the other contract could call
            // back into the current contract and modify the state or cause
            // effects (ether payout) to be performed multiple times.
            // If functions called internally include interaction with external
            // contracts, they also have to be considered interaction with
            // external contracts.
            // 这是很好的一个函数与其他合约交互的例子(调用函数或者交易Ether), 通过三步:
            // 1. 判断条件
            // 2. 执行操作, 包括更改当前的一些状态
            // 3. 与其他合约交互
            // 如果步骤搞混了, 其他合约将有可能通过多次回调到本合约, 修改合约状态, 或者引起恶性效果(账户被提空).
            // 如果有需要内部调用的方法需要考虑与外部的合约进行交互.

            // 1. Conditions, 条件
            require(now >= auctionEnd); // auction did not yet end, 竞拍结束时间到, 个人觉得原英文注释略有问题.
            require(!ended); // this function has already been called, 当前还未执行过结束方法.

            // Q&A: 底层是单进程模型还是多进程模型? 此方法是否是进程安全的? 高并发下 require(!ended); 会不会有问题?

            // 2. Effects, 操作
            ended = true;
            AuctionEnded(highestBidder, highestBid);

            // 3. Interaction, 交互
            beneficiary.transfer(highestBid);
        }
    }

Blind Auction 盲拍
==================

The previous open auction is extended to a blind auction
in the following. The advantage of a blind auction is
that there is no time pressure towards the end of
the bidding period. Creating a blind auction on a
transparent computing platform might sound like a
contradiction, but cryptography comes to the rescue.

下面将前面的公开竞价改成盲拍. 盲拍的优势在于拍卖结束前的一小段时间内没有突然的压力.
在一个公开的网络数据环境下创建盲拍似乎有些矛盾, 但是密码学使这得以实现.

During the **bidding period**, a bidder does not
actually send her bid, but only a hashed version of it.
Since it is currently considered practically impossible
to find two (sufficiently long) values whose hash
values are equal, the bidder commits to the bid by that.
After the end of the bidding period, the bidders have
to reveal their bids: They send their values
unencrypted and the contract checks that the hash value
is the same as the one provided during the bidding period.

在盲拍阶段, 竞价者并不是真的发送竞价, 只是发送价格的哈希值.
由于当前很难知道两个足够大的值使其哈希值相等, 竞价者据此来竞价.
在最后阶段, 竞价者需要上传未加密的竞价信息, 合约校验竞价信息的哈希值,
是否与原来提供的哈希值相等, 以判断盲拍阶段的竞价信息是否有效.

Another challenge is how to make the auction
**binding and blind** at the same time: The only way to
prevent the bidder from just not sending the money
after he won the auction is to make her send it
together with the bid. Since value transfers cannot
be blinded in Ethereum, anyone can see the value.

另一个有挑战的问题是如何同时保证竞拍的 **约束力和私密性**.
唯一能保证竞拍者拍中之后不毁约的方法就是竞拍时将价款发送过来,
但是 Ethereum 中消息的传递无法私密性, 任何人都可以看见传递的数据.

The following contract solves this problem by
accepting any value that is larger than the highest
bid. Since this can of course only be checked during
the reveal phase, some bids might be **invalid**, and
this is on purpose (it even provides an explicit
flag to place invalid bids with high value transfers):
Bidders can confuse competition by placing several
high or low invalid bids.

下面的合约通过接受任意高于当前最高价的值解决了这个问题. 由于校验只有到了
验证环节才能揭晓结果, 竞标者可能是故意做的虚假竞价来迷惑对手, 方式是
通过多次虚假出价(几个高价和几个低价)之间夹杂一个真实出价.


::

    pragma solidity ^0.4.11;

    contract BlindAuction {
        struct Bid {
            bytes32 blindedBid;
            uint deposit;
        }

        address public beneficiary;
        uint public biddingEnd;
        uint public revealEnd;
        bool public ended;

        mapping(address => Bid[]) public bids;

        address public highestBidder;
        uint public highestBid;

        // Allowed withdrawals of previous bids
        // 前面失去最高价的人的竞价历史
        mapping(address => uint) pendingReturns;

        event AuctionEnded(address winner, uint highestBid);

        /// Modifiers are a convenient way to validate inputs to
        /// functions. `onlyBefore` is applied to `bid` below:
        /// The new function body is the modifier's body where
        /// `_` is replaced by the old function body.
        /// modifier 修饰符使对函数输入校验变得方便.
        /// `onlyBefore` 用于修饰下面的 `bid` 方法,
        /// 添加修饰符后, 相当于将原方法放在修饰方法的 `_` 位置, 并且返回修饰方法.
        /// 调用原方法就相当于调用新方法.
        modifier onlyBefore(uint _time) { require(now < _time); _; }
        modifier onlyAfter(uint _time) { require(now > _time); _; }

        function BlindAuction(
            uint _biddingTime,
            uint _revealTime,
            address _beneficiary
        ) public {
            beneficiary = _beneficiary;
            biddingEnd = now + _biddingTime;
            revealEnd = biddingEnd + _revealTime;
        }

        /// Place a blinded bid with `_blindedBid` = keccak256(value,
        /// fake, secret).
        /// The sent ether is only refunded if the bid is correctly
        /// revealed in the revealing phase. The bid is valid if the
        /// ether sent together with the bid is at least "value" and
        /// "fake" is not true. Setting "fake" to true and sending
        /// not the exact amount are ways to hide the real bid but
        /// still make the required deposit. The same address can
        /// place multiple bids.
        /// 竞价时发送盲拍值 `_blindedBid` = keccak256(value, fake, secret).
        /// 发送过来的 ether 仅在叫价验证该 `_blindedBid` 正确时被退回, 校验失败该笔价款不退.
        /// 只在校验真实(!fake)出价小于等于预付价款时叫价有效.
        /// 设置fake为真, 或者真实出价大于预付价款, 是两种迷惑真实交易的方法, 但是仍然需要先交竞价款.
        /// 同一竞标者可以多次出价.
        function bid(bytes32 _blindedBid)
            public
            payable
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

        /// Reveal your blinded bids. You will get a refund for all
        /// correctly blinded invalid bids and for all bids except for
        /// the totally highest.
        /// 验证盲拍交易合法性. 验证结束后将得到退款(所有正确验证的迷惑交易
        /// 价款, 和超出竞价的预付款), 但不包括最高出价价款.
        function reveal(
            uint[] _values,
            bool[] _fake,
            bytes32[] _secret
        )
            public
            onlyAfter(biddingEnd)
            onlyBefore(revealEnd)
        {
            uint length = bids[msg.sender].length;
            require(_values.length == length);
            require(_fake.length == length);
            require(_secret.length == length);

            uint refund;
            for (uint i = 0; i < length; i++) {
                var bid = bids[msg.sender][i];
                var (value, fake, secret) =
                        (_values[i], _fake[i], _secret[i]);
                if (bid.blindedBid != keccak256(value, fake, secret)) {
                    // Bid was not actually revealed.
                    // Do not refund deposit.
                    // 如果哈希值校验失败, 非有效交易, 不退款.
                    continue;
                }
                refund += bid.deposit;
                // fake 允许虚拟出价, 先付款后拿回来;
                // bid.deposit 是预付款, value 是真实竞价款, 如果预付款少那么不允许竞价.
                // todo: 是否搞一个多次预付款总额的比较? 这样是否更盲拍?
                if (!fake && bid.deposit >= value) {
                    if (placeBid(msg.sender, value))
                        refund -= value;
                }
                // Make it impossible for the sender to re-claim
                // the same deposit.
                // 清零该笔交易的校验码, 让用户无法重复验证.
                bid.blindedBid = bytes32(0);
            }

            // 退款
            msg.sender.transfer(refund);
        }

        // This is an "internal" function which means that it
        // can only be called from the contract itself (or from
        // derived contracts).
        // "internal" 这意味着是一个内部方法, 只能通过自身或者继承合约来调用
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid) {
                return false;
            }
            if (highestBidder != 0) {
                // Refund the previously highest bidder.
                // 将原来最高报价数额退还到原最高报价者账号
                pendingReturns[highestBidder] += highestBid;
            }
            highestBid = value;
            highestBidder = bidder;
            return true;
        }

        /// Withdraw a bid that was overbid.
        /// 失败的竞标提取现金
        function withdraw() public {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `transfer` returns (see the remark above about
                // conditions -> effects -> interaction).
                // 防止重复调用提款
                pendingReturns[msg.sender] = 0;

                msg.sender.transfer(amount);
            }
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        /// 结束竞标, 将最高标价金额发给受益人
        function auctionEnd()
            public
            onlyAfter(revealEnd)
        {
            require(!ended);
            AuctionEnded(highestBidder, highestBid);
            ended = true;
            beneficiary.transfer(highestBid);
        }
    }


.. index:: purchase, remote purchase, escrow

********************
安全的远程购买
********************

::

    pragma solidity ^0.4.11;

    contract Purchase {
        uint public value;
        address public seller;
        address public buyer;
        enum State { Created, Locked, Inactive }
        State public state;

        // Ensure that `msg.value` is an even number.
        // Division will truncate if it is an odd number.
        // Check via multiplication that it wasn't an odd number.
        // 保证 `msg.value` 是一个偶数.
        // 定义 value 是 uint 类型, 将浮点型赋值给uint会舍去小数部分.
        // 使用乘法校验是否 `msg.value` 是偶数.
        function Purchase() public payable {
            seller = msg.sender;
            value = msg.value / 2;
            require((2 * value) == msg.value);
        }

        modifier condition(bool _condition) {
            require(_condition);
            _;
        }

        modifier onlyBuyer() {
            require(msg.sender == buyer);
            _;
        }

        modifier onlySeller() {
            require(msg.sender == seller);
            _;
        }

        modifier inState(State _state) {
            require(state == _state);
            _;
        }

        event Aborted();
        event PurchaseConfirmed();
        event ItemReceived();

        /// Abort the purchase and reclaim the ether.
        /// Can only be called by the seller before
        /// the contract is locked.
        /// 终止付费并且要回价款, 只能在合约锁定前通过卖家调用
        function abort()
            public
            onlySeller
            inState(State.Created)
        {
            Aborted();
            state = State.Inactive;
            seller.transfer(this.balance);
        }

        /// Confirm the purchase as buyer.
        /// Transaction has to include `2 * value` ether.
        /// The ether will be locked until confirmReceived
        /// is called.
        /// 买家确认付款. 交易必须包含 `2 * value` 价款.
        /// 价款将锁定直到调用 confirmReceived.
        function confirmPurchase()
            public
            inState(State.Created)
            condition(msg.value == (2 * value))
            payable
        {
            PurchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }

        /// Confirm that you (the buyer) received the item.
        /// This will release the locked ether.
        /// 买家确认收货, 解锁锁定的价款
        function confirmReceived()
            public
            onlyBuyer
            inState(State.Locked)
        {
            ItemReceived();
            // It is important to change the state first because
            // otherwise, the contracts called using `send` below
            // can call in again here.
            // 首先需要解锁, 因为后面的一些操作会判断该值
            state = State.Inactive;

            // NOTE: This actually allows both the buyer and the seller to
            // block the refund - the withdraw pattern should be used.
            // 注意: 现在本方法实际上买家和卖家都有权利调用. 应该扩展成提款模式.

            buyer.transfer(value);
            seller.transfer(this.balance);
        }
    }

********************
微支付通道
********************

To be written.

待完善.