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

