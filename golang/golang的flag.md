+++
date = "2016-01-07T23:05:09+08:00"
draft = true
title = "【原创】golang 的flag -- 标准类型和自定义类型"

+++
### golang 的flag
##### 自定义usage
flag可以通过自定义Usage()的方式来自定义usage的输出。注意`flag.PrintDefaults()`用来输出参数的help信息。

    var usageMsg string = `Usage of %s:zssh [flag] commands
    provide hosts by pipline or -hosts
    flag.Usage = func() {
		fmt.Fprintf(os.Stderr, usageMsg, os.Args[0])
		flag.PrintDefaults()
	}

##### flag
可以通过`TypeVar`和`Var`两种方式来定义flag参数：

- 注意`TypeVar`可以定义默认值
- `Var`可以定义自定义变量（如[]string）等的flag参数，但是无法定义默认值。

两种方式的代码如下：

    type FlagParam []string

    func (f *FlagParam) String() string {
        return "string method"
    }

    func (f *FlagParam) Set(value string) error {
        *f = strings.Split(value, ",")
        return nil
    }
    var (
        Hosts    FlagParam
	    port     int
	    Count   int
	    user string
	
	    usageMsg string = `Usage of %s:zssh [flag] commands
            provide hosts by pipline or -hosts
   
           `
    )

    func parasFlag() []string {
	    flag.Usage = func() {
	    	fmt.Fprintf(os.Stderr, usageMsg, os.Args[0])
		flag.PrintDefaults()
	    }
	    flag.Var(&Hosts, "hosts", "host list. e.g: host1,host2.  if not provide, please given in stdin")
	    flag.IntVar(&port, "p", 22, "Port of ssh")
	    flag.IntVar(&Count, "c", runtime.NumCPU(), "Parallel number, will run parallel on all the hosts if c=0")
	    flag.StringVar(&user, "l", os.Getenv("USER"), "Username for ssh")
	    flag.Parse()
	    return flag.Args()
	}
	func main() {
		args := parasFlag()
        fmt.Println(Hosts)
        fmt.Println(args)
    }
