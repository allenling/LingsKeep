一句话，go太诡异了
=======================

1.  init函数
----------------

    官方文档: https://golang.org/doc/effective_go.html#init
    每一个package中都可以有多个或者一个init函数，这些init函数会在所有全局变量和导入的包初始化(也就是调用对应包中的init函数)之后，顺序执行
    (真是好隐式啊，有点恶心)

2.  goroutine
---------------

how goroutine work(http://blog.nindalf.com/how-goroutines-work/)

把goroutine看成Python中curio中的一个task就好

goroutine好傻. 在go中的并发你必须主动去block一个goroutine才能让另外一个goroutine运行.

.. code-block:: 

    // 例子1
    func main() {
    	fmt.Println("my first goroutine, running goroutine")
    	go func () {
    		fmt.Println("+++++++++++++++++in goroutine first++++++++++++++++++")
    	}()
    	// time.Sleep(time.Second)
    	defer fmt.Println("main finish")
    	fmt.Println("still in main")
    }

    // 例子2
    func main() {
    	fmt.Println("my first goroutine, running goroutine")
    	go func () {
    		fmt.Println("+++++++++++++++++in goroutine first++++++++++++++++++")
    		time.Sleep(time.Second / 2)
    		fmt.Println("+++++++++++++++++goroutine first end++++++++++++++++++")
    	}()
    	go func () {
    		fmt.Println("+++++++++++++++++in goroutine two++++++++++++++++++")
    		time.Sleep(time.Second / 2)
    		fmt.Println("+++++++++++++++++goroutine sec end++++++++++++++++++")
    	}()
    	time.Sleep(time.Second)
    	defer fmt.Println("main finish")
    	fmt.Println("still in main")
    }
 
例子1中goroutine不会运行的，main会直接终止然后整个程序终止, 除非加上time.sleep，也就是说go最后不会去主动运行goroutine，即使你的main(也是一个goroutine)应该完成.
按道理来说一个routine运行block(主动/被动block或者运行完成)之后，应该轮到另外一个goroutine运行了的，go却没有这么做.
例子2中依然是不会打印出goroutine first end和goroutine sec end两个语句就结束了，明明还有两个goroutine没有运行完并且只运行了一半.
2.1 这其实和Python中GIL很像，比如Python中两个线程A和B，A是cpu密集型的程序，B是io密集型程序，然后A启动之后，B很可能在一段时间内拿不到GIL，就一直挂起不能运行.
    这是虽然因为A在一段时间内释放GIL之后，在B拿到GIL之前，A又拿到了GIL，因为A是cpu密集的嘛，不会被block，直接又拿到了. 所以你必须手动block A然后B才能拿到GIL才能运行.
    但是goroutine看起来更傻一点.
2.2 https://nathanleclaire.com/blog/2014/02/15/how-to-wait-for-all-goroutines-to-finish-executing-before-continuing/, 这篇文章里面的例子有竞态问题，具体的更正是在
    https://nathanleclaire.com/blog/2014/02/21/how-to-wait-for-all-goroutines-to-finish-executing-before-continuing-part-two-fixing-my-ooops/这里作者修改了一些错误，但是思路还算是正确的),
    以上基本上提出了思路是，在main中使用chan来接收goroutine返回的数据, 不太喜欢这种方式.
2.3 若channel中为空，无论是否设置了缓冲区，则读都会阻塞，若没有设置缓冲区，则若channel不为空，则写入阻塞，设置了缓冲区，则只有当缓冲区满了，则写阻塞.
    不设置缓冲区可以看成缓冲区为0的情况.

3.  string, byte, rune
----------------------------

go中的字符串和Python2中的一样，恶心呀, 虽然字符串打印出来是正常的，但是长度其实是byte的长度, 因为打印的时候依赖与编码格式，而go是utf8编码，所以打印出来是正确的字符
而range很奇怪，要小心, 对于字符串官方建议用unicode/utf8这个包来处理https://blog.golang.org/strings

.. code-block:: 

    // golang中字符串和byte的关系和python2中的一样，unicode跟rune一样，简直可怕
    unicode_string := "我们"
    fmt.Println(len(unicode_string))
    a := []byte(unicode_string)
    s := string(a)
    ru := []rune(unicode_string)
    // 输出长度分别为6,6,2, 打印出来都是我们
    fmt.Println(len(a), len(s), len(ru))
    // 单引号引起来的是rune类型, 并且只能是单个字符, 打印出来的就是字符的编码，如果是unicode，一样打出它的编码
    runne_char = '我'
    // 输出是25105
    fmt.Println(runne_char)
    /*
    range输出是unicode，但是index确实byte的index
    输出是
    0 25105
    3 20204
    */
    for index, value := range unicode_string {
        fmt.Println(index, value)
    }
    // 就算用了unicode/utf8这库，输出还是一样，感觉没什么用
    for i, w := 0, 0; i < len(unicode_string); i += w {
        runeValue, width := utf8.DecodeRuneInString(unicode_string[i:])
        fmt.Printf("%#U starts at byte position %d\n", runeValue, i)
        w = width
    }
    // unicode和ascii混用，结果还是一样
    fmt.Println("----------hybird string------------------")
    hybird_string := "anc我们"
    // 长度输出是9
    fmt.Println(hybird_string, len(hybird_string))
    for index, value := range hybird_string {
        fmt.Println(index, value)
    }
    fmt.Println("----------hybird string done------------------")

4.  类型赋值
---------------

go是强类型

同一个类型的变量可以任意赋值，而不同类型的变量则不行

.. code-block:: 

    //
    func main() {
        k := 1
        // 这样是可以的
        k = 2
        // 这样报错
        k = 'a'
    }

5.  array和slice
--------------------

这里我更是感到恶心. 


修改slice里面的元素可以看成引用修改，但是append就很诡异.

假设slicea是myslice/myarray的一部分，则
1. 修改slicea的时候，myrray/myslice对应的元素也会被修改
2. 而对slicea进行append操作之后，若此时slicea的长度没有超过myarray/myslice的长度，则myarray/myslice对应元素会被修改，若长度大于myarray/myslice，则myarray/myslice不变

原因应该是slice是共享内存，修改好理解，append的行为则是原共享内存添加元素，也就是为共享内存后面(连续的)开辟一块区域，此时指向的就是myarray/myslice后面的元素
也就是说myarray/myslice的后续元素也会被修改。并且反过来，对myarray/myslice做对应的修改也会对slicea有相应的效果(傻得一笔)

Slicing does not copy the slice's data(https://blog.golang.org/go-slices-usage-and-internals)

slice的存储结构:

slice                  array/slice

+-------+              +-------+
| 指针  | ---------->  | data  |
+-------+              +-------+
| 长度  |              | data  |
+-------+              +-------+
| 容量  |              | data  |
+-------+              +-------+

所有slice有起始位置而没有结束位置，只是用长度来标识结束.

还有，删除slice中的元素，恩，go不提供，有很多种方法，恩，再次傻得一比(https://groups.google.com/forum/#!topic/golang-china/Sk3NB_j1u9g)
其实删除的思路：
（1）被删除元素的后续元素前移
（2）被删除元素与最末元素交换，长度减一。
（3）如果只是删除两端的元素，重新slice即可（即改变array起始地址和长度），无需移动。

python是第一种，所以Python中list的删除为O(n)


.. code-block:: 

    func main() {
        myarray := [3]string{"a", "b"}
        myslice := []string{}
        myslice = append(myslice, "c")
        myslice = append(myslice, "d")
        myslice = append(myslice, "3")
        myslice = append(myslice, "f")
        myslice = append(myslice, "q")
        fmt.Println(myarray, myslice)
        fmt.Println("--------------------------------")
        // 这里slicea是原来slice的一部分, 这里把myslice换成myarray对下面的输出也没什么影响
        slicea := myslice[1:3]
        /*  输出
            [c d 3 f q] [d 3]
        */  
        fmt.Println(myslice, slicea)
        fmt.Println("--------------origin------------------")
        slicea[1] = "e"
        /*  输出
            [c d e f q] [d e]
        */
        fmt.Println(myslice, slicea)
        fmt.Println("--------------change subslice------------------")
        slicea = append(slicea, "l")
        /*  这里输出就是
            [c d e l q] [d e l]
            共享内存后面的元素被修改了
        */
        fmt.Println("--------------append subslice with l------------------")
        slicea = append(slicea, "n")
        /*  这里输出就是
            [c d e l n] [d e l n]
            共享内存后面的元素被修改了
        */
        fmt.Println("--------------append subslice with n-----------------")
        slicea = append(slicea, "m")
        /*  这里输出就是
            [c d e l n] [d e l n m]
            超过共享内存的大小，所以myslice不变
        */
        fmt.Println(myslice, slicea)
        fmt.Println("--------------append subslice with m------------------"
        myslice = append(myslice, "k")
        fmt.Println(myslice, slicea)
        /*  这里输出就是
            [c d e l n k] [d e l n k]
            既然是共享内存，所以myslice改变，则slicea也会跟着改变
        */
        fmt.Println("--------------append origin slice with k-----------------"))
    }



6. map
------------

map底层数据结构是hashmap，跟Python一样.

map可以返回一个值和两个值，根据你左边是一个或者两个来决定，这是语言上的特性, value := map[key], value, ok := map[key](傻得一比)

获取map中不存在的key的value会返回"", 不会报错，所以一般是value, ok := map[key]这样通过ok来判断(傻得一比)

关于map返回值是可变的这种情况，参考http://stackoverflow.com/questions/30129206/golang-return-multiple-values-issue 中lander的回答.



7. struct/function/method
-------------------------------

new关键字返回的是一个指针, 效果跟var := &Struct{}是一样的

new allocates zeroed storage for a new item or type whatever and then returns a pointer to it

7.1 函数传值
++++++++++++++++++++

go中，函数传值都是值传递，就算传递struct，也就是复制一份struct过去给函数，所以不会影响外面的struct，可以传递一个指针类型过去，这样也是值传递，
只是传递的是地址的值，这样你在函数中求出地址值的时候就得到对象，然后改变对象就会影响到外面的struct. 当struct很大的时候，传入指针就好很多.

.. code-block:: 

    type MethodTest struct {
        a, b string
    }

    // 传入的是struct的值，也就是复制struct
    func modify_value(m MethodTest) {
        m.a = "function modify a"
        m.b = "function modify b"
    }
    
    // 传入的是地址的值，也就是复制的是地址
    func modify_pointer(m *MethodTest) {
        m.a = "function modify a"
        m.b = "function modify b"
    }
    
    func main () {
        pointerM := &MethodTest{a: "a", b: "b"}
        valueM := MethodTest{a: "a", b: "b"}
        // 这里不会修改struct的a和b
        modify_value(valueM)
        modify_value(*pointerM)
        fmt.Println("--------------after modify_value pointerM and valueM---------")
        fmt.Println("pointerM:", pointerM, ", valueM:", valueM)
        // 这里会修改struct的a和b
        modify_pointer(&valueM)
        modify_pointer(pointerM)
        fmt.Println("--------------after modify_pointer pointerM and valueM---------")
        fmt.Println("pointerM:", pointerM, ", valueM:", valueM)
    }


7.2 定义struct方法
+++++++++++++++++++

一个方法的时候，最左边是值绑定还是指针绑定，若是指针绑定，则修改struct会修改引用的struct，若值绑定，则不会. 这根函数传值的情况一样，当结构大很大的是，指针绑定比较好(或许)

一个约定是:
Consistency: if some of the methods on the struct have pointer receivers, the rest should too. This allows predictability of behavior


.. code-block:: 

    type MethodTest struct {
        a, b string
    }
    
    func (m *MethodTest) Modify () {
        m.a = "in Modify a"
        m.b = "in Modify b"
    }
    
    // 值绑定，不会修改struct的
    func (m MethodTest) WouldNotModify() {
        m.a = "in WouldNotModify a"
        m.b = "in WouldNotModify b"
    }
    
    func main() {
        pointerM := &MethodTest{a: "a", b: "b"}
        valueM := MethodTest{a: "a", b: "b"}
        // 这里表示大小，指针比值小很多
        fmt.Println("size of pointerM: ", unsafe.Sizeof(pointerM), "size of valueM: ", unsafe.Sizeof(valueM))
        fmt.Println("pointerM:", pointerM, ", valueM:", valueM)
        // 直接修改struct都会生效
        pointerM.a = "modify directly a"
        pointerM.b = "modify directly b"
        fmt.Println("pointerM after modify property directly, a: ", pointerM.a, " b:", pointerM.b)
        valueM.a = "modify directly a"
        valueM.b = "modify directly b"
        fmt.Println("valueM after modify property directly, a: ", valueM.a, " b:", valueM.b)
        // 下面调用值绑定的方法和指针绑定的方法，效果跟函数传入的是值还是指针效果是一样的
        fmt.Println("++++++++++++in pointer m++++++++++++++")
        fmt.Println("beofre a and b", pointerM.a, pointerM.b)
        pointerM.Modify()
        fmt.Println("after call Modify")
        fmt.Println("a: ", pointerM.a, "b: ", pointerM.b)
        pointerM.WouldNotModify()
        fmt.Println("after call WouldNotModify")
        fmt.Println("a: ", pointerM.a, "b: ", pointerM.b)
        fmt.Println("++++++++++++in value m++++++++++++++")
        fmt.Println("beofre a and b", valueM.a, valueM.b)
        valueM.Modify()
        fmt.Println("after call Modify")
        fmt.Println("a: ", pointerM.a, "b: ", pointerM.b)
        valueM.WouldNotModify()
        fmt.Println("after call WouldNotModify")
        fmt.Println("a: ", pointerM.a, "b: ", pointerM.b)
    }


8. 可变参数以及类型转换
------------------------

http://www.jianshu.com/p/94710d8ab691

[]string{}和[]interface{}是两种类型，可变参数传参的规律是: 对于 func(first int, arg ...T)

1. 当不传可变参数时，对应的 arg 就是 nil

2. 传入单个可变参数时，实际上执行 [] T{arg1,arg2,arg3}

3. 传入...语法糖的 slice时，直接使用这个 slice

所以当一个函数的参数是data ...[]interface的时候，传入之前必须转成[]interface{}类型.

.. code-block:: 

    func test(data ...interface{}){
        fmt.Println(data)
        }

    func main() {
        x:=[]string{"a", "b", "c"}
        test(x...)
    }


上面的例子会报错: 

cannot use x (type []string) as type []interface {} in argument

所以必须转换类型

.. code-block:: 

    func test(data ...interface{}){
        fmt.Println(data)
    }
    
    
    func main() {
        x:=[]string{"a", "b", "c"}
        a := make([]interface{}, len(x))
        for index, value := range x{
            a[index] = value
        }
        test(a...)
        test("a")
    }


上面例子输出:
[a b c]
[a]


:=和=
---------

:=仅限于未声明的变量，=仅限于声明了的变量，但是如果函数返回多个值，比如两个，可以同时使用一个已经定义的和未定义的变量来接收返回值.

.. code-block:: 

    package main
    
    import (
        "fmt"
    )
    
    type Test struct {
        A, B string
    }
    
    func create() (*Test, error){
        tmp := new(Test)
        return tmp, nil
    }
    
    
    func main() {
        x := new(Test)
        fmt.Println(&x)
        x, err := create()
        fmt.Println(&x, err)
    }

这里接收create的返回值的时候，x是已经定义过的，err是未定义过的，这时也可以使用:=来赋值，结果就是golang会复用x, 生成一个新的err


go中作用域
-----------

http://studygolang.com/articles/9095


也就是每一个花括号都是一个隔离的作用域, 任何在花括号里面进行:=赋值的都是创建一个新的变量，如果用的是=来重新赋值，则golang会根据
作用域往上找

.. code-block:: 

    package main
    
    import (
        "fmt"
    )
    
    
    func main() {
    	x := 100
    	fmt.Println(x, &x)
    	for i := 0; i < 5; i++ {
    		x := i
    		fmt.Println(x, &x)
    	}
    	fmt.Println(x, &x)
    }


上面打印出来的结果是for中每一个x的地址都不一样，并且跟for外部的x也不一样.


这里值得注意的是创建对象之后，对对象变量的重新赋值上:

.. code-block:: 

  package main
  
  import (
      "fmt"
  )
  
  type Test struct {
      A, B string
  }
  
  func create() (*Test, error){
      tmp := new(Test)
      return tmp, nil
  }
  
  
  func main() {
      a := 1
      x := new(Test)
      if a == 2 {
          fmt.Println(a)
      }else{
          x, err := create()
          fmt.Println(err, &x)
      }
      fmt.Println(&x)
  }


这里在else打印出来的x和外部的x是不一样的，因为在else里面用了:=赋值，所以else里面就是新分配的变量, 如果else里面是用=, 则x都是一个变量(地址一样)

