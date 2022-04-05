---
title: How to invoke .Net in Jint
date: 2022-04-05 00:13:37
tags:
  - Jint
categories: Jint
---
# 描述
Jint可以实现和.Net的交互
- 从Javascript中修改CLR中的Object, 包括:
   -  Single values
   -  Objects
   -  Properties
   -  Methods
   -  Delegates
   -  Anonymous objects
-  从Javascript的Values转换成CLR的Object
   -  Primitive values
   -  Object -> expando objects (IDictionary<string, object> and dynamic)
   -  Array -> object[]
   -  Date -> DateTime
   -  number -> double
   -  string -> string
   -  boolean -> bool
   -  Regex -> RegExp
   -  Function -> Delegate
   -  Extensions methods

# Jint调用.Net
## 从JS中修改.Net的值
```CSharp
var p = new Person {
    Name = "Mickey Mouse"
};

var engine = new Engine()
    .SetValue("p", p)
    .Execute("p.Name = 'Minnie'");

Assert.AreEqual("Minnie", p.Name);
```

## 调用.Net方法
```CSharp
var engine = new Engine()
    .SetValue("log", new Action<object>(Console.WriteLine));
    
engine.Execute(@"
    function hello() { 
        log('Hello World');
    };
 
    hello();
");
```

## 调用Delegate

用Jint实现Wait，每次循环执行一次action，Javascript内的function在jint中实际会转换成.Net中的Action
```C#
using Jint.Native;
using System;
using System.Threading;
using System.Threading.Tasks;

namespace Jint.Demo
{
    class Program
    {
        static void Main(string[] args)
        {
            var engine = new Engine();
            var test2 = new Test2();
            var test = new Test(new Action(() => { test2.Callback(engine.GetValue); }));
            engine.SetValue("test", test);
            engine.SetValue("context", 1);
            engine.Execute(@"context = 10;test.Wait(2, 2, function(){ context++; });");
            var context = engine.GetValue("context");
            Console.WriteLine("context: " + context);
            Console.ReadLine();
        }

        public class Test
        {
            private readonly Action callback;

            public Test(Action callback)
            {
                this.callback = callback;
            }

            public void Wait(int times, int second, Action action)
            {
                var count = 0;
                while (count <= times)   
                {
                    action();
                    callback();
                    count++;
                    Thread.Sleep(second * 1000);
                }
            }
        }

        public class Test2
        {
            public void Callback(Func<string, JsValue> getContext)
            {
                Console.WriteLine("Callback is called" + getContext("context"));
            }
        }
    }
}

```




## 使用.Net中的Class
```CSharp
var engine = new Engine();
engine.SetValue("TheType", TypeReference.CreateTypeReference(engine, typeof(TheType)));
engine.Execute("var o = new TheType();")
```

# 从Jint中获取值
## 获取特定变量的值
```CSharp
var context = new Context();
var engine = new Engine()
    .SetValue("Context", context);
    
engine.Execute(@"Context.test = 1;");

var jsValue = engine.GetValue("Context");
```