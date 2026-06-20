炮夯胃痴械


  
 本文详细介绍了如何使用 java 原生 api（无需 bouncy castle）解密安全可靠 aws acm 导出的密码保护私钥（pkcs#8 加密格式)，重点解决 `java.io.ioexception: extra data given to dervalue constructor` 常见解析异常。
在使用 AWS ACM 导出证书时，私钥默认为 PEM 封装的 PKCS#8 Encrypted Private Key 格式输出(即以) -----BEGIN ENCRYPTED PRIVATE KEY----- 开头)。直接调用原始字符串。 new EncryptedPrivateKeyInfo(private_key.getBytes()) 之所以会失败，是因为：这个构造函数预计会被引入 DER 编码的二进制内容不包括在内 PEM 头尾和 Base64 编码的完整文本字符串。错误 extra data given to DerValue constructor 正是由于 PEM 封装头/尾、换行符及 Base64 元数据被误认为是 DER 由数据分析引起。
正确的方法是先剥离 PEM 包装，提取纯 Base64 并将内容解码为 DER 字节数组。推荐使用标准 PemReader（来自 Bouncy Castle 的 org.bouncycastle.openssl.PemReader）完成此步骤；如果需要纯 JDK 方案(无第三方依赖)也可手动分析 PEM——但需要谨慎处理换行、空格和 Base64 解码。
以下是基于验证的完整解密方法( Bouncy Castle）：import org.bouncycastle.openssl.PemReader;
import javax.crypto.Cipher;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;
import java.io.StringReader;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.EncryptedPrivateKeyInfo;
import java.security.spec.KeySpec;

public PrivateKey readPrivateKey(String privateKeyPem, String passphrase) throws Exception {
try (PemReader pemReader = new PemReader(new StringReader(privateKeyPem))) {
// 1. 解析 PEM 对象，获取原始 DER 编码字节
byte[] derBytes = pemReader.readPemObject().getContent();

// 2. 构建 EncryptedPrivateKeyInfo(此时输入是合法的 DER）
EncryptedPrivateKeyInfo encryptedInfo = new EncryptedPrivateKeyInfo(derBytes);

// 3. 初始密码衍生和解密
String algName = encryptedInfo.getAlgName();
Cipher cipher = Cipher.getInstance(algName);
PBEKeySpec keySpec = new PBEKeySpec(passphrase.toCharArray());
SecretKeyFactory factory = SecretKeyFactory.getInstance(algName);
cipher.init(Cipher.DECRYPT_MODE, factory.generateSecret(keySpec), encryptedInfo.getAlgParameters());

// 4. 解密得到 PKCS#8 私钥规范，并生成 PrivateKey 实例
KeySpec keyspecpcs8 = encryptedInfo.getKeySpec(cipher);
KeyFactory kf = KeyFactory.getInstance("RSA"); // 若为 EC 私钥，改为 "EC"
return kf.generatePrivate(keyspecpcs8);
}
}✅ 关键注意事项：  
			
		
；

✅ 必须使用 PemReader(或等效逻辑)提取 content()原始不能直接传入 PEM 字符串；  
✅ EncryptedPrivateKeyInfo 严格要求结构函数 DER 编码字节，非 Base64 或文本；  
✅ 密码必须与 AWS ACM 导出时指定的完全一致(区分大小写和空格)；  
✅ 支持的加密算法取决于导出配置(如常见) PBES2WithmacSHA256AndAES_256)，JDK 8+ 原生支持主流 PBE 算法；  
⚠️ 若私钥为 EC 类型（如 secp256r1) KeyFactory.getInstance("RSA") 替换为 "EC"；  
⚠️ 必须捕捉生产环境中的具体异常(例如) InvalidKeySpecException, BadPaddingException）并记录日志，避免无声失败。

该方案已在 JDK 11+ 和 AWS ACM 导出的各种密码保护私钥稳定运行，从集成证书到 Java 应用（如 Spring Boot HTTPS 配置，客户端 TLS 标准实践路径认证)。	 

泛豆克兴匪阉峡幕迟酵授绷邻罢籽

tv.blog.xlruof.cn/Article/details/359153.sHtML
tv.blog.xlruof.cn/Article/details/282682.sHtML
tv.blog.xlruof.cn/Article/details/084624.sHtML
tv.blog.yqvyet.cn/Article/details/244660.sHtML
tv.blog.yqvyet.cn/Article/details/608806.sHtML
tv.blog.yqvyet.cn/Article/details/662442.sHtML
tv.blog.yqvyet.cn/Article/details/866282.sHtML
tv.blog.yqvyet.cn/Article/details/288626.sHtML
tv.blog.yqvyet.cn/Article/details/262246.sHtML
tv.blog.yqvyet.cn/Article/details/248820.sHtML
tv.blog.yqvyet.cn/Article/details/028888.sHtML
tv.blog.yqvyet.cn/Article/details/642260.sHtML
tv.blog.yqvyet.cn/Article/details/040420.sHtML
tv.blog.yqvyet.cn/Article/details/868624.sHtML
tv.blog.yqvyet.cn/Article/details/648222.sHtML
tv.blog.yqvyet.cn/Article/details/420446.sHtML
tv.blog.yqvyet.cn/Article/details/240640.sHtML
tv.blog.yqvyet.cn/Article/details/620428.sHtML
tv.blog.yqvyet.cn/Article/details/826242.sHtML
tv.blog.yqvyet.cn/Article/details/824000.sHtML
tv.blog.yqvyet.cn/Article/details/440266.sHtML
tv.blog.yqvyet.cn/Article/details/826446.sHtML
tv.blog.yqvyet.cn/Article/details/628040.sHtML
tv.blog.yqvyet.cn/Article/details/468800.sHtML
tv.blog.yqvyet.cn/Article/details/224622.sHtML
tv.blog.yqvyet.cn/Article/details/002664.sHtML
tv.blog.yqvyet.cn/Article/details/868486.sHtML
tv.blog.yqvyet.cn/Article/details/204042.sHtML
tv.blog.yqvyet.cn/Article/details/842082.sHtML
tv.blog.yqvyet.cn/Article/details/840240.sHtML
tv.blog.yqvyet.cn/Article/details/408822.sHtML
tv.blog.yqvyet.cn/Article/details/804666.sHtML
tv.blog.yqvyet.cn/Article/details/288222.sHtML
tv.blog.yqvyet.cn/Article/details/199351.sHtML
tv.blog.yqvyet.cn/Article/details/246068.sHtML
tv.blog.yqvyet.cn/Article/details/040208.sHtML
tv.blog.yqvyet.cn/Article/details/246800.sHtML
tv.blog.yqvyet.cn/Article/details/428448.sHtML
tv.blog.yqvyet.cn/Article/details/204282.sHtML
tv.blog.yqvyet.cn/Article/details/488662.sHtML
tv.blog.yqvyet.cn/Article/details/995759.sHtML
tv.blog.yqvyet.cn/Article/details/171553.sHtML
tv.blog.yqvyet.cn/Article/details/793379.sHtML
tv.blog.yqvyet.cn/Article/details/355171.sHtML
tv.blog.yqvyet.cn/Article/details/335517.sHtML
tv.blog.yqvyet.cn/Article/details/337933.sHtML
tv.blog.yqvyet.cn/Article/details/733179.sHtML
tv.blog.yqvyet.cn/Article/details/173313.sHtML
tv.blog.yqvyet.cn/Article/details/915755.sHtML
tv.blog.yqvyet.cn/Article/details/755735.sHtML
tv.blog.yqvyet.cn/Article/details/353957.sHtML
tv.blog.yqvyet.cn/Article/details/935719.sHtML
tv.blog.yqvyet.cn/Article/details/911735.sHtML
tv.blog.yqvyet.cn/Article/details/573555.sHtML
tv.blog.yqvyet.cn/Article/details/335539.sHtML
tv.blog.yqvyet.cn/Article/details/355913.sHtML
tv.blog.yqvyet.cn/Article/details/377355.sHtML
tv.blog.yqvyet.cn/Article/details/511315.sHtML
tv.blog.yqvyet.cn/Article/details/115955.sHtML
tv.blog.yqvyet.cn/Article/details/133537.sHtML
tv.blog.yqvyet.cn/Article/details/915779.sHtML
tv.blog.yqvyet.cn/Article/details/997995.sHtML
tv.blog.yqvyet.cn/Article/details/773971.sHtML
tv.blog.yqvyet.cn/Article/details/599159.sHtML
tv.blog.yqvyet.cn/Article/details/197531.sHtML
tv.blog.yqvyet.cn/Article/details/951759.sHtML
tv.blog.yqvyet.cn/Article/details/971915.sHtML
tv.blog.yqvyet.cn/Article/details/371317.sHtML
tv.blog.yqvyet.cn/Article/details/757519.sHtML
tv.blog.yqvyet.cn/Article/details/537159.sHtML
tv.blog.yqvyet.cn/Article/details/515159.sHtML
tv.blog.yqvyet.cn/Article/details/573933.sHtML
tv.blog.yqvyet.cn/Article/details/919775.sHtML
tv.blog.yqvyet.cn/Article/details/175555.sHtML
tv.blog.yqvyet.cn/Article/details/153995.sHtML
tv.blog.yqvyet.cn/Article/details/511337.sHtML
tv.blog.yqvyet.cn/Article/details/159599.sHtML
tv.blog.yqvyet.cn/Article/details/335555.sHtML
tv.blog.yqvyet.cn/Article/details/119573.sHtML
tv.blog.yqvyet.cn/Article/details/771393.sHtML
tv.blog.yqvyet.cn/Article/details/759117.sHtML
tv.blog.yqvyet.cn/Article/details/371153.sHtML
tv.blog.yqvyet.cn/Article/details/931759.sHtML
tv.blog.yqvyet.cn/Article/details/751993.sHtML
tv.blog.yqvyet.cn/Article/details/177991.sHtML
tv.blog.yqvyet.cn/Article/details/799793.sHtML
tv.blog.yqvyet.cn/Article/details/775553.sHtML
tv.blog.yqvyet.cn/Article/details/375775.sHtML
tv.blog.yqvyet.cn/Article/details/935399.sHtML
tv.blog.yqvyet.cn/Article/details/319151.sHtML
tv.blog.yqvyet.cn/Article/details/979173.sHtML
tv.blog.yqvyet.cn/Article/details/153997.sHtML
tv.blog.yqvyet.cn/Article/details/775339.sHtML
tv.blog.yqvyet.cn/Article/details/999533.sHtML
tv.blog.yqvyet.cn/Article/details/119997.sHtML
tv.blog.yqvyet.cn/Article/details/733959.sHtML
tv.blog.yqvyet.cn/Article/details/739397.sHtML
tv.blog.yqvyet.cn/Article/details/117315.sHtML
tv.blog.yqvyet.cn/Article/details/359713.sHtML
tv.blog.yqvyet.cn/Article/details/117577.sHtML
tv.blog.yqvyet.cn/Article/details/151135.sHtML
tv.blog.yqvyet.cn/Article/details/791991.sHtML
tv.blog.yqvyet.cn/Article/details/991779.sHtML
tv.blog.yqvyet.cn/Article/details/755797.sHtML
tv.blog.yqvyet.cn/Article/details/759559.sHtML
tv.blog.yqvyet.cn/Article/details/953573.sHtML
tv.blog.yqvyet.cn/Article/details/317731.sHtML
tv.blog.yqvyet.cn/Article/details/397797.sHtML
tv.blog.yqvyet.cn/Article/details/353951.sHtML
tv.blog.yqvyet.cn/Article/details/919371.sHtML
tv.blog.yqvyet.cn/Article/details/937317.sHtML
tv.blog.yqvyet.cn/Article/details/559755.sHtML
tv.blog.yqvyet.cn/Article/details/379335.sHtML
tv.blog.yqvyet.cn/Article/details/179399.sHtML
tv.blog.yqvyet.cn/Article/details/599159.sHtML
tv.blog.yqvyet.cn/Article/details/395171.sHtML
tv.blog.yqvyet.cn/Article/details/335399.sHtML
tv.blog.yqvyet.cn/Article/details/571131.sHtML
tv.blog.yqvyet.cn/Article/details/131573.sHtML
tv.blog.yqvyet.cn/Article/details/537559.sHtML
tv.blog.yqvyet.cn/Article/details/915599.sHtML
tv.blog.yqvyet.cn/Article/details/935197.sHtML
tv.blog.yqvyet.cn/Article/details/151579.sHtML
tv.blog.yqvyet.cn/Article/details/719955.sHtML
tv.blog.yqvyet.cn/Article/details/757719.sHtML
tv.blog.yqvyet.cn/Article/details/955371.sHtML
tv.blog.yqvyet.cn/Article/details/959753.sHtML
tv.blog.yqvyet.cn/Article/details/319735.sHtML
tv.blog.yqvyet.cn/Article/details/911393.sHtML
tv.blog.yqvyet.cn/Article/details/353733.sHtML
tv.blog.yqvyet.cn/Article/details/719333.sHtML
tv.blog.yqvyet.cn/Article/details/995971.sHtML
tv.blog.yqvyet.cn/Article/details/915379.sHtML
tv.blog.yqvyet.cn/Article/details/575951.sHtML
tv.blog.yqvyet.cn/Article/details/153159.sHtML
tv.blog.yqvyet.cn/Article/details/735993.sHtML
tv.blog.yqvyet.cn/Article/details/971379.sHtML
tv.blog.yqvyet.cn/Article/details/311357.sHtML
tv.blog.yqvyet.cn/Article/details/777953.sHtML
tv.blog.yqvyet.cn/Article/details/957359.sHtML
tv.blog.yqvyet.cn/Article/details/737379.sHtML
tv.blog.yqvyet.cn/Article/details/975159.sHtML
tv.blog.yqvyet.cn/Article/details/357771.sHtML
tv.blog.yqvyet.cn/Article/details/331379.sHtML
tv.blog.yqvyet.cn/Article/details/711771.sHtML
tv.blog.yqvyet.cn/Article/details/335991.sHtML
tv.blog.yqvyet.cn/Article/details/977559.sHtML
tv.blog.yqvyet.cn/Article/details/135933.sHtML
tv.blog.yqvyet.cn/Article/details/377737.sHtML
tv.blog.yqvyet.cn/Article/details/717133.sHtML
tv.blog.yqvyet.cn/Article/details/531399.sHtML
tv.blog.yqvyet.cn/Article/details/335119.sHtML
tv.blog.yqvyet.cn/Article/details/171577.sHtML
tv.blog.yqvyet.cn/Article/details/791779.sHtML
tv.blog.yqvyet.cn/Article/details/177139.sHtML
tv.blog.yqvyet.cn/Article/details/973775.sHtML
tv.blog.yqvyet.cn/Article/details/117331.sHtML
tv.blog.yqvyet.cn/Article/details/731197.sHtML
tv.blog.yqvyet.cn/Article/details/799913.sHtML
tv.blog.yqvyet.cn/Article/details/591553.sHtML
tv.blog.yqvyet.cn/Article/details/359357.sHtML
tv.blog.yqvyet.cn/Article/details/995931.sHtML
tv.blog.yqvyet.cn/Article/details/153731.sHtML
tv.blog.yqvyet.cn/Article/details/751195.sHtML
tv.blog.yqvyet.cn/Article/details/571975.sHtML
tv.blog.yqvyet.cn/Article/details/797117.sHtML
tv.blog.yqvyet.cn/Article/details/535591.sHtML
tv.blog.yqvyet.cn/Article/details/377751.sHtML
tv.blog.yqvyet.cn/Article/details/579731.sHtML
tv.blog.yqvyet.cn/Article/details/795173.sHtML
tv.blog.yqvyet.cn/Article/details/113531.sHtML
tv.blog.yqvyet.cn/Article/details/531977.sHtML
tv.blog.yqvyet.cn/Article/details/193575.sHtML
tv.blog.yqvyet.cn/Article/details/517577.sHtML
tv.blog.yqvyet.cn/Article/details/733155.sHtML
tv.blog.yqvyet.cn/Article/details/397191.sHtML
tv.blog.yqvyet.cn/Article/details/175551.sHtML
tv.blog.yqvyet.cn/Article/details/519917.sHtML
tv.blog.yqvyet.cn/Article/details/971775.sHtML
tv.blog.yqvyet.cn/Article/details/137171.sHtML
tv.blog.yqvyet.cn/Article/details/133957.sHtML
tv.blog.yqvyet.cn/Article/details/599371.sHtML
tv.blog.yqvyet.cn/Article/details/353751.sHtML
tv.blog.yqvyet.cn/Article/details/339559.sHtML
tv.blog.yqvyet.cn/Article/details/919579.sHtML
tv.blog.yqvyet.cn/Article/details/735971.sHtML
tv.blog.yqvyet.cn/Article/details/157315.sHtML
tv.blog.yqvyet.cn/Article/details/573351.sHtML
tv.blog.yqvyet.cn/Article/details/339179.sHtML
tv.blog.yqvyet.cn/Article/details/513719.sHtML
tv.blog.yqvyet.cn/Article/details/377991.sHtML
tv.blog.yqvyet.cn/Article/details/335755.sHtML
tv.blog.yqvyet.cn/Article/details/111337.sHtML
tv.blog.yqvyet.cn/Article/details/193951.sHtML
tv.blog.yqvyet.cn/Article/details/955713.sHtML
tv.blog.yqvyet.cn/Article/details/753719.sHtML
tv.blog.yqvyet.cn/Article/details/139375.sHtML
tv.blog.yqvyet.cn/Article/details/971557.sHtML
tv.blog.yqvyet.cn/Article/details/373551.sHtML
tv.blog.yqvyet.cn/Article/details/979399.sHtML
tv.blog.yqvyet.cn/Article/details/595599.sHtML
tv.blog.yqvyet.cn/Article/details/791937.sHtML
tv.blog.yqvyet.cn/Article/details/199155.sHtML
tv.blog.yqvyet.cn/Article/details/557379.sHtML
tv.blog.yqvyet.cn/Article/details/713919.sHtML
tv.blog.yqvyet.cn/Article/details/377173.sHtML
tv.blog.yqvyet.cn/Article/details/113115.sHtML
tv.blog.yqvyet.cn/Article/details/717933.sHtML
tv.blog.yqvyet.cn/Article/details/115937.sHtML
tv.blog.yqvyet.cn/Article/details/553199.sHtML
tv.blog.yqvyet.cn/Article/details/371117.sHtML
tv.blog.yqvyet.cn/Article/details/931317.sHtML
tv.blog.yqvyet.cn/Article/details/135713.sHtML
tv.blog.yqvyet.cn/Article/details/135359.sHtML
tv.blog.yqvyet.cn/Article/details/939719.sHtML
tv.blog.yqvyet.cn/Article/details/799519.sHtML
tv.blog.yqvyet.cn/Article/details/191197.sHtML
tv.blog.yqvyet.cn/Article/details/773793.sHtML
tv.blog.yqvyet.cn/Article/details/355151.sHtML
tv.blog.yqvyet.cn/Article/details/177779.sHtML
tv.blog.yqvyet.cn/Article/details/995191.sHtML
tv.blog.yqvyet.cn/Article/details/519759.sHtML
tv.blog.yqvyet.cn/Article/details/991159.sHtML
tv.blog.yqvyet.cn/Article/details/311933.sHtML
tv.blog.yqvyet.cn/Article/details/731913.sHtML
tv.blog.yqvyet.cn/Article/details/793737.sHtML
tv.blog.yqvyet.cn/Article/details/539773.sHtML
tv.blog.yqvyet.cn/Article/details/371133.sHtML
tv.blog.yqvyet.cn/Article/details/919397.sHtML
tv.blog.yqvyet.cn/Article/details/357771.sHtML
tv.blog.yqvyet.cn/Article/details/971131.sHtML
tv.blog.yqvyet.cn/Article/details/773159.sHtML
tv.blog.yqvyet.cn/Article/details/331537.sHtML
tv.blog.yqvyet.cn/Article/details/313799.sHtML
tv.blog.yqvyet.cn/Article/details/319511.sHtML
tv.blog.yqvyet.cn/Article/details/931335.sHtML
tv.blog.yqvyet.cn/Article/details/993513.sHtML
tv.blog.yqvyet.cn/Article/details/715517.sHtML
tv.blog.yqvyet.cn/Article/details/391973.sHtML
tv.blog.yqvyet.cn/Article/details/753911.sHtML
tv.blog.yqvyet.cn/Article/details/157957.sHtML
tv.blog.yqvyet.cn/Article/details/997737.sHtML
tv.blog.yqvyet.cn/Article/details/133379.sHtML
tv.blog.yqvyet.cn/Article/details/191715.sHtML
tv.blog.yqvyet.cn/Article/details/533977.sHtML
tv.blog.yqvyet.cn/Article/details/735977.sHtML
tv.blog.yqvyet.cn/Article/details/399931.sHtML
tv.blog.yqvyet.cn/Article/details/379159.sHtML
tv.blog.yqvyet.cn/Article/details/395533.sHtML
tv.blog.yqvyet.cn/Article/details/975513.sHtML
tv.blog.yqvyet.cn/Article/details/573199.sHtML
tv.blog.yqvyet.cn/Article/details/913951.sHtML
tv.blog.yqvyet.cn/Article/details/593993.sHtML
tv.blog.yqvyet.cn/Article/details/731391.sHtML
tv.blog.yqvyet.cn/Article/details/599911.sHtML
tv.blog.yqvyet.cn/Article/details/751777.sHtML
tv.blog.yqvyet.cn/Article/details/933731.sHtML
tv.blog.yqvyet.cn/Article/details/735937.sHtML
tv.blog.yqvyet.cn/Article/details/397171.sHtML
tv.blog.yqvyet.cn/Article/details/759317.sHtML
tv.blog.yqvyet.cn/Article/details/995953.sHtML
tv.blog.yqvyet.cn/Article/details/177935.sHtML
tv.blog.yqvyet.cn/Article/details/571171.sHtML
tv.blog.yqvyet.cn/Article/details/711733.sHtML
tv.blog.yqvyet.cn/Article/details/793759.sHtML
tv.blog.yqvyet.cn/Article/details/311333.sHtML
tv.blog.yqvyet.cn/Article/details/197379.sHtML
tv.blog.yqvyet.cn/Article/details/117511.sHtML
tv.blog.yqvyet.cn/Article/details/759311.sHtML
tv.blog.yqvyet.cn/Article/details/799333.sHtML
tv.blog.yqvyet.cn/Article/details/391117.sHtML
tv.blog.yqvyet.cn/Article/details/519373.sHtML
tv.blog.yqvyet.cn/Article/details/337395.sHtML
tv.blog.yqvyet.cn/Article/details/191599.sHtML
tv.blog.yqvyet.cn/Article/details/595511.sHtML
tv.blog.yqvyet.cn/Article/details/953351.sHtML
tv.blog.yqvyet.cn/Article/details/915955.sHtML
tv.blog.yqvyet.cn/Article/details/117917.sHtML
tv.blog.yqvyet.cn/Article/details/595797.sHtML
tv.blog.yqvyet.cn/Article/details/793719.sHtML
tv.blog.yqvyet.cn/Article/details/751319.sHtML
tv.blog.yqvyet.cn/Article/details/199511.sHtML
tv.blog.yqvyet.cn/Article/details/775793.sHtML
tv.blog.yqvyet.cn/Article/details/535733.sHtML
tv.blog.yqvyet.cn/Article/details/159357.sHtML
tv.blog.yqvyet.cn/Article/details/597315.sHtML
tv.blog.yqvyet.cn/Article/details/355919.sHtML
tv.blog.yqvyet.cn/Article/details/919773.sHtML
tv.blog.yqvyet.cn/Article/details/795137.sHtML
tv.blog.yqvyet.cn/Article/details/979157.sHtML
tv.blog.yqvyet.cn/Article/details/195713.sHtML
tv.blog.yqvyet.cn/Article/details/111133.sHtML
tv.blog.yqvyet.cn/Article/details/155517.sHtML
tv.blog.yqvyet.cn/Article/details/739951.sHtML
tv.blog.yqvyet.cn/Article/details/791173.sHtML
tv.blog.yqvyet.cn/Article/details/171395.sHtML
tv.blog.yqvyet.cn/Article/details/599391.sHtML
tv.blog.yqvyet.cn/Article/details/113935.sHtML
tv.blog.yqvyet.cn/Article/details/377713.sHtML
tv.blog.yqvyet.cn/Article/details/337779.sHtML
tv.blog.yqvyet.cn/Article/details/133595.sHtML
tv.blog.yqvyet.cn/Article/details/717797.sHtML
tv.blog.yqvyet.cn/Article/details/397973.sHtML
tv.blog.yqvyet.cn/Article/details/373575.sHtML
tv.blog.yqvyet.cn/Article/details/737171.sHtML
tv.blog.yqvyet.cn/Article/details/799717.sHtML
tv.blog.yqvyet.cn/Article/details/315559.sHtML
tv.blog.yqvyet.cn/Article/details/391535.sHtML
tv.blog.yqvyet.cn/Article/details/555715.sHtML
tv.blog.yqvyet.cn/Article/details/555357.sHtML
tv.blog.yqvyet.cn/Article/details/359795.sHtML
tv.blog.yqvyet.cn/Article/details/999331.sHtML
tv.blog.yqvyet.cn/Article/details/977599.sHtML
tv.blog.yqvyet.cn/Article/details/553395.sHtML
tv.blog.yqvyet.cn/Article/details/513379.sHtML
tv.blog.yqvyet.cn/Article/details/799593.sHtML
tv.blog.yqvyet.cn/Article/details/915133.sHtML
tv.blog.yqvyet.cn/Article/details/915997.sHtML
tv.blog.yqvyet.cn/Article/details/533379.sHtML
tv.blog.yqvyet.cn/Article/details/773379.sHtML
tv.blog.yqvyet.cn/Article/details/319311.sHtML
tv.blog.yqvyet.cn/Article/details/773191.sHtML
tv.blog.yqvyet.cn/Article/details/995735.sHtML
tv.blog.yqvyet.cn/Article/details/517355.sHtML
tv.blog.yqvyet.cn/Article/details/739731.sHtML
tv.blog.yqvyet.cn/Article/details/333133.sHtML
tv.blog.yqvyet.cn/Article/details/791951.sHtML
tv.blog.yqvyet.cn/Article/details/555557.sHtML
tv.blog.yqvyet.cn/Article/details/959395.sHtML
tv.blog.yqvyet.cn/Article/details/191557.sHtML
tv.blog.yqvyet.cn/Article/details/337731.sHtML
tv.blog.yqvyet.cn/Article/details/795355.sHtML
tv.blog.yqvyet.cn/Article/details/115535.sHtML
tv.blog.yqvyet.cn/Article/details/773153.sHtML
tv.blog.yqvyet.cn/Article/details/577571.sHtML
tv.blog.yqvyet.cn/Article/details/597733.sHtML
tv.blog.yqvyet.cn/Article/details/579331.sHtML
tv.blog.yqvyet.cn/Article/details/373733.sHtML
tv.blog.yqvyet.cn/Article/details/139735.sHtML
tv.blog.yqvyet.cn/Article/details/759135.sHtML
tv.blog.yqvyet.cn/Article/details/199979.sHtML
tv.blog.yqvyet.cn/Article/details/199393.sHtML
tv.blog.yqvyet.cn/Article/details/937579.sHtML
tv.blog.yqvyet.cn/Article/details/939771.sHtML
tv.blog.yqvyet.cn/Article/details/199917.sHtML
tv.blog.yqvyet.cn/Article/details/555937.sHtML
tv.blog.yqvyet.cn/Article/details/171593.sHtML
tv.blog.yqvyet.cn/Article/details/773133.sHtML
tv.blog.yqvyet.cn/Article/details/171151.sHtML
tv.blog.yqvyet.cn/Article/details/515535.sHtML
tv.blog.yqvyet.cn/Article/details/371335.sHtML
tv.blog.yqvyet.cn/Article/details/157777.sHtML
tv.blog.yqvyet.cn/Article/details/357311.sHtML
tv.blog.yqvyet.cn/Article/details/599331.sHtML
tv.blog.yqvyet.cn/Article/details/977917.sHtML
tv.blog.yqvyet.cn/Article/details/939713.sHtML
tv.blog.yqvyet.cn/Article/details/133111.sHtML
tv.blog.yqvyet.cn/Article/details/979591.sHtML
tv.blog.yqvyet.cn/Article/details/755919.sHtML
tv.blog.yqvyet.cn/Article/details/151153.sHtML
tv.blog.yqvyet.cn/Article/details/139153.sHtML
tv.blog.yqvyet.cn/Article/details/331359.sHtML
tv.blog.yqvyet.cn/Article/details/113195.sHtML
tv.blog.yqvyet.cn/Article/details/319117.sHtML
tv.blog.yqvyet.cn/Article/details/151955.sHtML
tv.blog.yqvyet.cn/Article/details/157733.sHtML
tv.blog.yqvyet.cn/Article/details/791371.sHtML
tv.blog.yqvyet.cn/Article/details/717913.sHtML
tv.blog.yqvyet.cn/Article/details/995997.sHtML
tv.blog.yqvyet.cn/Article/details/739957.sHtML
tv.blog.yqvyet.cn/Article/details/359975.sHtML
tv.blog.yqvyet.cn/Article/details/755779.sHtML
tv.blog.yqvyet.cn/Article/details/795715.sHtML
tv.blog.yqvyet.cn/Article/details/533155.sHtML
tv.blog.yqvyet.cn/Article/details/535359.sHtML
tv.blog.yqvyet.cn/Article/details/959537.sHtML
tv.blog.yqvyet.cn/Article/details/979773.sHtML
tv.blog.yqvyet.cn/Article/details/319335.sHtML
tv.blog.yqvyet.cn/Article/details/377595.sHtML
tv.blog.yqvyet.cn/Article/details/539119.sHtML
tv.blog.yqvyet.cn/Article/details/317559.sHtML
tv.blog.yqvyet.cn/Article/details/335711.sHtML
tv.blog.yqvyet.cn/Article/details/373751.sHtML
tv.blog.yqvyet.cn/Article/details/793951.sHtML
tv.blog.yqvyet.cn/Article/details/515715.sHtML
tv.blog.yqvyet.cn/Article/details/991155.sHtML
tv.blog.yqvyet.cn/Article/details/591391.sHtML
tv.blog.yqvyet.cn/Article/details/715593.sHtML
tv.blog.yqvyet.cn/Article/details/939713.sHtML
tv.blog.yqvyet.cn/Article/details/737775.sHtML
tv.blog.yqvyet.cn/Article/details/533931.sHtML
tv.blog.yqvyet.cn/Article/details/359737.sHtML
tv.blog.yqvyet.cn/Article/details/337331.sHtML
tv.blog.yqvyet.cn/Article/details/515537.sHtML
tv.blog.yqvyet.cn/Article/details/171131.sHtML
