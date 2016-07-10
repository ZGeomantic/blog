---
layout: post
title:  "golang常见误区：如何解析json中的null字段"
date:   2016-07-10 20:57:15 +0800
categories: my article
---

不论是golang的官方文档，还是各种技术博客，在讲到json解析的时候，一般举例都很浅显，定义一个结构体，然后Marshal或者Unmarshal就可以，其实中间有一个容易忽略的问题。

比如在编写api的项目中，经常会遇到这样一个需求：

前后端的通过api通信的时候，通过定义一系列的结构体来解析。有些接口的功能很简单，协议只是为了对数据库进行增删改查，但是具体查的是哪些字段，希望可以通过结构体有无赋值自动查询出来，这样在使用orm进行数据库操作的时候可以省下很多的代码，比如用beego的orm进行update操作的时候，只要传一个指向table的结构体和需要更新的一组column的名称就可以。
这个功能看起来很简单，其实有几个问题需要解决：

- 如何判断json对象的某个字段值是否为空？
- 如何通过reflect包反射出结构体的所有成员变量（包括存在结构体嵌套关系的时候）

下面会按照这两个大的问题，去一步步的完善代码，实现最终的功能。

### 1. 如何区分json中的null

由于golang自带的json包在处理json中的NULL字段值时，会把此字段解析成默认值。比如```{"age":NULL,"name":NULL}```这样的一段json解析到结构体里，会变成```{age:0,name:""}```，无法区分是不是空值。很多时候这样是无法满足逻辑判断的，因为比如我不知道某次请求中，对方到底是没有传age，还是传来的age就是等于0.

如果是一个新系统从头设计，我们当然可以避免这些默认值出现在协议里，但这中理想的情况第一我们不太可能遇到，第二这样的设计本身就不合理。
还有一种更好的解决方法，就是把所有的字段值设为指针类型，如：

```go
type Human struct {
	Age  *int
	Name *string
}
```

此时就可以把null和默认值区分开来了，比如解析json的时候：


```go
// output:
// Name is nil: true
// Age: 10
func PtrUnmarshal() {
	str := `{"Age":10, "Name":null}`
	var human Human
	json.Unmarshal([]byte(str), &human)

	fmt.Printf("Name is nil: %v\n", human.Name == nil)
	fmt.Printf("Age: %d\n", *human.Age)
}
```

此时看起来好像完美了，但是会有两个影响代码可读性的"后果"。第一个是在于给结构赋值的时候，比如想要给此结构体赋值，就要先声明，再取引用了。

> 注意： 这里绝对不能在构造struct的的全部用new初始化，因为一旦初始化就就会赋默认值，又回到之前的无法区分nil和默认值的问题。

```go
func PtrMarshal() {
	var human Human
	// 错误1，直接这样赋值是错误的，因为Age是个指针类型，并且没有被初始化，
	// *human.Age = 10

	// 正解1，先初始化再赋值，这样就是正确的
	// human.Age = new(int)
	// *human.Age=10

	// 正解2，，原理还是初始化，这个也对，但是啰嗦
	// var age *int = new(int)
	// *age = 100
	// human.Age = age

	// 正解3，这是最便捷的做法
	var age int = 101
	human.Age = &age
	
	b, e := json.Marshal(human)
	fmt.Printf("body: %s, err: %s\n", b, e)
}
```

还有一个后果，就是在使用结构体的时候，当我们decode一个结构体出来之后，每一次使用它的值的时候都要小心翼翼，做一个是否是nil的判断，否则要抛空指针异常。

所以最佳实践是: 对nil不敏感的字段，直接用普通类型的变量，对于nil敏感的字段，用指针类型。不要试图统一，混杂的结构体也挺不错。 对于赋值的麻烦，可以通过封装一些的optional函数来实现，如：

```go
func OptionalString(input *string) string {
	if input == nil {
		return ""
	} else {
		return *input
	}
}

func OptionalInt(input *int) int {
	if input == nil {
		return 0
	} else {
		return *input
	}
}
```
这样写虽然有点low（更好的方法是用reflect.Zero函数，具体实现可见[github.com/ZGeomantic/go-optional](https://github.com/ZGeomantic/go-optional)包）。但是因为json中常用的也是就string和int，所以这个汤可以满足基本的功能需求。

### 2. 从结构体中挑选出所有的非空字段

#### 2.1 第一版的代码很简单，只挑选出结构体中的变量类型是指针，而且指针指向nil的字段：

```
// 内省出类型和值
func InspectOf(ptrStruct interface{}) (valueS reflect.Value, typeS reflect.Type) {

	valueS = reflect.ValueOf(ptrStruct)
	for valueS.Kind() == reflect.Ptr {
		valueS = valueS.Elem()
	}
	typeS = valueS.Type()

	if typeS.Kind() != reflect.Struct {
		panic("Argument is not a pointer to an struct")
	}

	return
}

func GetNonNil(src interface{}) []string {

	var columns []string

	v, t := InspectOf(src)
	//	标签是在type中的，与value无关
	for i := 0; i < t.NumField(); i++ {
		valueField := v.Field(i)
		typeField := t.Field(i)

		switch valueField.Kind() {
		case reflect.Ptr:
			// 如果是指针类型的结构体，需要判断是否是nil，
			if !valueField.IsNil() {
				columns = append(columns, typeField.Name)
			}
		}
	}
	return columns
}
```

#### 2.2 对于嵌套的结构体，reflect是无法展开嵌套的结构体的

上的代码有很大的局限。第一，只能判断指针类型的成员变量，第二，对有嵌套关系的结构体是无法获得深层的结构体关系的。在改进代码之前，我们先了解一下当reflect遇到有嵌套关系的结构体时会发生什么。

例如于下面的结构体，Student内嵌了一个General。当我们对Student调用反射的时候，会获得的成员变量是School、General，而不是School、Name、Age。其实这个结果很合理，General只是一个匿名成员变量，并不是把所有成员变量拷贝到Student中。

```go
type General struct {
	Name string
	Age  int
}

type Student struct {
	School  string
	General
}

// output：
//  type: School
//  type: General
func ShowInside() {
	var std Student

	v := reflect.ValueOf(std)
	t := v.Type()

	for i := 0; i < t.NumField(); i++ {
		valueField := v.Field(i)
		_ = valueField
		typeField := t.Field(i)
		fmt.Printf("type: %s\n", typeField.Name)
	}

}
```

这个特性并不影响json的解析，因为从json上看，Student仍然是一个平面的结构。General就好像并不存在一样，不存在与序列化后的字符串中。

```go
// output:
// {"School":"MIT","Name":"jack","Age":10}
func Json() {
	var std Student
	std.Name = "jack"
	std.Age = 10
	std.School = "MIT"
	
	b, _ := json.Marshal(std)
	fmt.Printf("%s\n", b)
}
```

于是继续改进代码，考虑到结构体之间的嵌套，改进了第二版代码：


```
func GetNonNil(src interface{}) []string {

	var columns []string

	v, t := InspectOf(src)
	//	标签是在type中的，与value无关
	for i := 0; i < t.NumField(); i++ {
		valueField := v.Field(i)
		typeField := t.Field(i)

		switch valueField.Kind() {
		case reflect.Ptr:
			// 如果是指针类型的结构体，需要判断是否是nil，
			if !valueField.IsNil() {
				columns = append(columns, typeField.Name)
			}
		case reflect.Struct:
			columns = append(columns, GetNonNil(valueField.Interface())...)
		}
	}
	return columns
}
```

#### 2.3 全类型、可定义字段的最终版

如果除了指针类型外，还要适配所有变量类型，既要int，bool这样的普通类型的变量，也要包括自定义结构体类型的变量。并且还要考虑到一个问题，就是结构体中也许有些字段并不需要参与到这个逻辑中，应该用一个tag来标记哪些字段需要参与到筛选逻辑中，否则这个结构体的定义就只能局限于一个映射的功能，不能添加仍和字段了，这显然是不合理的。所以改进到第三版，选出结构体中带有reflect标签，并且值为nonil的非空字段：

```go
type Human struct {
	Age    *int
	Name   *string `reflect:"nonil"`
	Gender int
	Alien  `reflect:"nonil"`
}

type Alien struct {
	Planet   *string `reflect:"nonil"`
	Dimesion *int
	Level    string
}

func GetNonNil(src interface{}) []string {

	var columns []string

	v, t := InspectOf(src)
	//	标签是在type中的，与value无关
	for i := 0; i < t.NumField(); i++ {
		valueField := v.Field(i)
		typeField := t.Field(i)

		if typeField.Tag.Get("reflect") != "nonil" {
			continue
		}

		switch valueField.Kind() {

		case reflect.Struct:
			// 如果是结构体，并且是嵌套的结构体，递归
			columns = append(columns, GetNonNil(valueField.Interface())...)
		default:
			if valueField.Interface() != reflect.Zero(typeField.Type).Interface() {
				columns = append(columns, typeField.Name)
			}
		}
	}
	return columns
}
```

至此，已经实现本文开头提出的需求。其实最本质的问题，是如何完整的反射一个复杂结构体所有字段，以及平时用json解析时容易走的误区。
