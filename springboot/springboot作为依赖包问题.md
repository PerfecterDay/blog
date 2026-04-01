# Springboot 报错问题
{docsify-updated}


## springboot作为依赖包问题
1. 需求：在一个springboot项目中依赖了另一个springboot项目。
2. 现象：在IDEA上，两个在一个project中时，依赖项可以正常运行。移除被依赖项后，报找不到依赖的类错误。
3. 原因：springboot项目默认达成jar包，使用maven的话，默认会使用springboot的打包插件达成springboot格式的jar包，与普通的jar包格式不一样，如果作为依赖包，则依赖方会找不到依赖的类。
4. 解决方案：移除被依赖项的pom文件中plugins节点下的spring-boot-maven-plugin。

        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <fork>true</fork>
            </configuration>
        </plugin>    

1. HttpMediaTypeNotAcceptableException: Could not find acceptable representation
   返回的对象中没有写 getter/seeter 方法。