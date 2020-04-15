---
date: 2020-04-06 16:32:40
layout: post
title: Hello, Instrumentation (Java)
subtitle: Getting started with instrumentation in Java.
description: A step-by-step tutorial that will help you get started with instrumentation in Java.
image: https://www.wns.com/Portals/0/Images/Articles/DesktopImages/600/159/Digital_ROI_HD-1.jpg
optimized_image: https://www.wns.com/Portals/0/Images/Articles/DesktopImages/600/159/Digital_ROI_HD-1.jpg
category: java
tags:
  - bytebuddy
  - IAST
author: secinst
paginate: true
---

In this article, we will set up a simple "Hello, World" type example for using instrumentation. Here's a simple JSP that reads the "name" parameter...

```
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<body>

<form>
  <div>Enter name:</div>
  <input name="name">
  <button name="submit" type="submit">Submit</button>
</form>

<h1>Hello, ${param.name}!</h1>

</body>
</html>
```

Try visiting http://localhost:8080/ticketbook/hello.jsp?name=Fred and it works just great. But unfortunately, you just introduced a cross-site scripting (XSS) vulnerability. This happens anytime you write untrusted data (the "name" parameter) to the HTTP response without escaping it. Try visiting http://localhost:8080/ticketbook/hello.jsp?name=&lt;script&gt;alert("hacked")&lt;/script&gt; and you'll see the problem.

> You could easily fix this by adding some escaping to the code. But the typical application reads hundreds or thousands of parameters. How can we take care of *all* of them at once?

# Instrumentation

Instrumentation to the rescue. Let's instrument a method that will eliminate any dangerous characters from all HTTP parameters. This is definitely overkill, but it's a good demonstration of the power and elegance of the instrumentation approach. Basically, if you can articulate a rule for where certain code should be used, instrumentation is a clean and easy way to make sure it's always there.

Let's build an SI agent. There are three basic steps.
1. Set up our POM to automatically build an agent jar
2. Create our agent code with pointcuts and advices
3. Run our test application with our new agent

# Step 1: Set up our POM to automatically build an agent jar

Here's a sample pom.xml file to put in your project directory. Running "mvn install" should do the trick. Pay attention to the manifest entries, this is what tells the Java Instrumentation API that this is an instrumentation agent jar.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://maven.apache.org/POM/4.0.0"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.contrastsecurity</groupId>
	<artifactId>validating-agent</artifactId>
	<version>0.9</version>

	<name>validating-agent</name>

	<properties>
		<java.version>1.8</java.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-lang3</artifactId>
			<version>3.0</version>
		</dependency>
		<dependency>
			<groupId>net.bytebuddy</groupId>
			<artifactId>byte-buddy</artifactId>
			<version>1.10.9</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.8.1</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-assembly-plugin</artifactId>
				<version>3.2.0</version>
				<configuration>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
					<archive>
						<manifestEntries>
							<Premain-Class>com.contrastsecurity.ValidatingAgent</Premain-Class>
							<Agent-Class>com.contrastsecurity.ValidatingAgent</Agent-Class>
							<Can-Redefine-Classes>true</Can-Redefine-Classes>
							<Can-Retransform-Classes>false</Can-Retransform-Classes>
							<Can-Set-Native-Method-Prefix>false</Can-Set-Native-Method-Prefix>
						</manifestEntries>
					</archive>
				</configuration>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

</project>
```

# Step 2: Create our agent code with pointcuts and advices*

In this code, we are using the excellent ByteBuddy library. The AgentBuilder.Default() uses the Java Instrumentation API to set up the transformations we want to perform. Here, we specify the .type() we want as anything extending "javax.servlet.ServletRequest."  And we tell ByteBuddy to transform any methods named "getParameter" in any matching classes. ByteBuddy supports a number of matchers and boolean operations that will allow you to very narrowly define your pointcuts.

We then tell ByteBuddy to add the contents of the SecurityAdvice class into any getParameter() methods that we find. We specified that this code should be inserted @Advice.OnMethodExit, or just before the method returns. And we tell ByteBuddy to replace the return value of the getParameter() method with a sanitized version where any special characters are replaced by '_'.  That's it!

```java
import static net.bytebuddy.matcher.ElementMatchers.hasGenericSuperType;
import static net.bytebuddy.matcher.ElementMatchers.named;
import java.lang.instrument.Instrumentation;
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.asm.Advice;

public class ValidatingAgent {

	public static void premain(String arg, Instrumentation inst) {
		System.out.println("ValidatingAgent installed");
		new AgentBuilder.Default()
			.type(hasGenericSuperType(named("javax.servlet.ServletRequest")))
			.transform((b, td, cl, m) -> 
          b.visit(Advice.to(SecurityAdvice.class).on(named("getParameter"))))
			.installOn(inst);
	}

	public static class SecurityAdvice {
		@Advice.OnMethodExit
		static void exit(@Advice.Return(readOnly = false) String p) {
			p = p.replaceAll("[^a-zA-Z0-9 ']", "_");
		}
	}
}
```

# Step 3: Run our test application with our new agent*

Finally, the good part. All you have to do is add the "-javaagent" flag to however you launch Java. Every server has a different place to specify JVM arguments, but the simplest way is to use the JAVA_TOOL_OPTIONS environment variable. Make sure you use the full path to the "target" directory with your new agent jar file.

export JAVA_TOOL_OPTIONS="-javaagent:/full/path/to/validating-agent-0.9-jar-with-dependencies.jar"

[[ IMAGE ]]

# Closing Thoughts

So that's a whirlwind introduction to Java SI. Please let me know if this was helpful to you in the comments below.

