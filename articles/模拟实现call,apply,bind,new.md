#### call方法是在一个指定的this和若干个指定的参数值的前提下调用某个函数或方法
例：
```
var obj = {
    a: 1
};
function bar() {
    console.log(this.a);
}
bar.call(obj);
// 结果是1
```
模拟call方法
```
Function.prototype.call = function(context) {
    var context = context || window;
    context.fn = this;
    
    var args = [];
    for(var i = 1, len = arguments.length; i  < len; i++) {
        args.push('arguments[' + i + ']');
    }
    
    var result = eval(context.fn(' + args +'));
    delete context.fn;
    return result;
}
```
#### apply 同call方法，只是在参数上，第二个参数是array类型
```
Function.prototype.apply = function(context, arr) {
    var context = Object(context) || window;
    context.fn = this;
    
    var result;
    if(!arr) {
        result = context.fn();
    } else {
        var args = []; 
        for(var i = 0, len = arr.length; i < len; i++) {
            args.push('arr[ + i + ']');
        }
        //此处相当于执行了 context.fn(arr[0]);
        result = eval('context.fn(' + args + ')');
    }
    return result;
}
```

#### bind方法创建一个新的函数，在bind被调用时，这个新函数的this被bind的第一个参数指定

```

```
#### new 运算符创建一个用户自定义的对象类型的实例或具有构造函数的内置对象类型之一
```
funtion objectFactory() {
    var obj = new Object();
    Constructor = [].shift.call(arguments);
    
    var F = function() {};
    F.prototype = Constructor.prototype;
    obj = new F();
    
    var ret = Constructor.apply(obj, arguments);
    
    return typeof ret === 'object' ? ret || obj : obj;
}
```