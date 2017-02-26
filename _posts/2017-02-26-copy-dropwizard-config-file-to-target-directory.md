---
layout: post
title:  "Copy Dropwizard configuration file to target directory on build"
date:   2017-02-26 14:50:00 -0500
categories: java dropwizard maven
---

The Dropwizard convention is to build your whole application as one
"fat" JAR file and to provide it a YAML configuration file on startup like so:

{% highlight java %}
java -jar yourapplication-1.0.0-SNAPSHOT.jar server configuration.yml
{% endhighlight %}

When we run the package phase using Maven, our .yml file is not automatically
copied to the target directory nor is it bundled with the JAR (and rightly so). 

To have Maven copy your YAML configuration, simply add the following to your pom.xml.
I am assuming your configuration.yml file is in src/main, if not then just modify 
the value inside the "directory" tags.

{% highlight xml %}
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>3.0.2</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${basedir}/target</outputDirectory>
                        <resources>
                            <resource>
                                <directory>${basedir}/src/main</directory>
                                <includes>
                                    <include>configuration.yml</include>
                                </includes>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
{% endhighlight %}
