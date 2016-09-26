---
---
#  Developing Applications With the Terracotta Toolkit


{toc|2:3}


## Introduction
The Terracotta Toolkit is intended for developers working on scalable applications, frameworks, and software tools. The Terracotta Toolkit provides the following features:

* **Ease-of-use** &ndash;  A stable API, fully documented classes (see the <a href="/apidocs/terracotta-toolkit/2.5/">Terracotta Toolkit Javadoc</a>), and a versioning scheme that's easy to understand.
* **Guaranteed compatibility** &ndash;  Verified by a Terracotta Compliance Kit that tests all classes to ensure backward compatibility.
* **Extensibility** &ndash;  Includes all of the tools used to create Terracotta products, such as concurrent maps, locks, counters, queues.
* **Flexibility** &ndash;  Can be used to build clustered products that communicate with multiple clusters.
* **Platform independence** &ndash;  Runs on any Java 1.6 or 1.7 JVM and requires no boot-jars, agents, or container-specific code.

The Terracotta Toolkit 2.x is available with Terracotta kits version 4.0.0 and higher.


## Installing the Terracotta Toolkit

The Terracotta Toolkit is contained in the following JAR file:

~~~
${BIGMEMORY_HOME}/apis/toolkit/lib/terracotta-toolkit-runtime-ee-<version>.jar
~~~

The Terracotta Toolkit JAR file should be on your application's classpath or in `WEB-INF/lib` if using a WAR file.

Maven users can add the Terracotta Toolkit as a dependency (shown for BigMemory Max 4.0.0):

    <dependency>
       <groupId>org.terracotta</groupId>
       <artifactId>terracotta-toolkit-runtime-ee</artifactId>
       <version>4.0.0</version>
    </dependency>

See the Terracotta kit version you plan to use for the correct API and JAR versions to specify in the dependency block.

The repository is given by the following:

    <repository>
     <id>terracotta-repository</id>
     <url>http://www.terracotta.org/download/reflector/releases</url>
     <releases>
        <enabled>true</enabled>
     </releases>
    </repository>


## Understanding Versions

The products you create with the Terracotta Toolkit depend on the API at its heart. The Toolkit's API has a version number with a major digit and a minor digit that indicate its compatibility with other versions. The major version number indicates a breaking change, while the minor version number indicates a compatible change. For example, Terracotta Toolkit API version 1.1 is compatible with version 1.0. Version 1.2 is compatible with both versions 1.1 and 1.0. Version 2.0 is not compatible with any version 1.x, but will be forward compatible with any version 2.x.
