#### Groth16验证设置系统(非完全版)

##### BigNumber 函数已被弃用，需要使用 toBigInt 函数(补给上一篇)

Groth16 没有使用“系数知识”（即在证明中每个多项式需要两个群元素），而是使用秘密域元素 α、β 来强制 A、B 和 C 使用相同的向量**w**。另外两个秘密字段元素 γ、δ 用于使公共输入独立于其他见证组件

ceremony文件中体现的随机性也体现在ZKey文件中

电路执行的input.json,通过讨论的转换进行处理，并被映射和评估到椭圆曲线上，一遍发送给验证者进行验证

确保在你为该文件贡献额外随机性之前，ceremony文件没有以任何方式被破坏

##### 实现一个电路，设置对这个电路的zkproof验证系统，生成证明并将其发送到链上的solidity后端(终端命令行操作)

##### Groth16的程序设置

1.通过仪式创建一个可信设置，Groth16的这个仪式称之为powers of Tau，使用bn128曲线创建一个新的仪式。将这个仪式文件对象传递给大量第三方，他们都将为这个仪式对象贡献额外的随机性，具体数字是随机的。这些随机值之后需要被删除。
2.要使用这个仪式文件为特定的电路设置验证设置，你需要以某种方式将它们交织在一起，在第二阶段设置特定方式。首先获得最终的仪式文件，然后将它与想设置的电路交织在一起，获取到一个zkey文件，这将是你的证明密钥，每次想为特定电路生成证明时都会需要它。
3.生成证明，需要电路、电流执行的输入json和ZKey文件

使用上一篇MIMC的代码

##### 1.创建一个仪式文件

```
snarkjs powersoftau new bn128 12 ceremoy_0000.ptau -v
```

12代表最大约束数。当我们编译电路时，会给你一个输出，详细说明电路有多少约束。12代表电路可以拥有的最大约束数是2^12。

最后命名为ceremoy_0000.ptau,将数字附加到仪式词的末尾是传统做法，这样就可以跟踪哪些参与者为这个仪式文件贡献了随机性。

##### 2.为仪式文件贡献随机性

你会将这个生成文件传递给其他人，他们将它们选择的随机性贡献给仪式文件，你不应该知道其他参与者的贡献。贡献之后这些数字和输入应该被丢弃，因为我们只需要最后的生成文件。

```
snarkjs powersoftau contribute ceremoy_0000.ptau ceremoy_0001.ptau -v
```

输入随机数或者文字完成后，删除 ceremoy_0000.ptau，继续提供随机性，多次操作

```
snarkjs powersoftau contribute ceremoy_0001.ptau ceremoy_0002.ptau -v
```

从别人那里获取到仪式文件时可以验证完整性，因为你需要确保为该文件贡献随机性之前，仪式文件没有以任何方式被破坏。

##### 验证仪式文件的完整性

```
snarkjs powersoftau verify ceremoy_0002.ptau
```

应打印出 Powers of tau file OK!

##### 文件随机性贡献完成，生成最终文件

```
snarkjs powersoftau prepare phase2 ceremoy_0002.ptau ceremoy_final.ptau -v
```

它将计算用来测试多项式的一组随机挑战

##### 验证最后的生成文件

```
snarkjs powersoftau verify ceremoy_final.ptau 
```

##### 3.使用Groth16设置，将仪式文件与电路交织在一起：

编译电路成R1CS，circom circuit.citcom --r1cs

生成密钥

```
snarkjs groth16 setup circuit.r1cs ceremony_final.ptau setup_0000.zkey
```

snarkjs groth16 电路在rics中的表示形式 仪式并命名输出ZKey文件

这个过程将提供一个Zkey，证明文件

为这个ZKey文件贡献一次额外的随机性

```
snarkjs zkey contribute setup_0000.zkey setup_final.zkey
```

##### 验证ZKey文件：

```
snarkjs zkey verify circuit.r1cs ceremony_final.ptau setup_final.zkey
```

##### 4.构建证明：

添加输入元素ison文件：(随机数)

```
{
    "x": "123456789019012345678901234",
    "k": "132456326574"
}
```

编译电路成web assembly形式：circom circuit.citcom --wasm

生成proof：

```
snarkjs groth16 fullprove input.json circuit.js/circuit.wasm setup_final.zkey proof.json public.json
```

电路表示形式wasm表示形式 设置zkey 命名outputs，证明本身和公共输出

##### 将验证搬到后端并上链(智能合约)：

根据ZKey文件和验证密钥生成一个验证器合约：

```
snarkjs zkey export solidityverifier setup_final.zkey Verifier.sol
```

Verifier.sol自己命名，成功生成的智能合约复制到remix部署，编译器版本选0.6.11(版本过旧)

回到终端，生成验证字符串

```
snarkjs zkey export soliditycalldata public.json proof.json
```

生成这个函数调用所需的参数，复制到verifyProof，会调用成功并给出最终结果