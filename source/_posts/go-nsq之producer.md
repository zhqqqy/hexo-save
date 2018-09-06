---
title: go-nsq之producer
date: 2018-08-23 11:02:11
tags: nsq，go-nsq
categories: nsq
---

## Publish发送消息

Producer结构体已经nsq实现Publish发送消息功能

```go
type producerConn interface {
	String() string
	SetLogger(logger, LogLevel, string)
	Connect() (*IdentifyResponse, error)
	Close() error
	WriteCommand(*Command) error
}

type Producer struct {
	id     int64   //初始化Producer数目。原子自增
	addr   string //连接的ip地址加端口号
    conn   producerConn //接口类型，连接需要实现的方法
	config Config   //配置

	logger   logger
	logLvl   LogLevel
	logGuard sync.RWMutex 

	responseChan chan []byte
	errorChan    chan []byte
	closeChan    chan int

	transactionChan chan *ProducerTransaction //
	transactions    []*ProducerTransaction
    state           int32 //生产者状态(0：未连接,1：断开连接,2：已连接)

	concurrentProducers int32
	stopFlag            int32
	exitChan            chan int //退出信号
	wg                  sync.WaitGroup
	guard               sync.Mutex
}
//发送主题和数据	
func (w *Producer) Publish(topic string, body []byte) error {
	  return w.sendCommand(Publish(topic, body))
}

```



### 1、接收topic和数据保存到Command类型的对象

将topic和body数据封装到Command类型的对象并返回

```go
type Command struct {
    Name   []byte   //大致有(PUB,DPUB,MPUB,SUB,RDY,REQ,FIN,CLS,NOP,TOUCH,PING,REGISTER,UNREGISTER,AUTH,IDENTIFY)全都在command.go中
	Params [][]byte //topic数组
	Body   []byte	//消息
}
//Publish()--->1.Publish，params存放topic,Name是PUB,body是发送的消息数据
func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}
```

### 2、sendCommand接收Publish函数的返回值(cmd *Command)

将封装了topic和body数据的Command类型的对象发送给sendCommandAsyn函数

```go
//Publish()--->2.sendCommand
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)
    //异步的发送Command
	err := w.sendCommandAsync(cmd, doneChan, nil)
	if err != nil {
		close(doneChan)
		return err
	}
    //阻塞，直到doneChan被关闭(sendCommandAsync返回值err不为nil的时候)
	t := <-doneChan
	return t.Error
}
```

#### 1、sendCommandAsync函数

接收封装了topic和body数据的Command类型的对象还有ProducerTransaction类型的chan

```go
const (
	StateInit = iota
	StateDisconnected
	StateConnected
)
func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction,args []interface{}) error {
    //跟踪我们正在处理多少producers，以便以后确保我们清理它们......,通过原子累加来计数
	atomic.AddInt32(&w.concurrentProducers, 1)
	defer atomic.AddInt32(&w.concurrentProducers, -1)
	//StateConnected默认是2
	if atomic.LoadInt32(&w.state) != StateConnected {
        // <<<<<<<<<<<<<<<<<<<<如果没有连接，建立连接,跳转到下面>>>>>>>>>>>>>>>>>>>>>
		err := w.connect()
		if err != nil {
			return err
		}
	}
    //构建一个ProducerTransaction，赋值cmd，doneChan(空的)，args(这里是nil)
	t := &ProducerTransaction{
		cmd:      cmd,
		doneChan: doneChan,
		Args:     args,
	}

	select {
        //将数据封装后又传给Producer的transactionChan，会在w.connect()中的w.router()函数被消费
	case w.transactionChan <- t:
	case <-w.exitChan:
		return ErrStopped
	}

	return nil
}
```

###  3、w.connect(),内部发送数据流程

```go
// keeps the exported Producer struct clean of the exported methods
// required to implement the ConnDelegate interface
type producerConnDelegate struct {
	w *Producer
}

func (w *Producer) connect() error {
    //加锁保证当前有线程执行的时候，其他线程无法执行,否则可能会在读取state的时候出现多次Init（因为没有原子增加操作）
	w.guard.Lock()
	defer w.guard.Unlock()
	//原子性，保证加载的时候当前计算机中的任何CPU都不会进行其它的针对此值的读或写操作
    //这样的约束是受到底层硬件的支持的。
	if atomic.LoadInt32(&w.stopFlag) == 1 {
		return ErrStopped
	}
    //查看当前状态，是0则直接执行下面的函数(执行发送数据操作)，2的话断开连接，返回nil，其他的话就返回error
	switch state := atomic.LoadInt32(&w.state); state {
	case StateInit:
	case StateConnected:
		return nil
	default:
		return ErrNotConnected
	}
	w.log(LogLevelInfo, "(%s) connecting to nsqd", w.addr)

	logger, logLvl := w.getLogger()
	//新建连接
	w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})
	w.conn.SetLogger(logger, logLvl, fmt.Sprintf("%3d (%%s)", w.id))
	//创建连接，并发送数据
	_, err := w.conn.Connect()
	if err != nil {
		w.conn.Close()
		w.log(LogLevelError, "(%s) error connecting to nsqd - %s", w.addr, err)
		return err
	}
    //连接成功，状态转换为连接状态
	atomic.StoreInt32(&w.state, StateConnected)
	w.closeChan = make(chan int)
	w.wg.Add(1)
	go w.router()

	return nil
}

```

#### 1、NewConn 

没有建立连接则生成*Conn类型对象

```go
//ConnDelegate是个接口需要实现OnResponse，OnError，OnMessage，OnMessageFinished，OnMessageRequeued，OnBackoff，OnContinue，OnResume，OnIOError，OnHeartbeat，OnClose这些方法
func NewConn(addr string, config *Config, delegate ConnDelegate) *Conn {
	if !config.initialized {
		panic("Config must be created with NewConfig()")
	}
	return &Conn{
		addr: addr,

		config:   config,
		delegate: delegate,

		maxRdyCount:      2500,
		lastMsgTimestamp: time.Now().UnixNano(),

		cmdChan:         make(chan *Command),
		msgResponseChan: make(chan *msgResponse),
		exitChan:        make(chan int),
		drainReady:      make(chan int),
	}
}

```

#### 2、w.conn.Connect() 

与服务端建立连接，并开启循环**读写**队列

```go
// Connect dials and bootstraps the nsqd connection
// (including IDENTIFY) and returns the IdentifyResponse
//conn   producerConn
type producerConn interface {
	String() string
	SetLogger(logger, LogLevel, string)
	Connect() (*IdentifyResponse, error)
	Close() error
	WriteCommand(*Command) error
}
//cnn.go下的Conn实现了producerConn接口
type Conn struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	messagesInFlight int64
	maxRdyCount      int64
	rdyCount         int64
	lastRdyCount     int64
	lastRdyTimestamp int64
	lastMsgTimestamp int64

	mtx sync.Mutex

	config *Config

	conn    *net.TCPConn
	tlsConn *tls.Conn
	addr    string

	delegate ConnDelegate

	logger   logger
	logLvl   LogLevel
	logFmt   string
	logGuard sync.RWMutex

	r io.Reader
	w io.Writer

	cmdChan         chan *Command
	msgResponseChan chan *msgResponse
	exitChan        chan int
	drainReady      chan int

	closeFlag int32
	stopper   sync.Once
	wg        sync.WaitGroup

	readLoopRunning int32
}

//Connect方法
func (c *Conn) Connect() (*IdentifyResponse, error) {
    //创建Dialer变量
	dialer := &net.Dialer{
		LocalAddr: c.config.LocalAddr,
		Timeout:   c.config.DialTimeout,
	}
    //和服务端建立tcp连接，连接到指定的网络地址
	conn, err := dialer.Dial("tcp", c.addr)
	if err != nil {
		return nil, err
	}
	c.conn = conn.(*net.TCPConn)
    //c.r和c.w是io.Reader和io.Write接口类型，net.TCPConn实现了这两个接口
	c.r = conn
	c.w = conn
	//将数据"  V2"写入底层缓冲中, var MagicV2 = []byte("  V2")
	_, err = c.Write(MagicV2)
	if err != nil {
		c.Close()
		return nil, fmt.Errorf("[%s] failed to write magic - %s", c.addr, err)
	}
	//将数据flush 将所有的数据刷到底层网络连接中。
	resp, err := c.identify()
	if err != nil {
		return nil, err
	}
	//身份认证
	if resp != nil && resp.AuthRequired {
		if c.config.AuthSecret == "" {
			c.log(LogLevelError, "Auth Required")
			return nil, errors.New("Auth Required")
		}
		err := c.auth(c.config.AuthSecret)
		if err != nil {
			c.log(LogLevelError, "Auth Failed %s", err)
			return nil, err
		}
	}

	c.wg.Add(2)
   
	atomic.StoreInt32(&c.readLoopRunning, 1)
    //开一个goroutine内部是for循环读取数据
	go c.readLoop()
      //开一个goroutine内部是for循环从cmdChan读取数据到cmd，并写入缓冲，并flush到网络连接
	go c.writeLoop()
	return resp, nil
}
```

#### 3、w.router()

将Publish的command数据写入到底层缓冲队列,等待flush被发送到网络上

```go
func (w *Producer) router() {
	for {
		select {
            //消费(w *Producer) sendCommandAsync(...)中的chan
		case t := <-w.transactionChan:
			w.transactions = append(w.transactions, t)
			err := w.conn.WriteCommand(t.cmd)
			if err != nil {
				w.log(LogLevelError, "(%s) sending command - %s", w.conn.String(), err)
				w.close()
			}
		case data := <-w.responseChan:
			w.popTransaction(FrameTypeResponse, data)
		case data := <-w.errorChan:
			w.popTransaction(FrameTypeError, data)
		case <-w.closeChan:
			goto exit
		case <-w.exitChan:
			goto exit
		}
	}

exit:
	w.transactionCleanup()
	w.wg.Done()
	w.log(LogLevelInfo, "exiting router")
}
```

