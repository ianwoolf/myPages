+++
date = "2016-10-06T16:30:09+08:00"
draft = true
title = "【分析】hold住百万qps"

+++

ndling 1 Million Requests per Minute with Go](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/)。

原文展示了一个demo，从简单的处理http请求逐渐演进到高qps的一个历程。



### 原始处理

最一开始仅仅是简单的http请求，采用routine进行处理。这种方式是非常糟糕的，原因是无法控制routine的数量，在高并发请求的时候程序很容易崩溃。

show you the code：



```
type PayloadCollection struct {   
	WindowsVersion  string    json:"version"   
	Token           string    json:"token"   
	Payloads        []Payload json:"data"   
}   

type Payload struct {   
    // [redacted]   
}   

func (p *Payload) UploadToS3() error {   
    // the storageFolder method ensures that there are no name collision in   
    // case we get same timestamp in the key name   
    storage_path := fmt.Sprintf("%v/%v", p.storageFolder, time.Now().UnixNano())   
	bucket := S3Bucket   
	b := new(bytes.Buffer)   
	encodeErr := json.NewEncoder(b).Encode(payload)   
	if encodeErr != nil {   
		return encodeErr   
	}   
    // Everything we post to the S3 bucket should be marked 'private'   
    var acl = s3.Private   
	var contentType = "application/octet-stream"   
	return bucket.PutReader(storage_path, b, int64(b.Len()), contentType, acl, s3.Options{})   
}           

func payloadHandler(w http.ResponseWriter, r *http.Request) {   
    if r.Method != "POST" {   
		w.WriteHeader(http.StatusMethodNotAllowed)   
		return   
	}   
    // Read the body into a string for json decoding   
	var content = &PayloadCollection{}   
	err := json.NewDecoder(io.LimitReader(r.Body, MaxLength)).Decode(&content)   
    if err != nil {   
		w.Header().Set("Content-Type", "application/json; charset=UTF-8")   
		w.WriteHeader(http.StatusBadRequest)   
		return   
	}   
    // Go through each payload and queue items individually to be posted to S3   
    for _, payload := range content.Payloads {   
        go payload.UploadToS3()   // <----- DON'T DO THIS   
    }   
    w.WriteHeader(http.StatusOK)   
}
```



### 简单演进

然后经过简单演进后，接受到请求后，会把任务压进channle（channle固定长度，否则情况和之前一样，以此开控制routine数量）。

show you the code:



```
var Queue chan Payload

func init() {
    Queue = make(chan Payload, MAX_QUEUE)
}

func payloadHandler(w http.ResponseWriter, r *http.Request) {
    ...
    // Go through each payload and queue items individually to be posted to S3
    for _, payload := range content.Payloads {
        Queue <- payload
    }
    ...
}

func StartProcessor() {
    for {
        select {
        case job := <-Queue:
            job.payload.UploadToS3()  // <-- STILL NOT GOOD
        }
    }
}
```



**这样仍旧会有问题**，因为队列长度限制了消费能力。数据量大的时候，buffer只是延缓了情况，但是问题并没有任何改善，仅仅是延后了而已！

情况如图：

<img src="http://marcio.io/img/cloudwatch-latency.png">



### 最终解决办法

最终采用了两层channle，一个用来缓存job，一个用来控制worker的数量。

```
var (
	MaxWorker = os.Getenv("MAX_WORKERS")
	MaxQueue  = os.Getenv("MAX_QUEUE")
)    

// Job represents the job to be run
type Job struct {
	Payload Payload
}    

// A buffered channel that we can send work requests on.
var JobQueue chan Job    

// Worker represents the worker that executes the job
type Worker struct {
	WorkerPool  chan chan Job
	JobChannel  chan Job
	quit    	chan bool
}    

func NewWorker(workerPool chan chan Job) Worker {
	return Worker{
		WorkerPool: workerPool,
		JobChannel: make(chan Job),
		quit:       make(chan bool)}
}    

// Start method starts the run loop for the worker, listening for a quit channel in
// case we need to stop it
func (w Worker) Start() {
	go func() {
		for {
			// register the current worker into the worker queue.
			w.WorkerPool <- w.JobChannel    

			select {
			case job := <-w.JobChannel:
				// we have received a work request.
				if err := job.Payload.UploadToS3(); err != nil {
					log.Errorf("Error uploading to S3: %s", err.Error())
				}    

			case <-w.quit:
				// we have received a signal to stop
				return
			}
		}
	}()
}    

// Stop signals the worker to stop listening for work requests.
func (w Worker) Stop() {
	go func() {
		w.quit <- true
	}()
}
```



httpHandler处理之后吧`Job`压进channle`JobQueue`，交给`worker`去处理 。

```
func payloadHandler(w http.ResponseWriter, r *http.Request) { \
    if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}    

    // Read the body into a string for json decoding
	var content = &PayloadCollection{}
	err := json.NewDecoder(io.LimitReader(r.Body, MaxLength)).Decode(&content)
    if err != nil {
		w.Header().Set("Content-Type", "application/json; charset=UTF-8")
		w.WriteHeader(http.StatusBadRequest)
		return
	}    

    // Go through each payload and queue items individually to be posted to S3
    for _, payload := range content.Payloads {    

        // let's create a job with the payload
        work := Job{Payload: payload}    

        // Push the work onto the queue.
        JobQueue <- work
    }    

    w.WriteHeader(http.StatusOK)
}
```



server初始化的时候，会初始化`dispatcher`并且调用`Run()`，会监听`JobQueue`中出现的`Job`。 

```
dispatcher := NewDispatcher(MaxWorker)
dispatcher.Run()
```



```
type Dispatcher struct {
	// A pool of workers channels that are registered with the dispatcher
	WorkerPool chan chan Job
}    

func NewDispatcher(maxWorkers int) *Dispatcher {
	pool := make(chan chan Job, maxWorkers)
	return &Dispatcher{WorkerPool: pool}
}    

func (d *Dispatcher) Run() {
    // starting n number of workers
	for i := 0; i < d.maxWorkers; i++ {
		worker := NewWorker(d.pool)
		worker.Start()
	}    

	go d.dispatch()
}    

func (d *Dispatcher) dispatch() {
	for {
		select {
		case job := <-JobQueue:
			// a job request has been received
			go func(job Job) {
				// try to obtain a worker job channel that is available.
				// this will block until a worker is idle
				jobChannel := <-d.WorkerPool    

				// dispatch the job to the worker job channel
				jobChannel <- job
			}(job)
		}
	}
}
```

注意这里采用环境变量的方式来限定最大`worker`数量和`job`队列数量。

```
var (
    MaxWorker = os.Getenv("MAX_WORKERS")
    MaxQueue  = os.Getenv("MAX_QUEUE")
)
```

效果是如此的明显

<img src="http://marcio.io/img/cloudwatch-console.png">


