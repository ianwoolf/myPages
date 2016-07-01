+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】一个函数解决http发送"

+++

golang发送http请求，是用Net/http包处理。http发送请求，其实都是由client.Do这个函数处理的。其他发送函数比如 Post、PostForm、Get函数，都是经过处理后调用client.DO进行处理。所以我们其实可以经过利用一个函数处理所有常用的发送请求。
附：

    #生成请求
    func NewRequest(method, urlStr string, body io.Reader) (*Request, error)
    #执行http请求
    func (c *Client) Do(req *Request) (resp *Response, err error)
    #生成带有超时的client（其他参数请自行google）
    client := &http.Client{Timeout: time.Duration(TimeoutDuration)}
   
# body参数形式--raw/form

http中 form|raw的post，其实只是一个头文件的区别，本质都是body中的字符串。form只是编码之后，放进body。比raw的多了两步骤：编码+加header

	type Check struct {
	        Timeout    time.Duration `json:"timeout"`
	        Method     string        `json:"measurement_type"`
	        Url        string
	        PostBody   string
	        BodyType   string `json:"body_type"`
	        Result     QueryResult
	}
	
	func (check *Check) Post() error {
	        TimeoutDuration := time.Duration(check.Timeout) * time.Second
	        client := &http.Client{Timeout: time.Duration(TimeoutDuration)}
	        queryHeader := ""
	        if check.BodyType == "form" {
	                queryHeader = "application/x-www-form-urlencoded"
	        } else if check.BodyType != "raw" {
	                return errors.New("Unkown body type")
	        }
	        res, err := client.Post(check.Url, queryHeader, strings.NewReader(check.PostBody))
	        if err != nil {
	                return err
	        }
	        defer res.Body.Close()
	        check.Result.Status = res.StatusCode
	        check.Result.Body, err = ioutil.ReadAll(res.Body)
	        return err
	}
如下面调用   第一个例子提交form的时候，提交编码后的string放进body并加头，跟body的post是完全一样的。

	tmpForm := `tstring=xxx&tint=1`
	testCheck = &check.Check{Url: "http://127.0.0.1:8080/room2/", Timeout: 3, Method: "POST", BodyType: "form", PostBody: tmpForm}
	err = testCheck.DoQuery()
	if err != nil {
	        t.Error("failed when get form query", testCheck)
	}
	
	tmpJson := `{"tstring": "x", "tint": 1}`
	testCheck = &check.Check{Url: "http://127.0.0.1:8080/room1/", Timeout: 3, Method: "POST", BodyType: "raw", PostBody: tmpJson}
	err = testCheck.DoQuery()
	if err != nil {
	        t.Error("failed when get json query", testCheck)
	}
	
	
# 汇总：一个函数解决所有http请求
	
	type HttpQuery struct {
	        Timeout  int
	        Method   string
	        Url      string
	        BodyType string
	        Body     []byte
	        Result   HttpResult
	}
	
	type HttpResult struct {
	        Status int
	        Body   []byte
	}
	
	func (query *HttpQuery) DoQuery() error {
	        url, _ := url.Parse(query.Url)
	        TimeoutDuration := time.Duration(query.Timeout) * time.Second
	        client := &http.Client{Timeout: time.Duration(TimeoutDuration)}
	        req, _ := http.NewRequest(query.Method, url.String(), bytes.NewBufferString(string(query.Body)))
	
	        if query.Method != "GET" {
	                queryHeader := ""
	                if query.BodyType == "form" {
	                        queryHeader = "application/x-www-form-urlencoded"
	                } else if query.BodyType != "raw" {
	                        return errors.New("Unkown body type")
	                }
	                req.Header.Set("Content-Type", queryHeader)
	        }
	        res, err := client.Do(req)
	        if err != nil {
	                return err
	        }
	        defer res.Body.Close()
	
	        query.Result.Status = res.StatusCode
	        query.Result.Body, err = ioutil.ReadAll(res.Body)
	        if err != nil {
	                return errors.New("error in read post body")
	        }
	        return nil
	}

    func GetToJson(url string, result interface{}) (err error) {
        getQuery := &HttpQuery{
            Url:     url,
            Timeout: 5,
            Method:  "GET",
        }
        err = getQuery.DoQuery()
        if err != nil {
            fmt.Println("failed when get form query, url:", getQuery.Url, " result body:", string(getQuery.Result.Body))
            return
        }
        if err = json.Unmarshal(getQuery.Result.Body, result); err != nil {
            fmt.Println("unmarshal dc catalog/services error, url:", getQuery.Url, " result:", string(getQuery.Result.Body))
        }
        return
    }


