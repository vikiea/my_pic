# PHP基础
## 正则表达式
邮箱：`/^[_a-z0-9-]+(\.[_a-z0-9-]+)*@[a-z0-9-]+(\.[a-z0-9-]+)*(\.[a-z]{2,})$`
## array_merge与数组相加的区别
- 用array_merge合并数组一个数组中的值附加在前一个数组的后面，返回作为结果的数组如果数组包含**数字键名**，**后面的值将不会覆盖原来的值**，而是附加到后面。 然而用加号来合并数组**如果键名相同**，则**取最先出现的数组值，后面的就直接忽略掉**
- 用array_merge合并数组一个数组中的值附加在前一个数组的后面。如果**非数字键名相同**，则**后面数组的值会覆盖前面数组的值**。 然而用加号来合并数组如果**键名相同**，则**取最先出现的数组值，后面的就直接忽略掉**


## PHP7
### PHP7新特性
- **标量类型声明**
    PHP 7 中的函数的形参类型声明可以是标量了。在 PHP 5 中只能是类名、接口、array 或者 callable (PHP 5.4，即可以是函数，包括匿名函数)，现在也可以使用 string、int、float和 bool 了。
    ```php
    <?php
    // 强制模式
    function sumOfInts(int ...$ints)
    {
        return array_sum($ints);
    }
    
    var_dump(sumOfInts(2, '3', 4.1));
    ```

- **返回值类型声明**
    PHP 7 增加了对返回类型声明的支持。 类似于参数类型声明，返回类型声明指明了函数返回值的类型。可用的类型与参数声明中可用的类型相同。
    ```php
    <?php
    
    function arraysSum(array ...$arrays): array
    {
        return array_map(function(array $array): int {
            return array_sum($array);
        }, $arrays);
    }
    
    print_r(arraysSum([1,2,3], [4,5,6], [7,8,9]));
    ```
    
- **NULL 合并运算符**
    由于日常使用中存在大量同时使用三元表达式和 isset()的情况，NULL 合并运算符使得变量存在且值不为NULL， 它就会返回自身的值，否则返回它的第二个操作数。
    
    ```php
    <?php
    // 如果 $_GET['user'] 不存在返回 'nobody'，否则返回 $_GET['user'] 的值
    $username = $_GET['user'] ?? 'nobody';
    // 类似的三元运算符
    $username = isset($_GET['user']) ? $_GET['user'] : 'nobody';
    ?>
    ```
    
- **太空船操作符（组合比较符）**
    太空船操作符用于比较两个表达式。当$a大于、等于或小于$b时它分别返回-1、0或1。
    ```php
    <?php
    // 整型
    echo 1 <=> 1; // 0
    echo 1 <=> 2; // -1
    echo 2 <=> 1; // 1
    
    // 浮点型
    echo 1.5 <=> 1.5; // 0
    echo 1.5 <=> 2.5; // -1
    echo 2.5 <=> 1.5; // 1
     
    // 字符串
    echo "a" <=> "a"; // 0
    echo "a" <=> "b"; // -1
    echo "b" <=> "a"; // 1
    ?>
    ```
- **通过 define() 定义常量数组**
    ```php
    <?php
    define('ANIMALS', [
        'dog',
        'cat',
        'bird'
    ]);
    echo ANIMALS[1]; // 输出 "cat"
    ?>
    ```
- **匿名类**
    现在支持通过new class 来实例化一个匿名类
    ```php
    <?php
    interface Logger {
        public function log(string $msg);
    }
    
    class Application {
        private $logger;
    
        public function getLogger(): Logger {
             return $this->logger;
        }
    
        public function setLogger(Logger $logger) {
             $this->logger = $logger;
        }
    }
    
    $app = new Application;
    $app->setLogger(new class implements Logger {
        public function log(string $msg) {
            echo $msg;
        }
    });
    
    var_dump($app->getLogger());
    ?>
    ```
- **Closure::call()**
    Closure::call() 现在有着更好的性能，简短干练的暂时绑定一个方法到对象上闭包并调用它。

    ```php
    <?php
    class A {private $x = 1;}
    
    // Pre PHP 7 代码
    $getXCB = function() {return $this->x;};
    $getX = $getXCB->bindTo(new A, 'A'); // intermediate closure
    echo $getX();
    
    // PHP 7+ 代码
    $getX = function() {return $this->x;};
    echo $getX->call(new A);
    ```
- **为unserialize()提供过滤**
    这个特性旨在提供更安全的方式解包不可靠的数据。它通过白名单的方式来防止潜在的代码注入。
    ```php
    <?php
    
    // 转换对象为 __PHP_Incomplete_Class 对象
    $data = unserialize($foo, ["allowed_classes" => false]);
    
    // 转换对象为 __PHP_Incomplete_Class 对象，除了 MyClass 和 MyClass2
    $data = unserialize($foo, ["allowed_classes" => ["MyClass", "MyClass2"]);
    
    // 默认接受所有类
    $data = unserialize($foo, ["allowed_classes" => true]);
    ```
- **use 加强**
    从同一 namespace 导入的类、函数和常量现在可以通过单个 use 语句 一次性导入了。
    ```php
    <?php
    //  PHP 7 之前版本用法
    use some\namespace\ClassA;
    use some\namespace\ClassB;
    use some\namespace\ClassC as C;
    
    use function some\namespace\fn_a;
    use function some\namespace\fn_b;
    use function some\namespace\fn_c;
    
    use const some\namespace\ConstA;
    use const some\namespace\ConstB;
    use const some\namespace\ConstC;
    
    // PHP 7+ 用法
    use some\namespace\{ClassA, ClassB, ClassC as C};
    use function some\namespace\{fn_a, fn_b, fn_c};
    use const some\namespace\{ConstA, ConstB, ConstC};
    ?>
    ```
- **Generator 加强**
    增强了Generator的功能，这个可以实现很多先进的特性
    ```php
    <?php
    function gen()
    {
        yield 1;
        yield 2;
        yield from gen2();
    }
    
    function gen2()
    {
        yield 3;
        yield 4;
    }
    
    foreach (gen() as $val)
    {
        echo $val, PHP_EOL;
    }
    ?>
    ```
- **整除**
    新增了整除函数 intdiv(),使用实例：
    
    ```php
    <?php
    var_dump(intdiv(10, 3));
    ?>
    //输出：int(3)
    ```
- **PHP 7 Session 选项**
    PHP 7 session_start() 函数可以接收一个数组作为参数，可以覆盖 php.ini 中 session 的配置项。

这个特性也引入了一个新的 php.ini 设置（session.lazy_write）, 默认情况下设置为 true，意味着 session 数据只在发生变化时才写入。

除了常规的会话配置指示项， 还可以在此数组中包含 read_and_close 选项。如果将此选项的值设置为 TRUE， 那么会话文件会在读取完毕之后马上关闭， 因此，可以在会话数据没有变动的时候，避免不必要的文件锁。

    ```php
    <?php
    session_start([
       'cache_limiter' => 'private',
       'read_and_close' => true,
    ]);
    ?>
    ```
- **PHP7可主动抛出异常**
    PHP 7 改变了大多数错误的报告方式。不同于 PHP 5 的传统错误报告机制，现在大多数错误被作为 Error 异常抛出。

这种 Error 异常可以像普通异常一样被 try / catch 块所捕获。如果没有匹配的 try / catch 块， 则调用异常处理函数（由 `set_exception_handler()` 注册）进行处理。 如果尚未注册异常处理函数，则按照传统方式处理：被报告为一个致命错误（Fatal Error）。

Error 类并不是从 Exception 类 扩展出来的，所以用 `catch (Exception $e) { ... } `这样的代码是捕获不 到 Error 的。你可以用 `catch (Error $e) { ... }` 这样的代码，或者通过注册异常处理函数（ `set_exception_handler()`）来捕获 Error。
### PHP7废弃特性
- 以静态的方式调用非静态方法
- password_hash() 随机因子选项
- PHP4风格的构造函数

### PHP7移除的扩展
- ereg
- mysql
- sybase_ct

### GENERATOR函数
#### yield
- 什么是 "yield"

> 生成器函数看上去就像一个普通函数， 除了不是返回一个值之外， 生成器会根据需求产生更多的值。

- "yield" & "return" 的区别

```php
function getValues() {
    return 'value';
}
var_dump(getValues()); // string(5) "value"

function getValues() {
    yield 'value';
}
var_dump(getValues()); // class Generator#1 (0) {}
```
> **生成器** 类实现了 **迭代器** 接口， 这意味着你必须遍历 getValue() 方法来取值：

```php
foreach (getValues() as $value) {
   echo $value;
}

// 使用变量也是好的
$values = getValues();
foreach ($values as $value) {
   echo $value;
}
```

- yield的好处

> 一个**生成器允许你使用循环来迭代一组数据，而不需要在内存中创建一个数组**，这可能会导致你超出内存限制。

- 什么是 "yield" 选项
    a. **使用 yield， 你也可以使用 return。**
    
    ```php
    function getValues() {
       yield 'value';
       return 'returnValue';
    }
    $values = getValues();
    foreach ($values as $value) {}
    echo $values->getReturn(); // 'returnValue'
    ```
    
    b. **返回键值对：**
    
    ```php
    function getValues() {
       yield 'key' => 'value';
    }
    $values = getValues();
    foreach ($values as $key => $value) {
       echo $key . ' => ' . $value;
    }
    ```