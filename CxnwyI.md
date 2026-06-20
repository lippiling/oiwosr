毖陕谇纫檬


  




本文介绍在 Spring 启动应用程序后，如何通过事件监控或 CommandLineRunner 只有在动态加载运行时，才能确定路径的多个外部 .properties 文件，并确保其属性可用 @Value 和 Environment 同时访问。



  本文介绍在 spring 启动应用程序后，如何通过事件监控或 `commandlinerunner` 只有在动态加载运行时，才能确定路径的多个外部 `.properties` 文件，并确保其属性可用 `@value` 和 `environment` 同时访问。
在 Spring 中，@PropertySource 和 @PropertySources 仅仅在编译过程中支持已知的静态资源路径，无法满足“启动时动态分析”的要求 VM 参数、扫描目录、加载未知数量的外部配置文件的场景。直接使用 PropertySourcesPlaceholderConfigurer(如问题所示)虽然可以注入 @Value，但属性不会注册 Spring 的 Environment 中-这是因为 PropertySourcesPlaceholderConfigurer 仅参与占位符分析，不修改 ConfigurableEnvironment 的 PropertySource 层级结构。
动态加载属性应真正纳入 Environment 并且全局可用，必须在那里 Spring 容器初始化完成后，手动方向 ConfigurableEnvironment 添加 PropertySource。建议在应用生命周期的适当阶段进行操作。最好的实践是监控 ApplicationStartedEvent 或 ApplicationReadyEvent，或实现 CommandLineRunner。
✅ 推荐方案：使用 ApplicationStartedEvent 监听器
ApplicationStartedEvent 在 ApplicationContext 创建完成，一切 BeanFactoryPostProcessor 执行之后、ApplicationRunner/CommandLineRunner 此前触发，此时 Environment 已安全获取和修改：@Component
public class DynamicPropertyLoader implements ApplicationRunner {

private final ConfigurableEnvironment environment;
private final ResourceLoader resourceLoader;

public DynamicPropertyLoader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
this.environment = environment;
this.resourceLoader = resourceLoader;
}

@Override
public void run(ApplicationArguments args) throws Exception {
// 1. 解析 VM 参数(例如:-Dconfig.locations=/opt/conf/a.properties,/opt/conf/b.properties）
String locations = System.getProperty(&quot;config.locations&quot;);
if (StringUtils.isBlank(locations)) {
return;
}

// 2. 分割路径，加载每个文件
String[] paths = locations.split(&quot;,&quot;);
for (String path : paths) {
try {
Resource resource = resourceLoader.getResource(&quot;file:&quot; + path.trim());
if (!resource.exists()) {
System.err.println(&quot;Warning: config file not found: &quot; + path);
continue;
}

// 3. 解析为 Properties 并注册为 PropertySource
Properties props = PropertiesLoaderUtils.loadProperties(resource);
environment.getPropertySources().addLast(
new PropertiesPropertySource(&quot;dynamic-&quot; + DigestUtils.md5Hex(path), props)
);
System.out.println(&quot;Loaded dynamic properties from: &quot; + path);

} catch (IOException e) {
throw new IllegalStateException(&quot;Failed to load property file: &quot; + path, e);
}
}
}
}
? 关键点说明:  

使用 resourceLoader.getResource("file:" + path) 支持绝对路径(如 /etc/app/config.properties）；  
environment.getPropertySources().addLast(...) 确保添加新属性源 Environment，后续调用 environment.getProperty("xxx") 或 @Value("${xxx}") 均可生效；  
PropertiesPropertySource 是 Spring 内置类型，语义清晰兼容 @ConfigurationProperties；  
使用唯一的名字(如带) MD5 避免重复注册冲突。


⚠️ 注意事项和最佳实践

注册时间非常重要：
若在 ApplicationContext 在初始化之前(如 BeanDefinitionRegistryPostProcessor）尝试修改 Environment，可能因上下文未就绪而失败；ApplicationRunner / CommandLineRunner 是最安全的选择（ApplicationStartedEvent 需配合 @EventListener，逻辑等效)。

属性优先控制：addLast() 表示最低优先级；如果需要高优先级(覆盖) application.properties），请改用 addFirst().建议按业务语义命名(如 "external-overrides调试方便：

environment.getPropertySources().addFirst(
new PropertiesPropertySource(&quot;external-overrides&quot;, props)
);
错误处理和可观测性：
动态加载失败不应导致应用程序启动失败（除非强烈依赖）。建议记录 WARN 日志而不是抛异常，并提供 fallback 机制(如默认值或健康检查报警)。
Spring Boot 2.4+ 兼容提示：兼容提示：
新版本引入 ConfigDataLocationResolver 和 ConfigDataLoader 机制，如果项目已经升级，建议自定义 ConfigDataLocationResolver 实现原生集成(实现原生集成(实现原生集成) org.springframework.boot.context.config.ConfigDataLocationResolver 接口)，但对于简单的场景，以上 ApplicationRunner 该方案仍然完全适用且更轻。

✅ 验证是否有效
注入 Environment 并测试读取：@Service
public class ConfigTester {

@Autowired
private Environment env;

public void test() {
String value = env.getProperty(&quot;my.property&quot;); // ✅ 这里将返回动态加载值
System.out.println(&quot;Dynamic property: &quot; + value);
}
}同时 @Value("${my.property:default}") 正常分析也将进行。
通过以上方式，您可以灵活加载任何数量和路径的外部属性文件，而无需重启应用程序，并确保其与 Spring 生态（@Value、@ConfigurationProperties、Environment API）无缝集成。	 

置好灰握胤挪识谋致膛冒霞霞铱魏

tv.blog.vgdrev.cn/Article/details/515775.sHtML
tv.blog.vgdrev.cn/Article/details/688484.sHtML
tv.blog.vgdrev.cn/Article/details/282204.sHtML
tv.blog.vgdrev.cn/Article/details/664800.sHtML
tv.blog.vgdrev.cn/Article/details/953179.sHtML
tv.blog.vgdrev.cn/Article/details/791991.sHtML
tv.blog.vgdrev.cn/Article/details/266404.sHtML
tv.blog.vgdrev.cn/Article/details/064228.sHtML
tv.blog.vgdrev.cn/Article/details/864846.sHtML
tv.blog.vgdrev.cn/Article/details/020404.sHtML
tv.blog.vgdrev.cn/Article/details/246240.sHtML
tv.blog.vgdrev.cn/Article/details/824266.sHtML
tv.blog.vgdrev.cn/Article/details/484204.sHtML
tv.blog.vgdrev.cn/Article/details/264448.sHtML
tv.blog.vgdrev.cn/Article/details/575795.sHtML
tv.blog.vgdrev.cn/Article/details/880468.sHtML
tv.blog.vgdrev.cn/Article/details/882440.sHtML
tv.blog.vgdrev.cn/Article/details/113179.sHtML
tv.blog.vgdrev.cn/Article/details/971337.sHtML
tv.blog.vgdrev.cn/Article/details/777951.sHtML
tv.blog.vgdrev.cn/Article/details/139913.sHtML
tv.blog.vgdrev.cn/Article/details/664624.sHtML
tv.blog.vgdrev.cn/Article/details/484048.sHtML
tv.blog.vgdrev.cn/Article/details/999939.sHtML
tv.blog.vgdrev.cn/Article/details/660248.sHtML
tv.blog.vgdrev.cn/Article/details/444288.sHtML
tv.blog.vgdrev.cn/Article/details/537359.sHtML
tv.blog.vgdrev.cn/Article/details/555339.sHtML
tv.blog.vgdrev.cn/Article/details/266820.sHtML
tv.blog.vgdrev.cn/Article/details/951931.sHtML
tv.blog.vgdrev.cn/Article/details/866844.sHtML
tv.blog.vgdrev.cn/Article/details/820246.sHtML
tv.blog.vgdrev.cn/Article/details/686662.sHtML
tv.blog.vgdrev.cn/Article/details/044488.sHtML
tv.blog.vgdrev.cn/Article/details/020000.sHtML
tv.blog.vgdrev.cn/Article/details/555937.sHtML
tv.blog.vgdrev.cn/Article/details/377519.sHtML
tv.blog.vgdrev.cn/Article/details/880666.sHtML
tv.blog.vgdrev.cn/Article/details/040028.sHtML
tv.blog.vgdrev.cn/Article/details/199957.sHtML
tv.blog.vgdrev.cn/Article/details/317553.sHtML
tv.blog.vgdrev.cn/Article/details/608400.sHtML
tv.blog.vgdrev.cn/Article/details/008402.sHtML
tv.blog.vgdrev.cn/Article/details/866804.sHtML
tv.blog.vgdrev.cn/Article/details/220086.sHtML
tv.blog.vgdrev.cn/Article/details/862284.sHtML
tv.blog.vgdrev.cn/Article/details/157533.sHtML
tv.blog.vgdrev.cn/Article/details/888822.sHtML
tv.blog.vgdrev.cn/Article/details/577775.sHtML
tv.blog.vgdrev.cn/Article/details/224004.sHtML
tv.blog.vgdrev.cn/Article/details/682666.sHtML
tv.blog.vgdrev.cn/Article/details/082826.sHtML
tv.blog.vgdrev.cn/Article/details/800822.sHtML
tv.blog.vgdrev.cn/Article/details/197757.sHtML
tv.blog.vgdrev.cn/Article/details/466428.sHtML
tv.blog.vgdrev.cn/Article/details/446248.sHtML
tv.blog.vgdrev.cn/Article/details/820040.sHtML
tv.blog.vgdrev.cn/Article/details/919335.sHtML
tv.blog.vgdrev.cn/Article/details/260480.sHtML
tv.blog.vgdrev.cn/Article/details/391791.sHtML
tv.blog.vgdrev.cn/Article/details/406028.sHtML
tv.blog.vgdrev.cn/Article/details/446044.sHtML
tv.blog.vgdrev.cn/Article/details/193113.sHtML
tv.blog.vgdrev.cn/Article/details/317595.sHtML
tv.blog.vgdrev.cn/Article/details/795395.sHtML
tv.blog.vgdrev.cn/Article/details/202804.sHtML
tv.blog.vgdrev.cn/Article/details/197179.sHtML
tv.blog.vgdrev.cn/Article/details/991557.sHtML
tv.blog.vgdrev.cn/Article/details/391739.sHtML
tv.blog.vgdrev.cn/Article/details/044402.sHtML
tv.blog.vgdrev.cn/Article/details/999955.sHtML
tv.blog.vgdrev.cn/Article/details/604884.sHtML
tv.blog.vgdrev.cn/Article/details/357979.sHtML
tv.blog.vgdrev.cn/Article/details/977341.sHtML
tv.blog.vgdrev.cn/Article/details/002064.sHtML
tv.blog.vgdrev.cn/Article/details/971113.sHtML
tv.blog.vgdrev.cn/Article/details/628402.sHtML
tv.blog.vgdrev.cn/Article/details/984082.sHtML
tv.blog.vgdrev.cn/Article/details/535797.sHtML
tv.blog.vgdrev.cn/Article/details/531931.sHtML
tv.blog.vgdrev.cn/Article/details/997648.sHtML
tv.blog.vgdrev.cn/Article/details/938588.sHtML
tv.blog.vgdrev.cn/Article/details/975395.sHtML
tv.blog.vgdrev.cn/Article/details/252717.sHtML
tv.blog.vgdrev.cn/Article/details/468642.sHtML
tv.blog.vgdrev.cn/Article/details/260428.sHtML
tv.blog.vgdrev.cn/Article/details/885518.sHtML
tv.blog.vgdrev.cn/Article/details/662628.sHtML
tv.blog.vgdrev.cn/Article/details/876547.sHtML
tv.blog.vgdrev.cn/Article/details/044602.sHtML
tv.blog.vgdrev.cn/Article/details/373191.sHtML
tv.blog.vgdrev.cn/Article/details/151153.sHtML
tv.blog.vgdrev.cn/Article/details/311199.sHtML
tv.blog.vgdrev.cn/Article/details/002628.sHtML
tv.blog.vgdrev.cn/Article/details/608206.sHtML
tv.blog.vgdrev.cn/Article/details/022606.sHtML
tv.blog.vgdrev.cn/Article/details/048664.sHtML
tv.blog.vgdrev.cn/Article/details/113557.sHtML
tv.blog.vgdrev.cn/Article/details/717313.sHtML
tv.blog.vgdrev.cn/Article/details/172870.sHtML
tv.blog.vgdrev.cn/Article/details/224319.sHtML
tv.blog.vgdrev.cn/Article/details/357593.sHtML
tv.blog.vgdrev.cn/Article/details/024373.sHtML
tv.blog.vgdrev.cn/Article/details/408006.sHtML
tv.blog.vgdrev.cn/Article/details/137155.sHtML
tv.blog.vgdrev.cn/Article/details/048190.sHtML
tv.blog.vgdrev.cn/Article/details/820806.sHtML
tv.blog.vgdrev.cn/Article/details/351933.sHtML
tv.blog.vgdrev.cn/Article/details/020064.sHtML
tv.blog.vgdrev.cn/Article/details/424046.sHtML
tv.blog.vgdrev.cn/Article/details/086406.sHtML
tv.blog.vgdrev.cn/Article/details/911593.sHtML
tv.blog.vgdrev.cn/Article/details/159335.sHtML
tv.blog.vgdrev.cn/Article/details/462426.sHtML
tv.blog.vgdrev.cn/Article/details/517557.sHtML
tv.blog.vgdrev.cn/Article/details/240044.sHtML
tv.blog.vgdrev.cn/Article/details/519999.sHtML
tv.blog.vgdrev.cn/Article/details/339953.sHtML
tv.blog.vgdrev.cn/Article/details/331575.sHtML
tv.blog.vgdrev.cn/Article/details/066402.sHtML
tv.blog.vgdrev.cn/Article/details/862622.sHtML
tv.blog.vgdrev.cn/Article/details/591937.sHtML
tv.blog.vgdrev.cn/Article/details/424882.sHtML
tv.blog.vgdrev.cn/Article/details/660606.sHtML
tv.blog.vgdrev.cn/Article/details/464666.sHtML
tv.blog.vgdrev.cn/Article/details/313797.sHtML
tv.blog.vgdrev.cn/Article/details/335375.sHtML
tv.blog.vgdrev.cn/Article/details/268260.sHtML
tv.blog.vgdrev.cn/Article/details/440620.sHtML
tv.blog.vgdrev.cn/Article/details/226640.sHtML
tv.blog.vgdrev.cn/Article/details/686842.sHtML
tv.blog.vgdrev.cn/Article/details/660444.sHtML
tv.blog.vgdrev.cn/Article/details/406402.sHtML
tv.blog.vgdrev.cn/Article/details/028880.sHtML
tv.blog.vgdrev.cn/Article/details/040426.sHtML
tv.blog.vgdrev.cn/Article/details/082882.sHtML
tv.blog.vgdrev.cn/Article/details/115951.sHtML
tv.blog.vgdrev.cn/Article/details/086666.sHtML
tv.blog.vgdrev.cn/Article/details/028602.sHtML
tv.blog.vgdrev.cn/Article/details/240004.sHtML
tv.blog.vgdrev.cn/Article/details/284826.sHtML
tv.blog.vgdrev.cn/Article/details/466488.sHtML
tv.blog.vgdrev.cn/Article/details/371337.sHtML
tv.blog.vgdrev.cn/Article/details/371573.sHtML
tv.blog.vgdrev.cn/Article/details/575397.sHtML
tv.blog.vgdrev.cn/Article/details/484824.sHtML
tv.blog.vgdrev.cn/Article/details/066024.sHtML
tv.blog.vgdrev.cn/Article/details/515715.sHtML
tv.blog.vgdrev.cn/Article/details/931179.sHtML
tv.blog.vgdrev.cn/Article/details/066460.sHtML
tv.blog.vgdrev.cn/Article/details/028046.sHtML
tv.blog.vgdrev.cn/Article/details/860268.sHtML
tv.blog.vgdrev.cn/Article/details/446246.sHtML
tv.blog.vgdrev.cn/Article/details/028280.sHtML
tv.blog.vgdrev.cn/Article/details/624460.sHtML
tv.blog.vgdrev.cn/Article/details/084842.sHtML
tv.blog.vgdrev.cn/Article/details/935713.sHtML
tv.blog.vgdrev.cn/Article/details/040846.sHtML
tv.blog.vgdrev.cn/Article/details/260846.sHtML
tv.blog.vgdrev.cn/Article/details/404022.sHtML
tv.blog.vgdrev.cn/Article/details/866266.sHtML
tv.blog.vgdrev.cn/Article/details/280604.sHtML
tv.blog.vgdrev.cn/Article/details/240868.sHtML
tv.blog.vgdrev.cn/Article/details/648446.sHtML
tv.blog.vgdrev.cn/Article/details/919117.sHtML
tv.blog.vgdrev.cn/Article/details/422026.sHtML
tv.blog.vgdrev.cn/Article/details/159757.sHtML
tv.blog.vgdrev.cn/Article/details/820880.sHtML
tv.blog.vgdrev.cn/Article/details/593177.sHtML
tv.blog.vgdrev.cn/Article/details/644248.sHtML
tv.blog.vgdrev.cn/Article/details/351375.sHtML
tv.blog.vgdrev.cn/Article/details/406260.sHtML
tv.blog.vgdrev.cn/Article/details/846664.sHtML
tv.blog.vgdrev.cn/Article/details/604480.sHtML
tv.blog.vgdrev.cn/Article/details/397757.sHtML
tv.blog.vgdrev.cn/Article/details/777751.sHtML
tv.blog.vgdrev.cn/Article/details/557373.sHtML
tv.blog.vgdrev.cn/Article/details/644442.sHtML
tv.blog.vgdrev.cn/Article/details/286088.sHtML
tv.blog.vgdrev.cn/Article/details/842620.sHtML
tv.blog.vgdrev.cn/Article/details/062420.sHtML
tv.blog.vgdrev.cn/Article/details/640664.sHtML
tv.blog.vgdrev.cn/Article/details/880802.sHtML
tv.blog.vgdrev.cn/Article/details/286668.sHtML
tv.blog.vgdrev.cn/Article/details/137519.sHtML
tv.blog.vgdrev.cn/Article/details/226206.sHtML
tv.blog.vgdrev.cn/Article/details/628602.sHtML
tv.blog.vgdrev.cn/Article/details/179397.sHtML
tv.blog.vgdrev.cn/Article/details/684688.sHtML
tv.blog.vgdrev.cn/Article/details/424822.sHtML
tv.blog.vgdrev.cn/Article/details/797191.sHtML
tv.blog.vgdrev.cn/Article/details/448840.sHtML
tv.blog.vgdrev.cn/Article/details/913773.sHtML
tv.blog.vgdrev.cn/Article/details/480040.sHtML
tv.blog.vgdrev.cn/Article/details/408406.sHtML
tv.blog.vgdrev.cn/Article/details/395595.sHtML
tv.blog.vgdrev.cn/Article/details/802488.sHtML
tv.blog.vgdrev.cn/Article/details/159511.sHtML
tv.blog.vgdrev.cn/Article/details/006642.sHtML
tv.blog.vgdrev.cn/Article/details/046000.sHtML
tv.blog.vgdrev.cn/Article/details/860406.sHtML
tv.blog.vgdrev.cn/Article/details/668666.sHtML
tv.blog.vgdrev.cn/Article/details/684642.sHtML
tv.blog.vgdrev.cn/Article/details/420626.sHtML
tv.blog.vgdrev.cn/Article/details/151751.sHtML
tv.blog.vgdrev.cn/Article/details/551999.sHtML
tv.blog.vgdrev.cn/Article/details/044608.sHtML
tv.blog.vgdrev.cn/Article/details/755359.sHtML
tv.blog.vgdrev.cn/Article/details/664602.sHtML
tv.blog.vgdrev.cn/Article/details/284444.sHtML
tv.blog.vgdrev.cn/Article/details/624860.sHtML
tv.blog.vgdrev.cn/Article/details/228422.sHtML
tv.blog.vgdrev.cn/Article/details/624448.sHtML
tv.blog.vgdrev.cn/Article/details/488020.sHtML
tv.blog.vgdrev.cn/Article/details/951993.sHtML
tv.blog.vgdrev.cn/Article/details/242262.sHtML
tv.blog.vgdrev.cn/Article/details/739331.sHtML
tv.blog.vgdrev.cn/Article/details/084486.sHtML
tv.blog.vgdrev.cn/Article/details/844460.sHtML
tv.blog.vgdrev.cn/Article/details/044486.sHtML
tv.blog.vgdrev.cn/Article/details/935359.sHtML
tv.blog.vgdrev.cn/Article/details/628684.sHtML
tv.blog.vgdrev.cn/Article/details/642626.sHtML
tv.blog.vgdrev.cn/Article/details/682242.sHtML
tv.blog.vgdrev.cn/Article/details/660224.sHtML
tv.blog.vgdrev.cn/Article/details/359791.sHtML
tv.blog.vgdrev.cn/Article/details/913771.sHtML
tv.blog.vgdrev.cn/Article/details/240246.sHtML
tv.blog.vgdrev.cn/Article/details/462866.sHtML
tv.blog.vgdrev.cn/Article/details/822088.sHtML
tv.blog.vgdrev.cn/Article/details/402486.sHtML
tv.blog.vgdrev.cn/Article/details/264446.sHtML
tv.blog.vgdrev.cn/Article/details/666848.sHtML
tv.blog.vgdrev.cn/Article/details/866040.sHtML
tv.blog.vgdrev.cn/Article/details/046060.sHtML
tv.blog.vgdrev.cn/Article/details/840408.sHtML
tv.blog.vgdrev.cn/Article/details/935935.sHtML
tv.blog.vgdrev.cn/Article/details/406860.sHtML
tv.blog.vgdrev.cn/Article/details/624664.sHtML
tv.blog.vgdrev.cn/Article/details/915955.sHtML
tv.blog.vgdrev.cn/Article/details/004268.sHtML
tv.blog.vgdrev.cn/Article/details/266402.sHtML
tv.blog.vgdrev.cn/Article/details/448440.sHtML
tv.blog.vgdrev.cn/Article/details/577531.sHtML
tv.blog.vgdrev.cn/Article/details/006868.sHtML
tv.blog.vgdrev.cn/Article/details/822242.sHtML
tv.blog.vgdrev.cn/Article/details/804646.sHtML
tv.blog.vgdrev.cn/Article/details/068244.sHtML
tv.blog.vgdrev.cn/Article/details/797913.sHtML
tv.blog.vgdrev.cn/Article/details/020028.sHtML
tv.blog.vgdrev.cn/Article/details/022022.sHtML
tv.blog.vgdrev.cn/Article/details/888840.sHtML
tv.blog.vgdrev.cn/Article/details/886482.sHtML
tv.blog.vgdrev.cn/Article/details/951591.sHtML
tv.blog.vgdrev.cn/Article/details/600842.sHtML
tv.blog.vgdrev.cn/Article/details/882680.sHtML
tv.blog.vgdrev.cn/Article/details/444824.sHtML
tv.blog.vgdrev.cn/Article/details/000466.sHtML
tv.blog.vgdrev.cn/Article/details/024826.sHtML
tv.blog.vgdrev.cn/Article/details/006684.sHtML
tv.blog.vgdrev.cn/Article/details/400864.sHtML
tv.blog.vgdrev.cn/Article/details/680066.sHtML
tv.blog.vgdrev.cn/Article/details/913197.sHtML
tv.blog.vgdrev.cn/Article/details/779579.sHtML
tv.blog.vgdrev.cn/Article/details/911719.sHtML
tv.blog.vgdrev.cn/Article/details/739939.sHtML
tv.blog.vgdrev.cn/Article/details/400820.sHtML
tv.blog.vgdrev.cn/Article/details/280006.sHtML
tv.blog.vgdrev.cn/Article/details/264848.sHtML
tv.blog.vgdrev.cn/Article/details/886004.sHtML
tv.blog.vgdrev.cn/Article/details/280646.sHtML
tv.blog.vgdrev.cn/Article/details/333351.sHtML
tv.blog.vgdrev.cn/Article/details/628464.sHtML
tv.blog.vgdrev.cn/Article/details/393755.sHtML
tv.blog.vgdrev.cn/Article/details/448628.sHtML
tv.blog.vgdrev.cn/Article/details/953731.sHtML
tv.blog.vgdrev.cn/Article/details/442404.sHtML
tv.blog.vgdrev.cn/Article/details/595535.sHtML
tv.blog.vgdrev.cn/Article/details/288466.sHtML
tv.blog.vgdrev.cn/Article/details/488822.sHtML
tv.blog.vgdrev.cn/Article/details/933539.sHtML
tv.blog.vgdrev.cn/Article/details/848864.sHtML
tv.blog.vgdrev.cn/Article/details/139755.sHtML
tv.blog.vgdrev.cn/Article/details/519351.sHtML
tv.blog.vgdrev.cn/Article/details/733535.sHtML
tv.blog.vgdrev.cn/Article/details/462048.sHtML
tv.blog.vgdrev.cn/Article/details/008024.sHtML
tv.blog.vgdrev.cn/Article/details/828466.sHtML
tv.blog.vgdrev.cn/Article/details/884464.sHtML
tv.blog.vgdrev.cn/Article/details/640008.sHtML
tv.blog.vgdrev.cn/Article/details/642688.sHtML
tv.blog.vgdrev.cn/Article/details/660424.sHtML
tv.blog.vgdrev.cn/Article/details/537591.sHtML
tv.blog.vgdrev.cn/Article/details/060666.sHtML
tv.blog.vgdrev.cn/Article/details/468848.sHtML
tv.blog.vgdrev.cn/Article/details/404682.sHtML
tv.blog.vgdrev.cn/Article/details/642424.sHtML
tv.blog.vgdrev.cn/Article/details/882824.sHtML
tv.blog.vgdrev.cn/Article/details/044466.sHtML
tv.blog.vgdrev.cn/Article/details/482400.sHtML
tv.blog.vgdrev.cn/Article/details/440884.sHtML
tv.blog.vgdrev.cn/Article/details/628604.sHtML
tv.blog.vgdrev.cn/Article/details/084664.sHtML
tv.blog.vgdrev.cn/Article/details/377339.sHtML
tv.blog.vgdrev.cn/Article/details/402286.sHtML
tv.blog.vgdrev.cn/Article/details/046682.sHtML
tv.blog.vgdrev.cn/Article/details/199973.sHtML
tv.blog.vgdrev.cn/Article/details/713333.sHtML
tv.blog.vgdrev.cn/Article/details/442682.sHtML
tv.blog.vgdrev.cn/Article/details/802668.sHtML
tv.blog.vgdrev.cn/Article/details/351731.sHtML
tv.blog.vgdrev.cn/Article/details/864088.sHtML
tv.blog.vgdrev.cn/Article/details/735599.sHtML
tv.blog.vgdrev.cn/Article/details/599997.sHtML
tv.blog.vgdrev.cn/Article/details/008228.sHtML
tv.blog.vgdrev.cn/Article/details/062424.sHtML
tv.blog.vgdrev.cn/Article/details/000480.sHtML
tv.blog.vgdrev.cn/Article/details/680682.sHtML
tv.blog.vgdrev.cn/Article/details/775531.sHtML
tv.blog.vgdrev.cn/Article/details/157759.sHtML
tv.blog.vgdrev.cn/Article/details/519339.sHtML
tv.blog.vgdrev.cn/Article/details/137737.sHtML
tv.blog.vgdrev.cn/Article/details/600620.sHtML
tv.blog.vgdrev.cn/Article/details/444026.sHtML
tv.blog.vgdrev.cn/Article/details/535151.sHtML
tv.blog.vgdrev.cn/Article/details/048860.sHtML
tv.blog.vgdrev.cn/Article/details/266822.sHtML
tv.blog.vgdrev.cn/Article/details/428248.sHtML
tv.blog.vgdrev.cn/Article/details/026220.sHtML
tv.blog.vgdrev.cn/Article/details/602660.sHtML
tv.blog.vgdrev.cn/Article/details/373713.sHtML
tv.blog.vgdrev.cn/Article/details/115957.sHtML
tv.blog.vgdrev.cn/Article/details/557959.sHtML
tv.blog.vgdrev.cn/Article/details/193999.sHtML
tv.blog.vgdrev.cn/Article/details/082244.sHtML
tv.blog.vgdrev.cn/Article/details/462260.sHtML
tv.blog.vgdrev.cn/Article/details/826624.sHtML
tv.blog.vgdrev.cn/Article/details/797319.sHtML
tv.blog.vgdrev.cn/Article/details/040608.sHtML
tv.blog.vgdrev.cn/Article/details/977115.sHtML
tv.blog.vgdrev.cn/Article/details/284224.sHtML
tv.blog.vgdrev.cn/Article/details/199195.sHtML
tv.blog.vgdrev.cn/Article/details/115915.sHtML
tv.blog.vgdrev.cn/Article/details/791353.sHtML
tv.blog.vgdrev.cn/Article/details/846484.sHtML
tv.blog.vgdrev.cn/Article/details/022062.sHtML
tv.blog.vgdrev.cn/Article/details/199739.sHtML
tv.blog.vgdrev.cn/Article/details/046244.sHtML
tv.blog.vgdrev.cn/Article/details/848624.sHtML
tv.blog.vgdrev.cn/Article/details/797739.sHtML
tv.blog.vgdrev.cn/Article/details/802060.sHtML
tv.blog.vgdrev.cn/Article/details/131939.sHtML
tv.blog.vgdrev.cn/Article/details/244620.sHtML
tv.blog.vgdrev.cn/Article/details/824400.sHtML
tv.blog.vgdrev.cn/Article/details/626606.sHtML
tv.blog.vgdrev.cn/Article/details/822442.sHtML
tv.blog.vgdrev.cn/Article/details/008660.sHtML
tv.blog.vgdrev.cn/Article/details/462428.sHtML
tv.blog.vgdrev.cn/Article/details/662068.sHtML
tv.blog.vgdrev.cn/Article/details/602262.sHtML
tv.blog.vgdrev.cn/Article/details/977739.sHtML
tv.blog.vgdrev.cn/Article/details/686442.sHtML
tv.blog.vgdrev.cn/Article/details/195773.sHtML
tv.blog.vgdrev.cn/Article/details/488020.sHtML
tv.blog.vgdrev.cn/Article/details/642042.sHtML
tv.blog.vgdrev.cn/Article/details/337115.sHtML
tv.blog.vgdrev.cn/Article/details/420080.sHtML
tv.blog.vgdrev.cn/Article/details/002642.sHtML
tv.blog.vgdrev.cn/Article/details/062840.sHtML
tv.blog.vgdrev.cn/Article/details/260840.sHtML
tv.blog.vgdrev.cn/Article/details/446488.sHtML
tv.blog.vgdrev.cn/Article/details/224666.sHtML
tv.blog.vgdrev.cn/Article/details/866022.sHtML
tv.blog.vgdrev.cn/Article/details/715593.sHtML
tv.blog.vgdrev.cn/Article/details/288668.sHtML
tv.blog.vgdrev.cn/Article/details/808042.sHtML
tv.blog.vgdrev.cn/Article/details/428600.sHtML
tv.blog.vgdrev.cn/Article/details/822608.sHtML
tv.blog.vgdrev.cn/Article/details/622486.sHtML
tv.blog.vgdrev.cn/Article/details/559113.sHtML
tv.blog.vgdrev.cn/Article/details/351993.sHtML
tv.blog.vgdrev.cn/Article/details/042002.sHtML
tv.blog.vgdrev.cn/Article/details/799371.sHtML
tv.blog.vgdrev.cn/Article/details/393919.sHtML
tv.blog.vgdrev.cn/Article/details/002866.sHtML
tv.blog.vgdrev.cn/Article/details/462804.sHtML
tv.blog.vgdrev.cn/Article/details/860088.sHtML
tv.blog.vgdrev.cn/Article/details/371539.sHtML
tv.blog.vgdrev.cn/Article/details/440442.sHtML
tv.blog.vgdrev.cn/Article/details/608062.sHtML
tv.blog.vgdrev.cn/Article/details/802208.sHtML
tv.blog.vgdrev.cn/Article/details/755797.sHtML
tv.blog.vgdrev.cn/Article/details/422044.sHtML
tv.blog.vgdrev.cn/Article/details/331517.sHtML
tv.blog.vgdrev.cn/Article/details/042868.sHtML
