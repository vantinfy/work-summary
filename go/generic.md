# go泛型使用示例记录

## 情况

对接某个第三方平台的接口时，其返回的数据

```go
// ApiResp 接口返回格式 common为通用结构 data可能有多种情况，一般是结构体、dataList或string
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

// GetCrossCar 调接口获取数据并存数据库 下同
func GetCrossCar() {
	// 忽略错误 只展示数据如何解析
	resp, _ := Post("http://localhost/cross_car/get")
	cc := CrossCar{}
	_ = json.Unmarshal(resp, &cc)

	// 写数据库
	DBStoreCrossCar(cc.List)
}

func GetVisitor() {
	resp, _ := Post("http://localhost/visitor/get")
	v := Visitor{}
	_ = json.Unmarshal(resp, &v)

	// 写数据库
	DBStoreVisitor(v.List)
}
```

可以看到CrossCar与Visitor的定义外层结构大同小异，如果算上其它的接口，冗余代码只会更多

另外，最后写数据库的部分（List字段）最坏也要对每个结构都专门写一个DBStoreXX方法

这两个问题都可以使用泛型解决，但是因为最开始学习泛型的时候没有搞懂，写数据库部分的问题取巧使用了接口的方式解决

### 示例

在看具体代码前这里先稍微了解下示例，便于后续的泛型理解

```go
func Add(x, y int) int {
	return x + y
}
```

上面的代码很简单，就是返回x+y的结果，但是有个局限就是只能处理int类型的数据

如果稍微加点需求，要实现float64、uint、int64这样，每个需求都要重新写代码，重复度特别高

这个时候就可以使用泛型了，改造后的Add方法支持了float64、uint、int64，如果要支持例如float32等也只需要对T进行拓展即可

这里的string也是可以的，返回结果就是两个字符串拼接的结果

```go
func Add[T int | float64 | uint | int64 | string /* | ...*/](a, b T) T {
	return a + b
}
```

这个就是简单的泛型方法。结构体中不确定的字段也可以使用泛型

```go
// 一个泛型类型的结构体，可用int或string类型实例化
type MyStruct[T int | string] struct {
	Name string
	Data T
}

func msNew()  {
	msi := MyStruct[int]{
		Name: "this is int",
		Data: 3,
	}
	mss := MyStruct[string]{
		Name: "this is string",
		Data: "xxx",
	}
	// ... 具体业务
}
```

比较重要的部分就是Go1.18之后interface的定义

下面资料来源于[Go 1.18 泛型全面讲解：一篇讲清泛型的全部_go语言泛型_nihaihaoma的博客-CSDN博客](https://blog.csdn.net/nihaihaoma/article/details/125601630) 6.3部分

```go
// 1.18之前的接口，【仅包含方法】 称为【Basic interface】
type rw interface {
	Read() []byte
	Write()
}

// 1.18之后的接口除了basic interface，还可以包含类型 称为【General interface】
type base interface {
	int | string | ~int64 // 带"~"表示底层类型为int64，如熟悉的time包的Duration实际上就是int64
}

// 既有方法 又有类型 同样属于General interface
type db interface { // 要实现此db接口需要为CrossCar或Visitor，且实现DBTableName方法
	CrossCar | Visitor

	DBTableName() string
}

// 补充：泛型接口 https://blog.csdn.net/weixin_42128977/article/details/127388016
type IPrintData[T int | float32 | string] interface {
	Print(data T)
}
```

知道了**Basic interface**与**General interface**之后再回头看前面写的Add方法，可以这样写

```go
type numOrString interface {
	int | float64 | uint | int64 | string // | ...
}

func Add[T numOrString](a, b T) T {
	return a + b
}

func main() {
	// fmt.Printf("%T %v\n", Add[float64](3.3, 3.3), Add[float64](3.3, 3.3))
	fmt.Printf("%T %v\n", Add(3.3, 3.3), Add(3.3, 3.3))
	fmt.Printf("%#v\n", Add("hello", "world"))
}
```

可以看到调用Add的时候没有指定T类型，这个是编译器帮忙推导的，实际上最终还是`Add[float64](3.3, 3.3)`这样

详见[Go 1.18 泛型全面讲解：一篇讲清泛型的全部_go语言泛型_nihaihaoma的博客-CSDN博客](https://blog.csdn.net/nihaihaoma/article/details/125601630) 5部分

### 分析 (Basic interface)

存到数据库的结构主要只是List部分，于是想到reflect包来取得结构体各个字段和对应的值，这里定义DBTableName接口，这样一来DBStore就可以将实现了该接口的结构作为参数了

另一个解决方式：各种数据库交互的结构可以整合为泛型，但是这部分代码已经成型后面就没有改动了（就算新增接口也只需要定义好结构并实现DBTableName接口即可，拓展性不是问题）

```go
// DBTableName 定义数据库结构体公共接口 便于insert ignore通用化传参
type DBTableName interface {
	TableName() string
}

// CrossCar 历史车辆记录数据库表 因为对数据库的insert是上面另外实现的 所以这里需要额外的orm的tag 而json的tag用来对应接口返回格式
type CrossCar struct {
	CrossRecordSyscode string `json:"crossRecordSyscode" orm:"cross_record_syscode"` // 过车记录唯一标识
	CrossTime          string `json:"crossTime" orm:"cross_time"`                    // 通过时间
	Type               string `json:"type" orm:"type"`
}

func (c CrossCar) TableName() string {
	return "cross_car"
}

// DataListCrossCar 这个结构用来解析调用接口时响应的数据
type DataListCrossCar struct {
	Total    int
	PageNo   int
	PageSize int
	List     []CrossCar // 最后存数据库的部分
}

func GetCC()  {
	// 忽略错误 只展示数据如何解析
	resp, _ := Post("http://localhost/cross_car/get")
	ar := ApiResp{} // 定义在前文
	_ = json.Unmarshal(resp, &ar)

	ccBytes, _ := json.Marshal(ar.Data)
	dataList := DataListCrossCar{}
	_ = json.Unmarshal(ccBytes, &dataList)

	dbname := make([]DBTableName, 0)
	for _, car := range dataList.List {
		// car.Translate() // 一些中间处理（如果有） 此处的translate只做了type映射（接口返回是123等，要转换成前端需要的类型）
		dbname = append(dbname, car)
	}
	// 写数据库
	InsertIgnore(dbname)
}

const DBTag = "orm"

// InsertIgnore 使用接口传参 方便支持多种结构体 目前的应用还是同时插入一种结构体的多个实例 理论上是可以同时插入多种结构体实例（但这样使用肯定不合适）
func InsertIgnore(items []DBTableName) {
	if len(items) == 0 {
		return
	}
	// 插入前先确定好结构体各字段在数据库中的字段名（且有顺序要求）
	tags, err := GetDBTags(items[0], DBTag)
	if err != nil {
		glog.Error("YMJ insert ignore: get tags failed", err)
		return
	}

	// insert ignore: 插入记录时（优先判断主键）如果已存在则跳过
	// ConvertInsertValue()方法将结构体切片转为sql字符串中的values 其中各个字段的值与GetDBTags()顺序一致
	sql := `insert ignore into ` + items[0].TableName() + ` (` + tags + `) values ` + ConvertInsertValues(items)
	if items[0].TableName() == "ymj_hw_alarm" {
		sql = strings.ReplaceAll(sql, "localtime", "`localtime`")
	}
	_, err = common.ExecSQL(sql)
	if err != nil {
		glog.Error("YMJ ExecSQL: db insert ignore failed:", err, "and sql is:", sql)
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
	// insert ignore into table (field1, ...) values (value1, ...), (value1, ...)
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

InsertIgnore方法利用了接口来实现多种结构体插入数据库，只是使用reflect包遍历结构体的field来取得key跟value，最后字符串拼接为sql语句并执行而已

此外，现有的公共方法中虽然有insert，但是sql在执行插入重复主键时会报错，于是才封装了这个InsertIgnore版本

其它要注意的就是主要我默认使用`orm`tag来解析结构体，如果依然使用json tag与数据库交互后期功能拓展对数据的存取可能会混乱

### 分析 (General interface)

引入接口的方式解决了存数据库需要写多个方法的问题，但是接口响应解析代码冗余问题依然存在，这个问题是后续重新啃了泛型才解决的

各种结构体因为存在共同Total、PageNo字段可以抽出来，仅List为各种结构，泛型的引入顺理成章

```go
// dataList 类型集
type baseList interface {
	[]CrossCar | []Visitor // | ...
}

// DataList 泛型结构体 接口返回格式的一种 通常是获取列表时 此时list字段为结构体数组
type DataList[T baseList] struct {
	Total    int `json:"total,omitempty"`
	PageNo   int `json:"pageNo,omitempty"`
	PageSize int `json:"pageSize,omitempty"`
	List     T   `json:"list"` // 表示List字段可以为baseList中声明的任意类型（[]CrossCar或者[]Visitor等）
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

// ------ 泛型 结合前面泛型结构体与General interface ------

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
```

这里定义了RespDataReduce泛型方法来统一解析不同接口（CrossCar、Visitor）的返回数据，要求第二个参数为Reduce中定义的类型，这样一来就不需要为每种结构都写一个解析方法

假设后续新增了要处理的接口以及对应新的返回结构（前提是格式一致，即{code msg data{total pageNo []list}}这种形式），只需要在Reduce中添加新定义的结构即可

```go
func GetCrossCars() {
	apiRespBytes, _ := Post("xxx")
	// 接口数据还原为结构体
	dl := DataList[[]CrossCar]{}
	// general interface
	dl, _ = RespDataReduce(apiRespBytes, dl)

	// insert ignore插入数据
	dbname := make([]DBTableName, 0)
	for _, owner := range dl.List {
		dbname = append(dbname, owner)
	}
	// basic interface
	InsertIgnore(dbname)  // InsertIgnore方法在前文
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
```

## 进阶

既有类型又有方法的general interface，这个是在后面写后台数据管理时应用的

```go
// XlsxDB 需要xlsx与结构体之间转换的泛型
type XlsxDB interface {
	Asset | Student // | ...

	DBName() string
	ExportFilename() string
	Translate(self any) (any, error) // 目前只用于结构体时间字段的格式化 因为具体结构体定义func(s *structure) Translate()这样会报错误 最后折中将结构体本身作为参数传递
}
```

因为涉及到excel文件的处理，包括数据库导入导出，ExportFileName用于导出下载xlsx模板时寻找文件

```go
// ------ 公共常量与方法 ------

const (
	requireTimeFormat      = "2006年01月02日 15:04:05" // 返回前端时间字符串格式
	timeFormatWithoutTZone = "2006-01-02 15:04:05"  // 没有带T与时区的时间格式
)

// ConvTimeStr 时间字符串格式转换 按照inFormat标准输入 以outFormat输出格式化后的时间字符串
func ConvTimeStr(date, inFormat, outFormat string) (string, error) {
	if date == "" {
		return "", fmt.Errorf("empty time string")
	}
	duetimecst, err := time.ParseInLocation(inFormat, date, time.Local)
	if err != nil {
		return "", err
	}
	return duetimecst.Format(outFormat), nil
}

// ------ 具体结构 只保留部分字段 ------

type Asset struct {
	Id               int    `xlsx:"id" json:"id"`                             // 物资编号
	Name             string `xlsx:"name" json:"name"`                         // 物资名称
	LastModifiedDate string `xlsx:"lastModifiedDate" json:"lastModifiedDate"` // 最后修改时间
}

func (m Asset) DBName() string {
	return "asset"
}

func (m Asset) ExportFilename() string {
	return "物资"
}

func (m Asset) Translate(selfArg any) (any, error) {
	self, ok := selfArg.(Asset)
	if !ok {
		return selfArg, fmt.Errorf("translate: Asset real type[%T]", selfArg)
	}
	last, err := ConvTimeStr(self.LastModifiedDate, timeFormatWithoutTZone, requireTimeFormat)
	if err != nil {
		return selfArg, err
	}
	self.LastModifiedDate = last
	return self, nil
}

type Student struct {
	Id               int64  `xlsx:"id" json:"id"`                             // 学员id
	Name             string `xlsx:"name" json:"name"`                         // 姓名
	LastModifiedDate string `xlsx:"lastModifiedDate" json:"lastModifiedDate"` // 最后修改时间
}

func (s Student) DBName() string {
	return "student"
}

func (s Student) ExportFilename() string {
	return "学员信息"
}

func (s Student) Translate(selfArg any) (any, error) {
	self, ok := selfArg.(Student)
	if !ok {
		return selfArg, fmt.Errorf("translate: Student real type[%T]", selfArg)
	}

	last, err := ConvTimeStr(self.LastModifiedDate, timeFormatWithoutTZone, requireTimeFormat)
	if err != nil {
		return selfArg, err
	}
	self.LastModifiedDate = last
	return self, nil
}
```

Store与前面InsertIgnore思想类似，都是使用reflect包解析结构体tag跟value

比较值得一提的是rows2Struct，泛型方法中如果要声明泛型T只能使用`var name T`这种方式，参数rows是xlsx表格读取出来的行列，同样结合reflect包来构造T，之后append到[]T中

```go
// "github.com/gogf/gf/net/ghttp"
// "github.com/qax-os/excelize"
// "mime/multipart"

const (
	Tag   string = "xlsx"   // 解析的结构体tag
	DBKey string = "db_key" // 标识导入/导出为何种类型结构
)

func GetUploadFiles(r *ghttp.Request) ([]multipart.File, error) {
	// 获取上传的文件
	files := r.GetUploadFiles(UploadFileKey)
	filesSlice := make([]multipart.File, 0)
	// 直接打开上传的文件流 不需要保存到磁盘再打开-解析-删除文件 但是注意使用File后需要手动关闭
	for _, file := range files {
		//file.FileHeader.Filename
		oFile, err := file.Open()
		if err != nil {
			return nil, err
		}
		filesSlice = append(filesSlice, oFile)
	}
	return filesSlice, nil
}

// DataImport excel数据导入数据库（不支持xls文件 保存时.xlsx重命名为.xls的没有问题）
func (y *Action) DataImport(r *ghttp.Request) {
	files, err := GetUploadFiles(r)
	if err != nil {
		panic(err)
	}
	// 实际上就一个文件
	for _, file := range files {
		// 打开xlsx文件
		f, err := excelize.OpenReader(file)
		if err != nil {
			panic(err)
		}
		// 解析xlsx文件 后续转为结构体切片
		rows, err := f.GetRows("Sheet1")
		if len(rows) == 0 || err != nil {
			panic(err)
		}
		_ = f.Close()

		// 根据请求key决定要转为哪种结构体
		dbType := r.Get(DBKey)
		v, _ := dbType.(string)
		switch v {
		case "assets": // 物资 判断上传的时间参数是否合法
			for _, row := range rows[1:] {
				_, err = time.ParseInLocation(timeFormatWithoutTZone, row[len(row)-1], time.Local)
				if err != nil {
					panic(err)
				}
			}
			err = Store[Asset](rows, Asset{})
		case "student": // 学员信息
			for _, row := range rows[1:] {
				_, err2 := time.ParseInLocation("2006-01-02", row[2], time.Local)
				_, err7 := time.ParseInLocation(timeFormatWithoutTZone, row[len(row)-2], time.Local)
				_, err8 := time.ParseInLocation(timeFormatWithoutTZone, row[len(row)-1], time.Local)
				if err2 != nil || err7 != nil || err8 != nil {
					func(errs ...error) {
						for _, erri := range errs {
							if erri != nil {
								panic(erri)
							}
						}
					}(err2, err7, err8)
				}
			}
			err = Store[Student](rows, Student{})
		default:
			err = fmt.Errorf("unsupported type[%v]", v)
		}
		if err != nil {
			panic(err)
		}

		// 关闭文件
		err = file.Close()
		if err != nil {
			glog.Error("data manage import[file close]:", err)
		}
	}
	common.Data(r, common.ERR_CODE_SUCCESS, common.LOG_PRINT_NO, "DataImport", "import success")
}

func Store[T XlsxDB](rows [][]string, dbType T) error {
	var fields, values string
	cType := reflect.TypeOf(dbType)
	// 需要替换掉xlsx解析的row第一行 因为模板为中文
	replace := make([]string, 0)
	// 获取sql insert时的tag
	for i := 0; i < cType.NumField()-1; i++ {
		v, ok := cType.Field(i).Tag.Lookup(Tag)
		if ok {
			fields += v + ","
			replace = append(replace, v)
		}
	}
	last, ok := cType.Field(cType.NumField() - 1).Tag.Lookup(Tag)
	if ok {
		replace = append(replace, last)
		fields += last
	}
	rows[0] = replace // 替换xlsx解析第一行的中文

	// 获取结构体的tag
	tags := initTag2FieldIdx(dbType, Tag)
	// 根据tag将[][]string转为结构体切片
	s := rowsToStruct(rows, tags, dbType)
	if len(s) == 0 {
		// 感觉这里可以不算错误...
		return fmt.Errorf("rows2Struct: result empty")
	}

	// 获取value
	for _, t := range s {
		// 切片的每个元素转后的值 例如: ("fields", 1, "fields2")
		tempV := ""
		reflectCarValue := reflect.ValueOf(t)
		CarType := reflect.TypeOf(t)

		for i := 0; i < CarType.NumField(); i++ {
			// 通过定义的结构体字段名获取值 配合GetDBTags()方法使用 （insert into table (...) values (...)前后字段及对应值顺序需要一致）
			v := reflectCarValue.FieldByName(CarType.Field(i).Name)
			// 数值类型无需加双引号
			if v.CanInt() || v.CanUint() || v.CanFloat() {
				tempV += fmt.Sprintf("%v", v) + ","
			} else {
				// 目前默认其他都是string
				appendStr := fmt.Sprintf(`%v`, v)
				tempV += `"` + strings.ReplaceAll(appendStr, `"`, `\"`) + `",` // 防止单元格内容带["]时后续insert错误
			}
		}
		// tempV[:len(tempV)-1] 去掉最后一个","号 后面return时同理
		values += "(" + tempV[:len(tempV)-1] + "),"
		values = values[:len(values)-1]
	}

	_, err := common.ExecSQL(`insert ignore into ` + dbType.DBName() + ` (` + fields + `) values ` + values)
	return err
}

// 提取结构体Stu的字段和标签
func initTag2FieldIdx(v interface{}, tagKey string) map[string]int {
	u := reflect.TypeOf(v)
	tag2fieldIndex := map[string]int{}
	for i := 0; i < u.NumField(); i++ {
		f := u.Field(i)
		tagValue, ok := f.Tag.Lookup(tagKey)
		if ok {
			tag2fieldIndex[tagValue] = i
		}
	}
	return tag2fieldIndex
}

// structure参数只作为泛型判断转换结果data类型用
func rowsToStruct[T XlsxDB](rows [][]string, tag2fieldIndex map[string]int, structure T) []T {
	var data []T
	// 默认第一行对应tag
	head := rows[0]
	for _, row := range rows[1:] {
		var stu T // 使用泛型情况下 这里只能使用var声明变量
		rv := reflect.ValueOf(&stu).Elem()
		for i := 0; i < len(row); i++ {
			colCell := row[i]
			// 通过 tag 取到结构体字段下标
			fieldIndex, ok := tag2fieldIndex[head[i]]
			if !ok {
				continue
			}
			colCell = strings.Trim(colCell, " ")
			// 通过字段下标找到字段放射对象
			v := rv.Field(fieldIndex)
			// 根据字段的类型，选择适合的赋值方法
			switch v.Kind() {
			case reflect.String:
				value := colCell
				v.SetString(value)
			case reflect.Int64, reflect.Int32, reflect.Int:
				value, err := strconv.Atoi(colCell)
				if err != nil {
					panic(err)
				}
				v.SetInt(int64(value))
			case reflect.Float64:
				value, err := strconv.ParseFloat(colCell, 64)
				if err != nil {
					panic(err)
				}
				v.SetFloat(value)
			}
		}
		data = append(data, stu)
	}
	return data
}
```

读取数据，没什么难点。其中返回前端之前for range了一次structure切片，对每个元素都调用了一次Translate方法

Translate在前面注释已经提到，只是对一些字段的时间字符串做格式化处理，转成前端要求的格式

```go
const (
	sqlAll       = "select * from "
	sqlSelectCnt = "select count(*) as thecount from "
)

type Generic[T XlsxDB] struct {
	Count int `json:"count"`
	List  []T `json:"list"`
}

func (y *Action) DataManageAsset(r *ghttp.Request) {
	DBRead(r, []Asset{})
}

func (y *Action) DataManageStudent(r *ghttp.Request) {
	DBRead(r, []Student{})
}

func DBRead[T XlsxDB](r *ghttp.Request, structure []T) {
	var temp T
	// 分页查询 这里省略错误检查
	pageNo := r.GetInt("currPage")
	pageNo--
	pageSize := r.GetInt("pageSize")
	limit := fmt.Sprintf(` limit %d, %d`, pageNo*pageSize, pageSize)

	// 部分数据会要求附带where进行过滤查询 这里也省略了
	sqlGrep := sqlAll + temp.DBName() + limit

	_, err := common.GetStructsFromSQL(sqlGrep, &structure)
	if err != nil {
		panic(err)
	}
	// 计算总数
	total, err := common.GetRecordCountBySQL(sqlSelectCnt + temp.DBName())
	if err != nil {
		panic(err)
	}

	// 处理每个子元素 （例如时间格式转换）
	for i, t := range structure {
		tx, err := t.Translate(t)
		if err != nil {
			panic(err)
		}
		structure[i], _ = tx.(T) // 断言+泛型
	}

	// 响应结构套一层list
	g := Generic[T]{
		Count: total,
		List:  structure,
	}
	common.Data(r, common.ERR_CODE_SUCCESS, common.LOG_PRINT_YES, temp.ExportFilename(), g)
}
```

## 参考

不分先后

[Go 1.18 泛型全面讲解：一篇讲清泛型的全部_go语言泛型_nihaihaoma的博客-CSDN博客](https://blog.csdn.net/nihaihaoma/article/details/125601630)

[Go泛型实战教程之如何在结构体中使用泛型_Golang_脚本之家 (jb51.net)](https://www.jb51.net/article/257079.htm)

[go泛型使用方法_go 泛型_doublewe的博客-CSDN博客](https://blog.csdn.net/qq_42062052/article/details/123840525)

[golang-泛型基础篇（一）_golang 泛型结构体_蔡蔡开始内卷的博客-CSDN博客](https://blog.csdn.net/weixin_42128977/article/details/127388016)

[Go 泛型-CSDN博客](https://blog.csdn.net/tearon/article/details/124960440)

[Golang泛型的巧妙应用，防止变量空指针错误，防止结构体字段空指针错误-CSDN博客](https://blog.csdn.net/Deng_Xian_Sheng/article/details/125512321)