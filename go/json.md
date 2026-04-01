# Json
{docsify-updated}


+ json.MarshalIndent
+ json.Marshal
+ json.Unmarshal

```
type Movie struct {
    Title  string
    Year   int  `json:"released"`
    Color  bool `json:"color,omitempty"`
    Actors []string
}

var movies = []Movie{
    {Title: "Casablanca", Year: 1942, Color: false,
        Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
    {Title: "Cool Hand Luke", Year: 1967, Color: true,
        Actors: []string{"Paul Newman"}},
    {Title: "Bullitt", Year: 1968, Color: true,
        Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
    // ...
}
```

这样的数据结构特别适合JSON格式，并且在两者之间相互转换也很容易。将一个Go语言中类似movies的结构体slice转为JSON的过程叫编组（marshaling）。编组通过调用json.Marshal函数完成：
```go
data, err := json.Marshal(movies)
if err != nil {
    log.Fatalf("JSON marshaling failed: %s", err)
}
fmt.Printf("%s\n", data)
data, _= json.MarshalIndent(movies, "", "    ")
fmt.Printf("%s\n", data)
```

一个结构体成员Tag是和在编译阶段关联到该成员的元信息字符串：
```go
Year  int  `json:"released"`
Color bool `json:"color,omitempty"`
```
结构体的成员Tag可以是任意的字符串面值，但是通常是一系列用空格分隔的key:"value"键值对序列；因为值中含有双引号字符，因此成员Tag一般用原生字符串面值的形式书写。json开头键名对应的值用于控制encoding/json包的编码和解码的行为，并且encoding/...下面其它的包也遵循这个约定。成员Tag中json对应值的第一部分用于指定JSON对象的名字，比如将Go语言中的TotalCount成员对应到JSON中的total_count对象。Color成员的Tag还带了一个额外的omitempty选项，表示当Go语言结构体成员为空或零值时不生成该JSON对象（这里false为零值）。果然，Casablanca是一个黑白电影，并没有输出Color成员。


```
var titles []struct{ Title string }
if err := json.Unmarshal(data, &titles); err != nil {
    log.Fatalf("JSON unmarshaling failed: %s", err)
}
fmt.Println(titles) // "[{Casablanca} {Cool Hand Luke} {Bullitt}]"
```

## 精度丢失问题
> https://www.51cto.com/article/697019.html

当Json 字符串:
```
{"dictCode":1554660410667769858,"dictLabel":"美容/保健服务"}
```
反序列化为 `map[string]interface{}` 类型时， 1554660410667769858 会变成 1554660410667769900 。  
测试代码:
```
func main() {
	var result map[string]interface{}
	body := "{\"dictCode\":1554660410667769858,\"dictLabel\":\"美容/保健服务\"}"
	bytes := []byte(body)
	json.Unmarshal(bytes, &result)
	fmt.Println(result)
	var a float64 = 1.5546604106677699e+18
	fmt.Printf("%18.f", a)
}
```