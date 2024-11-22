# Java Unmarshaller安全性

如果您是因为 Log4Shell/CVE-2021-44228 而来到这里，建议阅读有关漏洞利用向量和受影响的 Java 运行时版本的详细信息：[点击此处](https://mbechler.github.io/2021/12/10/PSA_Log4Shell_JNDI_Injection/)。

## 研究论文

两年多以前，Chris Frohoff 和 Gabriel Lawrence 发表了关于 Java 对象反序列化漏洞的研究，这一研究最终导致 Java 历史上最大的一波远程代码执行漏洞的爆发。

后续研究表明，这些漏洞不仅限于 Java 序列化或 XStream 这样的机制，还可以应用于其他机制。

本文对多种 Java 开源反序列化库进行了分析，包括一些利用细节，表明无论采用何种方式来实现反序列化，只要允许攻击者提供任意类型，这个过程都会有被类似攻击技术利用的风险。

完整论文请点击查看：[marshalsec.pdf](https://www.github.com/mbechler/marshalsec/blob/master/marshalsec.pdf?raw=true)

## 免责声明

所有信息和代码仅用于教育目的和/或测试您自己的系统是否存在这些漏洞。

## 使用方法

需要 Java 8 环境，使用 maven 进行构建：```mvn clean package -DskipTests```。运行命令如下：

```
mvn package -Dmaven.test.skip=true -fn
```

```shell
java -cp target/marshalsec-[版本号]-SNAPSHOT-all.jar marshalsec.<Marshaller> [-a] [-v] [-t] [<gadget_type> [<参数...>]]
```

其中：

* **-a** - 生成/测试该 marshaller 的所有 payloads。
* **-t** - 以测试模式运行，生成 payload 后对其进行反序列化。
* **-v** - 详细模式，例如在测试模式下显示生成的 payload。
* **gadget_type** - 指定的 gadget 类型标识符，如果省略则会显示该 marshaller 可用的 gadget 类型。
* **arguments** - gadget 特定的参数。

以下是支持的 marshaller 及其可能带来的影响：

| Marshaller                      | Gadget 影响
| ------------------------------- | ----------------------------------------------
| BlazeDSAMF(0&#124;3&#124;X)     | JDK 远程代码执行；第三方库远程代码执行
| Hessian&#124;Burlap             | 第三方库远程代码执行
| Castor                          | 依赖库远程代码执行
| Jackson                         | **可能的 JDK 远程代码执行**，第三方库远程代码执行
| Java                            | 第三方库远程代码执行
| JsonIO                          | **仅 JDK 远程代码执行**
| JYAML                           | **仅 JDK 远程代码执行**
| Kryo                            | 第三方库远程代码执行
| KryoAltStrategy                 | **仅 JDK 远程代码执行**
| Red5AMF(0&#124;3)               | **仅 JDK 远程代码执行**
| SnakeYAML                       | **仅 JDK 远程代码执行**
| XStream                         | **仅 JDK 远程代码执行**
| YAMLBeans                       | 第三方库远程代码执行

## 参数和其他前提条件

### 系统命令执行

* **cmd** - 要执行的命令
* **args...** - 作为参数传递的额外内容

无特殊前提条件。

### 远程类加载（普通方式）

* **codebase** - 远程代码库的 URL
* **class** - 要加载的类

**前提条件**：

* 设置一个提供 Java classpath 的 Web 服务器。
* 编译后的类文件必须按 Java classpath 规范提供。

### 远程类加载（ServiceLoader）

* **service_codebase** - 远程代码库的 URL

服务加载器当前硬编码为 *javax.script.ScriptEngineFactory*。

**前提条件**：

* 与普通远程类加载相同。
* 还需要在 *<codebase>*/META-INF/javax.script.ScriptEngineFactory 路径下提供一个包含目标类名的配置文件。
* 目标类必须实现接口 *javax.script.ScriptEngineFactory*。

### JNDI 引用重定向

* **jndiUrl** - 需要触发查找的 JNDI URL

**前提条件**：

* 设置远程代码库，和远程类加载相同。
* 运行一个指向该代码库的 JNDI 引用重定向服务 - 两种实现包括：*marshalsec.jndi.LDAPRefServer* 和 *RMIRefServer*。
  ```shell
  java -cp target/marshalsec-[版本号]-SNAPSHOT-all.jar marshalsec.jndi.(LDAP|RMI)RefServer <代码库>#<类> [<端口>]
  ```
* 使用 (ldap|rmi)://*主机*:*端口*/obj 作为 *jndiUrl*，指向服务的监听地址。

## 运行测试

运行测试时可以通过一些系统属性来控制参数（通过 maven 或使用 **-a** 时）：

* **exploit.codebase**，默认为 *http://localhost:8080/*
* **exploit.codebaseClass**，默认为 *Exploit*
* **exploit.jndiUrl**，默认为 *ldap://localhost:1389/obj*
* **exploit.exec**，默认为 */usr/bin/gedit*

测试时会安装一个 SecurityManager，用于检查系统命令执行以及远程代码库执行的代码。要使其工作，加载的类必须触发某些安全管理器检查。