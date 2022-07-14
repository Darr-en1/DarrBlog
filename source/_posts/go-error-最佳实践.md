---

title: go error 最佳实践
copyright: true
permalink: 1
top: 1
date: 2022-06-10 16:25:46
tags:
    - golang
    - error
    - design
    - 源码
categories: golang
password:
typora-root-url: ../../source
---

有别于其他语言错误和异常的概念模糊，Go语言直接从语法层面提供区分错误和异常的机制。错误通常是良性的，程序中可能出现的问题，因此错误处理也被视为业务的一部分。而异常则作为意料之外的存在而出现，通常是毁灭性的，直接导致程序的崩溃。本文将从多方面维度介绍golang 的 error ，实际应用和最佳实践，相信看完这文章后，你对 golang中 error 能有一个全面的理解。
<!--more-->



### golang error 设计理念

Go Error 的设计哲学主要有两点：

- 处理所有潜在的错误
- Errors Are Value



#### 处理所有潜在的错误

Go 从最初设计起就确定了一个原则：程序中的所有潜在的错误都必须被明确地处理。假如按照java的  `try catch`  异常处理机制设计：



```golang
func CopyFile(src, dst string) throws error {
	r := os.Open(src)
	defer r.Close()

	w := os.Create(dst)
	io.Copy(w, r)
	w.Close()
}
```

`os.Open`  `os.Create`  `io.Copy` 都可能失败引发错误，当错误发生时会抛出错误 `throws error` ，不同的错误处理的逻辑也不太一样，我们并不能直接知道具体哪个函数出错，因此需要依次做出相应的处理，属于隐晦地处理错误的方法。golang 希望避免这种情况, 它希望清楚地处理错误（error），而不是把它当作异常（exception）。因此golang 设计原则下的代码长这样：

```golang
func CopyFile(src, dst string) error {
	r, err := os.Open(src)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	defer r.Close()

	w, err := os.Create(dst)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if _, err := io.Copy(w, r); err != nil {
		w.Close()
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if err := w.Close(); err != il {
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
}
```

 golang会对每一个存在的潜在错误都做检查和处理，使程序更加健壮。严格的将error和excepiton区别开来。

但这也会带来一个问题：即代码中重复出现很多次 `if err != nil` ，可读性较差，是  每一个 Gopher 挥之不去的心病。



#### Errors Are Values

`Errors Are Values` 阐述 error 是可以被赋值，即通过代码来自定义 `error` 。



##### error接口

golang语言对错误的设计非常简洁，原生实现如下：

```golang
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

对于开发者来说，功能是明显不足的，比如我们需要知道出错的更多信息，在什么文件的，哪一行代码等等。怎么办呢？这正是golang设计的精妙之处。 



golang 中的 error 是其内置的一个接口类型，只声明了一个方法 `Error`，只要实现了这个方法，就是实现了`error`，通过自定义 Error 使得我们在处理错误时有很大的发挥空间。这是便是 `Errors Are Values`  的基础。

```golang
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```



##### Errors Are Values 实践

 `if err != nil` 一直被诟病在代码中出现太多次，可读性很差。但合理的设计是可以避免这种情况的。通过将error 封装的方式，就可以避免重复的` if err != nil`。



##### 错误处理优化 减少if err != nil

普通写法：

```golang
func count(r io.Reader) (int, error) {
	var (
		br    = bufio.NewReader(r)
		lines int
		err   error
	)

	for {
		// 读取到换行符就说明是一行
		_, err = br.ReadString('\n')
		lines++
		if err != nil {
			break
		}
	}

	// 当错误是 EOF 的时候说明文件读取完毕了
	if err != io.EOF {
		return 0, err
	}

	return lines, err
}
```

有效设计减少 `if err != nil` 的设计方式：

```golang
func count2(r io.Reader) (int, error) {
	var (
		sc = bufio.NewScanner(r)
		lines int
	)

	for sc.Scan() {
		lines++
	}
	
	return lines, sc.Err()
}
```

 

##### gorm 优秀案例

官方标准库及很多优秀的开源库([gorm](https://github.com/go-gorm/gorm)，[go-redis](https://github.com/go-redis/redis))就采用了这种方式，以gorm举例：

假设有以下场景，用户购买一件商品，数据库生成两条数据，向订单表插入一条数据，并且账户表对用户的余额进行更新(暂不考虑事物和先后顺序)，如果订单表插入失败则不进行后续操作。

笔者通过两次插入模拟该场景，一般的代码编写如下：

```golang
func test(db *gorm.DB) error {
	if ret := db.Model(&wemediaModel.TWeMediaRegion{}).Create(map[string]interface{}{
		"order": "test001",
	}); ret.Error != nil {
		return ret.Error
	}

	db = db.Model(&wemediaModel.TWeMediaRegion{}).Create(map[string]interface{}{
		"owner": "test_darr_en1",
	})
	return db.Error
}
```

因为表没有order，owner字段，所以会报错

```shell
2022/06/13 18:13:52 /Users/tiger/cmy-project/gocms/test/main.go:13 Error 1054: Unknown column 'order' in 'field list'
[82.561ms] [rows:0] INSERT INTO `t_we_media_region` (`order`) VALUES ('test001')
Error 1054: Unknown column 'order' in 'field list'
```

如果插入次数增多，就会变得十分累赘，gorm 支持以下写法：

```golang
func test(db *gorm.DB) error {
	db = db.Model(&wemediaModel.TWeMediaRegion{}).Create(map[string]interface{}{
		"order": "test001",
	})

	db = db.Model(&wemediaModel.TWeMediaRegion{}).Create(map[string]interface{}{
		"owner": "test_darr_en1",
	})
	return db.Error
}
```

同样报错，第二条sql并未执行

```shell
2022/06/13 18:19:17 /Users/tiger/cmy-project/gocms/test/main.go:24 Error 1054: Unknown column 'order' in 'field list'
[62.527ms] [rows:0] INSERT INTO `t_we_media_region` (`order`) VALUES ('test001')

2022/06/13 18:19:17 /Users/tiger/cmy-project/gocms/test/main.go:28 Error 1054: Unknown column 'order' in 'field list'; invalid transaction
[4.774ms] [rows:0] 
Error 1054: Unknown column 'order' in 'field list'; invalid transaction

```

###### 源码分析

```golang
// DB GORM DB definition
type DB struct {
   *Config
   Error        error // 将 error 封装到DB对象中
   RowsAffected int64
   Statement    *Statement
   clone        int
}
```

Gorm 的 db 对象封装了连接配置，错误，执行语句以及 返回结果。当执行时出错则调用 `AddError` 将错误赋值给 `db.Error`。

同时，Create 在执行sql之前会判断 db对象是否存在Error,存在则不执行

```golang
func Create(config *Config) func(db *gorm.DB) {
	if config.WithReturning {
		return CreateWithReturning
	}

	return func(db *gorm.DB) {
		if db.Error != nil { //  在执行sql之前会判断 db对象是否存在Error,存在则不执行
			return
		}
    ... // 略
    if !db.DryRun && db.Error == nil {
			result, err := db.Statement.ConnPool.ExecContext(db.Statement.Context, db.Statement.SQL.String(), db.Statement.Vars...)

			if err != nil {  // 当执行时出错则调用 AddError 将错误赋值给 db.Error
				db.AddError(err)
				return
      }
    }
    ... // 略
  }
}
```

执行Create 返回的db 时通过`db.getInstance()` 拷贝的副本，因此也会包含前db对象的Error,因此当发现`if db.Error != nil`,则不向下执行。

```golang
func (db *DB) getInstance() *DB {
	if db.clone > 0 {
		tx := &DB{Config: db.Config, Error: db.Error}  // 复制原db对象的数据

		if db.clone == 1 {
			// clone with new statement
			tx.Statement = &Statement{
				DB:       tx,
				ConnPool: db.Statement.ConnPool,
				Context:  db.Statement.Context,
				Clauses:  map[string]clause.Clause{},
				Vars:     make([]interface{}, 0, 8),
			}
		} else {
			// with clone statement
			tx.Statement = db.Statement.clone()
			tx.Statement.DB = tx
		}

		return tx
	}

	return db
}
```

这种对错误的处理方式优化了原有的重复判断的逻辑，也是`Errors Are Values` 设计的一种体现。



### golang error 使用

标准库 `errors.New` 和 `fmt.Errorf` 函数用于创建实现 `error` 接口的错误对象。返回`errors.errorString`的实例：

```golang
err1 := errors.New("error")
err2 = fmt.Errorf("%s", "fmt error")
```



#### golang1.13 以前的error

此时的 `error` 只能自带一串文本,可以用于定义一些包级别的错误变量，然后在调用的时候外部包可以直接对比变量进行判定，在标准库当中大量的使用了这种方式

```golang
ErrDivByZero := errors.New("division by zero")
if err == ErrDivByZero {
	//...
}
```

如果我们想给`error`增加一些附加文本，通常如下：

```golang
newErr = fmt.Errorf("new error:%v", ErrDivByZero)
```

但是通过`fmt.Errorf`函数，是基于已经存在的`err`再生成一个新的`newErr`，我们将丢失了原来的`err`，失去这层关联。

```golang
var ErrDivByZero = errors.New("division by zero")

func div(x, y int) (int, error) {
	if y == 0 {
		return 0, ErrDivByZero
	}
	return x / y, nil
}
func main() {
	_, err := div(10, 0)
	newErr:= fmt.Errorf("div 错误：%s error", err)
	fmt.Println(err == ErrDivByZero) // true
  fmt.Println(newErr == ErrDivByZero) // false
}
```

通常只能通过字符串比较实现：

```golang
var ErrDivByZero = errors.New("division by zero")

func div(x, y int) (int, error) {
	if y == 0 {
		return 0, ErrDivByZero
	}
	return x / y, nil
}
func main() {
	_, err := div(10, 0)
	newErr:= fmt.Errorf("div 错误：%s error", err)
  fmt.Println(strings.Contains(err.Error(), ErrDivByZero.Error())) // true
	fmt.Println(strings.Contains(newErr.Error(), ErrDivByZero.Error())) // true
}
```

但这样显然造成了更高的耦合度，不利于扩展性，是个很失败的实现方式。



##### 封装

```golang
func main() {
  newErr := NewError{err, "执行出错："}
  fmt.Println(newErr.err == ErrDivByZero) // true
}

var ErrDivByZero = errors.New("division by zero")

type NewError struct {
	err error
	msg string
}

func (e *NewError) Error() string {
	return e.msg + e.err.Error() 
}
```

 我们可以通过将原有的error 封装在新的结构体下，从而保证不丢失原本error增加新的特性。在种方式对于提升error 真的很有用，是 `Errors Are Values`的实现。但显然对于追加描述的实现上显得过重了，依旧不理想.



####  Wrapping Error

golang 1.13 为我们提供了Error Wrapping，即 Error嵌套。可以实现一个 `error`嵌套另一个`error`，生成一个`error`错误跟踪链，也可以理解为错误堆栈信息，追溯根本原因在哪里。



##### fmt.Errorf扩展

golang 并没有提供什么`Wrap`函数，而是扩展了`fmt.Errorf`函数，加了一个`%w`实现 Wrapping Error。

```golang
baseError := errors.New("base error")
wrapError := fmt.Errorf("Wrap error: %w", e)
```

阅读源码

```golang
func Errorf(format string, a ...interface{}) error {
   p := newPrinter()
   p.wrapErrs = true
   p.doPrintf(format, a) // 如果使用 %w, 则 会将 传递的 error 赋值给 wrappedErr，最后实例化成wrapError
   s := string(p.buf)
   var err error
   if p.wrappedErr == nil {
      err = errors.New(s)
   } else {
      err = &wrapError{s, p.wrappedErr}
   }
   p.free()
   return err
}

type wrapError struct {
   msg string
   err error
}

func (e *wrapError) Error() string {
   return e.msg
}

func (e *wrapError) Unwrap() error { // wrapError 实现了Unwrap，用于返回被嵌套的error
   return e.err
}
```



##### Unwrap 函数

伴随着Wrapping Error，golang1.13 也引入了Unwrap机制获取被嵌套 error

```go
func main() {
	fmt.Println(errors.Unwrap(wrapError) == baseError) // true
}
```

阅读源码

```golang
func Unwrap(err error) error {
   // 类型包含一个返回错误的 Unwrap 方法,即为 Wrapping Error
   u, ok := err.(interface {
      Unwrap() error
   })
   // 否则，Unwrap 返回 nil
   if !ok {
      return nil
   }
   //  调用Wrapping Error 的 Unwrap方法返回被嵌套的error，调用一次 Unwrap 函数只能返回最外面的一层 error
   return u.Unwrap()
}
```



##### Is函数

上面所有的判断都是 `==`, 但是Error  嵌套后判断需要进行 Unwrap带来了复杂性，且针对多层嵌套需要多次 Unwrap。官方于是提供了`errors.Is`函数加以支持。

源码：

```golang
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
  // for循环，Unwrap 拆解多层嵌套
	for {
		if isComparable && err == target {
			return true
		}
    // 自定义error如果实现 Is方法，就可以实现自己的比较逻辑
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}
		// Unwrap通过 拆解一层嵌套，返回被嵌套的error
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}
```



##### As函数

在golang1.13之前没有wrapping error，把error转为特定error，通过类型断言实现，包括type assertion 或者 type switch：

```golang
// type assertion
if newErr, ok := err.(*ErrDivByZero); ok {
	...
}
//  type switch
switch newErr := err.(type) {
  case *Err1:
      ...
  case *Err2:
     ...
}
```

但倘若error 出现嵌套， 这种方式就不能用了，所以golang 为我们在`errors `包里提供了`As`函数实现类型转化。

使用：

```golang
var divByZero *ErrDivByZero
if errors.As(err, &divByZero) {
	...
}
```

源码:

```golang
func As(err error, target interface{}) bool {
	if target == nil {
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
  
  // target必须是一个非nil指针
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
  
  // target是一个接口或者实现了error接口
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
    
    //类型断言的反射写法，反射判断是否可被赋予，如果可以就赋值并且返回true
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
    // 自定义error如果实现 As方法，就可以实现自己的比较逻辑
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
			return true
		}
    // for循环，Unwrap 拆解多层嵌套
		err = Unwrap(err)
	}
	return false
}
```



#### github.com/pkg/errors

标准库的 error 是 通过 errorString 实现的，非常基础，倘若我们需要附加一些其他信息（比如堆栈信息）如何实现呢？

在errorString 中加入用于存储堆栈信息的`stack`字段，实例化error 时，将调用的堆栈信息存储在这个字段里。，

```golang
// 标准库 fmt 包 接口。实现控制状态和符文的解释方式，用于写入生成的格式化文本。并且可以调用 Sprint(f) 或 Fprint(f) 等来生成它的输出。
// %v    值的默认格式表示。当输出结构体时，扩展标志（%+v）会添加字段名
// %#v   值的Go语法表示
// %T    值的类型的Go语法表示
type Formatter interface {
	Format(f State, verb rune)
}

// 定义结构体记录程序计数器的堆栈
type stack []uintptr

// 实现 Formatter interface
func (s *stack) Format(st fmt.State, verb rune) {
	switch verb {
	case 'v':
		switch {
		case st.Flag('+'):
			for _, pc := range *s {
				f := Frame(pc)
				fmt.Fprintf(st, "\n%+v", f)
			}
		}
	}
}

// 获取当前堆栈赋值给 stack
func callers() *stack {
   const depth = 32
   var pcs [depth]uintptr
   n := runtime.Callers(3, pcs[:])
   var st stack = pcs[0:n]
   return &st
}

// 记录 stack的 error struct
type fundamental struct {
   msg string
   *stack
}

// 实现 error interface
func (f *fundamental) Error() string { return f.msg }

// 实现 Formatter interface，可以通过fmt.Printf函数输出对应的错误信息
// %s,%v 输出错误信息，不包含堆栈
// %+v  输出错误信息和堆栈
func (f *fundamental) Format(s fmt.State, verb rune) {
	switch verb {
	case 'v':
		if s.Flag('+') {
			io.WriteString(s, f.msg)
			f.stack.Format(s, verb)
			return
		}
		fallthrough
	case 's':
		io.WriteString(s, f.msg)
	case 'q':
		fmt.Fprintf(s, "%q", f.msg)
	}
}

// 通过new 返回携带 stack 的 error
func New(message string) error {
	return &fundamental{
		msg:   message,
		stack: callers(),
	}
}
```

是不是也不难，参照这种方式可以实现自定义扩展。

以上的代码其实都不是我写的，我只是将 `github.com/pkg/errors`的实现原封不动的copy了一份，它功能非常强大，因此大家可以用它来替代基础库的 error。



### web error 开发最佳实践 - gin-errors

web开发一定离不开HTTP状态码，不同的HTTP状态码通常表示的含义也不太一样，但都是通用的表达方式，并不能直接反应业务上的问题。当业务复杂后，通常需要设置错误码来标识api请求的问题，优化问题排查以及与调用方的交互。

当业务中抛出错误，自定义异常能包含该错误信息，为问题排查提供依据。

业务的错误，能够以堆栈信息保存，提高问题排查的效率。



因此我觉得一个好的自定义error需要具备以下特性：

- 包含错误码
- 报错错误的堆栈信息
- 如果是其他错误引发的，同时携带原生错误的信息



#### 自定义error实现



##### error 定义 

首先我们需要定义错误码，错误码也应该保存描述信息message，通常实现方式是通过 map 维护错误码与message的映射关系，这里我们是用 [stringer](https://pkg.go.dev/golang.org/x/tools/cmd/stringer) 实现，它可以通过注释 自动生成struct 的 String，代码和 message 写在一起，维护和查阅都更加方便。 `stringer`的使用自行查阅。

下载stringer：

```shell
go get golang.org/x/tools/cmd/stringer
```



定义错误码

```golang
type errCode int64 // 错误码

// 定义errorCo
// 执行 go generate 生成 String 方法
//go:generate stringer -type errCode -linecomment
const (
	ServerError       errCode = 10101 // 服务器内部错误
	TooManyRequests   errCode = 10102 // 请求太多
	ParamBindError    errCode = 10103 // 参数信息有误
	MySQLNoQueryError errCode = 20505 // 查询数据库无结果
	MySQLNoFieldError errCode = 20506 // 查询数据库字段不存在
)	
```

通过`stringer`生成 `String` 函数

```golang
// Code generated by "stringer -type errCode -linecomment"; DO NOT EDIT.

package errors

import "strconv"

func _() {
	// An "invalid array index" compiler error signifies that the constant values have changed.
	// Re-run the stringer command to generate them again.
	var x [1]struct{}
	_ = x[ServerError-10101]
	_ = x[TooManyRequests-10102]
	_ = x[ParamBindError-10103]
	_ = x[MySQLNoQueryError-20505]
	_ = x[MySQLNoFieldError-20506]
}

const (
	_errCode_name_0 = "服务器内部错误请求太多参数信息有误"
	_errCode_name_1 = "查询数据库无结果查询数据库字段不存在"
)

var (
	_errCode_index_0 = [...]uint8{0, 21, 33, 51}
	_errCode_index_1 = [...]uint8{0, 24, 54}
)

func (i errCode) String() string {
	switch {
	case 10101 <= i && i <= 10103:
		i -= 10101
		return _errCode_name_0[_errCode_index_0[i]:_errCode_index_0[i+1]]
	case 20505 <= i && i <= 20506:
		i -= 20505
		return _errCode_name_1[_errCode_index_1[i]:_errCode_index_1[i+1]]
	default:
		return "errCode(" + strconv.FormatInt(int64(i), 10) + ")"
	}
}

```

定义自定义error , 其包含了errCode 和 原本的 error，这样我们就拥有了错误码，并且还携带原生错误信息

```golang
type customError struct {
	Code errCode `json:"code"` // 业务码
	Err  error
}

func (e *customError) Error() string {
	errMsg := ""
	if e.Err != nil {
		errMsg = e.Err.Error()
	}
	return fmt.Sprintf("code: %d, message: %s, error: %s ", e.Code, e.Code.String(), errMsg)
}
```

当然我们还需要得到错误的堆栈信息，这时`github.com/pkg/errors`便派上用场了

```golang
func (i errCode) WrapWithMessage(err error, message string) error { // 为 customError 添加 stack 并附加信息
	return errors.Wrap(&customError{
		Code: i, Err: err,
	}, message)
}

func (i errCode) Wrap(err error) error { // 为customError 添加 stack
	return errors.Wrap(&customError{
		Code: i, Err: err,
	}, "")
}
```

由此我们的自定义error 则基本实现



##### response定义

接下来对接口的返回格式进行定义

```golang
type Response struct {
	Data      interface{} `json:"data,omitempty"`     // 返回结果
	ErrMsg    string      `json:"err_msg,omitempty"`  // 错误信息
	ErrCode   errCode     `json:"err_code,omitempty"` // 错误状态码
	IsSuccess bool        `json:"is_success"`         // 请求结果
}

const (
	Fail    = false
	SUCCESS = true
)

// Result 通用返回格式
func Result(httpCode int, data interface{}, errCode errCode, ErrMsg string, isSuccess bool, c *gin.Context) {
	c.JSON(httpCode, Response{
		data,
		ErrMsg,
		errCode,
		isSuccess,
	})
}

// ResultWithErrCode 通用返回格式  携带errCode
func ResultWithErrCode(httpCode int, data interface{}, errCode errCode, isSuccess bool, c *gin.Context) {
	Result(httpCode, data, errCode, errCode.String(), isSuccess, c)
}

// ParamBindErrorResult 参数绑定异常返回
func ParamBindErrorResult(ErrMsg string, c *gin.Context) {
	c.JSON(http.StatusBadRequest, Response{
		ErrMsg:  ErrMsg,
		ErrCode: ParamBindError,
	})
}

func ResultOk(data interface{}, c *gin.Context) {
	c.JSON(http.StatusOK, Response{
		Data:      data,
		IsSuccess: SUCCESS,
	})
}

func ResultFail(httpCode int, errCode errCode, c *gin.Context) {
	c.JSON(httpCode, Response{
		ErrMsg:    errCode.String(),
		ErrCode:   errCode,
		IsSuccess: Fail,
	})
}
```



##### error 统一处理

在golang 中，函数同python一样被视为一等公民，因此我们可以将错误处理进行统一的封装

```golang
func IsValidationError(err error) bool {
	_, ok := err.(validator.ValidationErrors)
	return ok
}

func ErrWrapper(handler func(g *gin.Context) (interface{}, error)) func(*gin.Context) {
	return func(context *gin.Context) {
		data, err := handler(context)
		if err != nil {
			var customErr = new(customError)

			if errors2.As(err, &customErr) {
				ResultFail(http.StatusBadRequest, customErr.Code, context)
			} else if IsValidationError(err) {
				ParamBindErrorResult(err.Error(), context)
			} else {
				ResultFail(http.StatusInternalServerError, ServerError, context)
			}
			// 自行更改成特定的log
			log.Printf("url: %s ,erorr: %+v", context.Request.URL, err)

		} else {
			ResultOk(data, context)
		}

	}
}
```

##### demo

我们可以将自定义error 应用于gin 框架中，这里提供一份我写的demo，仅供参考： [gin-errors-demo](https://github.com/Darr-en1/gin-errors/tree/main/demo)



##### tips

最近，golang  社区有发起了一个新提案 

![1460000041888769](/images/go-error-最佳实践/1460000041888769.png)

很多时候我们在收到错误信息时，返回堆栈并提供其他信息（例如：业务状态码）时，就只能向我们前面定义的`customError `那样将原生error 包含进去。

新提案是希望在标准库 errors 中实现一个更简单的函数来达到这种效果，支持将任何错误与任何其他错误包装在一起，从而使它们形成一个新的包装错误列表。

如下代码

```go
// With returns an error that wraps err with other.  
func With(err, other error) error
```

这个被包裹起来的错误类似于链表，可以复用 `errors.Unwrap` 来遍历列表。而类链表存储，就有先后顺序的问题。

在 `With` 函数中，`other` 参数的错误将会放在包装错误列表的头部。如果在调用 `With` 函数时是 `With(b->a, d->c)`，呈现在内的错误列表是：d->c->b->a。



errors.With(err, other).Error()： other.Error() + ": " + err.Error()



如果在不久的将来，这个提案真进入了golang 的原生库，我们的自定义错误实现就更简单了，它让error实现只需要考虑自身的逻辑即可，以下为具体实现

```golang
type errCode int64

func (i *errCode) Error() string {
	return fmt.Sprintf("code: %d, message: %s ", e, e.String())
}

func (i errCode) Wrap(err error) error { 
	return errors.Wrap(errors.With(i, err))
}
```



以上就是我对 go error 的理解，希望对大家能有启发



参考资料

[Go Error 的设计哲学](https://chasecs.github.io/posts/the-philosophy-of-go-error-handling/)

[如何优雅的 Golang 错误处理[自定义异常]](https://learnku.com/go/t/33210)

[Go错误处理最佳实践](https://lailin.xyz/post/go-training-03.html)

[Go语言(golang)的错误(error)处理的推荐方案](https://www.flysnow.org/2019/01/01/golang-error-handle-suggestion.html)

[Go语言(golang)新发布的1.13中的Error Wrapping深度分析]( https://www.flysnow.org/2019/09/06/go1.13-error-wrapping.html)

[骚爆了... Go 错误处理中再套个娃，能解决烦恼不？](https://segmentfault.com/a/1190000041888767)

[最佳实践之Golang错误处理](https://segmentfault.com/a/1190000040917186)
