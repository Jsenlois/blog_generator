---
title: Mockito单元测试
tags: 测试
date: 2017-10-07 13:14:07
comments: true
---

## Mokcito单元测试 
> reference : [mockito doc](http://static.javadoc.io/org.mockito/mockito-core/2.10.0/org/mockito/Mockito.html)

### 单元测试why？  
- 发现问题的成本最低
- 调教你的编程习惯
- 保证代码质量
### 优点
- 语法简洁，使用简单，易上手
- 比较有测试的思维（无法随意Mock）
### 限制
- 不能mock静态方法
- 不能mock构造方法
- 不能mock equals()，hashCode()
- 不能mock final类 ， final方法
- 不能mock私有方法，可以把私有，变成protected
，若你需要mock私有，可能是你的程序还不够OO (Object Oriented)
### 常规用法
#### 大体蓝图  
当你需要桩（取代某个依赖）的时候，你可以用两种方式来模拟桩，spy & mock
- spy
```
MockedObject object = new MockedObject();  
MockedObject spiedObject = Mockito.spy(object);
```
- mock  
```
MockedObject mockedObject = Mockito.mock(MockedObject.class);
```
- 两者的区别：    

  当你使用spy来获取到一个桩的时候，你在执行某个方法的时候，spy的对象都会==真正的去执行方法内的代码==。  
  
  而当桩是通过mock获取到的话，mock的对象的所有属性都为空，所有方法的返回值都是返回类型的默认值（如对象类型的默认值是null），==方法里的代码不会执行==。 
  
  其实这点区别是可以从代码层面上看出来的，spy的时候先拿到了一个对象，然后对对象进行spy，而mock的时候使用的则是字节码。  
- 两者之间的效果也可以互相转化  
    spy类型的可以通过 ``` doReturn() ``` ，使得代码不被执行，mock类型可以通过 ``` doRealCallMethod() ``` ,使得代码被执行，所以spy和mock的界限其实并不清晰
- 代码示例  
  被mock和spy的类 

```
class TestMock{

    public void t1 (String ss) {
        System.out.println("hh"+ss);
    }

    public String t2 (String ss) {
        return "hh"+ss;
    }

    public Boolean t3() {
        return true;
    }

    public String t4(MockParam mockParam) {

        return mockParam.name;
    }

    public String t5(MockParam mockParam , String s) {
        return s;
    }
}
```

  参数类（在上面的类中的某些方法的参数）   

```
class MockParam{

    String name;

    int age;

    public MockParam(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object obj) {
        obj = (MockParam)obj;
        if(this == obj || this.name.equals(((MockParam) obj).name)) {
            return true;
        }
        return false;
    }
}
```

  mock样例  
  所有方法的返回都是方法的默认类型，方法内的代码不会被执行

```
public void test2() {

    TestMock mock = mock(TestMock.class);

    mock.t1("a");

    System.out.println(mock.t2("b"));

    System.out.println(mock.t3());

    when(mock.t2("c")).thenReturn("666");

    when(mock.t2("c")).thenReturn("777");

    System.out.println(mock.t2("c"));
}

```
  
  spy样例  
  所有方法的代码都会被执行，并正常返回  

```
public void test3() {

    TestMock mock1 = new TestMock();

    TestMock spyTest = spy(mock1);

    spyTest.t1("c");

    System.out.println(spyTest.t2("d"));

    System.out.println(spyTest.t3());

}
```
  
#### 参数匹配 
- 通常情况下，参数匹配使用的equals方法  

```
 @Test
public void test4() {

    TestMock mock = mock(TestMock.class);

    when(mock.t2("c")).thenReturn("777");

    System.out.println(mock.t2("d"));//d,c equals = false
}

@Test
public void test5() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("555");

    when(mock.t4(mockParam)).thenReturn("555");

    System.out.println(mock.t4(mockParam));//equals true

    System.out.println(mock.t4(new MockParam("555")));//equals false 可以重写equals方法，使之为ture

}
```

- 内置参数匹配(常用)  

```
@Test
public void test6() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("555");

    when(mock.t4(any())).thenReturn("555");

    System.out.println(mock.t4(mockParam));//true

    System.out.println(mock.t4(new MockParam("555")));//true

    when(mock.t2(anyString())).thenReturn("dddd");

    System.out.println(mock.t2("aaa"));//return dddd

    System.out.println(mock.t2("bbb"));//return dddd
}
```

- 注意
⚠ :如果有一个参数使用了参数匹配器，那所有参数都得使用参数匹配器

```
@Test
public void test7() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("555");

    //when(mock.t5(any() , "6666")).thenReturn("666"); //no
    when(mock.t5(any() , anyString())).thenReturn("666");  //yes

    System.out.println(mock.t5(mockParam , "ddd"));
}
```

- 自定义参数匹配器，先实现参数匹配器的接口，根据需要在matches里返回true or false  

```
//定义参数匹配器
class MockParamArgMatcher implements ArgumentMatcher<MockParam> {

    @Override
    public boolean matches(MockParam mockParam) {

        if(mockParam.name.equals("666")) {
            return true;
        }

        return false;
    }
}
//使用匹配器
@Test
public void test9() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("666");

    when(mock.t4(argThat(new MockParamArgMatcher()))).thenReturn("666666666");

    System.out.println(mock.t4(mockParam));

}

```

- 附加参数匹配器(参数验证)   

```
@Test
public void test8() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("555");

    //在此处可以做入参的逻辑校验
    //when(mock.t5(any() , eq("6666"))).thenReturn("666");  //yes
    
    when(mock.t5(any() , not( and( eq("6666"),eq("444"))) ) ).thenReturn("666");  //yes

    System.out.println(mock.t5(mockParam , "6667"));
}
```

- 官网推荐的参数验证方式（很好用，使用这个差不多就可以取代断点调试了）

```
@Test
public void test10() {

    TestMock mock = mock(TestMock.class);

    MockParam mockParam = new MockParam("666");

    MockParam mockParam1 = new MockParam("777");

    when(mock.t4(any())).thenReturn("666666666");

    System.out.println(mock.t4(mockParam1));

    //参数捕捉器，可以捕捉该类型的入參
    ArgumentCaptor<MockParam> captor = ArgumentCaptor.forClass(MockParam.class);

    //这条语句一定要执行
    verify(mock).t4(captor.capture());

    System.out.println(captor.getAllValues().size());

    assertEquals("666" , captor.getValue().name);
}
```

#### 其他常规套路
- 验证调用方法的次数  

```
@Test
public void test11() {

    TestMock mock = mock(TestMock.class);

    mock.t1("1");
    mock.t1("2");
    mock.t1("3");
    mock.t1("4");
    mock.t1("5");
    mock.t1("6");
    mock.t1("7");
    mock.t1("8");

    verify(mock,times(8)).t1(anyString());
    verify(mock,atLeast(2)).t1(anyString());
    verify(mock,atMost(9)).t1(anyString());

}
```

- 验证方法执行的顺序  

```
@Test
public void test14() {

    TestMock mock = mock(TestMock.class);

    mock.t1("");
    mock.t2("");

    InOrder inOrder = Mockito.inOrder(mock);

    inOrder.verify(mock).t1("");
    inOrder.verify(mock).t2("");

}
```

- doReturn  

```
 @Test
public void test16() {

    List list = new LinkedList();
    List spy = spy(list);
    //spy 模式会执行代码，但目前list为空
    //when(spy.get(0)).thenReturn("foo");   //IndexOutOfBoundsException
    //相当于mock
    doReturn("foo").when(spy).get(0);
}

```

---

    
