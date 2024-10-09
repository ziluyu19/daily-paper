将数组 in[nElements] 和索引`index`作为输入，并输出 in[index] 的值

```
include "CalculateTotal.circom"

template QuinSelector(choices) {
    signal input in[choices];
    signal input index;
    signal output out;
    
    // 确保索引小于选项
    component lessThan = LessThan(4);
    lessThan.in[0] <== index;
    lessThan.in[1] <== choices;
    lessThan.out === 1;

    component calcTotal = CalculateTotal(choices);
    component eqs[choices];

    // 对于每个项目，检查其索引是否等于输入索引
    for (var i = 0; i < choices; i ++) {
        eqs[i] = IsEqual();
        eqs[i].in[0] <== i;
        eqs[i].in[1] <== index;

        // 如果索引匹配，则 eqs[i].out 为 1
        // calcTotal is not 0.
        calcTotal.in[i] <== eqs[i].out * in[i];
    }

    out <== calcTotal.out;
}
```



##### ZK-group-sigs

让参与者能够在不放弃隐私的情况下证明自己的可信度

第一个电路 `sign` 是最重要的，并且是模拟群签名协议严格必需的唯一电路。该协议可以通过`revealSigner`和`denySigning`电路作为附加组件进行增强。

sign:在证明组成员身份的同时证明消息

revealSigner:证明您的特定密钥用于签署特定的组签名消息

denySigning:证明您的特定密钥*未*用于签署特定的组签名消息

##### sign:

首先获取用户的秘密并对其应用MIMC哈希（SNARK 友好的哈希），从而派生公共哈希，

然后，获取用户的公共哈希值并将其与所有存在的公共哈希值列表进行比较。一旦验证公共哈希确实存在于该列表中，这意味着用户是该组的有效成员，就使用密钥“签署”预期消息并将其返回

```

/*
  Inputs:
  - hash1 (pub)
  - hash2 (pub)
  - hash3 (pub)
  - msg (pub)
  - secret

  Intermediate values:
  - x (supposed to be hash of secret)
  
  Output:
  - msgAttestation
  
  Prove:
  - mimc(secret) == x
  - (x - hash1)(x - hash2)(x - hash3) == 0
  - msgAttestation == mimc(msg, secret)
*/

template Main() {
  signal private input secret;
  signal input hash1;
  signal input hash2;
  signal input hash3;
  signal input msg;

  signal x;

  signal output msgAttestation;
//电路的第一部分将 MiMC 哈希应用于用户的密钥，并将结果（公共哈希）存储在`myHash`信号中
  component mimcSecret = MiMCSponge(1, 220, 1);
  mimcSecret.ins[0] <== secret;
  mimcSecret.k <== 0;
  x <== mimcSecret.outs[0];
//检查myHash（用户密钥的 MiMC 哈希）是否等于公共哈希之一。如果是这样，则意味着该用户是该组的一部分，正在验证成员身份
  signal temp;
  temp <== (x - hash1) * (x - hash2);
  0 === temp * (x - hash3);
//通过使用用户的密钥对消息应用 MiMC 哈希，我们对消息进行签名并将其返回。输出取决于用户想要证明的特定消息，以防止重放攻击；如果没有这个，相同的证据可以用来证明不同的消息
  component mimcAttestation = MiMCSponge(2, 220, 1);
  mimcAttestation.ins[0] <== msg;
  mimcAttestation.ins[1] <== secret;
  mimcAttestation.k <== 0;
  msgAttestation <== mimcAttestation.outs[0];
}

component main = Main();
```

首先声明一些输入，一个是私有的，其余的是公共的。 `hash1` 、 `hash2` 、 `hash3`是代表该组的其他用户的公共哈希值。 `secret`是发送者的密钥， `msg`是用户证明的消息。

##### revealSigner:

用来证明一个人的特定密钥被用来生成消息的组签名，从而允许用户以加密方式暴露自己

代码：https://github.com/0xPARC/zk-group-sigs/blob/6c68294897149c65ae11a6fc283fa28cc6a089e5/circuits/reveal.circom

该电路与sign电路类似。首先使用 MiMC 验证用户密钥的哈希值确实是公钥。然后验证消息和用户密钥之间的 MiMC 哈希是否是消息证明
这证明该消息已由正确的用户正确签名

##### denySigning：

允许用户拒绝特定消息的所有权。例如，如果匿名成员发送了不当内容，其他成员可能会想要拒绝该消息，以洗清自己的罪名并挑出肇事者

该电路与revealSigner电路几乎完全相同，只是不是证明用户发送了消息，而是证明用户没有发送消息。输入与上述电路相同，而证明语句略有修改：

```
msgAttestation != MiMC(msg, secret)
myHash == MiMC(secret)
```

