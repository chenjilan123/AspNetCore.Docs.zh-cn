---
title: ASP.NET Core 中的上下文标题
author: rick-anderson
description: 了解 ASP.NET Core 数据保护上下文标题的实现细节。
ms.author: riande
ms.date: 10/14/2016
no-loc:
- Blazor
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: security/data-protection/implementation/context-headers
ms.openlocfilehash: 381cc137d1de87e87f36c3b32a6a551a318ed3cf
ms.sourcegitcommit: 70e5f982c218db82aa54aa8b8d96b377cfc7283f
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 05/04/2020
ms.locfileid: "82776950"
---
# <a name="context-headers-in-aspnet-core"></a>ASP.NET Core 中的上下文标题

<a name="data-protection-implementation-context-headers"></a>

## <a name="background-and-theory"></a>背景和理论

在数据保护系统中，"密钥" 是指可提供经过身份验证的加密服务的对象。 每个密钥都是由唯一 id （GUID）标识的，它附带了算法信息和 entropic 材料。 它的目的是，每个密钥都具有唯一的平均信息量，但系统不能强制实施这一点，并且我们还需要考虑到通过修改密钥环中现有密钥的算法信息来手动更改密钥环的开发人员。 为实现安全要求，在这种情况下，数据保护系统具有[加密灵活性](https://www.microsoft.com/en-us/research/publication/cryptographic-agility-and-its-relation-to-circular-encryption/)，这允许使用跨多个加密算法的单个 entropic 值安全地使用。

大多数支持加密灵活性的系统通过在有效负载中包含有关算法的一些识别信息来实现此目的。 通常，该算法的 OID 是一个不错的候选项。 但我们遇到的一个问题是，有多种方法可以指定相同的算法： "AES" （CNG），托管的 Aes、AesManaged、AesCryptoServiceProvider、AesCng 和 RijndaelManaged （给定特定参数）类实际上是相同的，因此我们需要维护所有这些类的映射到正确的 OID。 如果开发人员想要提供自定义算法（甚至是 AES！的另一实现），则必须告诉我们其 OID。 此额外注册步骤使系统配置特别令人头痛。

再回顾一步，我们决定我们从错误的方向接近问题。 OID 告诉您算法是什么，但我们并不真正关心这一点。 如果需要以两个不同的算法安全地使用单个 entropic 值，我们不需要知道算法的实际含义。 我们真正关心的是它们的行为方式。 任何适当的对称块加密算法也是一种强大的伪随机排列（PRP）：修复输入（键、链接模式、IV、纯文本），并且密码文本输出的概率与任何其他对称块加密算法相比，提供的输入相同。 同样，任何适当的键控哈希函数也是强伪随机函数（PRF），并且给定固定输入集，其输出将充分与任何其他键控哈希函数不同。

我们使用强 PRPs 和 PRFs 这一概念来构建上下文标头。 此上下文标头本质上可充当用于任何给定操作的算法的稳定指纹，并提供数据保护系统所需的加密灵活性。 此标头可重复使用，稍后将用作[子项派生过程](xref:security/data-protection/implementation/subkeyderivation#data-protection-implementation-subkey-derivation)的一部分。 可以通过两种不同的方式生成上下文标题，具体取决于基础算法的操作模式。

## <a name="cbc-mode-encryption--hmac-authentication"></a>CBC 模式加密 + HMAC 身份验证

<a name="data-protection-implementation-context-headers-cbc-components"></a>

上下文标头包含以下组件：

* [16 位]值 00 00，它是一个标记，表示 "CBC 加密 + HMAC 身份验证"。

* [32 位]对称块加密算法的密钥长度（以字节为单位）。

* [32 位]对称块加密算法的块大小（以字节为单位）。

* [32 位]HMAC 算法的密钥长度（以字节为单位）。 （目前密钥大小始终与摘要大小匹配。）

* [32 位]HMAC 算法的摘要大小（以字节为单位）。

* EncCBC （K_E，IV，""），表示对称块加密算法的输出（给定空字符串输入），其中 IV 为全零向量。 下面描述了 K_E 的构造。

* MAC （K_H ""），它是在给定空字符串输入的情况下为 HMAC 算法的输出。 下面描述了 K_H 的构造。

理想情况下，我们可以将所有零向量传递给 K_E 和 K_H。 但是，我们希望避免在执行任何操作（特别是 DES 和3DES）之前，基础算法检查是否存在弱密钥，这会排除使用简单或可重复的模式（如全零矢量）。

相反，我们以计数器模式使用 NIST SP800-108 KDF （请参阅[NIST SP800-108](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf)，Sec. 5.1），其长度为零、标签和上下文，HMACSHA512 为基础 PRF。 派生 |K_E |+ |K_H |输出字节数，然后将结果分解为 K_E 并 K_H 自身。 从数学上来说，这种情况如下所示。

（K_E | |K_H） = SP800_108_CTR （prf = HMACSHA512，key = ""，label = ""，context = ""）

### <a name="example-aes-192-cbc--hmacsha256"></a>示例： AES-192-CBC + HMACSHA256

作为示例，请考虑对称块加密算法为 AES-192-CBC 并且验证算法为 HMACSHA256 的情况。 系统将使用以下步骤生成上下文标头。

首先，让（K_E | |K_H） = SP800_108_CTR （prf = HMACSHA512，key = ""，label = ""，context = ""），其中 |K_E |= 192 位 and |K_H |= 256 位/指定的算法。 这会导致 K_E = 5BB6。21DD 和 K_H = A04A.。以下示例中的00A9：

```
5B B6 C9 83 13 78 22 1D 8E 10 73 CA CF 65 8E B0
61 62 42 71 CB 83 21 DD A0 4A 05 00 5B AB C0 A2
49 6F A5 61 E3 E2 49 87 AA 63 55 CD 74 0A DA C4
B7 92 3D BF 59 90 00 A9
```

接下来，计算 192-CBC K_E 给定的-CBC Enc_CBC （K_E，IV，""），如上所述。

result： = F474B1872B3B53E4721DE19C0841DB6F

接下来，为 HMACSHA256 给定 K_H 计算 MAC （K_H ""），如上所述。

result： = D4791184B996092EE1202F36E8608FA8FBD98ABDFF5402F264B1D7211536220C

这会生成以下完整的上下文标头：

```
00 00 00 00 00 18 00 00 00 10 00 00 00 20 00 00
00 20 F4 74 B1 87 2B 3B 53 E4 72 1D E1 9C 08 41
DB 6F D4 79 11 84 B9 96 09 2E E1 20 2F 36 E8 60
8F A8 FB D9 8A BD FF 54 02 F2 64 B1 D7 21 15 36
22 0C
```

此上下文标头是经过身份验证的加密算法对（AES-192-CBC encryption + HMACSHA256 验证）的指纹。 如上所述，组件如下所[述：](xref:security/data-protection/implementation/context-headers#data-protection-implementation-context-headers-cbc-components)

* 标记（00 00）

* 块加密密钥长度（00 00 00 18）

* 块加密块大小（00 00 00 10）

* HMAC 密钥长度（00 00 00 20）

* HMAC 摘要大小（00 00 00 20）

* 块密码 PRP 输出（F4 74-DB 6F）和

* HMAC PRF 输出（D4 79 端）。

> [!NOTE]
> 无论算法实现由 Windows CNG 还是由托管的 System.security.cryptography.symmetricalgorithm 和 KeyedHashAlgorithm 类型提供，CBC 模式加密 + HMAC 身份验证上下文标题都采用相同的方式生成。 这使得在不同操作系统上运行的应用程序可以可靠地生成相同的上下文标头，即使这些算法的实现在操作系统之间有所不同。 （实际上，KeyedHashAlgorithm 不必是正确的 HMAC。 它可以是任何密钥哈希算法类型。）

### <a name="example-3des-192-cbc--hmacsha1"></a>示例： 3DES-192-CBC + HMACSHA1

首先，让（K_E | |K_H） = SP800_108_CTR （prf = HMACSHA512，key = ""，label = ""，context = ""），其中 |K_E |= 192 位 and |K_H |= 160 位/指定的算法。 这会导致 K_E = A219。E2BB 和 K_H = DC4A.。以下示例中的 B464：

```
A2 19 60 2F 83 A9 13 EA B0 61 3A 39 B8 A6 7E 22
61 D9 F8 6C 10 51 E2 BB DC 4A 00 D7 03 A2 48 3E
D1 F7 5A 34 EB 28 3E D7 D4 67 B4 64
```

接下来，计算 192-CBC 给定的 IV = 0 * Enc_CBC （K_E，IV，""），并按上述方式 K_E。

result： = ABB100F81E53E10E

接下来，为 HMACSHA1 给定 K_H 计算 MAC （K_H ""），如上所述。

result： = 76EB189B35CF03461DDF877CD9F4B1B4D63A7555

这将生成完整的上下文标题，该标头是经过身份验证的加密算法对（3DES-192-CBC encryption + HMACSHA1 验证）的指纹，如下所示：

```
00 00 00 00 00 18 00 00 00 08 00 00 00 14 00 00
00 14 AB B1 00 F8 1E 53 E1 0E 76 EB 18 9B 35 CF
03 46 1D DF 87 7C D9 F4 B1 B4 D6 3A 75 55
```

组件将按如下方式进行分解：

* 标记（00 00）

* 块加密密钥长度（00 00 00 18）

* 块加密块大小（00 00 00 08）

* HMAC 密钥长度（00 00 00 14）

* HMAC 摘要大小（00 00 00 14）

* 块密码 PRP 输出（AB B1-E1 0E）和

* HMAC PRF 输出（76 EB-end）。

## <a name="galoiscounter-mode-encryption--authentication"></a>Galois/Counter 模式加密 + 身份验证

上下文标头包含以下组件：

* [16 位]值 00 01，它是一个标记，表示 "GCM 加密 + 身份验证"。

* [32 位]对称块加密算法的密钥长度（以字节为单位）。

* [32 位]经过身份验证的加密操作期间使用的 nonce 大小（以字节为单位）。 （对于我们的系统，这是在 nonce 大小 = 96 位固定的。）

* [32 位]对称块加密算法的块大小（以字节为单位）。 （对于 GCM，这是在块大小 = 128 位固定的。）

* [32 位]经过身份验证的加密功能生成的身份验证标记大小（以字节为单位）。 （对于我们的系统，这是在标记大小 = 128 位固定的。）

* [128 位]Enc_GCM （K_E，nonce，""）的标记，它是对称块加密算法的输出，提供的是空字符串输入，其中 nonce 为96位全零矢量。

使用与在 CBC 加密 + HMAC 身份验证方案中相同的机制来派生 K_E。 不过，由于这里没有任何 K_H，因此我们实质上有 |K_H |= 0，算法折叠到下面的窗体。

K_E = SP800_108_CTR （prf = HMACSHA512，key = ""，label = ""，context = ""）

### <a name="example-aes-256-gcm"></a>示例： AES-256-GCM

首先，让 K_E = SP800_108_CTR （prf = HMACSHA512，key = ""，label = ""，context = ""），其中 |K_E |= 256 位。

K_E： = 22BC6F1B171C08C4AE2F27444AF8FC8B3087A90006CAEA91FDCFB47C1B8733B8

接下来，计算 256-GCM 给定 nonce = 096 的 Enc_GCM （K_E，nonce，""）的身份验证标记，并按上述方式 K_E。

result： = E7DCCE66DF855A323A6BB7BD7A59BE45

这会生成以下完整的上下文标头：

```
00 01 00 00 00 20 00 00 00 0C 00 00 00 10 00 00
00 10 E7 DC CE 66 DF 85 5A 32 3A 6B B7 BD 7A 59
BE 45
```

组件将按如下方式进行分解：

* 标记（00 01）

* 块加密密钥长度（00 00 00 20）

* nonce 大小（00 00 00 0C）

* 块加密块大小（00 00 00 10）

* 身份验证标记大小（00 00 00 10）和

* 运行块密码（E7 DC）的身份验证标记。
