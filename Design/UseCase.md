# CloudFlow 使用实例/标准

## UseCase

### Common

#### Python

##### Init `cf` and `app`

```python
import cloudflow as cf
from typing import TypeAlias

def pre_run():
    cf.init("cf_cfg.yaml")                                  # When cf is not running
                                                            # simply need to configure

def run_app():
    app = cf.app.new("cf_app_cfg.yaml")
    # app.service include
    app.serve()                                             # async start app

if __name__ == "__main__":
    cf.pre_run(pre_run)
    cf.run_app(run_app)
```

##### execute session

```python
def execute(app: cf.App):
    session = app.session.new("session_cfg.yaml")
    # session.source create
    # session.filter create
    # session.flow create
    session.run()                                           # async run session

# session exection
app = cf.app.get("App Name")
app.execute(execute)
if app.session.get("Session Name").status == cf.Status.FINISHED:
    print("Execute Successfully")
    app.session.clean("Session Name")

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
import "fmt"
var cf cloudflow.CF

func preRun() {
    cf.init("cf_cfg.yaml")
}

func runApp()) {
    app := cf.app.new("cf_app_cfg.yaml")
    // app.service include
    app.serve()
}

func main() {
    cf.preRun(PreRun)
    cf.runApp(RunApp)
}
```

##### execute session

```go
func execute(app cloudflow.CF.App) {
    session := app.session.New("session_cfg.yaml")
    // session.source create
    // session.filter create
    // session.flow create
    session.Run()                                           // async run session
}

// session exection
func utility() {
    app := cf.app.Get("App Name")
    app.Execute(execute)
    if app.session.Get("Session Name").status == cf.Status.FINISHED:
        fmt.Println("Execute Successfully")
        app.session.Clean("Session Name")
    
    # other utilities
    app.ForceTerminate()
    app.Log("Path")
    app.session.ForceQuite("Session Name")
    app.session.Log("Session Name", "Path")
}
```

### Source

### Flow

如何将函数之间参数和返回值对应起来？

明确规定`flow`中函数参数顺序：

1. `session`
2. 之前函数的输出参数，需全部包含
3. 用户额外指定的参数，使用`append_args`指定

#### MapReduce: WordCount

##### Python

```python
def data_read(session: cf.App.Session) -> str:
    pid = session.get_pid()
    num = session.get_num()
    src = session.get_source(pid, num)
    data = ""
    for f in src[pid * num: (pid + 1) * num]:
        for l in open(f).readlines():
            data = l
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
def wc_reduce(session: cf.App.Session, data: dict[str, int], merge: MergeFunc) -> dict[str, int]:
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
    session app.session.new("session_cfg.yaml")
    files = os.listdir("File Path")
    session.source.init(files)
    # 1st way
    session.flow.init("cncf_workflow.yaml")
    # 2nd way
    flow = session.flow.init("wc")
    flow = flow.input(data_read, count=20)
               .map(word_count, batch_size=100, count=20)
               .reduce(wc_reduce, append_args=(merge_dict), is_final=False)
               .output(output, append_args=(False), way="stdout")
               .reduce(wc_reduce, append_args=(merge_dict), is_final=True)
               .output(output, append_args=(True), way="file", path="res.txt")

# session execution like above
```

##### Go

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
func wcReduce(session: cf.App.Session, data map[string]int, merge MergeFunc {
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

# execution
func execute(app cf.App) {
    session app.session.new("session_cfg.yaml")
    files,_ := ioutil.ReadDir("File Path")
    session.source.Init(files)
    // 1st way
    session.flow.Init("cncf_workflow.yaml")
    // 2nd way
    flow := session.flow.init("wc")
    var reduceInterface []interface{}
    reduceInterface = append(reduceInterface, merge_dict)
    var firstOutputInterface []interface{}
    firstOutputInterface = append(firstOutputInterface, false, "stdout")
    var secondOutputInterface []interface{}
    secondOutputInterface = append(secondOutputInterface, true, "file", "res.txt")
    flow = flow.input(data_read, 20)
               .map(word_count, 100, 20)
               .reduce(wc_reduce, false, reduceInterface)
               .output(output, firstOutputInterface)
               .reduce(wc_reduce, true, reduceInterface)
               .output(output, secondOutputInterface)
}

# session execution like above
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
                                                  
===============================================================================
An Inclusive Framework Supporting FLow, MicroServices and Distributed Computing

Usage:
    cloudflow [command]

Available Commands:
    build    CloudFlow will build your go project
    config
    create
    deploy
    invoke
    logs
    info
    remove
    print
    status
```