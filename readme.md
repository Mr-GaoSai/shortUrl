## 今日头条2017年秋招设计题
## 长链转短链

> 时间：2017/08/25
> 
> 作者：Mr-GaoSai
> 
> 项目地址：https://github.com/Mr-GaoSai/shortUrl

### 1. 题目要求
- 数据量500亿
- 短链转长链100w/qps（尚未测试）
- 长链转短链10w/qps(尚未测试)
- 长链只对应一个短链
- 短链之对应一个长链

### 2. 短链长度

0-9，a-z，A-Z，总共62位数字，62的6次方与等于500亿，那么短链长度为6位

### 3. 长链转短链算法
1. 对长链进行md5加密生成32位字符串
2. 将32位字符串分成4组，每8个一组，每组是8个字符，将8个字符串16进制生成10进制，可以在32位数字表示，用unsiged int 可以表示，这里是32位数。
3. 选择第一组进行下面处理，如果重复出现短链，选择下一组
4. 将32位的数，将它与0x3FFFFFFF进行位与运算，取其低30位的数据
5. 把得到的30位与0x0000003D进行位与运算，再把得到的结果作为下标在字符表中选取字符
6. 再把得到30位数右移5位进行第5步操作，总共执行6次。
7. 选择出一个6位字符串作为短链，如果这6位字符串已经是其他长链的短链，则回到第3步

java代码

```java

package com.lufy.lucene.action;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class ShortUrlGenerator {

	public static String getMd5(String plainText) {
		try {
			MessageDigest md = MessageDigest.getInstance("MD5");
			md.update(plainText.getBytes());
			byte b[] = md.digest();
			int i;
			StringBuffer buf = new StringBuffer("");
			for (int offset = 0; offset < b.length; offset++) {
				i = b[offset];
				if (i < 0)
					i += 256;
				if (i < 16)
					buf.append("0");
				buf.append(Integer.toHexString(i));
			}
			// 32位加密
			return buf.toString();
			// 16位的加密
			// return buf.toString().substring(8, 24);
		} catch (NoSuchAlgorithmException e) {
			e.printStackTrace();
			return null;
		}

	}

	public static String[] shortUrl(String url) {
		// 要使用生成 URL 的字符
		String[] chars = new String[] { "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p",
				"q", "r", "s", "t", "u", "v", "w", "x", "y", "z", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A",
				"B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V",
				"W", "X", "Y", "Z" };
		// 对传入网址进行 MD5 加密，获取32位字符串
		String hex = getMd5(url);
		String[] resUrl = new String[4];
		for (int i = 0; i < 4; i++) {
			// 把加密字符按照 8 位一组 16 进制与 0x3FFFFFFF 进行位与运算
			String sTempSubString = hex.substring(i * 8, i * 8 + 8);
			// 这里需要使用 long 型来转换，因为 Inteper .parseInt() 只能处理 31 位 , 首位为符号位 , 如果不用
			// long ，则会越界
			long lHexLong = 0x3FFFFFFF & Long.parseLong(sTempSubString, 16);
			String outChars = "";
			for (int j = 0; j < 6; j++) {
				// 把得到的值与 0x0000003D 进行位与运算，取得字符数组 chars 索引
				long index = 0x0000003D & lHexLong;
				// 把取得的字符相加
				outChars += chars[(int) index];
				// 每次循环按位右移 5 位
				lHexLong = lHexLong >> 5;
			}
			// 把字符串存入对应索引的输出数组
			resUrl[i] = outChars;
		}
		return resUrl;
	}

}


```

### 4. 数据库设计
由于redis进行计算500亿行数据的话需要400G的内存，物理硬件可能出现瓶颈，所以采用了数据库。

对于500亿数据肯定要分表，那么分表的依据就是：短链可以通过62进制可以转化成唯一的long型，可以通过62进制数/50w的商分表。

这样就会让插入和查询都能以最快的速度。

索引：短链，长链都要加唯一索引，不是复合索引。

可能要进行数据库集群

### 5. 对应用服务器做负载均衡

可以采用nginx软负载，或者，f5等硬负载。

### 6. 应用层设计
***长链转短链***

用户上传长链，然后服务端判断长链是否合法，然后通过上述算法生成hash码（即短链），然后根据短链分表，长链短链插入数据库，如果出现字段重复的error，那么先判断是长链重复然后再短链重复，如果长链重复，直接返回已经生成的短链，如果短链重复，那么选择下一个短链，依然重复循环

### 7. 项目成果展示

项目采用ssm作为服务端，vue搭建的view层，详细请看源码

![demo](screen.jpeg)


