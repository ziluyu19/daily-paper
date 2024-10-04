##### MIMC：最小乘法复杂度

MIMC的加密函数为：

![image-20241005042917669](C:\Users\Jewel\AppData\Roaming\Typora\typora-user-images\image-20241005042917669.png)

x∈Fq是明文，r是轮数，Fi is the round function for round i≥0，k∈Fqk∈Fq is the key，每一个Fi(同样作为实现方法)定义为

![image-20241005042823768](C:\Users\Jewel\AppData\Roaming\Typora\typora-user-images\image-20241005042823768.png)

随机常量在MIMC实例化时被选择为有限域Fq中的随机元素，然后固定下来。

意味着当实现MIMC哈希函数时，需要为每轮加密生成一堆常量，然后可以将它们硬编码到代码中

![img](https://byt3bit.github.io/primesym/mimc/mimc.png)

哈希函数电路实现：

```
pragma circom  2.0.0;

template MIMC5() {

    // Declaration of signals.
    signal input x; 
    signal input k;
    signal output h;

    var nRounds = 10;
//轮次第一个常量总是0，后续为随机生成的大数
    var c[nRounds] = [
        0,
        18306687113883968518773305135814488358176083808813054563815085114441907421609,
        28775042267900106674004003818311383890356920779197943873558139717867667301403,
        10646004036085173582161772598322508733033235722142696401677275812451383218426,
        79386406729580134365639080545738781721728472515692126537259957703398136088192,
        62242654296843257469589256646358677564706947459511971583298897633042897307430,
        100757216228217238239007064371612967739349752303491399624903885788260695209128,
        507509826840708981343713881905337579016630481226461005569912286517617204348,
        84151666979564219931596845972660527041047222493679972951736475653661048649949,
        63371771133525813111661365558608359052049356932502237121489308974094194458244 
    ];

    signal lastOutput[nRounds + 1];

    var base[nRounds];
    signal base2[nRounds];
    signal base4[nRounds];

    lastOutput[0] <== x;

    for (var i = 0; i < nRounds; i++){
        base[i] = lastOutput[i] + k + c[i];
        base2[i] <== base[i] * base[i];
        base4[i] <== base2[i] * base2[i];
        lastOutput[i+1] <== base4[i] * base[i]; 
    
    }
}

component main = MIMC5();
```

生成bignumber实现：

```
const { ethers } = require("ethers");

const num = 10;

async function generate() {
    for (let i = 0; i < num; i++) {
        // uint256 256/8 = 32
        let n = ethers.BigNumber.from(ethers.utils.randomBytes(32));
        console.log(n.toString());
    }
}

generate().catch((err) => { console.log(err); process.exit(1); });
```

