---
title: '[Junit][1][Junit介绍]'
date: 2020-02-09 16:39:48
tags:
    - Junit
categories:
    - Junit
---
## Junit

- [idea+Junit单元测试](https://blog.csdn.net/antony9118/article/details/51736135)

- [Junit教程](http://wiki.jikexueyuan.com/project/junit/overview.html)

### 1 概述

#### 1.1 什么是单元测试

所谓单元测试是测试应用程序的功能是否能够按需要正常运行。单元测试是一个对单一实体（类或方法）的测试。单元测试是每个软件公司提高产品质量、满足客户需求的重要环节。

单元测试分为人工测试和自动测试

|人工测试|自动测试|
|--|--|
|手动执行测试用例并不借助任何工具的测试被称为人工测试|借助工具支持并且利用自动工具执行用例被称为自动测试|
|消耗时间并单调|快速自动化运行测试用例时明显比人力资源快|
|人力资源上投资巨大|人力资源投资较少|
|可信度较低|可信度更高，自动化测试每次运行时精确地执行相同的操作|
|非程式化|程式化|

#### 1.1 什么是Junit

`JUnit`是一个基于`Java`的单元测试框架。`JUnit`在测试驱动开发的方面有很重要的发展。

JUnit 促进了“先测试后编码”的理念，强调建立测试数据的一段代码，可以先测试，然后再应用。这个方法就好比“测试一点，编码一点，测试一点，编码一点……”，增加了程序员的产量和程序的稳定性，可以减少程序员的压力和花费在排错上的时间。

#### 1.2 为什么使用Junit

这里不说过于晦涩难懂的原因，只从我们使用感受的角度来讲。
我们都知道，`main`方法是一个程序的入口，通常来说，没有`main`方法，程序就无法运行。我们经常会写一些`class`文件（如下图所示），他们并没有自己的main方法。那么我们如何检测这些class写的对不对？难道每次测试一个class都要自己写一个main方法？这样显然代价太大。Junit单元测试给我们提供了这样的便捷，可以直接对没有main方法的class进行测试

#### 1.3 特点

- JUnit 是一个开放的资源框架，用于编写和运行测试。
- 提供**注释**来**识别测试方法**。
- 提供**断言**来**测试预期结果**。
- 提供**测试运行**来运行测试。
- JUnit 测试允许你编写代码更快，并能提高质量。
- JUnit 优雅简洁。没那么复杂，花费时间较少。
- JUnit 测试**可以自动运行并且检查自身结果**并**提供即时反馈**。所以也**没有必要人工梳理测试结果的报告**。
- JUnit 测试可以被组织为**测试套件**，包含测试用例，甚至其他的测试套件。
- JUnit 在一个条中**显示进度**。如果运行良好则是绿色；如果运行失败，则变成红色。

#### 1.4 什么是一个单元测试用例

单元测试用例是一部分代码，可以确保另一段代码（方法）按预期工作。为了迅速达到预期的结果，就需要测试框架。**JUnit 是 java 编程语言理想的单元测试框架。**

一个正式的编写好的单元测试用例的特点是：**已知输入和预期输出**，**即在测试执行前就已知**。已知输入需要测试的先决条件，预期输出需要测试后置条件。

**每一项需求至少需要两个单元测试用例：一个正检验，一个负检验。**如果一个需求有子需求，每一个子需求必须至少有正检验和负检验两个测试用例。

### 2 重要API

#### 2.1 Assert类

这个类提供了一系列的编写测试的有用的声明方法。只有失败的声明方法才会被记录。

````java
@Test
    public void testAdd(){
        int num = 5;
        String temp = null;
        String str = "Junit is working fine";
        assertEquals("Junit is working fine", str);
        assertFalse(num > 6);
        assertNotNull(str);
    }
````

|id|签名|描述|
|-|-|-|
|1|void assertEquals(Object expected, Object actual) |检查两个变量是否相等|
|2|void assertFalse(boolean condition) |检查条件是假的|
|3|void assertNotNull(Object object) |检查对象不为空|
|4|void assertNull(Object object) |检查对象为空|
|4|void assertTrue(boolean condition) |检查条件为真|

#### 2.2 TestCase类

测试样例定义了运行多重测试的固定格式

````java
import junit.framework.TestCase;
import org.junit.Before;
import org.junit.Test;
public class TestJunit2 extends TestCase  {
   protected double fValue1;
   protected double fValue2;

   @Before 
   public void setUp() {
      fValue1= 2.0;
      fValue2= 3.0;
   }

   @Test
   public void testAdd() {
      //count the number of test cases
      System.out.println("No of Test Case = "+ this.countTestCases());

      //test getName 
      String name= this.getName();
      System.out.println("Test Case Name = "+ name);

      //test setName
      this.setName("testNewAdd");
      String newName= this.getName();
      System.out.println("Updated Test Case Name = "+ newName);
   }
   //tearDown used to close the connection or clean up activities
   public void tearDown(  ) {
   }
}
````

|id|签名|描述|
|-|-|-|
|1|int countTestCases()|为被run(TestResult result) 执行的测试案例计数|
|2|TestResult createResult()|创建一个默认的 TestResult 对象|
|3|String getName()|获取 TestCase 的名称|
|4|TestResult run()|一个运行这个测试的方便的方法，收集由TestResult 对象产生的结果|
|5|void run(TestResult result)|在TestResult 中运行测试案例并收集结果|
|6|void setUp()|创建固定装置，例如，打开一个网络连接|
|7|void tearDown()|拆除固定装置，例如，关闭一个网络连接|

#### 2.3 TestResult类

TestResult类 收集所有执行测试案例的结果。它区分失败(Failure)和错误(Error)。失败是可以预料的并且可以通过假设来检查。错误是不可预料的问题就像ArrayIndexOutOfBoundsException。

#### 2.4 TestSuite类

TestSuite 类是测试的组成部分。它运行了很多的测试用例。

````java
import junit.framework.*;
public class JunitTestSuite {
   public static void main(String[] a) {
      // add the test's in the suite
      TestSuite suite = new TestSuite(TestJunit1.class, TestJunit2.class, TestJunit3.class );
      TestResult result = new TestResult();
      suite.run(result);
      System.out.println("Number of test cases = " + result.runCount());
    }
}
````

### 3 样例-使用JUnit来测试数据库的CRUD功能

````java
public class T extends AbstractTransactionalJUnit4SpringContextTests{
	
	@Resource
	private ClientService clientService;
	
	@Test
	public void testAdd(){
		Client c = new Client();
		c.setCode("88976-1");
		c.setFullName("单元测试");
		c.setShortName("测试");
		c.setLinkMan("张三");
		c.setAddressName("西城区");
		c.setCompanyUrl("www.baidu.com");
		clientService.save(c);
		Assert.assertTrue(c.getClientId()>0);
	}

	@Test
	public void testFindById(){
		Client c = new Client();
		c.setFullName("单元测试");
		clientService.save(c);
		Client c2 = clientService.getClientById(c.getClientId());
		Assert.assertEquals(c2.getFullName(), "单元测试");
	}

	@Test
	public void testUpdate(){
		Client c = new Client();
		c.setFullName("单元测试");
		clientService.save(c);
		c.setFullName("单元测试更新");
		clientService.update(c);
		System.out.println(c.getFullName());
		Assert.assertEquals(c.getFullName(), "单元测试更新");
	}
	
	@Test
	public void testDelete(){
		Client c = new Client();
		clientService.save(c);
		Long id = c.getClientId();
		clientService.delete(id);
		System.out.println(id);
		Assert.assertNull(clientService.getClientById(id));
	}
}
````
