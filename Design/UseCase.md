# CloudFlow 使用实例/标准

## UseCase

### Common

#### Python

##### Init `cf` and `app`

使用`yaml`配置对`app`进行初始化，`app`持续运行

```python
import cloudflow as cf

def run_app():
    app = cf.create_app("cf_app_cfg.yaml")
    # app.service include
    app.serve()                                             # async start app

if __name__ == "__main__":
    cf.run_app(run_app)
```

##### execute session

共三种方式，一是以flow为单位运行`execute()`方法，进行每一步flow的运行，这种方式利于debug；其余方式是形成完成flow之后统一运行所有步骤，更符合图计算的运行模式。[go的运行模式相同](<##### go execute session>)

```python
# I. single process calculation
app = cf.app.get("App Name")
session = app.create_session("session_cfg.yaml")
# session.source create
# session.filter create
# session.flow create
# 1st way run
flow.execute();
# session.flow create
flow.execute();

# II. whole process calculation
def execute(app: cf.App):
    session = app.create_session("session_cfg.yaml")
    # session.source create
    # session.filter create
    # session.flow create

# session exection
app = cf.app.get("App Name")
# 2nd way run
app.execute(execute)
# 3rd way run
app.session.get("Session Name").execute_all()
while app.session.get("Session Name").status != cf.Status.FINISHED:
    import time
    time.sleep(1)
print("Execute Successfully")
app.clean_session("Session Name")

# other utilities
app = cf.app.get("App Name")
app.force_terminate()
app.log("Path")
app.session.force_quite("Session Name")
app.session.log("Session Name", "Path")
```

#### go

##### Init `cf` and `app`

```go
import "cloudflow"
var cf cloudflow.CF

func runApp()) {
    app := cf.CreateApp("cf_app_cfg.yaml")
    // app.service include
    app.serve()
}

func main() {
    cf.RunApp(runApp)
}
```

##### go execute session

```go
// I. single process calculation
func main() {
    app := cf.app.Get("App Name")
    session := app.CreateSession("session_cfg.yaml")
    // session.source create
    // session.filter create
    // session.flow create
    // 1st way run
    flow.Execute();
    // session.flow create
    flow.Execute();
}

// II. whole process calculation
func execute(app cloudflow.CF.App) {
    session := app.CreateSession("session_cfg.yaml")
    // session.source create
    // session.filter create
    // session.flow create
}

// session exection
func utility() {
    app := cf.app.Get("App Name")
    // 2nd way run
    app.Execute(execute)
    // 3rd way run
    app.session.Get("Session Name").Execute()
    for app.session.Get("Session Name").Status != cf.Status.FINISHED:
        time.Sleep(time.Second)
    fmt.Println("Execute Successfully")
    app.session.Clean("Session Name")
    
    // other utilities
    app.ForceTerminate()
    app.Log("Path")
    app.session.ForceQuite("Session Name")
    app.session.Log("Session Name", "Path")
}
```

### Flow

如何将函数之间参数和返回值对应起来？

明确规定`flow`中函数参数顺序：

1. `session`
2. 之前函数的输出参数，需全部包含
3. 用户额外指定的参数，使用`append_args`指定

### Examples

#### MicroServices

类同[https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)

1. clone repo

```sh
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd microservices-demo/
```

2. config cloudflow

```sh
cf config ./config/cf_cfg.yaml
```

3. build and deploy the app

```sh
cf build
cf deploy MyApp
```

4. check dashboard, run below and access [localhost:24678](http://localhost:24678)

```sh
cf dashboard open
```

5. access your API Server url to use your service
6. if you want to delete your service

```sh
cf delete app MyApp
```

#### MapReduce: WordCount

##### Python

最标准的MapReduce问题，定义四种函数：输入、Map、Reduce、输出，利用`flow`提供的`input, map, reduce, output`函数进行处理，无需关心中间结果。

```python
def data_read(session: cf.App.Session) -> list[str]:
    pid = session.get_pid()
    num = session.get_num()
    src = session.get_source(pid, num)
    data = []
    for f in src[pid * num: (pid + 1) * num]:
        for l in open(f).readlines():
            data.append(l)
    return data

# map
def word_count(session: cf.App.Session, data: str) -> dict[str, int]:
    wc = {}
    for w in data.split(' '):
        if w in wc:
            wc[w] += 1
    return wc

def merge_dict(a: dict[str, int], b: dict[str, int]) -> dict[str, int]:
    c = copy(a)
    for k,v in b.items():
        if k in c:
            c[k] += b[k]
    return c

MergeFunc = Callable[[dict[str, int], dict[str, int]], dict[str, int]]

# reduce
def wc_reduce(session: cf.App.Session, data: list[dict[str, int]], merge: MergeFunc) -> dict[str, int]:
    wc = {}
    for d in data:
        wc = merge_dict(wc, d)
    return wc

def output(session: cf.App.Session, data: dict[str, int], is_final: bool) -> dict[str, int]:
	if is_final:
        print("final[%d] > %s" % (session.get_pid(), data))
    else:
        print("index[%d] > finished" % (session.get_pid()))
    return data

# execution
def execute(app: cf.App):
    session = app.create_session("session_cfg.yaml")
    files = os.listdir("File Path")
    session.create_source(files, type="files")
    # 1st way
    session.import_flow("cncf_workflow.yaml")
    # 2nd way
    flow = session.create_flow("wc")
    # 1st way run
    flow.input(data_read, count=20).execute()
    flow.map(word_count, batch_size=100, count=20).execute()
    flow.reduce(wc_reduce, append_args=(merge_dict), is_final=False).execute()
    flow.output(output, append_args=(False), way="stdout").execute()
    flow.reduce(wc_reduce, append_args=(merge_dict), is_final=True).execute()
    flow.output(output, append_args=(True), way="file", path="res.txt").execute()
    # 2nd/3rd way run
    flow = flow.input(data_read, count=20)
               .map(word_count, batch_size=100, count=20)
               .reduce(wc_reduce, append_args=(merge_dict), is_final=False)
               .output(output, append_args=(False), way="stdout")
               .reduce(wc_reduce, append_args=(merge_dict), is_final=True)
               .output(output, append_args=(True), way="file", path="res.txt")

# 2nd/3rd way of session execution like above
```

##### Go

go语言函数中无法指定可选参数，因此使用空接口代替`append_args`

```go
func dataRead(session cf.App.Session) string {
    pid := session.GetPid()
    num := session.GetNum()
    src := session.GetSource(pid, num)
    data := ""
    for _,f := range src[pid * num: (pid + 1) * num + 1]:
        file, err := os.Open(f)
        if err != nil {
            fmt.Println(err)
            return
        }
        defer file.Close()
        fileInfo, _ := file.Stat()
        bytes := make([]byte, fileInfo.Size())
        _, err = file.Read(bytes)
        if err != nil {
            fmt.Println(err)
            return
        }
        data += string(bytes)
    return data
}

// map
func wordCount(session cf.App.Session, data string) map[string]int {
    wc := make(map[string]int)
    for w in string.Split(data, " "):
        if w in wc:
            wc[w] += 1
    return wc
}

func mergeDict(a map[string]int, b map[string]int) map[string]int {
    c := make(map[string]int)
    for k,v in b.items():
        _, exist := a[k]
        if exist {
            c[k] += a[k] + b[k]
        }
        else {
            c[k] = b[k]
        }
    return c
}

type MergeFunc func(map[string]int, map[string]int) map[string]int) map[string]int

// reduce
func wcReduce(session: cf.App.Session, data []map[string]int, merge MergeFunc) map[string]int {
    wc := make(map[string]int)
    for d in data:
        wc = mergeDict(wc, d)
    return wc
}

func output(session cf.App.Session, data map[string]int], is_final bool) map[string]int {
	if is_final:
        fmt.printf("final[%d] > %s", (session.get_pid(), data))
    else:
        fmt.printf("index[%d] > finished", (session.get_pid()))
    return data
}

// execution
func execute(app cf.App) {
    session := app.CreateSession("session_cfg.yaml")
    files,_ := ioutil.ReadDir("File Path")
    session.CreateSource(files)
    // 1st way
    session.ImportFlow("cncf_workflow.yaml")
    // 2nd way
    flow := session.CreateFlow("wc")
    var reduceInterface []interface{}
    reduceInterface = append(reduceInterface, merge_dict)
    var firstOutputInterface []interface{}
    firstOutputInterface = append(firstOutputInterface, false, "stdout")
    var secondOutputInterface []interface{}
    secondOutputInterface = append(secondOutputInterface, true, "file", "res.txt")
    // 1st way run
    flow.Input(data_read, 20).Execute()
    flow.Map(wordCount, 100, 20).Execute()
    flow.Reduce(wcReduce, false, reduceInterface).Execute()
    flow.Output(output, firstOutputInterface).Execute()
    flow.Reduce(wcReduce, true, reduceInterface).Execute()
    flow.Output(output, secondOutputInterface).Execute()
    // 2nd/3rd way run
    flow = flow.Input(data_read, 20)
               .Map(wordCount, 100, 20)
               .Reduce(wcReduce, false, reduceInterface)
               .Output(output, firstOutputInterface)
               .Reduce(wcReduce, true, reduceInterface)
               .Output(output, secondOutputInterface)
}

// session execution like above
```

#### Pi Estimation

随机统计法

在$$x,y\in (0, 1)$$的正方形中随机抛硬币，在四分之一单位圆中的概率为$$\frac{\pi}{4}$$，利用这一点求解$$\pi$$。利用分布式模拟硬币投掷，将结果进行汇总。

##### Python

```python
def calculate_one_point(session: cf.App.Sesssion) -> int:
    x, y = random.random(), random.random()
    return x * x + y * y < 1 ? 1 : 0

def count(session: cf.App.Session, points: list[int]) -> int:
    res = 0
    for i in points:
        res += i
    return res

def calculate_res(session: cf.App.Session, count_res: int, sample_num: int) -> float:
    # 1st way
    return count_res * 4 / session.flow_count("calculate_one_point")
    # 2nd way
    return count_res * 4 / sample_num

def execute(app cf.App):
    session = app.create_session("session_cfg.yaml")
    # 1st way
    session.import_flow("cncf_workflow.yaml")
    # 2nd way
    flow = session.create_flow("pi")
    # 1st way run
    flow.map(calculate_one_point, count=NUM_SAMPLES).execute()
    flow.reduce(count).execute()
    flow.output(calculate_res, append_args=(NUM_SAMPLES), way="stdout").execute()
```

##### Go

```go
func calculateOnePoint(session cf.App.Sesssion) int {
    var x, y = rand.Float64(), rand.Float64()
    if x * x + y * y < 1 {
        return 1
    }
    return 0
}

func count(session cf.App.Session, points: []int) int:
    res = 0
    for _, i := range points {
        res += i
    }
    return res

func calculateRes(session: cf.App.Session, countRes: int, sampleNum: int) -> float64:
    // 1st way
    return countRes * 4 / session.FlowCount("calculate_one_point")
    // 2nd way
    return countRes * 4 / sampleNum

func execute(app cf.App) {
    session := app.CreateSession("session_cfg.yaml")
    // 1st way
    session.ImportFlow("cncf_workflow.yaml")
    // 2nd way
    flow := session.CreateFlow("pi")
    // 1st way run
    flow.Map(calculateOnePoint, NUM_SAMPLES).Execute()
    flow.Reduce(count).Execute()
    var outputInterface []interface{}
    outputInterface = append(outputInterface, NUM_SAMPLES, "stdout")
    flow.Output(calculateRes, outputInterface).Execute()
}
```


#### Sort

排序算法分为两部，将乱序数组分布式排序为局部有序数组，再对多个局部有序数组进行排序，时间复杂度$$O(\log{N})$$，其中N为数组总数

##### Python

```python
def data_read(session: cf.App.Session) -> list[int]:
    pid = session.get_pid()
    num = session.get_num()
    f, range_list = session.get_source(pid, num)
    data = []
    contents = open(f).readlines()
    for i in range(range_list[0], range_list[1]):
        for str in contents[i].split(' '):
            data.append(int(str))
    return data

def sort_unordered(session: cf.App.Session, raw: list[int]) -> list[int]:
    raw.sort()
    return raw

def sort_ordered(session: cf.App.Session, raw: list[list[int]]) -> list[int]:
    res = []
    for l in raw:
        res.append(min(l))

# execution
def execute(app: cf.App):
    session = app.create_session("session_cfg.yaml")
    file = "File Path"
    session.create_source(file, type="file")
    # 1st way
    session.import_flow("cncf_workflow.yaml")
    # 2nd way
    flow = session.create_flow("wc")
    # 1st way run
    flow.input(data_read, count=20).execute()
    flow.map(sort_local, batch_size=100, count=20).execute()
    flow.reduce(wc_reduce, is_final=False).execute()
    res = flow.reduce(wc_reduce, is_final=True).execute()
    print(res)
```

#### ALS

最小二乘，执行两个flow的MapReduce问题

##### Python

```python
def data_read(session: cf.App.Session) -> list[int]:
    pid = session.get_pid()
    num = session.get_num()
    src = session.get_source(pid, num)
    data = []
    for f in src:
        data.append(f)
    return data

def update(session: cf.App.Session, i: int, mat: np.ndarray, ratings: np.ndarray) -> np.ndarray:
    uu = mat.shape[0]
    ff = mat.shape[1]
    XtX = mat.T * mat
    Xty = mat.T * ratings[i, :].T
    for j in range(ff):
        XtX[j, j] += LAMBDA * uu
    return np.linalg.solve(XtX, Xty)

def rmse(session: cf.App.Session, R: np.ndarray, ms: np.ndarray, us: np.ndarray) -> np.float64:
    diff = R - ms * us.T
    return np.sqrt(np.sum(np.power(diff, 2)) / (M * U))

def execute(app: cf.App):
    session := app.CreateSession("session_cfg.yaml")
    M, U, F, ITERATIONS, partitions = 100, 500, 10, 5, 2

    R = matrix(rand(M, F)) * matrix(rand(U, F).T)
    ms: matrix = matrix(rand(M, F))
    us: matrix = matrix(rand(U, F))

    Rs = session.share(R)
    mss = session.share(ms)
    uss = session.share(us)
    
    flow = session.create_flow("pi")

    for i in range(ITERATIONS):
        ms_ = np.array()
        session.create_source(M, type="data")
        flow.input(data_read, count="auto").execute()
        ms_ = flow.map(update, append_args=(usb.value, Rb.value), count="auto").execute()
        # collect() returns a list, so array ends up being
        # a 3-d array, we take the first 2 dims for the matrix
        ms = matrix(np.array(ms_)[:, :, 0])
        mss = session.share(ms)
        flow.clean()

        us_ = np.array()
        session.create_source(U, type="data")
        flow.input(data_read, count="auto").execute()
        us_ = flow.map(update, append_args=(msb.value, Rb.value.T), count="auto").execute()
        us = matrix(np.array(us_)[:, :, 0])
        uss = session.share(us)
        flow.clean()

        error = rmse(R, ms, us)
        print("Iteration %d:" % i)
        print("\nRMSE: %5.4f\n" % error)
```

#### K-Means

K均值算法，需要一些数据的收集整理，使用`collect`操作将数据汇总到一个节点的内存中

```python
def data_read(session: cf.App.Session) -> list[str]:
    pid = session.get_pid()
    num = session.get_num()
    f, range_list = session.get_source(pid, num)
    data = []
    contents = open(f).readlines()
    for i in range(range_list[0], range_list[1]):
        for str in contents[i].split(' '):
            data.append(str)
    return data

def parse_vector(session: cf.App.Session, line: str) -> np.ndarray:
    return np.array([float(x) for x in line.split(' ')])

def take_samples(session: cf.App.Session, data: np.ndarray, in_size: int, in_replace: bool) -> np.ndarray:
    return np.random.choice(data, size=in_size, replace=in_replace)

def closest_point(session: cf.App.Session, p: np.ndarray, centers: List[np.ndarray]) -> tuple[int, tuple[np.ndarray, int]]:
    bestIndex = 0
    closest = float("+inf")
    for i in range(len(centers)):
        tempDist = np.sum((p - centers[i]) ** 2)
        if tempDist < closest:
            closest = tempDist
            bestIndex = i
    return (bestIndex, (p, 1))

def reduce_func(session: cf.App.Session, pc_list: list[tuple[np.ndarray, int]]) -> tuple[int, tuple[np.ndarray, int]]:
    res = (np.zeros(pc_list[0][0].shape()), 0)
    for pc in pc_list:
        res[0] += pc[0]
        res[1] += pc[1]
    return res

def new_points_map(st: tuple[int, tuple[np.ndarray, int]]) -> tuple[int, tuple[np.ndarray, int]]:
    return (st[0], st[1][0]/st[1][1])

def execute(app: cf.App):
    session = app.create_session("session_cfg.yaml")
    file = "File Path"
    session.create_source(file, type="file")
    # 1st way
    session.import_flow("cncf_workflow.yaml")
    # 2nd way
    flow = session.create_flow("wc")
    # 1st way run
    converge_dist = 1e-10
    K = 10
    temp_dist = 1
    flow.input(data_read, count=20).execute()
    data = flow.branch().map(parse_vector, count="auto").execute()
    k_points = flow.branch().add(take_samples, append_args=(K, False)).collect().execute()
    while temp_dist > converge_dist:
        flow.merge().add(closest_point).execute()
        flow.reduce(reduce_func).execute()
        flow.map(new_points_map, count="auto").execute()
        new_points = flow.collect().execute()
        temp_dist = sum(np.sum((k_points[iK] - p) ** 2) for (iK, p) in new_points)
        for (iK, p) in new_points:
            k_points[iK] = p
    print("Final centers: " + str(k_points))
```

#### Logistic Regression

迭代进行的MapReduce操作

```python
def data_read(session: cf.App.Session) -> list[np.ndarray]:
    strs = list(iterator)
    matrix = np.zeros((len(strs), D + 1))
    for i, s in enumerate(strs):
        matrix[i] = np.fromstring(s.replace(',', ' '), dtype=np.float32, sep=' ')
    return [matrix]

 def gradient(session: cf.App.Session, matrix: np.ndarray, w: np.ndarray) -> np.ndarray:
    Y = matrix[:, 0]    # point labels (first column of input file)
    X = matrix[:, 1:]   # point coordinates
    # For each point (x, y), compute gradient function, then sum these up
    return ((1.0 / (1.0 + np.exp(-Y * X.dot(w))) - 1.0) * Y * X.T).sum(1)

def add(session: cf.App.Session, x: list[np.ndarray]) -> np.ndarray:
    raw = np.zeros(x[0].shape)
    for i in x:
        raw += i
    return raw

def execute(app: cf.App):
    session := app.CreateSession("session_cfg.yaml")
    files = os.listdir("File Path")
    session.create_source(files, type="files")
    D = 10  # Number of dimensions
    flow = session.create_flow("lr")
    iterations = 10
    w = 2 * np.random.ranf(size=D) - 1
    for i in range(iterations) :
        flow.input(data_read, count="auto").execute()
        flow.map(gradient, append_args=(w), count="auto").execute()
        w -= flow.reduce(add).execute()
    print("Final w: " + str(w))
```

## CLI

ascii font: slant

```sh
~ cf --cli
# command cloudflow, alias cf
> help

   ________                __________             
  / ____/ /___  __  ______/ / ____/ /___ _      __
 / /   / / __ \/ / / / __  / /_  / / __ \ | /| / /
/ /___/ / /_/ / /_/ / /_/ / __/ / / /_/ / |/ |/ / 
\____/_/\____/\__,_/\__,_/_/   /_/\____/|__/|__/  
                                                  
=====================================================================================
An Inclusive Framework Supporting MicroServices, Distributed and Serverless Computing

Usage:
    cloudflow [command]

Available Commands:
    init            CloudFlow init in working directory, add files that you can edit
                    [Usage]: init [/path/to/project]
    build           CloudFlow will build your go project
                    [Usage]: build [/path/to/project]
    config          Configure CloudFlow Platform/App/Session
                    [Usage]: config [/path/to/cf_cfg.yaml]
                    [Usage]: config app [/path/to/cf_cfg.yaml]
                    [Usage]: config session [/path/to/cf_cfg.yaml]
    dashboard       Open Dashboard to Monitor Behavior in Browser
                    [Usage]: dashboard open [-p|--port [port-num]]      # async open dashboard,
                                                                        # access from ip:[portnum]
                             dashboard close
    deploy          Deploy app
                    [Usage]: deploy [name] [options]
    invoke          Invoke Session Operation
                    [Usage]: invoke [session.py|session.go] [options]
    log             Show CloudFlow/app/session Logs
                    [Usage]: log                                        # cloudflow log
                             log app [app-name]                         # app log
                             log session [app-name:session-name]        # session log
    delete          Delete app/session
                    [Usage]: delete app [app-name]
                             delete session [app-name:session-name]
    status          Show status of CloudFlow/app/session, same as dashboard showing
                    [Usage]: status                                     # cloudflow log
                             status app [app-name]                      # app log
                             status session [app-name:session-name]     # session log
    help            Show Commands Help

In non-cli mode, all commands below can be used with `cloudflow` or `cf` command, i.e.

cf build /path/to/project
```