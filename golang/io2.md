+++
date = "2015-10-20T23:05:09+08:00"
draft = true
title = "【原创】golang 的io(二)"

+++

合理利用io包，可以优雅代码，
比如直接写beego里controller的context，可以更直观、将生产结果更好的嵌入自己的逻辑里

用法很多很多，个人日常用了集中，不断总结中，也欢迎大家给我发[pr](https://github.com/ianwoolf/myPages).

1.直接写

	func GenJsonRes(w http.ResponseWriter, status int, msg string, data interface{}) {
		w.Header().Set("Content-Type", "application/json; charset=utf-8")
		w.WriteHeader(status)
		content, err := json.Marshal(Response{Status_code: status, Msg: msg, Data: data})
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		fmt.Fprintf(w, "%s", string(content))
	}
	
2.把生成结果的的过程嵌入到自己的业务逻辑里

	type Response struct {
		Success bool          `json:"success"`
		Msg     string        `json:"msg"`
		Data    []interface{} `json:"data"`
	}
	
	func (r *Response) GenResponse(w http.ResponseWriter, status int) error {
		w.Header().Set("Content-Type", "application/json; charset=utf-8")
		w.WriteHeader(status)
		if content, err := json.Marshal(r); err != nil {
			return err
		} else {
			fmt.Fprintf(w, "%s", string(content))
		}
		return nil
	}
	
3.官网例子，输出到标准输出

	func main() {
	// A Buffer can turn a string or a []byte into an io.Reader.
		buf := bytes.NewBufferString("R29waGVycyBydWxlIQ==")
		dec := base64.NewDecoder(base64.StdEncoding, buf)
		io.Copy(os.Stdout, dec)
	}

4.复制文件时，直接用file写入file

	func CopyFile(srcName, dstName string) (written int64, err error) {
		src, err := os.Open(srcName)
		defer src.Close()
		if err != nil {
			return
		}
		dst, err := os.OpenFile(dstName, os.O_WRONLY|os.O_CREATE, 0644)
		defer dst.Close()
		if err != nil {
			return
		}
		return io.Copy(dst, src)
	}


