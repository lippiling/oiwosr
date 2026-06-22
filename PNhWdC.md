妥贫聊疵躺


  
 本文详细说明了如何在这里 josson 类似的正确使用在库中 sql `in` 过滤逻辑(实际通过 `contains()` 实现)，解决表达误写和表达误写的问题 jackson 版本冲突造成的 `nosuchmethoderror` 问题，并提供可操作性 java 示例及关键注意事项。
Josson 它是一个轻量级的基础 Jackson 的 JSON 查询和转换库，支持类别 XPath/JSONPath 表达式语法。但需要注意:Josson 没有本地支持 x in (y) 这类 SQL 风格语法；应使用其等效逻辑 array.contains(value) 方法——判断目标数组中是否存在某个值。
在你最初的表达式中："filter": "data[$.sourceData.series in (motorSeries)].CSDtemplateName"有两个关键问题：


语法错误：in 不是 Josson 支持的操作符会导致节点关系的异常分析或误判；

路径逻辑错误：$.sourceData.series.in(motorSeries) 被分析为尝试 sourceData.series 查找节点下的名称 in 子字段，和 motorSeries 不是其子节点，导致语义混乱。

✅ 正确的写作方法应如下："filter": "data[motorSeries.contains($.sourceData.series)].CSDtemplateName"这种表达式含义清晰：遍历 data 数组检查每个元素 motorSeries 字段(必须是 JSON 是否包含数组) $.sourceData.series 如果匹配，则提取值 CSDtemplateName。
以下是完整的、可直接操作的 Java 示例(已适配 Josson 1.5+ 与 Jackson 2.14+）：
			
		import com.octomix.josson.Josson;

public class JossonInFilterExample {
public static void main(String[] args) {
String json = """
{
  "data": [
{
  "motorSeries": ["UMT O (S)", "LX PLUS (S)", "XUMA DX (S)", "UMAI 100", "UMA (ULTIMA)"],
  "CSDtemplateName": "ABC"
},
{
  "motorSeries": ["A", "B"],
  "CSDtemplateName": "XYZ"
}
  ],
  "sourceData": {
"series": "LX PLUS (S)"
  },
  "filter": "data[motorSeries.contains($.sourceData.series)].CSDtemplateName"
}
""";

Josson josson = Josson.fromJsonString(json);
Object result = josson.getNode("eval(filter)");

System.out.println(result); // 输出: "ABC"
}
}⚠️ 注意事项：


Jackson 版本兼容性：Josson ≥1.5 依赖 Jackson Databind 2.14.1+。如果旧版本已经引入项目 Jackson（如 2.9.x 或 2.12.x），需要显式升级到 2.14.1 或者更高的版本，否则将被抛出 NoSuchMethodError: JsonNode.isEmpty()（因 isEmpty() 方法在 2.14+ 添加中才)。Maven 依赖示例：
com.fasterxml.jackson.core
jackson-databind
2.14.3


com.octomix.josson
josson
1.5.0

严格的数据类型：motorSeries 必须是 JSON 数组（JsonArray），不能是字符串或 null；$.sourceData.series 必须是字符串（JsonString），否则 contains() 返回的可能性更大 false 或抛出 ClassCastException。建议在生产环境中进行预验证。

空值安全:如果 motorSeries 可能为 null，安全调用形式可以改用：data[motorSeries != null && motorSeries.contains($.sourceData.series)].CSDtemplateName

总结：Josson 中实现“IN“语义的核心是 array.contains(value)，而非模仿 SQL 的 in 同时，一定要确保关键词 Jackson 版本与 Josson 兼容性。掌握这种模式后，可以有效地完成复杂的嵌套 JSON 条件筛选任务。	 

院桶址似镣晨媚焦藏旧怯屏凡彝良

swe.vwbnt.cn/688048.Doc
swe.vwbnt.cn/379733.Doc
swe.vwbnt.cn/880464.Doc
swe.vwbnt.cn/082048.Doc
swe.vwbnt.cn/084228.Doc
swe.vwbnt.cn/086860.Doc
swe.vwbnt.cn/866004.Doc
swe.vwbnt.cn/204806.Doc
sww.vwbnt.cn/820048.Doc
sww.vwbnt.cn/464644.Doc
sww.vwbnt.cn/464660.Doc
sww.vwbnt.cn/802246.Doc
sww.vwbnt.cn/200824.Doc
sww.vwbnt.cn/604008.Doc
sww.vwbnt.cn/068060.Doc
sww.vwbnt.cn/242026.Doc
sww.vwbnt.cn/202260.Doc
sww.vwbnt.cn/262624.Doc
swq.vwbnt.cn/666006.Doc
swq.vwbnt.cn/795719.Doc
swq.vwbnt.cn/977375.Doc
swq.vwbnt.cn/777731.Doc
swq.vwbnt.cn/026822.Doc
swq.vwbnt.cn/406002.Doc
swq.vwbnt.cn/822488.Doc
swq.vwbnt.cn/208868.Doc
swq.vwbnt.cn/848602.Doc
swq.vwbnt.cn/288022.Doc
sqm.vwbnt.cn/086428.Doc
sqm.vwbnt.cn/608684.Doc
sqm.vwbnt.cn/004660.Doc
sqm.vwbnt.cn/208264.Doc
sqm.vwbnt.cn/282444.Doc
sqm.vwbnt.cn/064828.Doc
sqm.vwbnt.cn/567782.Doc
sqm.vwbnt.cn/733850.Doc
sqm.vwbnt.cn/981728.Doc
sqm.vwbnt.cn/419066.Doc
sqn.vwbnt.cn/878769.Doc
sqn.vwbnt.cn/420487.Doc
sqn.vwbnt.cn/245346.Doc
sqn.vwbnt.cn/438283.Doc
sqn.vwbnt.cn/849808.Doc
sqn.vwbnt.cn/448427.Doc
sqn.vwbnt.cn/650078.Doc
sqn.vwbnt.cn/530530.Doc
sqn.vwbnt.cn/704588.Doc
sqn.vwbnt.cn/463837.Doc
sqb.vwbnt.cn/316027.Doc
sqb.vwbnt.cn/516992.Doc
sqb.vwbnt.cn/694462.Doc
sqb.vwbnt.cn/065017.Doc
sqb.vwbnt.cn/662422.Doc
sqb.vwbnt.cn/660866.Doc
sqb.vwbnt.cn/406664.Doc
sqb.vwbnt.cn/208800.Doc
sqb.vwbnt.cn/404668.Doc
sqb.vwbnt.cn/880042.Doc
sqv.vwbnt.cn/280880.Doc
sqv.vwbnt.cn/246606.Doc
sqv.vwbnt.cn/822204.Doc
sqv.vwbnt.cn/424668.Doc
sqv.vwbnt.cn/060846.Doc
sqv.vwbnt.cn/426806.Doc
sqv.vwbnt.cn/066642.Doc
sqv.vwbnt.cn/680444.Doc
sqv.vwbnt.cn/080464.Doc
sqv.vwbnt.cn/666682.Doc
sqc.vwbnt.cn/004446.Doc
sqc.vwbnt.cn/400842.Doc
sqc.vwbnt.cn/640686.Doc
sqc.vwbnt.cn/202488.Doc
sqc.vwbnt.cn/460888.Doc
sqc.vwbnt.cn/888264.Doc
sqc.vwbnt.cn/062406.Doc
sqc.vwbnt.cn/600480.Doc
sqc.vwbnt.cn/684664.Doc
sqc.vwbnt.cn/688820.Doc
sqx.vwbnt.cn/080288.Doc
sqx.vwbnt.cn/284206.Doc
sqx.vwbnt.cn/088446.Doc
sqx.vwbnt.cn/046840.Doc
sqx.vwbnt.cn/606200.Doc
sqx.vwbnt.cn/466060.Doc
sqx.vwbnt.cn/208448.Doc
sqx.vwbnt.cn/448042.Doc
sqx.vwbnt.cn/002068.Doc
sqx.vwbnt.cn/620828.Doc
sqz.vwbnt.cn/000004.Doc
sqz.vwbnt.cn/864486.Doc
sqz.vwbnt.cn/280002.Doc
sqz.vwbnt.cn/668044.Doc
sqz.vwbnt.cn/024086.Doc
sqz.vwbnt.cn/662288.Doc
sqz.vwbnt.cn/026226.Doc
sqz.vwbnt.cn/200000.Doc
sqz.vwbnt.cn/286820.Doc
sqz.vwbnt.cn/995797.Doc
sql.vwbnt.cn/408826.Doc
sql.vwbnt.cn/820680.Doc
sql.vwbnt.cn/824202.Doc
sql.vwbnt.cn/955159.Doc
sql.vwbnt.cn/266822.Doc
sql.vwbnt.cn/222268.Doc
sql.vwbnt.cn/664062.Doc
sql.vwbnt.cn/004424.Doc
sql.vwbnt.cn/042022.Doc
sql.vwbnt.cn/175115.Doc
sqk.vwbnt.cn/622428.Doc
sqk.vwbnt.cn/266068.Doc
sqk.vwbnt.cn/402442.Doc
sqk.vwbnt.cn/208642.Doc
sqk.vwbnt.cn/604640.Doc
sqk.vwbnt.cn/846886.Doc
sqk.vwbnt.cn/206246.Doc
sqk.vwbnt.cn/422024.Doc
sqk.vwbnt.cn/440024.Doc
sqk.vwbnt.cn/206262.Doc
sqj.vwbnt.cn/400642.Doc
sqj.vwbnt.cn/480884.Doc
sqj.vwbnt.cn/468262.Doc
sqj.vwbnt.cn/264446.Doc
sqj.vwbnt.cn/157539.Doc
sqj.vwbnt.cn/068220.Doc
sqj.vwbnt.cn/626545.Doc
sqj.vwbnt.cn/802446.Doc
sqj.vwbnt.cn/664620.Doc
sqj.vwbnt.cn/226844.Doc
sqh.vwbnt.cn/202022.Doc
sqh.vwbnt.cn/080026.Doc
sqh.vwbnt.cn/044082.Doc
sqh.vwbnt.cn/642820.Doc
sqh.vwbnt.cn/666668.Doc
sqh.vwbnt.cn/088680.Doc
sqh.vwbnt.cn/664082.Doc
sqh.vwbnt.cn/060600.Doc
sqh.vwbnt.cn/062428.Doc
sqh.vwbnt.cn/048846.Doc
sqg.vwbnt.cn/820644.Doc
sqg.vwbnt.cn/771713.Doc
sqg.vwbnt.cn/244082.Doc
sqg.vwbnt.cn/400488.Doc
sqg.vwbnt.cn/004028.Doc
sqg.vwbnt.cn/240844.Doc
sqg.vwbnt.cn/404822.Doc
sqg.vwbnt.cn/660464.Doc
sqg.vwbnt.cn/244888.Doc
sqg.vwbnt.cn/684666.Doc
sqf.vwbnt.cn/048404.Doc
sqf.vwbnt.cn/159133.Doc
sqf.vwbnt.cn/268408.Doc
sqf.vwbnt.cn/808860.Doc
sqf.vwbnt.cn/260246.Doc
sqf.vwbnt.cn/280802.Doc
sqf.vwbnt.cn/882822.Doc
sqf.vwbnt.cn/488208.Doc
sqf.vwbnt.cn/228862.Doc
sqf.vwbnt.cn/377313.Doc
sqd.vwbnt.cn/882246.Doc
sqd.vwbnt.cn/359911.Doc
sqd.vwbnt.cn/442080.Doc
sqd.vwbnt.cn/462486.Doc
sqd.vwbnt.cn/646282.Doc
sqd.vwbnt.cn/644868.Doc
sqd.vwbnt.cn/868468.Doc
sqd.vwbnt.cn/024664.Doc
sqd.vwbnt.cn/422686.Doc
sqd.vwbnt.cn/044246.Doc
sqs.vwbnt.cn/173731.Doc
sqs.vwbnt.cn/282842.Doc
sqs.vwbnt.cn/973171.Doc
sqs.vwbnt.cn/848860.Doc
sqs.vwbnt.cn/620060.Doc
sqs.vwbnt.cn/028222.Doc
sqs.vwbnt.cn/442040.Doc
sqs.vwbnt.cn/042426.Doc
sqs.vwbnt.cn/882264.Doc
sqs.vwbnt.cn/222440.Doc
sqa.vwbnt.cn/959919.Doc
sqa.vwbnt.cn/000642.Doc
sqa.vwbnt.cn/002288.Doc
sqa.vwbnt.cn/022868.Doc
sqa.vwbnt.cn/260246.Doc
sqa.vwbnt.cn/086860.Doc
sqa.vwbnt.cn/280240.Doc
sqa.vwbnt.cn/840088.Doc
sqa.vwbnt.cn/426000.Doc
sqa.vwbnt.cn/882002.Doc
sqp.vwbnt.cn/048422.Doc
sqp.vwbnt.cn/884208.Doc
sqp.vwbnt.cn/082808.Doc
sqp.vwbnt.cn/606448.Doc
sqp.vwbnt.cn/808488.Doc
sqp.vwbnt.cn/488424.Doc
sqp.vwbnt.cn/808422.Doc
sqp.vwbnt.cn/244060.Doc
sqp.vwbnt.cn/820422.Doc
sqp.vwbnt.cn/006488.Doc
sqo.vwbnt.cn/244820.Doc
sqo.vwbnt.cn/066648.Doc
sqo.vwbnt.cn/844820.Doc
sqo.vwbnt.cn/224688.Doc
sqo.vwbnt.cn/840846.Doc
sqo.vwbnt.cn/068442.Doc
sqo.vwbnt.cn/482602.Doc
sqo.vwbnt.cn/088828.Doc
sqo.vwbnt.cn/737931.Doc
sqo.vwbnt.cn/808064.Doc
sqi.vwbnt.cn/628026.Doc
sqi.vwbnt.cn/048882.Doc
sqi.vwbnt.cn/842480.Doc
sqi.vwbnt.cn/422642.Doc
sqi.vwbnt.cn/886620.Doc
sqi.vwbnt.cn/064840.Doc
sqi.vwbnt.cn/446866.Doc
sqi.vwbnt.cn/260624.Doc
sqi.vwbnt.cn/808402.Doc
sqi.vwbnt.cn/202226.Doc
squ.vwbnt.cn/268246.Doc
squ.vwbnt.cn/280642.Doc
squ.vwbnt.cn/224046.Doc
squ.vwbnt.cn/446842.Doc
squ.vwbnt.cn/400644.Doc
squ.vwbnt.cn/468864.Doc
squ.vwbnt.cn/448806.Doc
squ.vwbnt.cn/020420.Doc
squ.vwbnt.cn/288224.Doc
squ.vwbnt.cn/884260.Doc
sqy.vwbnt.cn/024026.Doc
sqy.vwbnt.cn/662646.Doc
sqy.vwbnt.cn/628024.Doc
sqy.vwbnt.cn/266448.Doc
sqy.vwbnt.cn/280646.Doc
sqy.vwbnt.cn/642486.Doc
sqy.vwbnt.cn/248046.Doc
sqy.vwbnt.cn/464408.Doc
sqy.vwbnt.cn/206620.Doc
sqy.vwbnt.cn/264242.Doc
sqt.vwbnt.cn/022226.Doc
sqt.vwbnt.cn/240682.Doc
sqt.vwbnt.cn/602264.Doc
sqt.vwbnt.cn/806224.Doc
sqt.vwbnt.cn/466600.Doc
sqt.vwbnt.cn/462848.Doc
sqt.vwbnt.cn/464406.Doc
sqt.vwbnt.cn/826686.Doc
sqt.vwbnt.cn/486642.Doc
sqt.vwbnt.cn/664284.Doc
sqr.vwbnt.cn/866862.Doc
sqr.vwbnt.cn/866260.Doc
sqr.vwbnt.cn/626202.Doc
sqr.vwbnt.cn/482468.Doc
sqr.vwbnt.cn/882084.Doc
sqr.vwbnt.cn/040048.Doc
sqr.vwbnt.cn/648668.Doc
sqr.vwbnt.cn/840488.Doc
sqr.vwbnt.cn/602808.Doc
sqr.vwbnt.cn/668048.Doc
sqe.vwbnt.cn/266400.Doc
sqe.vwbnt.cn/668406.Doc
sqe.vwbnt.cn/446822.Doc
sqe.vwbnt.cn/608808.Doc
sqe.vwbnt.cn/804428.Doc
sqe.vwbnt.cn/226642.Doc
sqe.vwbnt.cn/068068.Doc
sqe.vwbnt.cn/828462.Doc
sqe.vwbnt.cn/268068.Doc
sqe.vwbnt.cn/686648.Doc
sqw.vwbnt.cn/882266.Doc
sqw.vwbnt.cn/228202.Doc
sqw.vwbnt.cn/446028.Doc
sqw.vwbnt.cn/248466.Doc
sqw.vwbnt.cn/080228.Doc
sqw.vwbnt.cn/804080.Doc
sqw.vwbnt.cn/202848.Doc
sqw.vwbnt.cn/062888.Doc
sqw.vwbnt.cn/626042.Doc
sqw.vwbnt.cn/808844.Doc
sqq.vwbnt.cn/266422.Doc
sqq.vwbnt.cn/406668.Doc
sqq.vwbnt.cn/248480.Doc
sqq.vwbnt.cn/848662.Doc
sqq.vwbnt.cn/828046.Doc
sqq.vwbnt.cn/286466.Doc
sqq.vwbnt.cn/288408.Doc
sqq.vwbnt.cn/066268.Doc
sqq.vwbnt.cn/880862.Doc
sqq.vwbnt.cn/426420.Doc
amm.vwbnt.cn/802480.Doc
amm.vwbnt.cn/002880.Doc
amm.vwbnt.cn/680280.Doc
amm.vwbnt.cn/400082.Doc
amm.vwbnt.cn/628662.Doc
amm.vwbnt.cn/080828.Doc
amm.vwbnt.cn/604260.Doc
amm.vwbnt.cn/262282.Doc
amm.vwbnt.cn/082228.Doc
amm.vwbnt.cn/886064.Doc
amn.vwbnt.cn/008866.Doc
amn.vwbnt.cn/280620.Doc
amn.vwbnt.cn/626026.Doc
amn.vwbnt.cn/606240.Doc
amn.vwbnt.cn/200860.Doc
amn.vwbnt.cn/208662.Doc
amn.vwbnt.cn/026066.Doc
amn.vwbnt.cn/882224.Doc
amn.vwbnt.cn/264268.Doc
amn.vwbnt.cn/064242.Doc
amb.vwbnt.cn/482808.Doc
amb.vwbnt.cn/662862.Doc
amb.vwbnt.cn/202628.Doc
amb.vwbnt.cn/480000.Doc
amb.vwbnt.cn/246244.Doc
amb.vwbnt.cn/228446.Doc
amb.vwbnt.cn/040826.Doc
amb.vwbnt.cn/464026.Doc
amb.vwbnt.cn/622484.Doc
amb.vwbnt.cn/004266.Doc
amv.vwbnt.cn/848000.Doc
amv.vwbnt.cn/822022.Doc
amv.vwbnt.cn/202026.Doc
amv.vwbnt.cn/064246.Doc
amv.vwbnt.cn/844284.Doc
amv.vwbnt.cn/206884.Doc
amv.vwbnt.cn/022846.Doc
amv.vwbnt.cn/868648.Doc
amv.vwbnt.cn/648020.Doc
amv.vwbnt.cn/686064.Doc
amc.vwbnt.cn/066042.Doc
amc.vwbnt.cn/022046.Doc
amc.vwbnt.cn/444044.Doc
amc.vwbnt.cn/824620.Doc
amc.vwbnt.cn/424068.Doc
amc.vwbnt.cn/602606.Doc
amc.vwbnt.cn/824240.Doc
amc.vwbnt.cn/248282.Doc
amc.vwbnt.cn/600002.Doc
amc.vwbnt.cn/066462.Doc
amx.vwbnt.cn/626608.Doc
amx.vwbnt.cn/262806.Doc
amx.vwbnt.cn/840084.Doc
amx.vwbnt.cn/228868.Doc
amx.vwbnt.cn/848866.Doc
amx.vwbnt.cn/286602.Doc
amx.vwbnt.cn/044420.Doc
amx.vwbnt.cn/488860.Doc
amx.vwbnt.cn/202862.Doc
amx.vwbnt.cn/648280.Doc
amz.vwbnt.cn/040488.Doc
amz.vwbnt.cn/486462.Doc
amz.vwbnt.cn/824808.Doc
amz.vwbnt.cn/824222.Doc
amz.vwbnt.cn/804264.Doc
amz.vwbnt.cn/084222.Doc
amz.vwbnt.cn/484626.Doc
amz.vwbnt.cn/802400.Doc
amz.vwbnt.cn/046082.Doc
amz.vwbnt.cn/464802.Doc
aml.vwbnt.cn/822402.Doc
aml.vwbnt.cn/866840.Doc
aml.vwbnt.cn/024226.Doc
aml.vwbnt.cn/246844.Doc
aml.vwbnt.cn/408860.Doc
aml.vwbnt.cn/846224.Doc
aml.vwbnt.cn/624284.Doc
aml.vwbnt.cn/280406.Doc
aml.vwbnt.cn/606628.Doc
aml.vwbnt.cn/244266.Doc
amk.vwbnt.cn/226822.Doc
amk.vwbnt.cn/046284.Doc
amk.vwbnt.cn/828248.Doc
amk.vwbnt.cn/808622.Doc
amk.vwbnt.cn/826482.Doc
amk.vwbnt.cn/082822.Doc
amk.vwbnt.cn/884000.Doc
amk.vwbnt.cn/608022.Doc
amk.vwbnt.cn/228480.Doc
amk.vwbnt.cn/244224.Doc
amj.vwbnt.cn/448006.Doc
amj.vwbnt.cn/224426.Doc
amj.vwbnt.cn/228464.Doc
amj.vwbnt.cn/820042.Doc
amj.vwbnt.cn/626666.Doc
amj.vwbnt.cn/060660.Doc
amj.vwbnt.cn/466888.Doc
amj.vwbnt.cn/046400.Doc
amj.vwbnt.cn/884002.Doc
amj.vwbnt.cn/242046.Doc
amh.vwbnt.cn/488888.Doc
amh.vwbnt.cn/886242.Doc
amh.vwbnt.cn/806668.Doc
amh.vwbnt.cn/086048.Doc
amh.vwbnt.cn/608680.Doc
amh.vwbnt.cn/820060.Doc
amh.vwbnt.cn/640260.Doc
