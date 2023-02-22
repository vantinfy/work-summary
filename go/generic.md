# go泛型使用示例记录

## 情况

对接某个第三方平台的接口时，其返回的数据

```go
// ApiResp 接口返回格式 common为通用结构 data可能有多重情况，一般是结构体（切片）、dataList或string
type ApiResp struct {
	Common
	Data any `json:"data,omitempty"`
}

// Common 接口通用返回格式(部分)
type Common struct {
	Code string `json:"code,omitempty"`
	Msg  string `json:"msg,omitempty"`
}
```

ApiResp的Data通常如下

```go
// 示例两个 省略了tag与一些字段

type CrossCar struct {
	Total    int
	PageNo   int
	PageSize int
	List     []struct {
		CrossRecordSyscode string
		Type               string
		CrossTime          string
	}
}

type Visitor struct {
	Total    int
	PageNo   int
	PageSize int
	List     []struct {
		OrderId        string
		VisitorName    string
		VisitStartTime string
	}
}

func GetCrossCar() {
	// 忽略错误 只展示数据如何解析
	resp, _ := Post("http://localhost/cross_car/get")
	cc := CrossCar{}
	json.Unmarshal(resp, &cc)

	// 写数据库
	DBStoreCrossCar(cc.List)
}

func GetVisitor() {
	// 忽略错误 只展示数据如何解析
	resp, _ := Post("http://localhost/visitor/get")
	v := Visitor{}
	json.Unmarshal(resp, &v)

	// 写数据库
	DBStoreVisitor(v.List)
}
```

可以看到CrossCar与Visitor外层结构大同小异，实际业务数据处理起来解析还不是麻烦的地方，反而是最后写数据库，需要对每个结构都专门写一个方法，代码重复冗长

当然也可以DBStore传any参数，具体在方法里面再类型断言为各种结构，但效果甚微

分析：最后存到数据库的结构主要只是List部分，于是想到reflect包来取得结构体各个字段和对应的值，这样一来DBStore就可以将几种结构体作为参数了

于是引入接口，与数据库交互就只要实现TableName方法即可，而各种结构体因为存在共同Total、PageNo字段也可以抽出来，仅List为各种结构，泛型的引入顺理成章

```go
// 泛型 dataList
type baseList interface {
	[]CrossCar | []Visitor
}

// DataList 返回格式的一种 通常是获取列表时 此时list字段为结构体数组
type DataList[T baseList] struct {
	Total    int `json:"total,omitempty"`
	PageNo   int `json:"pageNo,omitempty"`
	PageSize int `json:"pageSize,omitempty"`
	List     T   `json:"list"` // 表示List字段可以为baseList中声明的任意类型（[]CrossCar或者[]Visitor）
}

// DBTableName 定义数据库结构体公共接口 便于insert ignore通用化传参
type DBTableName interface {
	TableName() string
}

// CrossCar 历史车辆记录数据库表 因为对数据库的insert是上面另外实现的 所以这里需要额外的orm的tag 而json的tag用来对应接口返回格式
type CrossCar struct {
	CrossRecordSyscode string `json:"crossRecordSyscode" orm:"cross_record_syscode"` // 过车记录唯一标识
	CrossTime          string `json:"crossTime" orm:"cross_time"`                    // 通过时间
	Type               string `json:"type" orm:"type"`                               // 车辆类型字符串格式
}

func (y CrossCar) TableName() string {
	return "cross_car"
}

type Visitor struct {
	OrderId        string `json:"orderId" orm:"order_id"`                // 访客记录id
	VisitorName    string `json:"visitorName" orm:"visitor_name"`        // 访客姓名
	VisitStartTime string `json:"visitStartTime" orm:"visit_start_time"` // 来访时间
}

func (a Visitor) TableName() string {
	return "visitor"
}
```

之后就可以定义一个方法来统一解析不同接口的返回数据写数据库

```go
// Reduce 泛型 停车场、访客记录...
type Reduce interface {
	DataList[[]CrossCar] | DataList[[]Visitor] // | ...
}

// RespDataReduce 将http请求返回的数据还原为Datalist结构体
func RespDataReduce[T Reduce](apiRespBytes []byte, target T) (T, error) {
	// 接口数据还原为结构体 DataList上面还有一层（code msg data等信息 其中data就是前面定义的DataList）
	apiResp := ApiResp{}
	err := json.Unmarshal(apiRespBytes, &apiResp)
	if err != nil {
		return target, err
	}

	// 序列化apiResp.data 因为apiResp结构体为了兼容各种返回数据类型定义为any 这里使用json包 免去后续各种类型断言
	dataBytes, err := json.Marshal(apiResp.Data)
	if err != nil {
		return target, err
	}
	return target, json.Unmarshal(dataBytes, &target)
}

func GetCrossCars() {
	apiRespBytes, _ := Post("xxx")
	// 接口数据还原为结构体
	dl := DataList[[]CrossCar]{}
	dl, _ = RespDataReduce(apiRespBytes, dl)

	// insert ignore插入数据
	dbname := make([]DBTableName, 0)
	for _, owner := range dl.List {
		dbname = append(dbname, owner)
	}
	InsertIgnore(dbname)
}

func GetVisitors() {
	apiRespBytes, _ := Post("yyy")
	// 接口数据还原为结构体 访客信息
	dl := DataList[[]Visitor]{}
	dl, _ = RespDataReduce(apiRespBytes, dl)

	// insert ignore插入数据
	dbname := make([]DBTableName, 0)
	for _, owner := range dl.List {
		dbname = append(dbname, owner)
	}
	InsertIgnore(dbname)
}

const DBTag = "orm"

// InsertIgnore 使用接口传参 方便支持多种结构体 目前的应用还是同时插入一种结构体的多个实例 （理论上是可以同时插入多种结构体实例，这个最好做校验）
func InsertIgnore(items []DBTableName) {
	if len(items) == 0 {
		return
	}
	// 插入前先确定好结构体各字段在数据库中的字段名（且有顺序要求）
	tags, err := GetDBTags(items[0], DBTag)
	if err != nil {
		glog.Error("insert ignore: get tags failed", err)
		return
	}

	// insert ignore: 插入记录时（优先判断主键）如果已存在则跳过
	// ConvertInsertValue()方法将结构体切片转为sql字符串中的values 其中各个字段的值与GetDBTags()顺序一致
	sql := `insert ignore into ` + items[0].TableName() + ` (` + tags + `) values ` + ConvertInsertValues(items)
	_, err = common.ExecSQL(sql)
	if err != nil {
		glog.Error("ExecSQL: db insert ignore failed:", err, "and sql is:", sql)
	}
}

func GetDBTags(items DBTableName, dbTag string) (string, error) {
	if dbTag == "" {
		dbTag = "json" // 如果dbTag为空 则使用json tag
	}
	str := ""

	cType := reflect.TypeOf(items)
	for i := 0; i < cType.NumField(); i++ {
		// 取得结构体定义中的tag（跟数据库定义的字段对应）
		tagValue := cType.Field(i).Tag.Get(dbTag)
		if tagValue == "" {
			return "", fmt.Errorf("struct have not tag[%v]", dbTag)
		}
		// 最后一个无需加","号
		if i == cType.NumField()-1 {
			str += tagValue
		} else {
			str += tagValue + ","
		}
	}
	return str, nil
}

// ConvertInsertValues 将结构体切片转换为mysql插入语句insert时的values值
func ConvertInsertValues(cars []DBTableName) string {
	// 示例结果: `insert ignore into table (field1, ...) values (value1, ...), (value1, ...)`
	values := ""

	for _, car := range cars {
		// 切片的每个元素转后的值 例如: ("fields", 1, "fields2")
		tempV := ""
		reflectCarValue := reflect.ValueOf(car)
		CarType := reflect.TypeOf(car)

		for i := 0; i < CarType.NumField(); i++ {
			// 通过定义的结构体字段名获取值 配合GetDBTags()方法使用 （insert into table (...) values (...)前后字段及对应值顺序需要一致）
			v := reflectCarValue.FieldByName(CarType.Field(i).Name)
			// 数值类型无需加双引号
			if v.CanInt() || v.CanUint() || v.CanFloat() {
				tempV += fmt.Sprintf("%v", v) + ","
			} else {
				// 目前默认其他都是string
				tempV += fmt.Sprintf(`"%v"`, v) + ","
			}
		}
		// tempV[:len(tempV)-1] 去掉最后一个","号 后面return时同理
		values += "(" + tempV[:len(tempV)-1] + "),"
	}
	return values[:len(values)-1]
}
```

RespDataReduce就是一个泛型方法，要求第二个参数为Reduce中定义的类型，这样一来就不需要为每种结构都写一个解析方法

假设后续新增了要处理的接口以及对应新的返回结构（前提是格式一致，即code msg data{total pageNo []list}这种形式），只需要在Reduce中添加新定义的结构即可

而InsertIgnore方法则是利用了接口来实现多种结构体插入数据库（当时写这个方法的时候还没有研究泛型，~~好吧其实研究了但是没搞懂最后用这个方式取巧~~）

具体实现看代码大概就懂了~~吧~~，只是使用reflect包遍历结构体的field来取得key跟value，最后字符串拼接为sql语句并执行而已

比较值得一提的是，现有的公共方法中虽然有insert，但是没有ignore，sql在执行插入重复主键时会报错，于是才封装了这个InsertIgnore版本

其它要注意的就是主要我默认使用`orm`tag来解析结构体，如果依然使用json tag与数据库交互后期功能拓展对数据的存取可能会混乱

## 参考

todo 后面贴上当时看的go泛型博客 没有泛型基础直接看这篇文章可能不知所云（我当时刚研究就是这样）