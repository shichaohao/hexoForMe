---
title: solidity 合约状态变量存储规则分析
date: 2017-10-16 18:15:06
tags: [Go, 区块链]
categories: [技术]
---

### 内容简介

目前在较火的区块链技术中，使用 **solidity** 语言编写智能合约较为普遍，本文主要讲述智能合约中状态变量的存储规则。

<!-- more -->

### 状态变量的定义与类别

#### 状态变量是什么

状态变量是永久存储在合约存储空间中的值。如下代码所示，类型为uint、名称为num的变量就是一个状态变量。所以状态变量有点像Java中的类成员变量。

	pragma solidity ^0.4.0;

	contract SimpleStorage {
   		uint num; // State variable
    	// ...
	}

#### 状态变量的类型

**solidity** 是一门静态类型的语言，这意味着在编译期间，每个变量的类型（不管是状态变量还是局部变量）都是确定的，至少是可以推断出来的。在这里，主要讲述状态变量，局部变量与状态变量类似，不再阐述。

状态变量的类型分为静态类型和动态类型。

##### 动态类型

动态类型包括 **mapping** 类型和 **动态数组** 类型。它们在存储空间中所占据的位置需要通过计算才可获知。

**mapping** 类型可以声明为 **mapping(_KeyType => _ValueType)**。**_KeyType** 可以是除了 **mapping**，**动态数组**，**合约**，**枚举类型**，**结构类型** 之外的所有类型。 **_ValueType** 可以是任何类型。**mapping** 类型可以看作是一张哈希表。

**动态数组** 类型可以声明为 **T[]**，**T** 是类型。

如下所示，num是 **动态数组** 类型，people是 **mapping** 类型。

	contract Demo {
   		uint[] num;                       // State variable
   		mapping(bytes32 => uint) people;  // State variable
    	// ...
	}

##### 静态类型

静态类型是除了动态类型之外所有的类型。包括：

> 布尔类型: bool。
> 
> 整数类型: 包括有符号数(int8, int16, ..., int256)和无符号数(uint8, uint16, ..., uint256)。int 和 uint 分别等于 int256 和 uint256。
> 
> 地址类型: address。
> 
> 固定大小的字节数组: bytes1, bytes2, ..., bytes32。 byte 等于 bytes1。
> 
> 动态大小的字节数组（字符串）: bytes 和 string。
> 
> 枚举类型： enum。
> 
> 静态数组：T[k]。
> 
> 结构体类型：struct。

其中 **静态数组** 和 **结构体类型** 是否是静态，还需要看数组的类型(T)或者结构体定义才能判断。这里先归为静态类型。例子如下所示。

	contract Demo {
   		bool isEmpty;           // 变量类型为布尔类型，变量名称为isEmpty。
   		uint num;               // 变量类型为整数类型，变量名称为num。
   		address accountAddress; // 变量类型为地址类型，变量名称为accountAddress。
   		bytes32 id;             // 变量类型为32个字节大小的数组，变量名称为id。
   		string str;             // 变量类型为字符串类型，变量名称为str。
   		uint8[] money;          // 变量类型为uint8类型的动态数组，变量名称为money。
   		struct People {         // 结构体People的定义。
    		bytes32 name;
    		int height;
    		int weight; 
    	}
    	People people;          // 变量类型为结构体类型，变量名称为people。
    	enum AccountType{Customer, Merchant}  // 枚举类型定义。
    	AccountType accountType;  // 变量类型为枚举类型，变量名称为accountType。
    	// ...
	}

### 状态变量的存储规则

在了解了什么是状态变量以及状态变量的类别后，我们进入本文的主题，状态变量的存储规则讲解。

#### 存储空间

首先，我们需要了解整个存储空间是什么样子。如下图所示，存储空间可以理解为由许多个槽空间组成，每个槽空间都是键值对，键表示槽的地址，也就是图中绿色部分，表示状态变量的地址，值表示该槽所存储的内容，也就是图中黄色部分，表示状态变量的值。

存储空间的地址总是从0开始，每个槽的键、值都是256位，换算成十六进制，也就是32长度。不同类型的状态变量所占据的槽的位数不同，详情见下文。

少于32个字节的连续多个状态变量将会压缩进一个槽中，也就是存在多个状态变量存储在一个槽中。压缩时遵守如下规则：

> 1. 每个状态变量存储时总是从低序开始分配（也就是从右到左开始分配256位）
> 2. 每个状态变量只分配该状态变量需要的位数。
> 3. 如果一个槽中剩余可分配的位数不满足该状态变量的需求，那么分配新的槽（槽地址+1）给该状态变量。
> 4. 结构体类型和数组类型的状态变量总是分配新槽并且占据整个槽空间，但是结构体或者数组内部的存储仍然遵守压缩的规则。

![Storage](/images/solidity-合约状态变量存储规则/storage-1.jpg)

下面我们详细分析状态变量的存储规则。

#### 静态类型的状态变量

如下表所示，静态类型的状态变量所占据的位数与类型有关。

> 1. 布尔类型存储位数为8位。
> 2. 整数类型存储位数与具体类型有关。
> 3. 地址类型存储位数为160位。
> 4. 固定大小的字节数组存储位数与具体类型有关。
> 5. 枚举类型的存储位数与具体枚举个数有关。少于8个相当于uint8类型，8～16个相当于uint16类型，以此类推。
> 6. 静态数组 T[k] 中元素的存储位数与 T 相关。

bytes 和 string 类型较为特殊，会在下文分析。结构体类型的状态变量的存储位数与具体的结构体定义相关。

类型 | 存储位数 | 存储顺序 | 例子
---| ---| ---| ---
bool | 8 | 从右至左，0表示false，否则表示true | false：0x00 
int8 | 8 | 从右至左 | 10: 0x0a 
int16 | 16 | 从右至左 | 10: 0x000a 
int... | ... | 从右至左 | 10: 0x00...000a 
int256 | 256 | 从右至左 | 10: 0x0000...000a(256位) 
uint8 | 8 | 从右至左 | 10: 0x0a 
uint16 | 16 | 从右至左 | 10: 0x000a 
uint... | ... | 从右至左 | 10: 0x00...000a 
uint256 | 256 | 从右至左 | 10: 0x0000...000a（256位）
bytes1 | 8 | 从左到右 | "1": 0x31 
bytes2 | 16 | 从左到右 | "1": 0x3100 
bytes... | ... | 从左到右 | "1": 0x3100...00 
bytes32 | 256 | 从左到右 | "1": 0x3100...0000（256位）
address | 160 | 从右至左 | 0x 0346 de49 0a2e bd37 e1f6 f216 b137 6c60 f19c b9d6 
enum | 与具体枚举类型的个数有关 | 相当于uint类型的存储规则 | enum AccountType{Customer, Merchant} 的定义，类型为uint8 
静态数组T[k] | T具体类型相关 | T具体类型相关 | uint8[10]的存储结构相当于存储10个uint8类型

#### 动态类型的状态变量

动态类型的状态变量包括 **mapping** 类型和 **动态数组** 类型。由于动态类型的大小不可预测，所以采用了 **Keccak256** 哈希计算方式来找到map值或者数组元素的地址。

##### mapping 动态类型

**mapping** 类型首先会根据以上规则分配到一个槽，这个槽空间并不会拿来存储什么值。但是这个槽空间是必须的，因为需要用该槽的地址进行哈希计算。因此就算有两个相同的**mapping**，它们的起始槽空间也不一样，所以进行哈希计算后会有不同的哈希分布。

地址哈希计算公式：`position = keccak256(k . p)` （k为 **mapping** 的键，p为起始槽空间，. 为连接运算）。

如果 **mapping** 类型的值为非基本类型，那么槽地址加上位移。

如下代码所示，data[4][9].b 的地址就是：
`keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1`。
状态变量 **x** 的地址为0x00...00，状态变量 **data** 的的地址为0x00...01。所以 data[4][9] 先计算key为4的mapping，即`keccak256(uint256(4) . uint256(1))`。然后再计算key为9的mapping，即`keccak256(uint256(9) . keccak256(uint256(4) . uint256(1)))`，哈希计算出来的地址是s的地址，也是结构体中 a 的地址。最后因为想要得到结构体中 b 的地址，所以加1。

	pragma solidity ^0.4.0;
	
	contract C {
	  struct s { uint a; uint b; }
	  uint x;
	  mapping(uint => mapping(uint => s)) data;
	}

写成Golang的大致代码：（

	kec256Hash := crypto.NewKeccak256Hash("keccak256")
	address1 := kec256Hash.ByteHash([]byte(common.Hex2Bytes("00...04" + "00...01"))).Hex()
	address2 := kec256Hash.ByteHash([]byte(common.Hex2Bytes("00...09" + address1))).Hex()
	result := address2 + 1
	
> 注意，这里的k和p都要是64长度的字符串（也就是把256位的地址转换为64长度的16进制）。

##### 动态数组类型

对于动态数组类型，起始槽空间存储着数组元素的个数，也就是数组的长度。

地址哈希计算公式：`position = keccak256(p)`

得到地址之后，再根据数组长度，依次得到数组元素的具体地址。这里也符合压缩规则。

> 值得注意的是，二维数组data[2][3]其实第一维度是3，第二维度是2，与常见的编程语言（C，Java等）相反。

#### 字符串类型

对于字符串类型，因为字符串长度不知，而一个槽最多存储256位。所以对于字符串：

> 1. 如果字符串长度不超过31个字节，那么值存储在该槽中。该槽的最后一个字节表示长度，值为 2 * length。
> 
> 2. 如果字符串长度超过31个字节，那么值存储在 keccak256(p) 地址中（如果一个槽不够存放值，那么分配新槽）。该槽存储字符串长度，值为 2 * length + 1。

所以对于字符串类型，可以先获取最后1个字节的值，判断是否是偶数，然后再根据不同方式获取字符串的值。

如果类型是非基本元素，那么递归调用以上规则。

### 举个例子

现在，我们来举个例子。

	contract Demo {
		uint8    var1;
		uint16   var2;
		uint16   var3;
		uint     var4;
		bytes8   var5;
		bytes8   var6;
		bytes32  var7;
		string   var8;
		string   var9;
		mapping(bytes32 => mapping(bytes32 => uint)) userMap;
		mapping(bytes32 => uint[]) bankMap;
		uint[3]  array1;
		uint[]   array2;
		uint16[2][3]  array3;
		uint16[][]  array4;
		string[3]   strArray1;
		string[]    strArray2;
	
	    function Demo() {
	    	var1 = 1;
	    	var2 = 2;
	    	var3 = 3;
	    	var4 = 4;
	    	var5 = "a";
	    	var6 = "b";
	    	var7 = "c";
	    	var8 = "abc";
	    	var9 = "aaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeffffffffffgggggggggg";
	    	userMap["001"]["002"] = 5;
	    	userMap["003"]["004"] = 6;
	    	bankMap["10086"] = [7,8,9];
	    	bankMap["10087"] = [10,11,12,13,14];
	    	array1[0]=100;
	    	array1[1]=200;
	    	array2.push(300);
	    	array2.push(400);
	    	array3[0] = [500];
	    	array3[1] = [600,700];
	    	array4.push([700,800]);
	    	array4.push([900,1000,1100]);
	    	strArray1[0]="a";
			strArray1[1]="bbbbbbbbbbccccccccccddddddddddeeeeeeeeeeffffffffffgggggggggg";
			strArray2.push("b");
			strArray2.push("aaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeffffffffffgggggggggg");
	    }
	}
	
以上合约部署后，由于构造函数的调用，存储空间的内容为：

槽地址 | 槽内容
--- | ---
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 | 0300 0201
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0001 | 04
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0002 | 6200 0000 0000 0000 6100 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0003 | 6300 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0004 | 6162 6300 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0006
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0005 | 8d
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0008 | 64
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0009 | c8
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 000b | 02
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 000c | 01f4
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 000d | 02bc 0258
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 000f | 02
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0010 | 6100 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0002
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0011 | 79
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0013 | 02
0175 b7a6 3842 7703 f0db e7bb 9bbf 987a 2551 717b 34e7 9f33 b5b1 008d 1fa0 1db9 | 012c
0175 b7a6 3842 7703 f0db e7bb 9bbf 987a 2551 717b 34e7 9f33 b5b1 008d 1fa0 1dba | 0190
036b 6384 b5ec a791 c627 6115 2d0c 79bb 0604 c104 a5fb 6f4e b070 3f31 54bb 3db0 | 6161 6161 6161 6161 6161 6262 6262 6262 6262 6262 6363 6363 6363 6363 6363 6464
036b 6384 b5ec a791 c627 6115 2d0c 79bb 0604 c104 a5fb 6f4e b070 3f31 54bb 3db1 | 6464 6464 6464 6464 6565 6565 6565 6565 6565 6666 6666 6666 6666 6666 6767 6767
036b 6384 b5ec a791 c627 6115 2d0c 79bb 0604 c104 a5fb 6f4e b070 3f31 54bb 3db2 | 6767 6767 6767 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000
31ec c21a 745e 3968 a04e 9570 e442 5bc1 8fa8 019c 6802 8196 b546 d166 9c20 0c68 | 6262 6262 6262 6262 6262 6363 6363 6363 6363 6363 6464 6464 6464 6464 6464 6565
31ec c21a 745e 3968 a04e 9570 e442 5bc1 8fa8 019c 6802 8196 b546 d166 9c20 0c69 | 6565 6565 6565 6565 6666 6666 6666 6666 6666 6767 6767 6767 6767 6767 0000 0000
49e8 84de b6b3 6c8e e2bf 2731 8acb 9757 d4a9 6c66 e866 3459 9844 b329 ada5 69df | 044c 03e8 0384
66de 8ffd a797 e3de 9c05 e8fc 57b3 bf0e c28a 930d 40b0 d285 d93c 0650 1cf6 a090 | 6200 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0002
66de 8ffd a797 e3de 9c05 e8fc 57b3 bf0e c28a 930d 40b0 d285 d93c 0650 1cf6 a091 | 8d
7399 827b 1e68 b497 430a 72e6 7725 4393 1f27 51c9 09d0 1c20 1733 82a4 a947 263c | 05
8d11 08e1 0bcb 7c27 dddf c02e d9d6 93a0 7403 9d02 6cf4 ea42 40b4 0f7d 581a c802 | 02
8d11 08e1 0bcb 7c27 dddf c02e d9d6 93a0 7403 9d02 6cf4 ea42 40b4 0f7d 581a c803 | 03
91d5 0dd0 47ad bb71 a820 341a ebb5 83bf 75e9 232e 81f5 6500 841b 8f1d 70c1 632d | 6161 6161 6161 6161 6161 6262 6262 6262 6262 6262 6363 6363 6363 6363 6363 6464
91d5 0dd0 47ad bb71 a820 341a ebb5 83bf 75e9 232e 81f5 6500 841b 8f1d 70c1 632e | 6464 6464 6464 6464 6565 6565 6565 6565 6565 6666 6666 6666 6666 6666 6767 6767
91d5 0dd0 47ad bb71 a820 341a ebb5 83bf 75e9 232e 81f5 6500 841b 8f1d 70c1 632f | 6767 6767 6767 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000
9f0a 1398 8bdb eae5 6f1b 4f41 e53e d9b3 de4a 6902 722f 517b 4ef2 39da bba0 b34c | 07
9f0a 1398 8bdb eae5 6f1b 4f41 e53e d9b3 de4a 6902 722f 517b 4ef2 39da bba0 b34d | 08
9f0a 1398 8bdb eae5 6f1b 4f41 e53e d9b3 de4a 6902 722f 517b 4ef2 39da bba0 b34e | 09
b8e9 88fb db32 de37 1983 ecc5 ec88 6716 ea07 1793 398d 3b10 b18b 83b6 86c9 da7d | 06
bee3 7410 fc5b 77e7 98bd 0138 5d45 7e04 2711 adb9 07f1 5c82 467b b80d 0602 b1c8 | 03
c928 8bb4 db3b e5df ff26 9e68 fef8 65ec 8c48 c6d7 6d1b 377f 8fd9 9275 0ef9 7293 | 0a
c928 8bb4 db3b e5df ff26 9e68 fef8 65ec 8c48 c6d7 6d1b 377f 8fd9 9275 0ef9 7294 | 0b
c928 8bb4 db3b e5df ff26 9e68 fef8 65ec 8c48 c6d7 6d1b 377f 8fd9 9275 0ef9 7295 | 0c
c928 8bb4 db3b e5df ff26 9e68 fef8 65ec 8c48 c6d7 6d1b 377f 8fd9 9275 0ef9 7296 | 0d
c928 8bb4 db3b e5df ff26 9e68 fef8 65ec 8c48 c6d7 6d1b 377f 8fd9 9275 0ef9 7297 | 0e
f31e e89b 8f31 15a2 b487 50c8 c8a2 d7a5 b875 a0ce 5d42 ff75 45f3 2953 db9e 4aa3 | 0320 02bc
f56c 7ffb 723d b564 2b3d 1ea0 a20d e5a1 4314 9f33 0f4c c5b4 ffbe cb57 031a cd6c | 05

下面我们使用上文提到的存储规则详细解析这份存储内容。

> 1. var1、var2、var3分别占据8、16、16个位（bit）的存储空间，所以由于压缩规则，它们在一个槽中。也就是在`0000000000000000000000000000000000000000000000000000000000000000`槽中。槽的内容为`03000201 `，按照整数类型从右到左的存储顺序，所以var1的值为0x01，var2的值为0x0002，var3的值为0x03（其实是0x0003，但是从leveldb取数据时会把前面的00忽略）。
> 
> 2. var4占据256个位的存储空间，由于前面0x00..00的槽空间剩余可分配位数为256-8-16-16=216。所以不足以分配给var4状态变量，因此分配新槽给var4。var4槽的地址为`0000000000000000000000000000000000000000000000000000000000000001`，值为 `04`。所以var4的值为0x04。
> 
> 3. var5、var6分别占据64、64个位的存储空间，所以它们会存储在一个槽中，也就是`0000000000000000000000000000000000000000000000000000000000000002`中，槽的值为`62000000000000006100000000000000`。由于字节数组存储顺序为从左到右，所以var5的值为`6100000000000000`（0x00不表示字符，所以值为0x61，也就是a），var6的值为`6200000000000000`。
> 
> 4. var7占据256个位的存储空间。所以存储在`0000000000000000000000000000000000000000000000000000000000000003`，值为`6300000000000000000000000000000000000000000000000000000000000000`（也就是c）。
> 
> 5. var8是string类型，存储在`0000000000000000000000000000000000000000000000000000000000000004`槽中。所以判断最后一个字节是否是偶数，发现是6，那么可知var8的值就存储在该槽中，并且长度为6/2=3，也就是`616263`。
> 
> 6. var9是string类型，所以读取`0000000000000000000000000000000000000000000000000000000000000005`槽的值，发现最后一个字节是0x8d（十进制141）。所以需要做哈希计算，keccak256(0000000000000000000000000000000000000000000000000000000000000005) = 036b6384b5eca791c62761152d0c79bb0604c104a5fb6f4eb0703f3154bb3db0。并且可以知道字符串长度为(141-1)／2=70个字节。而一个槽最多256位，32个字节。因此后面还有2个槽（共3个槽）也是用来存储var9的值。
> 
> 7. userMap是mapping类型，所以需要做哈希计算。根据代码可知，使用了两组key，分别是{"001", "002"}和{"003", "004"}。我们先看第一组。如下计算哈希，所以存储在`f56c7ffb723db5642b3d1ea0a20de5a143149f330f4cc5b4ffbecb57031acd6c`上的`05`就是值。其余mapping类型的方式类似，不再阐述。
> > keccak256("303031000000...0000" + "0000...000000006") = 
> > 0xdec3a5381b3472ea0094010190dc94e40c4380ae77d46829b10b3f37e9f09dae 
> > 
> > keccak256("303032000000...0000" + 
> > "b2795388d06dd8f9b12622b0a4086887289759945e634f6fad6e946543627b48") =
> > 0xf56c7ffb723db5642b3d1ea0a20de5a143149f330f4cc5b4ffbecb57031acd6c
> 
> 8. array1是长度为3的静态数组。对于数组，首先需要开新槽，然后内部再做压缩。因为数组类型为uint，占据256位，所以其存储在`00...08`，`00...09`，`00...08`，`00...0a`槽上。
> 
> 9. array2是类型为uint的动态数组，所以`00...0b`上存储了该数组长度，也就是2个数组元素。然后通过地址哈希计算，得到地址`0175b7a638427703f0dbe7bb9bbf987a2551717b34e79f33b5b1008d1fa01db9`，因为是长度为2，所以后面一个槽也属于array2。
> 
> 10. array3是类型为uint16的二维静态数组。所以先看第一维度，值为3，也就是有3个元素。再看第二维度，值为2，也就是有2个元素。然后使用压缩规则，可以发现第二维度的数据是可以放在一个槽中的，因为对于第一维度，其数组元素类型为uint16[2]，仍然是数组。而对于第二维度，数组元素类型为uint16，所以符合压缩规则。所以`00...0c`，`00...0d`，`00...0e（值为零）`是属于array3的。`00...0c`中存储的是0x01f4(500)，`00...0d`中存储的是0x02bc0258，2个元素，0x02bc(700) 和 0x0258(600)。
> 
> 其余状态变量类似，这里不作阐述。

### 有趣的事情

#### 压缩规则对数据声明的影响

当元素的大小小于32个字节时，合约的gas消耗可能会更高。这是因为EVM每次操作32个字节。因此，如果元素大小小于32个字节，那么EVM必须花费更多的操作去裁剪32个字节来得到想要的大小。

所以这个只有在处理storage类型的数据时才会有益。因为编译器会对多个元素进行压缩处理。这样，就会将多次的读或写放在一次操作中。

因此在定义storage变量时，顺序很重要。比如要声明uint128, uint128, uint256的顺序，而不是uint128, uint256, uint128的顺序。前者只需要2次读取操作，而后者需要3次。

#### 数据冲突

如果看了上文，你会发现mapping是会通过哈希来计算地址。那么如果哈希出来的地址与其他变量冲突，那会发生什么？

假定有以下这个合约：

	contract Demo {
		mapping(bytes32 => uint) bankMap;
		function Demo() {
		    bankMap["001"] = 123;
		}
	}
	
其存储空间内容为：

槽地址 | 槽内容
--- | --- |
9114 e9f4 d72d 33a5 1c13 1b85 6d3c 3303 fce9 ef28 c684 2523 b0d4 51ac e6a0 4dbb | 7b

那么如果有以下这个合约：

	contract Demo {
		mapping(bytes32 => uint) bankMap;
		uint[0x9114e9f4d72d33a51c131b856d3c3303fce9ef28c6842523b0d451ace6a04dbb] data;
		function Demo() {
		    bankMap["001"] = 123;
		    data[0x9114e9f4d72d33a51c131b856d3c3303fce9ef28c6842523b0d451ace6a04dba] = 10;
		}
	}

其存储空间内容为：

槽地址 | 槽内容
--- | --- |
9114 e9f4 d72d 33a5 1c13 1b85 6d3c 3303 fce9 ef28 c684 2523 b0d4 51ac e6a0 4dbb | 0a

我们发现当发生地址冲突时，solidity采取是直接覆盖的方式。


### 参考资料

1. http://solidity.readthedocs.io/en/latest/structure-of-a-contract.html
2. http://solidity.readthedocs.io/en/latest/types.html
3. http://solidity.readthedocs.io/en/latest/miscellaneous.html