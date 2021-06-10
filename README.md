# nifi
## 如何自定义处理器
### 执行器抽象类AbstractProcessor
任何自定义 processor 都必须 直接或者间接 继承 AbstractProcessor，整个 Processor 的核心部分 是 onTrigger 方法，onTrigger方法会在一个flow file被传入处理器时调用。  
类：https://github.com/apache/nifi/blob/master/nifi-api/src/main/java/org/apache/nifi/processor/AbstractProcessor.java  
```
package org.apache.nifi.processor;

import org.apache.nifi.processor.exception.ProcessException;

public abstract class AbstractProcessor extends AbstractSessionFactoryProcessor {

    @Override
    public final void onTrigger(final ProcessContext context, final ProcessSessionFactory sessionFactory) throws ProcessException {
        final ProcessSession session = sessionFactory.createSession();
        try {
            onTrigger(context, session);
            session.commit();
        } catch (final Throwable t) {
            session.rollback(true);
            throw t;
        }
    }

    public abstract void onTrigger(final ProcessContext context, final ProcessSession session) throws ProcessException;
}
```
## 如何自定义处理器
官方wiki有说明：  
https://cwiki.apache.org/confluence/display/NIFI/Maven+Projects+for+Extensions

### 概述
apachenifi扩展打包在NARs（NiFi存档）中。NAR允许将多个组件及其依赖项打包到一个包中。然后，NAR包被提供与其他NAR包隔离的类加载器。有关NARs的更多详细信息，请参阅开发人员指南。

### 处理器项目
apachenifi处理器通常以处理器包的形式组织。处理器包通常由以下部分组成：
* 一个Maven项目，它产生一个jar处理器
* 将处理器打包到NAR中的Maven项目
* 构建处理器和NAR项目的捆绑包的Parent pom
NiFi发行版中的一个处理器包示例是NiFi-kafka包，它包含子项目nifi-kafka-processors和nifi-kafka-nar。

#### 通过Maven命令构建处理器原型项目
NiFi提供了一个Maven原型，可以方便地创建处理器包项目结构。要使用原型，请运行以下Maven命令：
```
mvn archetype:generate -DarchetypeGroupId=org.apache.nifi -DarchetypeArtifactId=nifi-processor-bundle-archetype -DarchetypeVersion=1.11.4 -DnifiVersion=1.11.4
```
archetypeVersion与正在使用的nifi处理器包原型的版本相对应，当前该版本与顶级nifi版本相同（即对于nifi 0.7.0版本，archetypeVersion将为0.7.0，依此类推）。nifiVersion是新处理器将依赖的NiFi版本  
注意：可以通过检查Maven中央存储库来找到原型的最新版本。如果从源代码构建NiFi，还可以通过快照依赖关系访问原型的最新版本。  
运行上述命令将提示您在控制台输入以下属性：
| 属性 | 描述 |
| :-----| :----- |
| artifactBaseName | artifactId中的基础名称。例如，要创建helloworld包，artifactBaseName需要是helloworld，artifactId将是nifi-helloworld-bundle包。 |
| artifactId | 新包的Maven artifactId。通常类似于“nifi-basename-bundle”，basename是特定于你的bundle的 |
| groupId | 新包的Maven groupId |
| package | 处理器的Java package |
| version | maven版本号 |

按照命令和参数提示创建好Maven项目结构：
```
nifi-basename-bundle
    ├── nifi-basename-nar
    │   └── pom.xml
    ├── nifi-basename-processors
    │   ├── pom.xml
    │   └── src
    │       ├── main
    │       │   ├── java
    │       │   │   └── org.apache.nifi.processors.basename
    │       │   │       └── MyProcessor.java
    │       │   └── resources
    │       │       ├── META-INF
    │       │       │   └── services
    │       │               └── org.apache.nifi.processor.Processor
    │       └── test
    │           └── java
    │               └── org.apache.nifi.processors.basename
    │                   └── MyProcessorTest.java
    └── pom.xml
```
原型生成的默认处理器称为MyProcessor。如果您决定重命名这个处理器，请确保也更新org.apache.nifi.processor.Processor文件。NiFi使用这个文件来定位可用的处理器，这里必须列出处理器的完全限定类名，以便在用户界面中可以使用它.
#### 开始编码
##### 如何增加可配置参数
```
/**
     * 处理成功
     */
    public static final Relationship REL_SUCCESS = new Relationship.Builder()
            .name("success")
            .description("Success, all done")
            .build();

    /**
     * 处理失败
     */
    public static final Relationship REL_FAIL = new Relationship.Builder()
            .name("failure")
            .description("处理失败")
            .build();

    /**
     * 原始数据
     */
    public static final Relationship REL_ORIGINAL = new Relationship.Builder()
            .name("original")
            .description("原始数据")
            .build();

```
##### 如何增加输出结果路由关系
```
    public static final PropertyDescriptor DATA_RELATIONSHIP = new PropertyDescriptor
            .Builder().name("dataRelationship")
            .displayName("数据关系")
            .description("格式：数据1：B,C,D")
            .required(true)
            .addValidator(StandardValidators.NON_EMPTY_VALIDATOR)
            .build();
```
##### 如何加载可配置参数和路由关系
```
    protected void init(final ProcessorInitializationContext context) {
        final List<PropertyDescriptor> descriptors = new ArrayList<>();
        descriptors.add(DATA_RELATIONSHIP);
        this.descriptors = Collections.unmodifiableList(descriptors);
        final Set<Relationship> relationships = new HashSet<>();
        relationships.add(REL_SUCCESS);
        relationships.add(REL_FAIL);
        relationships.add(REL_ORIGINAL);
        this.relationships = Collections.unmodifiableSet(relationships);
    }
```
##### 如何处理数据
重构方法onTrigger()
```
    public void onTrigger(final ProcessContext context, final ProcessSession session) throws ProcessException {
        FlowFile flowFile = session.get();
        if (flowFile == null) {
            return;
        }
        flowFile = session.write(flowFile, new StreamCallback() {
            @Override
            public void process(InputStream inputStream, OutputStream outputStream) throws IOException {
                String infoStr = IOUtils.toString(inputStream);
                IOUtils.write(infoStr.toUpperCase(), outputStream);
            }
        });
          session.transfer(flowFile,REL_SUCCESS);
        // TODO implement
    }

```
* ProcessContext context:上线文，可以获取上下文数据
* ProcessSession session：会话，读取和写FlowFile对象
1. session.get()：获取FlowFile对象
2. session.transfer()：将FlowFile写到指定路由关系中

##### 如何打包
修改processors项目的packing为nar，原本是jar
注意maven的setting文件中要添加库（nifi依赖了groovy，普通maven库下载不了）
```
     			</repository> 
				  <repository>  
		          <id>nexus2</id>  
		          <name>nexus2</name>  
		          <url>https://dl.bintray.com/groovy/maven/</url> 
     			</repository> 
```
添加打包工具
```
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.nifi</groupId>
      <artifactId>nifi-nar-maven-plugin</artifactId>
      <version>1.0.0-incubating</version>
      <extensions>true</extensions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>2.15</version>
    </plugin>
  </plugins>
</build>
```
