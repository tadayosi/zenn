# Operating Java Applications Verbally with LLMs via Jolokia MCP Server

https://www.youtube.com/watch?v=Ed0UdlF_erg

Java has the standard API for application management, [JMX (Java Management Extensions)](https://docs.oracle.com/en/java/javase/21/docs/api/java.management/javax/management/package-summary.html). Most Java application servers and frameworks provide the JMX-based management APIs, which have been used by system management tools for various monitoring purposes.

By leveraging JMX, any Java application can be operated with complex commands from an LLM via MCP, without introducing yet another mechanism or framework into Java.

To operate Java applications with MCP, we can use Jolokia, a JMX-over-HTTP technology.

https://github.com/jolokia/jolokia-mcp-server

## What is Jolokia?

[Jolokia](https://jolokia.org/) is a library that provides HTTP access to JMX. Jolokia offers [agents](https://jolokia.org/reference/html/manual/agents.html) and [clients](https://jolokia.org/reference/html/manual/clients.html). Essentially, attaching a Jolokia agent to a Java application enables JMX over HTTP.

There are two main types of agents: JVM agents and WAR agents. JVM agents are attached to the JVM at startup, while WAR agents deploy a Jolokia endpoint to a web application.

![Jolokia Architecture](https://jolokia.org/reference/html/manual/_images/architecture.png)
*Jolokia Architecture [^1]*

[^1]: Quoted from <https://jolokia.org/reference/html/manual/architecture.html#agent-mode>.

## Jolokia MCP Server

[Jolokia MCP Server](https://github.com/jolokia/jolokia-mcp-server) transforms a Jolokia-attached JVM into an MCP server.

First, let's watch the video at the beginning of the post to see what you can do by connecting a Java app with Jolokia MCP Server. In the video, the instruction is:

> check the memory state of the java app

This demo just runs a simple operation only using `java.lang:type=Memory`, which is provided by Java by default. However, if application servers or frameworks provide more advanced MBeans (i.e. JMX API), the AI model can automatically check available JMX MBean attributes and operations, then devise and run the necessary operations to achieve the task for you.

This also means that if you want to operate a Java application under development in a specific way from an LLM prompt, you only need to implement that specific functionality as an MBean and register it with the MBean server.

### Features/Tools

Each instance of Jolokia MCP Server connects to a single JVM and serves the following functionalities to the MCP host:

- Get a list of available MBeans
- Get a list of an MBean's operations
- Get a list of an MBean's attributes
- Read and write an MBean's attributes
- Execute an MBean's operations

They correspond to the following 6 MCP tools:

- **listMBeans**
  - Get a list of MBeans available from the JVM
  - Output (`List<String>`): A list of all MBean object names in the JVM
- **listMBeanOperations**
  - Get a list of available operations for the specified MBean
  - Input:
    - `mbean` (`String`): MBean name
  - Output (`String`): Definition of available operations for the specified MBean (JSON format)
- **listMBeanAttributes**
  - Get a list of available attributes for the specified MBean
  - Input:
    - `mbean` (`String`): MBean name
  - Output (`String`): Definition of available attributes for the specified MBean (JSON format)
- **readMBeanAttribute**
  - Read the value of the specified MBean attribute
  - Input:
    - `mbean` (`String`): MBean name
    - `attribute` (`String`): Attribute name
  - Output (`String`): String representation of the specified attribute's value, or "null"
- **writeMBeanAttribute**
  - Set the value of the specified MBean attribute
  - Input:
    - `mbean` (`String`): MBean name
    - `attribute` (`String`): Attribute name
    - `value` (`Object`): Attribute value
  - Output (`String`): String representation of the specified attribute's previous value, or "null"
- **executeMBeanOperation**
  - Execute an operation of the specified MBean
  - Input:
    - `mbean` (`String`): MBean name
    - `operation` (`String`): Operation name
    - `args` (`Object[]`): Arguments
  - Output (`String`): String representation of the operation's return value, or "null"

## How to Use

### 1. Add Jolokia Agent to Your Java App

Here, we will use the JVM agent so that existing Java applications can be used as is. First, download `jolokia-agent-jvm-<version>-javaagent.jar`.

⬇️ [jolokia-agent-jvm-2.2.9-javaagent.jar](https://repo1.maven.org/maven2/org/jolokia/jolokia-agent-jvm/2.2.9/jolokia-agent-jvm-2.2.9-javaagent.jar)

Next, launch your Java app (`app.jar`) with the `-javaagent` option. [^2]

[^2]: For Spring Boot, ensure `spring-boot-starter-actuator` and JMX are enabled.
https://docs.spring.io/spring-boot/reference/actuator/jmx.html

```console
java -javaagent:jolokia-agent-jvm-2.2.9-javaagent.jar -jar app.jar
```

If the Jolokia agent is attached successfully, you will see a message like this at the beginning. By default, the Jolokia endpoint URL will be: http://localhost:8778/jolokia/

```console
I> No access restrictor found, access to any MBean is allowed
Jolokia: Agent started with URL http://127.0.0.1:8778/jolokia/
```

### 2. Install Jolokia MCP Server

First, download the Jolokia MCP Server JAR.

⬇️ [jolokia-mcp-server-0.3.5-runner.jar](https://github.com/jolokia/jolokia-mcp-server/releases/download/v0.3.5/jolokia-mcp-server-0.3.5-runner.jar)

Next, add it to the configuration of your MCP host (e.g., Claude, Cline, Cursor, etc.). The Jolokia MCP Server itself can be launched as follows:

```console
java -jar jolokia-mcp-server-0.3.5-runner.jar
```

So, configure your MCP host like below. Replace `<path-to-the-runner-jar>` with the appropriate absolute path.

```json
{
  "mcpServers": {
    "jolokia": {
      "command": "java",
      "args": [
        "-jar",
        "<path-to-the-runner-jar>/jolokia-mcp-server-0.3.5-runner.jar"
      ]
    }
  }
}
```

:::message
If you set the Jolokia endpoint to something other than the default (http://localhost:8778/jolokia/), specify that endpoint URL as the third argument in `args`.
:::

### 3. Launch MCP Host

Launch your MCP host and confirm that Jolokia MCP Server is properly installed.

For Claude Desktop, it's successfully installed if `jolokia` is recognised and the tools are loaded as shown below.

![Claude Desktop](/images/202505-jolokia-mcp/claude-jolokia.png)
*For Claude Desktop*

Then, try entering a prompt like:

> check memory usage

If it works, try various other prompts.

## Wrap up

We introduced Jolokia MCP Server, which allows you to operate Java applications from LLMs.

By attaching the Jolokia agent to your Java application, existing JMX MBeans can be transformed into MCP tools, enabling direct interaction and operation of Java applications through LLMs.

This opens up new possibilities for Java application management and automation using LLMs and MCP. It can be considered one future direction for LLM utilisation in enterprise Java.

Jolokia MCP Server is planned for further improvements in the future. For example, the following are under consideration:

- [Integration of Jolokia agent and MCP Server](https://github.com/jolokia/jolokia-mcp-server/issues/12)
- [Support for streaming HTTP](https://github.com/jolokia/jolokia-mcp-server/issues/13)
- [Provision of boilerplate prompts](https://github.com/jolokia/jolokia-mcp-server/issues/14)

Please give Jolokia MCP Server a try!
