# go服务get请求url带";"号时报关于semicolon的警告问题

## 问题描述

这几天工作项目遇到了一个算不上bug的问题，在花了些时间解决后打算记录一下，也好提醒自己继续学习。

> [ERROR] http: URL query contains semicolon, which is no longer a supported separator; parts of the query may be stripped when parsed; see golang.org/issue/25192

错误日志输出如上，下面简单概括一下我写的那个接口的逻辑：

后端监听了一个/api/test/picture/get接口，用于前端请求获取图片，但是这个图片并不在后端的服务器，而是需要另外调一个第三方接口获取（而且还需要签名等验证），于是只能在前端请求图片的时候由handle方法代为解析参数转发。

举个例子，假设前端要请求某张图片A，请求url:

> /api/test/picture/get?picUri=xxx&&indexCode=bbb

一般handle方法GetPicture()解析picUri跟indexCode并另外构建请求转发到第三方接口，将接口返回直接发送到前端就没什么事了。问题就出在picUri参数中，这个"xxx"中通常包含了";"号，如果不做特殊处理的话在处理完最后就会在日志里面输出一行上面的警告。

### 下面是有问题的main.go

```go
package main

import (
	"encoding/json"
	"net/http"
	"strings"

	"github.com/gogf/gf/v2/frame/g"
	"github.com/gogf/gf/v2/net/ghttp"
)

func main() {
	s := g.Server()

	h := http.Handler(http.HandlerFunc(GetPicture))
	// 允许url请求带";"号（海康提供的picUri参数通常带有分号 不处理会报log 请求图片算是比较频繁的接口会影响日志） https://github.com/golang/go/issues/25192
	// 这里如果不使用http.AllowQuerySemicolons()方法就会在后台打印警告
	// h = http.AllowQuerySemicolons(h) // AllowQuerySemicolons()方法处理后';'被替换为了'&' 如果把替换结果传给海康接口会报错 需要在handler中再把';'还原回来

	// WrapH()将标准库的http.Handler转换为ghttp.Server支持的服务方法类型
	s.BindHandler("/api/test/picture/get", ghttp.WrapH(h))

	s.SetPort(8899)
	s.Run()
}

func GetPicture(w http.ResponseWriter, r *http.Request) {
	// 结构体定义在此文章最后的代码块里
	h := ThirdPlateform{}

	// r.URL.Query()结果不是期望的 这里手动解析两个参数
	splitServerIndexCode := strings.Split(r.URL.RawQuery, "&serverIndexCode=")
	if len(splitServerIndexCode) != 2 { // 对切割结果做长度判断 避免panic
		_, _ = w.Write([]byte("there is no have serverIndexCode"))
		return
	}
	serverIndexCode := splitServerIndexCode[1]
	splitPicUri := strings.Split(splitServerIndexCode[0], "picUri=")
	if len(splitPicUri) != 2 {
		_, _ = w.Write([]byte("there is no have picUri"))
		return
	}
	// 直接替换可能会导致http.AllowQuerySemicolons(h)中本来就是'&'的地方也换成了';'
	// picUri := strings.ReplaceAll(splitPicUri[1], "&", ";")

	// 用解析的两个参数构建请求传参
	req := map[string]string{
		"picUri":          splitPicUri[1], // picUri, // 如果main中启用http.AllowQuerySemicolons()那一行则这里用picUri
		"serverIndexCode": serverIndexCode,
	}
	reqBytes, _ := json.Marshal(req)

	// 第三方获取图片接口
	resp, err := h.PostUrl("/api/resource/v1/person/picture", 3, string(reqBytes))
	if err != nil {
		// /pic?B400505D0F604306667E*hcs1167ac4380844ebe92022/222/acs;167278985450192214169?pic*18416214*91570*484*B400505D0F604306667E-2*1672792246
		_, _ = w.Write([]byte("get img failed: " + err.Error()))
		return
	}
	// 直接将post返回结果写回响应即可
	_, _ = w.Write(resp)
}
```

## 问题分析与解决

查了一些资料发现是go1.17之后的版本报的问题`src/net/http/server.go 2911行 - go1.18.8 windows/amd64`，具体是1.17因为潜在安全问题修复了url传参识别解析方式，不再将分号作为分隔标识符号。

本来一行警告而已，也不是大问题，不打算管的，但是最近测试的时候发现那个获取图片接口调用频率还是不低的，日志又关不掉很影响体验，所以才决定要处理这个问题。

go issue下不少人也有同样的困扰，最后顺着https://tip.golang.org/doc/go1.17 提供的AllowQuerySemicolons方法开始确实解决了问题，不再报警告日志。

歪个题，注意AllowQuerySemicolons()的写法，不是简单调用一遍即可，它是有返回结果的，使用例如下

```go
func main(){
	// ...
	// 正确例
	h = AllowQuerySemicolons(h)

	// 错误例
	AllowQuerySemicolons(h)
	//...
}
```

这个地方使用例是看官方test文件`src/net/http/serve_test.go:6641`的。其实官方例子就是最好的文章了，只是一般经常被忽略（至少我是这样）

回到正题，对应上面代码只需要将`h = http.AllowQuerySemicolons(h)`与`picUri := strings.ReplaceAll(splitPicUri[1], "&", ";")`取消注释，并把req传参的splitPicUri[1]改为picUri即可解决这个报错问题

但是考虑到代码44行处使用`strings.ReplaceAll(splitPicUri[1], "&", ";")`无脑将'&'替换';'后期可能会出现问题。

例如假设原先url中传参`picUri=123&456;789`，经过AllowQuerySemicolons()处理后`picUri=123&456&789`，再使用ReplaceAll()就会变成`picUri=123;456;789`，请求到第三方接口那边对不上自然就会报错。

## 问题拓展（context）

于是又花了一些时间研究——如何在不报警告日志的情况下正确地解析还原URL请求的分号的位置，最后想到了context包。一开始打算越过http.AllowQuerySemicolons，先用正则匹配查找分号位置，将位置信息记录到context中，最后自行将分号替换，接口确实正确还原了mark的分号位置，但是依然报日志。

```go
func main() {
	s := g.Server()

	h := http.Handler(http.HandlerFunc(GetPicture))
	// 这个方法基本copy自go官方的http.AllowQuerySemicolons 中间正则是增加的逻辑 因为第三方接口如果用'&'替换后的参数传会报错
	markQuerySemicolons := func(h http.Handler) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			if strings.Contains(r.URL.RawQuery, ";") {
				r2 := new(http.Request)
				*r2 = *r
				r2.URL = new(url.URL)
				*r2.URL = *r.URL

				// 正则匹配 将原url中的';'索引位置标记放到context中
				semicolonsIndexes := regexp.MustCompile(";").FindAllStringIndex(r.URL.RawQuery, -1)
				// 索引信息放到context中 在具体处理方法GetHKPicture时再根据context中存的下标还原';'的位置
				marks := context.WithValue(r2.Context(), "semicolonMarks", semicolonsIndexes)
				r2 = r2.WithContext(marks)

				r2.URL.RawQuery = strings.ReplaceAll(r.URL.RawQuery, ";", "&")
				h.ServeHTTP(w, r2)
			} else {
				h.ServeHTTP(w, r)
			}
		}
	}
	// 允许url请求带";"号（海康提供的picUri参数通常带有分号 不处理会报log 请求图片算是比较频繁的接口会影响日志） https://github.com/golang/go/issues/25192
	h = markQuerySemicolons(h) // 越过http.AllowQuerySemicolons()自行处理
	//h = http.AllowQuerySemicolons(h) // AllowQuerySemicolons()方法处理后';'被替换为了'&' 如果把替换结果传给海康接口会报错 需要在handler中再把';'还原回来

	// WrapH()将标准库的http.Handler转换为ghttp.Server支持的服务方法类型
	s.BindHandler("/api/test/picture/get", ghttp.WrapH(h))

	s.SetPort(8899)
	s.Run()
}

func GetPicture(w http.ResponseWriter, r *http.Request) {
	// 结构体定义在此文章最后的代码块里
	h := ThirdPlatform{}

	// url经过http.AllowQuerySemicolons()后分号都被替换为'&' 这里需要根据ctx存的索引将替换的位置还原为';'
	urlReduceSemicolons := bytes.Runes([]byte(r.URL.RawQuery))
	// 根据context将标记位置的';'还原
	marksContextV := r.Context().Value("semicolonMarks")
	semicolonsIndexes, ok := marksContextV.([][]int)
	if ok {
		for _, index := range semicolonsIndexes {
			if len(index) != 0 {
				urlReduceSemicolons[index[0]] = ';'
			}
		}
	}

	// r.URL.Query()结果不是期望的 这里手动解析两个参数
	splitServerIndexCode := strings.Split(string(urlReduceSemicolons), "&serverIndexCode=")
	if len(splitServerIndexCode) != 2 { // 对切割结果做长度判断 避免panic
		_, _ = w.Write([]byte("there is no have serverIndexCode"))
		return
	}
	serverIndexCode := splitServerIndexCode[1]
	splitPicUri := strings.Split(splitServerIndexCode[0], "picUri=")
	if len(splitPicUri) != 2 {
		_, _ = w.Write([]byte("there is no have picUri"))
		return
	}
	// 直接替换可能会导致http.AllowQuerySemicolons(h)中本来就是'&'的地方也换成了';'
	//picUri := strings.ReplaceAll(splitPicUri[1], "&", ";")
	picUri := splitPicUri[1]

	// 用解析的两个参数构建请求传参
	req := map[string]string{
		"picUri":          picUri,
		"serverIndexCode": serverIndexCode,
	}
	reqBytes, _ := json.Marshal(req)

	// 第三方获取图片接口
	resp, err := h.PostUrl("/api/resource/v1/person/picture", 3, string(reqBytes))
	if err != nil {
		// /pic?B400505D0F604306667E*hcs1167ac4380844ebe92022/222/acs;167278985450192214169?pic*18416214*91570*484*B400505D0F604306667E-2*1672792246
		_, _ = w.Write([]byte("get img failed: " + err.Error()))
		return
	}
	// 直接将post返回结果写回响应即可
	_, _ = w.Write(resp)
}
```

如果把main中的AllowQuerySemicolons()那行取消注释，虽然不会报警告，但是GetPicture方法中context又不能正常解析还原分号位置了。

后面注释AllowQuerySemicolons()情况下排查，发现自行魔改的markQuerySemicolons方法缺失了如下代码

```go
// silenceSemWarnContextKey是http包一个私有变量
var silenceSemWarnContextKey = &contextKey{"silence-semicolons"}

// src/net/http/server.go
func AllowQuerySemicolons(h Handler) Handler {
	return HandlerFunc(func(w ResponseWriter, r *Request) {
		if silenceSemicolonsWarning, ok := r.Context().Value(silenceSemWarnContextKey).(func()); ok {
				silenceSemicolonsWarning()
		}
		// ...
}
```

下面两段是对http的ServeHTTP方法的解读比较【重要】

就是关键的silenceSemWarnContextKey变量无法自行存到context中于是才会依然报日志。于是看http包的实现，如果发现url请求参数携带了分号，ServeHTTP时会为请求context套一个silenceSemWarnContextKey，value是一个方法（该方法执行时将allowQuerySemicolonsInUse置为1,），并注册一个defer方法，如果ServeHTTP结束时allowQuerySemicolonsInUse值检查为0就报警告日志。

如果执行AllowQuerySemicolons则会将silenceSemWarnContextKey对应的value方法取出并执行（也就是将allowQuerySemicolonsInUse置1），这样ServeHTTP结束时检查allowQuerySemicolonsInUse值为1就不会报日志，这个设计真的妙啊，我解释地并不是很好，可能需要多看几遍才能理解（还是建议有条件的自行跑一下测试代码，会有更深的理解）

```go
// src/net/http/server.go:2895 go1.18.8 windows/amd64
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}

	if req.URL != nil && strings.Contains(req.URL.RawQuery, ";") {
		var allowQuerySemicolonsInUse int32
		req = req.WithContext(context.WithValue(req.Context(), silenceSemWarnContextKey, func() {
			atomic.StoreInt32(&allowQuerySemicolonsInUse, 1)
		}))
		defer func() {
			if atomic.LoadInt32(&allowQuerySemicolonsInUse) == 0 {
				sh.srv.logf("http: URL query contains semicolon, which is no longer a supported separator; parts of the query may be stripped when parsed; see golang.org/issue/25192")
			}
		}()
	}

	handler.ServeHTTP(rw, req)
}
```

让我觉得go妙的地方还不止上面。虽然到这里我理解了这个告警日志如何打印出来，但是依然需要面对如何解析分号位置的问题，因为在AllowQuerySemicolons执行完后所有的分号都会被替换掉，我自然不可能去修改go的源码，于是继续修改markQuerySemicolons方法。

本来想着如果在AllowQuerySemicolons之前使用markQuerySemicolons提前处理好存到context中即可，但是无论我怎么写最后依然报日志，甚至markQuerySemicolons中的

```go
if strings.Contains(r.URL.RawQuery, ";") {}
```

都没有执行，我想了很久都没搞明白，明明代码顺序应该是先执行markQuerySemicolons再执行的AllowQuerySemicolons，最后在我差点放弃准备使用strings.ReplaceAll的时候做了最后一次尝试，将markQuerySemicolons与AllowQuerySemicolons的执行顺序对调（~~猪脑子现在才想到~~），就是这个对调终于成功处理了整个流程，但是为啥我依然云里雾里。

改进后的实现main.go

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"net/http"
	"net/url"
	"regexp"
	"strings"

	"github.com/gogf/gf/v2/frame/g"
	"github.com/gogf/gf/v2/net/ghttp"
)

func main() {
	s := g.Server()

	h := http.Handler(http.HandlerFunc(GetPicture))
	// 这个方法基本copy自go官方的http.AllowQuerySemicolons 中间正则是增加的逻辑 因为第三方接口如果用'&'替换后的参数传会报错
	markQuerySemicolons := func(h http.Handler) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			if strings.Contains(r.URL.RawQuery, ";") {
				r2 := new(http.Request)
				*r2 = *r
				r2.URL = new(url.URL)
				*r2.URL = *r.URL

				// 正则匹配 将原url中的';'索引位置标记放到context中
				semicolonsIndexes := regexp.MustCompile(";").FindAllStringIndex(r.URL.RawQuery, -1)
				// 索引信息放到context中 在具体处理方法GetHKPicture时再根据context中存的下标还原';'的位置
				marks := context.WithValue(r2.Context(), "semicolonMarks", semicolonsIndexes)
				r2 = r2.WithContext(marks)

				r2.URL.RawQuery = strings.ReplaceAll(r.URL.RawQuery, ";", "&")
				h.ServeHTTP(w, r2)
			} else {
				h.ServeHTTP(w, r)
			}
		}
	}
	// 允许url请求带";"号（海康提供的picUri参数通常带有分号 不处理会报log 请求图片算是比较频繁的接口会影响日志） https://github.com/golang/go/issues/25192
	h = markQuerySemicolons(http.AllowQuerySemicolons(h)) // AllowQuerySemicolons()方法处理后';'被替换为了'&' 如果把替换结果传给海康接口会报错 需要在handler中再把';'还原回来

	// WrapH()将标准库的http.Handler转换为ghttp.Server支持的服务方法类型
	s.BindHandler("/api/test/picture/get", ghttp.WrapH(h))

	s.SetPort(8899)
	s.Run()
}

func GetPicture(w http.ResponseWriter, r *http.Request) {
	h := ThirdPlatform{}

	// url经过http.AllowQuerySemicolons()后分号都被替换为'&' 这里需要根据ctx存的索引将替换的位置还原为';'
	urlReduceSemicolons := bytes.Runes([]byte(r.URL.RawQuery))
	// 将标记位置的';'还原
	marksContextV := r.Context().Value("semicolonMarks")
	semicolonsIndexes, ok := marksContextV.([][]int)
	if ok {
		for _, index := range semicolonsIndexes {
			if len(index) != 0 {
				urlReduceSemicolons[index[0]] = ';'
			}
		}
	}

	// r.URL.Query()结果不是期望的 这里手动解析两个参数
	splitServerIndexCode := strings.Split(string(urlReduceSemicolons), "&serverIndexCode=")
	if len(splitServerIndexCode) != 2 { // 对切割结果做长度判断 避免panic
		_, _ = w.Write([]byte("there is no have serverIndexCode"))
		return
	}
	serverIndexCode := splitServerIndexCode[1]
	splitPicUri := strings.Split(splitServerIndexCode[0], "picUri=")
	if len(splitPicUri) != 2 {
		_, _ = w.Write([]byte("there is no have picUri"))
		return
	}
	// 直接替换可能会导致http.AllowQuerySemicolons(h)中本来就是'&'的地方也换成了';'
	//picUri := strings.ReplaceAll(splitPicUri[1], "&", ";")
	picUri := splitPicUri[1] // 经过http.AllowQuerySemicolons()与context的组合这里已经没有问题

	// 用解析的两个参数构建请求传参
	req := map[string]string{
		"picUri":          picUri,
		"serverIndexCode": serverIndexCode,
	}
	reqBytes, _ := json.Marshal(req)

	// 第三方获取图片接口
	resp, err := h.PostUrl("/api/resource/v1/person/picture", 120, string(reqBytes))
	if err != nil {
		// /pic?B400505D0F604306667E*hcs1167ac4380844ebe92022/222/acs;167278985450192214169?pic*18416214*91570*484*B400505D0F604306667E-2*1672792246
		_, _ = w.Write([]byte("get img failed: " + err.Error()))
		return
	}
	// 直接将post返回结果写回响应即可
	_, _ = w.Write(resp)
}
```

如果不深究为什么对调后一切都通了的话，文章看到这里就已经足够解决问题了。

## 问题总结

1. 转发接口写完，发现调用的时候传参中存在";"时执行完后台会打印一行警告日志，查了官方文档发现可以使用http.AllowQuerySemicolons解决
2. 使用http.AllowQuerySemicolons后，发现该方法只是使用strings.ReplaceAll将url中所有的";"替换成了"&"，而且无法还原，所以记录";"替换前的位置
3. 自行仿照AllowQuerySemicolons写了markQuerySemicolons方法，使用context上下文记录ReplaceAll前的";"位置，并在handle处理时根据context还原，发现context处理正常但是依然打印警告
4. markQuerySemicolons与AllowQuerySemicolons组合（注意两个调用顺序）使用最终正确处理

## 问题再拓展

但是我一定要搞懂对调的逻辑，研究下来才发现虽然代码顺序上确实是markQuerySemicolons先执行，但是实际上该方法返回结果是一个方法，既然是方法，就只有调用时才会执行，如果把这个方法再丢到AllowQuerySemicolons中处理，因为前面的方法在AllowQuerySemicolons中依然不执行，又返回了一个方法，说可能不是很清楚，直接上代码看运行结果就能理解了。

```go
package main

import (
	"fmt"
)

// 简化官方AllowQuerySemicolons
func official(f func(i int)) func(int) {
	fmt.Println("official -- external")
	return func(i int) {
		fmt.Println("official -- internal")
		f(i)
	}
}

// 简化业务逻辑GetPicture
func mock(i int) {
	fmt.Println("mock run", i)
}

func main() {
	// 简化markQuerySemicolons
	modified := func(f func(i int)) func(i int) {
		fmt.Println("modified official -- external")
		return func(i int) {
			fmt.Println("modified official -- internal")
			f(i)
		}
	}

	// "先"执行我魔改的官方逻辑
	f := modified(mock)
	f = official(f)
	f(2)

	fmt.Println()

	// "先"执行官方逻辑
	f2 := official(mock)
	f2 = modified(f2)
	f2(3)

	// 运行结果
	// modified official -- external
	// official -- external
	// official -- internal
	// modified official -- internal
	// mark 2

	// official -- external
	// modified official -- external
	// modified official -- internal
	// official -- internal
	// mark 3
}
```

实际上是方法调用栈的嵌套，所以是先进后出，只是换个写法，所以看起来比较晦涩。这个过程下来，理解颇多，学习到了不少。

## third_platform.go

最后补充前面main.go同级目录文件third_platform.go，其实主要就是签名并post请求第三方接口而已

```go
package main

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"crypto/tls"
	"encoding/base64"
	"io/ioutil"
	"net/http"
	"time"

	"github.com/gogf/gf/v2/os/glog"
)

type ThirdPlatform struct {
	IsInit int
	Url    string `json:"url"`
	Key    string
	Secret string
}

func (y *ThirdPlatform) Init() {
	if y.IsInit < 1 {
		y.Url = "https://192.10.2.1:443"
		y.Key = "123456"
		y.Secret = "DaFoAu234on20"
		y.IsInit = 1
	}
}

// 简化了一些逻辑 只保留核心的构造http request与client.Do
func (y *ThirdPlatform) PostUrl(url string, timeout int, params string) ([]byte, error) {
	y.Init()

	// 构造http请求
	buf := bytes.NewBuffer([]byte(params))
	req, err := http.NewRequest("POST", y.Url+url, buf)
	if err != nil {
		glog.Error(context.Background(), "thirdPlatform post new request failed", err)
		return nil, err
	}

	// 为请求添加ak、sk信息并签名
	y.AddAuthentication(req, "POST", url, params)

	// 调用第三方接口
	client := http.Client{
		Transport: &http.Transport{
			TLSClientConfig:   &tls.Config{InsecureSkipVerify: true},
			DisableKeepAlives: true,
		},
		Timeout: time.Second * time.Duration(timeout),
	}
	resp, err := client.Do(req)
	if err != nil {
		glog.Error(context.Background(), "thirdPlatform post failed", err)
		return nil, err
	}
	defer resp.Body.Close()

	// 处理响应
	b, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}
	client.CloseIdleConnections()
	return b, nil
}

// 同样简化了一些逻辑
func (y *ThirdPlatform) AddAuthentication(req *http.Request, method, url, params string) {
	// 换行符
	newLine := "\n"
	httpHeader := method + newLine
	acceptStr := "*/*"
	req.Header.Set("Accept", acceptStr)
	httpHeader += acceptStr + newLine

	// 设置请求格式
	contentTypeStr := "application/json"
	req.Header.Set("Content-Type", contentTypeStr)
	httpHeader += contentTypeStr + newLine

	// 签名
	signature := ComputeHmac256(httpHeader, y.Secret)
	req.Header.Set("x-ca-signature", signature)
}

func ComputeHmac256(message string, secret string) string {
	key := []byte(secret)
	h := hmac.New(sha256.New, key)
	h.Write([]byte(message))
	return base64.StdEncoding.EncodeToString(h.Sum(nil))
}
```

## 参考资料（不分先后）

[Go issue 25192](https://github.com/golang/go/issues/25192 "go issue 25192")

[Go 1.17 Release Notes - The Go Programming Language](https://tip.golang.org/doc/go1.17)

[Go can only get better: v1.17 hones in on performance and security](https://devclass.com/2021/08/17/go-can-only-get-better-v1-17-hones-in-on-performance-and-security/)

[Go 1.17 因安全问题修改的 URL query parsing](https://new.qq.com/rain/a/20210622A041MG00)

[Go net/http/server.go](https://github.com/golang/go/blob/master/src/net/http/server.go)

[Go net/http/serve_test.go](https://github.com/golang/go/blob/master/src/net/http/serve_test.go)

[mattn on Twitter: "net/url のセミコロンの扱いが変わる ...](https://twitter.com/mattn_jp/status/1427427593955930159)