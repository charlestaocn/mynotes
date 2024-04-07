# lua 函数

#lua函数
## 全局或者局部函数

```lua

(function()  
  
    local function innerFc()  
        print("innerFc")  
    end  
    innerFc()  --正常执行
end  
)()  
  
innerFc() -- 报错

```

## 多返回值

```lua
print((function()  
    return 1, 2, 3  
end  
)())
```

## 当参数传递

```lua
function testFun(testStr, fun)  
    fun(testStr);  
end  
  
testStr = "hello word";  
testFun(testStr,  
        function(val)  
            --匿名函数  
            print(val);  
        end  
);
```


## 变长参数

```lua

function testFun(...)  
    print('参数长度:'.. #{...})  
end  
  
print(testFun(1,2,3))

参数长度:3


function testFun(...)  
    for k,v  in pairs({...})  do  
        print(k..":"..v)  
    end  
end  
  
print(testFun('v1','v2','v3'))

1:v1
2:v2
3:v3

```

#### 字符串格式化

```lua
function fwrite(fmt, ...)  ---> 固定的参数fmt  
    return io.write(string.format(fmt, ...))      
end  
  
fwrite("runoob\n")       --->fmt = "runoob", 没有变长参数。    
fwrite("%d%d\n", 1, 2)   --->fmt = "%d%d", 变长参数为 1 和 2
```