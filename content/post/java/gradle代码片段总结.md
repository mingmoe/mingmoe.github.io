+++
title = "Gradle代码片段总结"
date = 2021-10-31T10:27:33+08:00
draft = false
archives = 2021
categories = "programming"
tags = ["gradle","java"]
+++

汇总了一些gradle常用代码片段，包括:
 - 汇集所有子项目的javadoc
 - 汇集所有子项目的source/javadoc jar

至于汇集所有依赖到同一个jar包里的，笔者依然是使用插件的方式(shadowJar)实现。

<!--more-->

将以下代码放至根项目的build.gradle

```groovy
// 声明
tasks.register("allJavadoc",Javadoc)
tasks.register("allJavaSourceJar",Jar)
tasks.register("allJavadocJar",Jar)

// 解析项目过后的设置
gradle.projectsEvaluated {
    gradle ->
        {
            // 生成所有子项目的javadoc到同一个目录
            tasks.named("allJavadoc",Javadoc) {
                description "Collect all subproject javadoc into one directory"

                it.source subprojects.collect { project -> project.sourceSets.main.allJava }

                it.classpath = files(subprojects.collect { project -> project.sourceSets.main.compileClasspath })
            }
            // 生成所有子项目的源代码的jar
            tasks.named("allJavaSourceJar",Jar){
                description "Collect all subproject source code into one jar"

                it.dependsOn(tasks.named("classes"))

                it.from subprojects.collect { project -> project.sourceSets.main.allSource }
            }
            // 生成所有子项目的javadoc的jar
            tasks.named("allJavadocJar",Jar){
                description "Collect all subproject javadoc into one jar"

                it.dependsOn(tasks.named("allJavadoc"))

                it.from "${rootProject.rootDir.toString()}/${javadocReleasePath.toString()}"
            }
        }
}
```

如果子项目要依赖这些任务，可以以此形式调用:
```groovy
tasks.named("test"){
    dependsOn rootProject.tasks.named("allJavaSourceJar")
}
```

`ref:` https://stackoverflow.com/questions/63287292/how-to-combine-multiple-javadoc-into-one-using-gradle

`ref:` https://www.heqiangfly.com/2016/03/18/development-tool-gradle-lifecycle/

`ref:` https://stackoverflow.com/questions/11474729/how-to-build-sources-jar-with-gradle/11475089
