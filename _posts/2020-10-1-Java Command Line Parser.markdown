---
layout: post
title:  "彩虹海盗's Blog Java命令行解析"
category: Java
---
# 使用Java反射机制解析命令行 
最近在使用Java，需要解析命令行，原来用了一种很蠢的方式：
```Java
 class main{  
   public static void main(String []args){  
       if(args.length == 1 && args[0].equal("-X")){  
       //do some things  
       }        
  //.....       
   }  
 }  
 ```
后来为了扩展（原来的做法几乎不可维护）以及
~~懒~~
更加方便，所以使用另一种方法：即反射。  
大体思路是通过反射来更改成员变量。
```Java
/**
     * 通用解析反射赋值函数
     * -DName=Vulan 给Name赋值Vulan
     * @param Cls 类类型
     * @param obj 实例
     * @param Param 字符串
     * @throws Exception 解析异常
     */
    public static void ReflectionAssignment(Class<?> Cls,Object obj,String Param) throws Exception{
        if(Param.startsWith("-D")){
            var s = Param.substring(2);

            var VulanName = s.substring(0,s.indexOf("="));
            var Vulan = s.substring(s.indexOf("=")+1);

            Field f = Cls.getField(VulanName);
            f.set(obj, Tools.GetFieldVulan(Vulan,f));
        }
        else throw new Exception("Param Not Found -D");
    }
```
这样即可通过ReflectionAssignment(this.class,this,你的命令行参数);来修改类内成员变量。 
同时我还编写了一个工具函数把String转化为Field所属类型
```Java
/**
     * 将String转换为field所属类型
     *
     * @param Source 要转换的String
     * @param field  field
     * @return 转换后的类型
     * @throws Exception 未知类型
     */
    public static Object GetFieldVulan(String Source, Field field) throws Exception {
        String TypeStr = field.getGenericType().toString();

        //前排提示，使用了新的Java语法(我的JDk版本为15)
        //可能不兼容老版本Jdk
        return switch (TypeStr) {
            case "class java.lang.String" -> Source;
            case "boolean" -> Boolean.parseBoolean(Source);
            case "int" -> Integer.getInteger(Source);
            case "double" -> Double.parseDouble(Source);
            case "float" -> Float.parseFloat(Source);
            case "short" -> Short.parseShort(Source);
            case "long" -> Long.parseLong(Source);
            case "byte" -> Byte.parseByte(Source);
            default -> throw new Exception("unknow type:" + TypeStr);
        };
    }
```
下面展示一个实例:
```Java
public class TestMain{
    public static int TestInt = 0;
    public static double TestDouble = 0.0;
    public static String TestStr = "NULL";

    public static void main(String []args){
        for(var s:args){
            ReflectionAssignment(this.class,this,s);
        }
        System.out.println("Int:" + TestInt);
        System.out.println("Double:" + TestDouble);
        System.out.println("String:" + TestStr);
    }
}
```
使用这个方法即可很方便地解析命令行参数，但是也有一些局限性，比如无法解析数组之类的（可以通过增强解析器实现解析数组）。
不过目前目的已经达到了。  
完整代码:
```Java

import java.lang.reflect.Field;


public class Test{
    public  int TestInt = 0;
    public  double TestDouble = 0.0;
    public  String TestStr = "NULL";

    public static void main(String []args){
        try{
        Test t = new Test();
        for(var s:args){
            ReflectionAssignment(Test.class,t,s);
        }
        System.out.println("Int:" + t.TestInt);
        System.out.println("Double:" + t.TestDouble);
        System.out.println("String:" + t.TestStr);
    }
    catch(Exception e){
        e.printStackTrace();
    }
    }

    /**
     * 将String转换为field所属类型
     *
     * @param Source 要转换的String
     * @param field  field
     * @return 转换后的类型
     * @throws Exception 未知类型
     */
    public static Object GetFieldVulan(String Source, Field field) throws Exception {
        String TypeStr = field.getGenericType().toString();

        //前排提示，使用了新的Java语法(我的JDk版本为15)
        //可能不兼容老版本Jdk
        return switch (TypeStr) {
            case "class java.lang.String" -> Source;
            case "boolean" -> Boolean.parseBoolean(Source);
            case "int" -> Integer.valueOf(Source);
            case "double" -> Double.parseDouble(Source);
            case "float" -> Float.parseFloat(Source);
            case "short" -> Short.parseShort(Source);
            case "long" -> Long.parseLong(Source);
            case "byte" -> Byte.parseByte(Source);
            default -> throw new Exception("unknow type:" + TypeStr);
        };
    }

    /**
     * 通用解析反射赋值函数
     * -DName=Vulan 给Name赋值Vulan
     * @param Cls 类类型
     * @param obj 实例
     * @param Param 字符串
     * @throws Exception 解析异常
     */
    public static void ReflectionAssignment(Class<?> Cls,Object obj,String Param) throws Exception{
        if(Param.startsWith("-D")){
            var s = Param.substring(2);

            var VulanName = s.substring(0,s.indexOf("="));
            var Vulan = s.substring(s.indexOf("=")+1);

            Field f = Cls.getField(VulanName);
            f.set(obj, GetFieldVulan(Vulan,f));
        }
        else throw new Exception("Param Not Found -D");
    }

}
```
控制台：
```
前排提示：使用了Java15
$ javac -encoding utf-8 .\Test.java
$ java Test
Int:0
Double:0.0
String:NULL
$ java Test "-DTestStr=Hello World" "-DTestInt=10" "-DTestDouble=10.0"
Int:10
Double:10.0
String:Hello World
```