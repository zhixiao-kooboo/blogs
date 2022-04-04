---
title: How to invoke .Net in Jint
date: 2022-04-05 00:13:37
tags:
  - Jint
categories: Jint
---

## With Action and Callback
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