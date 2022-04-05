---
title: 'Jint Bug: Setting Value to Array inside an object is not work'
date: 2022-04-05 00:10:21
tags:
  - Jint
categories: Jint
---
## Github issue
https://github.com/sebastienros/jint/issues/846

## 问题
The code is like following
```CSharp
            var list = new List<int>();
            for(var i = 0; i < 10; i++)
            {
                list.Add(0);
            }
            var array = list.ToArray();
            array[2] = 2;
            var engine = new Jint.Engine()
                .SetValue("Test", new { Array = array });

            var script = @"
Test.Array[0]=1;
Test.Array[1]=2;
Test.Array[2]=3;";
            var arrayResult = engine.Execute(script).GetValue("Test").ToObject();

            Console.WriteLine(JsonConvert.SerializeObject(arrayResult));
```
All the values which I set to `Test.Array` are not working as expected.
Here is the output
`{"Array":[0,0,2,0,0,0,0,0,0,0]}`

But if I change the array on the root, it works
```CSharp
            var list = new List<int>();
            for(var i = 0; i < 10; i++)
            {
                list.Add(0);
            }
            var array = list.ToArray();
            array[2] = 2;
            var engine = new Jint.Engine()
                .SetValue("Test", array);

            var script = @"
Test[0]=1;
Test[1]=2;
Test[2]=3;";
            var arrayResult = engine.Execute(script).GetValue("Test").ToObject();

            Console.WriteLine(JsonConvert.SerializeObject(arrayResult));
            Console.ReadLine();
```

Unit test to reproduce the issue
```CSharp
        [Fact]
        public void SetArrayValue()
        {
            var list = new List<int>();
            for (var i = 0; i < 10; i++)
            {
                list.Add(0);
            }
            var array = list.ToArray();
            _engine.SetValue("Test", new { Array = array });

            var script = @"
Test.Array[0]=1;
Test.Array[1]=2;
Test.Array[2]=3;
Test.Array[2]";
            var executedResult = _engine.Execute(script).GetValue("Test").ToObject();
            var executedResult2 = _engine.Execute(script).GetCompletionValue().ToObject();

            Assert.Equal(1, 1);
        }
```

## WalkAruond
Found a new idea to work around of this bug
```CSharp
        static void Main(string[] args)
        {
            var context = new { Variables = new { Array = new int[] { 1, 2, 3 }, EmptyArray = new int[] { } }  };

            var array = new int[] { };
            var engine = new Jint.Engine()
                .SetValue("log",new Action<object>(Console.WriteLine));

            var script = @"Context.Variables.Array[0]=2;
Context.Variables.Array[3]=4;
Context.Variables.EmptyArray[0]=5
Context.Variables.EmptyArray[3]=5";

            var preScript = $"var Context = {JsonConvert.SerializeObject(context)};{Environment.NewLine}";

            var executedResult = engine.Execute(preScript + script).GetValue("Context").ToObject();

            Console.WriteLine(JsonConvert.SerializeObject(executedResult));
            Console.ReadLine();
        }
```

The result is `{"Variables":{"Array":[2.0,2.0,3.0,4.0],"EmptyArray":[5.0,null,null,5.0]}}` 

## Root cause from author
I did a quick analysis yester day and the root cause is that Jint wraps array as JS array, I think in interop scenario it should be left as regular object wrapper allowing indexing. I think one would expect `engine.SetValue("myvar", new int [] { 1, 2, 3});` to be changed to JS array instance (allowing push/find etc), but not when accessing object's property.