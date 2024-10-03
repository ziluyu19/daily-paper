#### Tornado Cash

##### 整体架构

如下，组成该应用的角色有以下几个：

- 用户：使用该应用进行混币的发起人，可以在池子中存款、取款
- dApp：可以理解为 TornadoCash 的前端 dApp 页面，提供连接钱包、生成 zk 证明、与 relayer 或者直接与 Pool 合约交互
- relayer：用于进一步增强隐私性，代替用户重放交易的服务器
- TornadoPool：部署在以太坊上的智能合约，提供链上混币的功能

##### 电路输入

```
signal input root; // 默克尔树根
signal input nullifierHash;    // nullifier的哈希值
signal input recipient;        // 提现的接收地址（不参与电路计算）
signal input relayer;          // relayer地址（不参与电路计算）
signal input fee;              // relayer手续费（不参与电路计算）
signal input refund;           // 退款（不参与电路计算）
signal private input nullifier;   // nullifier值
signal private input secret;      // secret值
signal private input pathElements[levels];    // 默克尔树路径元素
signal private input pathIndices[levels];    // 默克尔树路径元素的index
```

##### 电路构建

```
// 计算一遍nullifierHash是否和公开信号中的一致
component hasher = CommitmentHasher();
hasher.nullifier <== nullifier;
hasher.secret <== secret;
hasher.nullifierHash === nullifierHash;  

// 利用默克尔树的路径元素和index来计算一个默克尔树根，这个树根和公开信号中的root一致
component tree = MerkleTreeChecker(levels);
tree.leaf <== hasher.commitment;
tree.root <== root;
for (var i = 0; i < levels; i++) {
    tree.pathElements[i] <== pathElements[i];
    tree.pathIndices[i] <== pathIndices[i];
}

// Add hidden signals to make sure that tampering with recipient or fee will invalidate the snark proof
// Most likely it is not required, but it's better to stay on the safe side and it only takes 2 constraints
// Squares are used to prevent optimizer from removing those constraints
signal recipientSquare;
signal feeSquare;
signal relayerSquare;
signal refundSquare;
recipientSquare <== recipient * recipient;
feeSquare <== fee * fee;
relayerSquare <== relayer * relayer;
refundSquare <== refund * refund;
```

##### 匿名加密货币转账的工作原理：mixing

多个用户将他们的加密货币提交到一个地址，将他们的存款mixing在一起，他们以存款人和取款人不能捆绑在一起的方式取款

将以太币发送到 龙卷风cash时，这是完全公开的。当您从龙卷风cash提款时，这也是完全公开的。不公开的是，所涉及的两个地址是相互关联的。

所有人都可以知道一个地址是“这个地址从龙卷风cash获得了以太币”或者“这个地址存入了 龙卷风cash”。当一个地址从龙卷风cash中提取时，人们无法分辨该加密货币来自哪个储户

##### 使用merkle树存储哈希值：

存款人：

生成两个秘密数字并根据它们的串联创建承诺哈希

提交承诺哈希

将加密货币转移到龙卷风cash

龙卷风cash：

在存款阶段，将承诺哈希添加到默克尔树中

提款人：

为 Merkle 根生成有效的 Merkle 证明

从两个秘密数字生成承诺哈希

生成上述计算的zk证明

向龙卷风cash提交证明

##### Nullifier方案

为了防止多次提款，智能合约使用所谓的“无效方案”

在提款期间，用户必须提交无效符的哈希值，即nullifierHash和证明，他们将nullifier和secret连接起来并对其进行哈希处理以产生叶子之一。然后，智能合约可以验证（使用零知识算法）发送者确实知道无效者哈希的原像证明

将nullifier哈希添加到映射中以确保它永远不会被复用

用户必须证明：

他们知道叶子也原像

之前没有使用过nullifier(这是一个简单的 Solidity 映射，不是 zk 验证步骤）

他们可以产生nullifier哈希和nullifier的原像

##### 将哈希函数部署为原始字节码

circomlib js 存储库包含用于创建原始字节码哈希的 JavaScript 工具。这是生成[MiMC](https://github.com/iden3/circomlibjs/blob/main/src/mimcsponge_gencontract.js)和[Poseidon Hash](https://github.com/iden3/circomlibjs/blob/main/src/poseidon_gencontract.js)的代码

##### 从龙卷风cash中提取：

用户必须使用[updateTree 脚本](https://github.com/tornadocash/tornado-classic-ui/blob/master/scripts/updateTree.js)在本地重建 Merkle 树。该脚本将下载所有相关的[Solidity 事件](https://www.rareskills.io/post/ethereum-events)并重建 Merkle 树。然后，用户将生成 Merkle 证明和叶承诺原像的零知识证明。如前所述，Tornado Cash 存储最后 30 个 Merkle 根，因此用户有足够的时间提交证明。

![tornado cash withdraw workflow diagram](https://devel-82-v2.tongkolspace.com/rareskills/wp-content/uploads/2024/09/935a00_e392067a72614272907a69b012da4898~mv2.png)