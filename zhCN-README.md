# Java Unmarshaller Security - 将数据转化为代码执行

如果您来这里是为了 Log4Shell/CVE-2021-44228，您可能想阅读 漏洞利用向量和受影响的 Java 运行时版本：
<https://mbechler.github.io/2021/12/10/PSA_Log4Shell_JNDI_Injection/>

## Paper

自 Chris Frohoff 和 Garbriel Lawrence 提出他们对 Java 对象反序列化漏洞的研究以来，已经过去两年多了，最终导致了 Java 历史上最大的一波远程代码执行错误。

对该问题的研究表明，这些漏洞并非像 Java 序列化或 XStream 这样具有表现力的机制所独有，但有些漏洞也可能适用于其他机制。


本文对各种 Java 开源编组库进行了分析（包括利用细节），这些库允许对攻击者提供的任意类型进行解组，并表明无论此过程如何执行以及存在哪些隐式约束，它很容易出现类似的利用技术。


全文位于 [marshalsec.pdf](https://www.github.com/mbechler/marshalsec/blob/master/marshalsec.pdf?raw=true)

## 免责声明

所有信息和代码仅用于教育目的和/或测试您自己的系统是否存在这些漏洞。



## Usage

需要 Java 8。使用 maven进行构建  .
```mvn clean package -DskipTests```

运行为
```shell
java -cp target/marshalsec-[VERSION]-SNAPSHOT-all.jar marshalsec.<Marshaller> [-a] [-v] [-t] [<gadget_type> [<arguments...>]]
```

在哪里

* **-a** -  生成/测试该marshaller的所有有效负载
* **-t** - 在测试模式下运行，生成有效负载后对其进行unmarshalling 。
* **-v** - 详细模式，例如还显示测试模式下生成的有效负载。
* **gadget_type** - 特定小工具的标识符，如果省略，将显示该特定marshaller的可用标识符。
* **arguments** - 小工具特定参数

包括以下marshallers 的有效负载生成器：<br />

| Marshaller                  | 小工具的影响                                                |
| --------------------------- | ------------------------------------------------------------ |
| BlazeDSAMF(0&#124;3&#124;X) | JDK仅升级为Java序列化<br/>各种第三方库 RCEs |
| Hessian&#124;Burlap         | 各种第三方 RCEs                                     |
| Castor                      | 依赖库RCE                                       |
| Jackson                     | **可能仅 JDK RCE**, 各种第三方 RCEs          |
| Java                        | 又一个第三方 RCE                                  |
| JsonIO                      | **仅限 JDK 的 RCE**                                             |
| JYAML                       | **仅限 JDK 的 RCE**                                             |
| Kryo                        | 第三方 RCEs                                             |
| KryoAltStrategy             | **仅限 JDK 的 RCE**                                             |
| Red5AMF(0&#124;3)           | **仅限 JDK 的 RCE**                                             |
| SnakeYAML                   | **仅限 JDK 的 RCEs**                                            |
| XStream                     | **仅限 JDK 的 RCEs**                                            |
| YAMLBeans                   | 第三方RCE                                              |

# 参数和附加先决条件

## 系统命令执行

* **cmd** - 要执行的命令
* **args...** - 作为参数传递的附加参数

没有先决条件。

## 远程类加载（普通）

* **codebase** - 远程代码库的 URL
* **class** - 要加载的类

**先决条件**:

* 在某个路径下设置一个托管 Java 类路径的 Web 服务器。
* 要加载的编译类文件需要根据 Java 类路径约定提供服务。

## 远程类加载（ServiceLoader）

* **service_codebase** - 远程代码库的 URL

要加载的服务当前硬编码为*javax.script.ScriptEngineFactory*.




**先决条件**:

* 与普通远程类加载相同。
* 还需要位于 *<codebase>*/META-INF/javax.script.ScriptEngineFactory
  的提供程序配置文件，其中包含纯文本的目标类名。
* 指定的目标类需要实现服务接口*javax.script.ScriptEngineFactory*.

### JNDI 引用间接

* **jndiUrl** - 用于触发查找的 JNDI URL


**先决条件**:

* 设置远程代码库，与远程类加载相同
* 运行指向该代码库的 JNDI 引用重定向器服务 - 其中包括两个实现: *marshalsec.jndi.LDAPRefServer* 和*RMIRefServer*.
  ```shell
  java -cp target/marshalsec-[VERSION]-SNAPSHOT-all.jar marshalsec.jndi.(LDAP|RMI)RefServer <codebase>#<class> [<port>]
  ```
* 使用 (ldap|rmi)://*host*:*port*/obj 作为 *jndiUrl*, 指向该服务的侦听地址。

## 运行测试

有几个系统属性可以在运行测试时控制参数(通过 Maven 或使用 **-a**时)

* **exploit.codebase**, 默认为*http://localhost:8080/*
* **exploit.codebaseClass**, 默认为*Exploit*
* **exploit.jndiUrl**, 默认为*ldap://localhost:1389/obj*
* **exploit.exec**, 默认为*/usr/bin/gedit*

测试运行时安装了 SecurityManager，用于检查系统命令执行情况以及从远程代码库执行的代码情况。为此，正在使用的加载类必须触发一些安全管理器检查。
