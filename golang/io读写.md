+++
date = "2016-01-07T23:05:09+08:00"
draft = true
title = "【原创】golang 的io读写"

+++

## 第一种读： 转为bufio.reader
	bio := bufio.NewReader(os.Stdin)
	line, _, err := bio.ReadLine()
	fmt.Println(line, err)

## 第二种读：io实现了read，读取len()长度的内容
	p := make([]byte, 11)
	readLen, _ := os.Stdin.Read(p)
	fmt.Println("read len from stdin:", readLen)
	line := string(p)

## 第三种读： ioutil
        // 这种方法会一直读到EOF，注意   管道进去或者读文件
	readb, _ := ioutil.ReadAll(os.Stdin)
	fmt.Println("the third read, ioutil.Readall() result:", string(readb))

## 第一种写，fmt写进io.Write
	w := bufio.NewWriter(os.Stdout)
	fmt.Fprint(w, string(line))
	w.Flush() // Don't forget to flush!

## 第二种写： io.write的方法
	os.Stdout.Write(p)

## 第三种： io.copy  不用读，直接写
	io.Copy(os.Stdout, os.Stdin)
