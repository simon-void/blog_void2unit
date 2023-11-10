+++
author = "Stephan Schröder"
title = "How to add Kotlin as a second language to a SpringBoot app using Maven multimodule"
date = 2023-10-17T14:54:00+02:00
description = "showcasing how to configure a SpringBoot app using Maven multimodule to allow Kotlin next to Java"
tags = [
    "kotlin", "maven", "spring boot"
]
+++

The official Kotlin Documentation has a section on how to
[configure Maven so that it compiles Kotlin code next to Java code](https://kotlinlang.org/docs/maven.html#compile-kotlin-and-java-sources).
But that setup assumes a configuration without Maven modules. Maybe it's just me, but it took me quite some time to
adapt that solution.

So this is what I learned:

## Assumptions

You have a Java Project using Spring Boot which is seperated into several Maven modules. The parent pom of all modules
is the pom-file in the base directory and the parent of that pom is **spring-boot-starter-parent**. Here's a [demo project](https://github.com/simon-void/java_to_kotlin_config_demo/tree/java_config)
configured just like that.

## How to modify the main pom-file

- add the following properties:

```xml
    <properties>
        ...
        <java.version>17</java.version>
        <kotlin.version>1.9.20</kotlin.version>
        <kotlin.compiler.incremental>true</kotlin.compiler.incremental>    <!-- optional: for faster builds -->
        ...
    </properties>
```
The properties `maven.compiler.source` and `maven.compiler.target` can be removed - should they be present in your configuration -
since we'll also update the configuration of the **maven-compiler-plugin**. 

- prepare the dependencies for **kotlin-reflect** and **kotlin-stdlib**:

```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.jetbrains.kotlin</groupId>
                <artifactId>kotlin-reflect</artifactId>
                <version>${kotlin.version}</version>
                <scope>runtime</scope>
            </dependency>
            <dependency>
                <groupId>org.jetbrains.kotlin</groupId>
                <artifactId>kotlin-stdlib</artifactId>
                <version>${kotlin.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

- prepare the following configuration of the **kotlin-maven-plugin** and the **maven-compiler-plugin**:

```xml
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.jetbrains.kotlin</groupId>
                    <artifactId>kotlin-maven-plugin</artifactId>
                    <version>${kotlin.version}</version>
                    <extensions>true</extensions>
                    <executions>
                        <execution>
                            <id>compile</id>
                            <!--<goals><goal>compile</goal></goals> not needed because of <extensions>true<.../>-->
                            <configuration>
                                <sourceDirs>
                                    <sourceDir>${project.basedir}/src/main/kotlin</sourceDir>
                                    <sourceDir>${project.basedir}/src/main/java</sourceDir>
                                </sourceDirs>
                            </configuration>
                        </execution>
                        <execution>
                            <id>test-compile</id>
                            <!--<goals><goal>test-compile</goal></goals> not needed because of <extensions>true<.../>-->
                        <configuration>
                            <sourceDirs>
                                <sourceDir>${project.basedir}/src/test/kotlin</sourceDir>
                                <sourceDir>${project.basedir}/src/test/java</sourceDir>
                            </sourceDirs>
                        </configuration>
                    </execution>
                </executions>
                <configuration>
                    <args>
                        <arg>-Xjsr305=strict</arg>
                    </args>
                    <compilerPlugins>
                        <plugin>spring</plugin>
                    </compilerPlugins>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.jetbrains.kotlin</groupId>
                        <artifactId>kotlin-maven-allopen</artifactId>
                        <version>${kotlin.version}</version>
                    </dependency>
                </dependencies>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <executions>
                    <!-- Replacing default-compile as it is treated specially by maven -->
                        <execution>
                            <id>default-compile</id>
                            <phase>none</phase>
                        </execution>
                        <!-- Replacing default-compile as it is treated specially by maven -->
                        <execution>
                            <id>default-testCompile</id>
                            <phase>none</phase>
                        </execution>
                        <execution>
                            <id>java-compile</id>
                            <phase>compile</phase>
                            <goals>
                                <goal>compile</goal>
                            </goals>
                        </execution>
                        <execution>
                            <id>java-test-compile</id>
                            <phase>test-compile</phase>
                            <goals>
                                <goal>testCompile</goal>
                            </goals>
                        </execution>
                    </executions>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

## How to modify all the module pom-files

first add **kotlin-maven-plugin** and the **maven-compiler-plugin** to all modules dependency list
```xml
	<dependencies>
		<dependency>
			<groupId>org.jetbrains.kotlin</groupId>
			<artifactId>kotlin-reflect</artifactId>
		</dependency>
		<dependency>
			<groupId>org.jetbrains.kotlin</groupId>
			<artifactId>kotlin-stdlib</artifactId>
		</dependency>
        ...
    </dependencies>
```
and then if you're in a module which uses the **spring-boot-maven-plugin**, use this order:
```xml
	<build>
		<plugins>
			<plugin>
				<groupId>org.jetbrains.kotlin</groupId>
				<artifactId>kotlin-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
			</plugin>
            ...
		</plugins>
	</build>
```
if you're in a module without that plugin the order of the other two remains:
```xml
	<build>
		<plugins>
			<plugin>
				<groupId>org.jetbrains.kotlin</groupId>
				<artifactId>kotlin-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
			</plugin>
            ...
		</plugins>
	</build>
```
The order of these plugins is important. Otherwise, your Kotlin code might see your Java code but not the other way around.

Now you can add a `src/main/kotlin`-Path in your project and convert some of your Java classes to Kotlin.

And that's already all there is to do to change your Java project to a Java and Kotlin project. You can check the [Java
and Kotlin config branch](https://github.com/simon-void/java_to_kotlin_config_demo/tree/java_and_kotlin_config) of my demo project.

## Bonus: pure Kotlin configuration

So once all the source files have all been converted to Kotlin, can we simplify the Java&Kotlin configuration? Yes we can:

- remove the **maven-compiler-plugin**-config from all the pom-files
- remove the `<executions>`-block from the **kotlin-maven-plugin** in the main pom-file
- configure `sourceDirectory` and `testSourceDirectory` at the start of the `build`-block in the main pom-file
```xml
    <build>
        <sourceDirectory>${project.basedir}/src/main/kotlin</sourceDirectory>
        <testSourceDirectory>${project.basedir}/src/test/kotlin</testSourceDirectory>
        <pluginManagement>
            ...
        </pluginManagement>
    </build>
```
- keep the `<java.version>` in the properties block, the **spring-boot-maven-plugin** will still use it to compile the Kotlin code
to the configured JVM bytecode version.

Here's the [Kotlin config branch](https://github.com/simon-void/java_to_kotlin_config_demo/tree/kotlin_config) of my demo project to double-check.

## What about JDK21 support

When you check my demo project, you'll see that I've used `<java.version>17</java.version>` for all the different branches/configurations.
So assuming a JDK21 is installed on your system, can you simply set that property to `21` so that your Kotlin code can use Java 21 APIs
and your code gets compiled to JVM 21 bytecode?

### pure Java config

Yes

### mixed Java and Kotlin config

Yes, as long as your `kotlin.version` is at least `1.9.20`,
since Kotlin `1.9.20-Beta1` was the first version to support a jvmTarget of `21`.

### pure Kotlin config

No, or more precisely, not before Kotlin 2.0 which will be released in early 2024.

#### Background

Apparently a `CommandLine` class, that Kotlin uses, was moved/removed in Java 21, so Kotlin currently doesn't find it.
The error, which is thrown, does look like this:
```text
Caused by: java.lang.IllegalAccessError: superclass access check failed: class org.jetbrains.kotlin.kapt3.base.javac.KaptJavaCompiler (in unnamed module @0x17de6b) cannot access class com.sun.tools.javac.main.JavaCompiler (in module jdk.compiler) because module jdk.compiler does not export com.sun.tools.javac.main to unnamed module @0x17de6b
```
The [bug](https://youtrack.jetbrains.com/issue/KT-60507/Kapt-IllegalAccessError-superclass-access-check-failed-using-java-21-toolchain)
has already been fixed, but only in Kotlin version `2.0.0-Beta1` and onwards.

Your project might not trigger this bug, if it doesn't use Spring or Maven. E.g. I've got a
[demo project showcasing Kotlin using Java 21 APIs](https://github.com/simon-void/vthreads_with_kotlin_demo/blob/main/build.gradle.kts)
using Gradle and not using Spring that works without a problem. So give it a try.

Maybe there's a pure Kotlin config that does work even with SpringBoot, but I didn't find it. The workaround I see, if you want to use Java 21 and Spring Boot from Kotlin right now, is to us the Java & Kotlin config, even though you don't have any Java source files in your project.

But luckily, Kotlin 2.0 isn't far off :)