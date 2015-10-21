+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】无缓存channel及日常用法"

+++
##### 当没有对channel读取的操作的时候，发送操作会阻塞。看下面例子

	package main
	import (
	"fmt"
	"time"
	)
	
	var (
	    messages = make(chan string)//无缓存channel
	    signals = make(chan bool)//无缓存channel
	)
	
	func res(cs chan string) {
	    fmt.Println("begin listin...")
	    for {
	       msg := <-cs//监听channel
	       fmt.Println("received message", msg)
	    }
	}
	
	func main() {
	    go res(messages)//监听channel
	    time.Sleep(1 * 1e9)// 注释掉这句，就send fail，执行default。，为啥
	
	    msg := "hi 2005"
	    select {
	    case messages <- msg:
	        fmt.Println("sent message", msg)
	    default:
	        fmt.Println("no message sent")
	    }
	
	    select {
	    case msg := <-messages:
	        fmt.Println("received message", msg)
	    case sig := <-signals:
	        fmt.Println("received signal", sig)
	    default:
	        fmt.Println("no activity")
	    }
	time.Sleep(1 * 1e9)
	}
	
##### close channel
表明没有数据再发往通道了（任务完成）。这对于channel接受者是一个信号，可以被用来表明任务完成。
    
	package main
	import "fmt"
	func main() {
    jobs := make(chan int, 5)
    done := make(chan bool)
	    go func() {
        for {
            j, more := <-jobs
            if more {
                fmt.Println("received job", j)
            } else {
                fmt.Println("received all jobs")
                done <- true
                return
            }
        }
    }()
	    for j := 1; j <= 3; j++ {
        jobs <- j
        fmt.Println("sent job", j)
    }
    close(jobs)
    fmt.Println("sent all jobs")
	    <-done
	}
结果：    

	$ go run closing-channels.go 
	sent job 1
	received job 1
	sent job 2
	received job 2
	sent job 3
	received job 3
	sent all jobs
	received all jobs

##### Range over channel
range channle之前，需要close掉channel，否则会hang住

	package main
	import "fmt"
	func main() {
	    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
	        //This range iterates over each element as it’s received fromqueue. 
            //Because we closed the channel above, the iteration terminates after receiving the 2 elements. 
            //If we didn’tclose it we’d block on a 3rd receive in the loop.
	    close(queue)
	    for elem := range queue {
        fmt.Println(elem)
    }
	}
结果：

	$ go run range-over-channels.go
	one
	two
	
##### 生产者+消费者：

	var done = make(chan bool)
	var msgs = make(chan int)
	func produce() {
		for i := 0; i < 10; i++ {
    	    msgs <- i
		}
		fmt.Println("Before closing channel")
		close(msgs)
		fmt.Println("Before passing true to done")
		done <- true
		}
	func consume() {
		for {
    	    msg := <-msgs
    	    time.Sleep(100 * time.Millisecond)
    	    fmt.Println("Consumer: ", msg)
     	  }
		}
	func main() {
		go produce()
		go consume()
		<-done
		fmt.Println("After calling DONE")
	}
    
结果：

	Consumer:  0
	Consumer:  1
	Consumer:  2
	Consumer:  3
	Consumer:  4
	Consumer:  5
	Consumer:  6
	Consumer:  7
	Consumer:  8
	Before closing channel
	Before passing true to done
	After calling DONE	sending

这里由于是无缓存channel，前一个没有读取的时候，再发送会阻塞，类似fifo。同样接收也会卡住，为了验证，增加输出，变成如下代码：

	func produce() {
    	for i := 0; i < 4; i++ {
    	    fmt.Println("sending")
    	    msgs <- i
    	    fmt.Println("sent")
   		}
    	fmt.Println("Before closing channel")
    	close(msgs)
    	fmt.Println("Before passing true to done")
    	done <- true
	}
	func consume() {
    	for msg := range msgs {
     	    fmt.Println("Consumer: ", msg)
      	    time.Sleep(100 * time.Millisecond)
		}
	}

结果

	Consumer:  0
	sent
	sending
	Consumer:  1
	sent
	sending
	Consumer:  2
	sent
	sending
	Consumer:  3
	sent
	Before closing channel
	Before passing true to done
	After calling DONE
    
 用close做完成通知,执行结果无返回
	package main
	
	import (
	        "fmt"
	        "time"
	)
	
	var done = make(chan bool)
	var msgs = make(chan int)
	
	func produce(num int) {
	        for i := 0; i < num; i++ {
	                msgs <- i
	                time.Sleep(2 * time.Second)
	        }
	        done <- true
	}
	
	func consume() {
	        for {
	                select {
	                case msg, status := <-msgs:
	                        if status {
	                                fmt.Println("Consumer: ", msg)
	                        } else {
	                                fmt.Println("all worker is done")
	                                return
	                        }
	                case <-time.After(1 * time.Second):
	                        fmt.Println("1s")
	                case <-done:
	                        fmt.Println("all reveived is done")
	                }
	        }
	}
	
	func main() {
	        num := 5
	        go produce(num)
	        go consume()
	        time.Sleep(time.Duration(num) * 3 * time.Second)
	        close(msgs)
	        fmt.Println("quit....")
	}
	
【常用】执行结果用channel通知执行结果 
	package main
	
	import (
	        "fmt"
	        "time"
	)
	
	type Result struct {
	        Success bool
	        Msg     string
	}
	
	var WorkResult = make(chan Result)
	var msgs = make(chan int)
	
	func produce(num int) {
	        for i := 0; i < num; i++ {
	                msgs <- i
	                time.Sleep(2 * time.Second)
	        }
	}
	
	func consume() {
	        for {
	                select {
	                case msg, status := <-msgs:
	                        if status {
	                                fmt.Println("Consumer: ", msg)
	                                WorkResult <- Result{Msg: "work is done", Success: true}
	                        } else {
	                                fmt.Println("work channel is close")
	                                return
	                        }
	                case <-time.After(1 * time.Second):
	                        fmt.Println("1s have no work to reseive")
	                }
	        }
	}
	
	func main() {
	        num := 5
	        go produce(num)
	        go consume()
	        for resultCnt := 0; resultCnt < num; resultCnt++ {
	                workResult := <-WorkResult
	                if workResult.Success {
	                        fmt.Println("work success:", workResult.Msg)
	                } else {
	                        fmt.Println("work fail:", workResult.Msg)
	                }
	                fmt.Printf("reseive %d result, all is%d\n", resultCnt+1, num)
	        }
	        //time.Sleep(time.Duration(num) * 4 * time.Second)
	        fmt.Println("all work return, reseive is done")
	        close(msgs)
	        fmt.Println("quit....")
	}

