+++
date = "2016-1-10T16:30:09+08:00"
draft = true
title = "【原创】reflect func in golang (get func by name)"

+++
#### 本屌丝先补习下知识

    type Type interface {
		......
        // Bits returns the size of the type in bits.
        // It panics if the type's Kind is not one of the
        // sized or unsized Int, Uint, Float, or Complex kinds.
        Bits() int
        // Elem returns a type's element type.
        // It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
        Elem() Type
        // In returns the type of a function type's i'th input parameter.
        // It panics if the type's Kind is not Func.
        // It panics if i is not in the range [0, NumIn()).
        In(i int) Type
        // NumIn returns a function type's input parameter count.
        // It panics if the type's Kind is not Func.
        NumIn() int
       // NumField returns a struct type's field count.
        // It panics if the type's Kind is not Struct.
        NumField() int
    }
    另外还有ValueOf  Type  Call等函数`
#### 用string拿func

之前写的python程序中，使用过根据string调用相关函数。这次写个东西需要这个，go中肯定也能实现。咋弄呢，对肯定是reflect，但是用万能的reflect具体咋实现呢？之前一直晕晕的，反正reflect这东西效率低，尽量不用。所以Google出来就贴代码里就完了，没有具体研究过reflect。这次小小看了下关于func的reflect。

先说结果，我了解到的有两种方式

- 获取struct或者其他可以指向的函数（暂且先这么说，等我再细致研究下，再更正下叫法）
- 直接获取一个函数，此种方法需要实现将函数注册

    func (v Value) MethodByName(name string) Value
    MethodByName returns a function value corresponding to the method of v with the given name. The arguments to a Call on the returned function should not include a receiver; the  returned function will always use v as the receiver. It returns the zero Value if no method was found.

我个人感觉应该使用interface来规范和保证struct的接口，所以这里主要介绍第二种。
#### 咋用呢

go里面，需要先注册函数，然后通过interface或者reflect.Type拿到func的type，然后进行处理。

那么如何调用 map 中的函数呢？像这样吗：

    funcs := map[string]func() {"foobar":foobar}
    funcs["foobar"]()
绝对不行！这无法工作！你不能直接调用存储在空接口中的函数。

万能而慢慢的reflect来了！总的思路就是`先拿到函数的type`（不论你通过注册interface传过来再reflect.TypeOf,还是把reflect.ValueOf(FuncInterface)转为reflect.Type之后再注册，不过主意判断是否是func的type，你可以通过InNum来校验），然后把参数列表反射出来转为`[]reflect.Value`，最后`Call`。

fork代码请移步，https://github.com/ianwoolf/golib/tree/master/funMap。欢迎pr

懒得打文字了，上代码

#### 上代码！

    type Funcs map[string]interface{}
    func NewFuncs(size int) Funcs {
    	return make(Funcs, size)
    }
    func (f Funcs) Bind(name string, fn interface{}) (err error) {
	    defer func() {
		    if e := recover(); e != nil {
			    err = errors.New(name + " is not callable.")
    		}
    	}()
    	v := reflect.ValueOf(fn)
    	// panic if non-func type
    	v.Type().NumIn()
    	f[name] = fn
    	return
    }
    func (f Funcs) Call(name string, params ...interface{}) (result []reflect.Value, err error) {
    	if _, ok := f[name]; !ok {
    		err = errors.New(name + " does not exist.")
    		return
    	}
    	return Invoke(f[name], params...)
    }
    func Invoke(FuncInterface interface{}, args ...interface{}) (result []reflect.Value, err error) {
	    funcType := reflect.ValueOf(FuncInterface)
	    if len(args) != funcType.Type().NumIn() {
	    	err = ErrParamsNotAdapted
	    	return
	    }
	    in := make([]reflect.Value, len(args))
	    for k, arg := range args {
	    	in[k] = reflect.ValueOf(arg)
	    }
	    result = funcType.Call(in)
	    return
    }

不过应该是转为reflect.Type再注册比较好。第二版代码来了

    type Funcs map[string]reflect.Value

    func NewFuncs(size int) Funcs {
    	return make(Funcs, size)
    }

    func (f Funcs) Bind(name string, fn interface{}) (err error) {
	    defer func() {
		    if e := recover(); e != nil {
			    err = errors.New(name + " is not callable.")
		    }
	    }()
    	v := reflect.ValueOf(fn)
    	// panic if non-func type
    	v.Type().NumIn()
    	f[name] = v
    	return
    }

    func (f Funcs) Call(name string, params ...interface{}) (result []reflect.Value, err error) {
    	if _, ok := f[name]; !ok {
    		err = errors.New(name + " does not exist.")
	    	return
    	}
    	if len(params) != f[name].Type().NumIn() {
	    	err = ErrParamsNotAdapted
		    return
    	}
    	result = Invoke(f[name], params...)
	    return
    }

    func Invoke(FuncValue reflect.Value, args ...interface{}) []reflect.Value {
    	in := make([]reflect.Value, len(args))
    	for k, arg := range args {
    		in[k] = reflect.ValueOf(arg)
    	}
    	return f[name].Call(in)
    }


#### by the way

###### 一个主意点
    func foobar() {
    // bla...bla...bla...
    }
    funcs := map[string]func() {"foobar":foobar}
    funcs["foobar"]()
但这里有一个限制：这个 map 仅仅可以用原型是“func()”的没有输入参数或返回值的函数。
如果想要用这个方法实现调用不同函数原型的函数，需要用到 interface{}。

###### 发现一类有意思的函数，还没研究
`MapOf、ChanOf、PtrOf`等

    func FuncOf
    func FuncOf(in, out []Type, variadic bool) Type
    FuncOf returns the function type with the given argument and result types. For example if k represents int and e represents string, FuncOf([]Type{k}, []Type{e}, false) represents     func(int) string.
    The variadic argument controls whether the function is variadic. FuncOf panics if the     in[len(in)-1] does not represent a slice and variadic is true.

###### 你也许想看的东西

[The Go Programming Language Specification:Interface types](https://golang.org/ref/spec#Interface_types)

[Laws of reflection](http://blog.golang.org/laws-of-reflection)

[反射的规则](http://mikespook.com/2011/09/%E5%8F%8D%E5%B0%84%E7%9A%84%E8%A7%84%E5%88%99/)


