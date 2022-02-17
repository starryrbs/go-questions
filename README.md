# GO常见的使用场景和问题解决方案

## 并发
1. 最大任务数的并发控制
    
   - waitgroup + chan实现的并发控制
   
       ```go
       count := 100
       sum := 1000
       wg := sync.WaitGroup{}
    
       message := make(chan struct{}, count)
       defer close(message)
    
       for i := 0; i < sum; i++ {
           wg.Add(1)
           message <- struct{}{}
           go func(j int) {
               defer wg.Done()
               time.Sleep(time.Second)
               fmt.Printf("get value %v\n", j)
               <-message
           }(i)
       }
       wg.Wait()
       ```
     
   - 第三方库`github.com/Jeffail/tunny`实现
    
      ```go
      numCpu := runtime.NumCPU()
      fmt.Printf("cpu count : %d\n", numCpu)
      pool := tunny.NewFunc(numCpu, func(i interface{}) interface{} {
         fmt.Printf("input %v\n", i)
         time.Sleep(time.Second)
         return i
      })
      defer pool.Close()
    
      wg := sync.WaitGroup{}
      for i := 0; i < 80; i++ {
         wg.Add(1)
         go func(j int) {
             defer wg.Done()
             pool.Process(j)
         }(i)
      }
      wg.Wait()
      ```
   
     - 第三方库`github.com/panjf2000/ants`实现

2. 并发从多个数据库读取，并返回收到的第一个响应

```go
func Query(conns []*Connection) int {
	resultCh := make(chan int, 1)

	for _, conn := range conns {
		go func(c *Connection) {
			select {
			case resultCh <- c.DoQuery():
			default:
			}
		}(conn)
	}
	return <-resultCh
}
```

## 设计模式
1. 实现单例模式

    - 使用atomic和mutex实现的并发安全的单例
    
        ```go
        type singleton struct{}
        
        var mu sync.Mutex
        var done uint32
        
        var instance *singleton
        
        func New() *singleton {
            // 添加if判断减少加锁开销，使用atomic做原子判断
            if atomic.LoadUint32(&done) == 1 {
                return instance
            }
            mu.Lock()
            defer mu.Unlock()
            if done == 0 {
                fmt.Printf("done : %d\n", done)
                instance = &singleton{}
                atomic.StoreUint32(&done, 1)
            }
            return instance
        }
        ```
   
   - 使用sync.Once实现的单例

        ```go
        type singleton struct{}
        
        var once sync.Once
        var instance *singleton
        
        func New() *singleton {
            once.Do(func() {
                instance = &singleton{}
            })
            return instance
        }
        ```

## time
1. 耗时统计

    ```go
    startTime := time.Now()
    cost := time.Since(startTime)
    fmt.Printf("耗时: %v", cost)
    ```

2. time.Ticker使用

3. 超时控制

   - 使用time.After实现
   
     ```go
     func GetIP(timeout time.Duration) (ip string, err error) {
         ch := make(chan error, 1)
    
         go func() {
             time.Sleep(timeout * 2)
             ch <- nil
         }()
    
         select {
         case <-ch:
             return "1.1.1.1", nil
         case <-time.After(timeout):
             return "", errors.New("timeout")
         }
     }
    
     func main() {
         ip, err := GetIP(time.Second * 1)
         if err != nil {
             panic(err)
         }
         fmt.Printf("get ip : %v", ip)
     }
     ```

## map

1. map拷贝

   - 通过for循环进行拷贝

        ```go
        a := make(map[string]int)
        a["haha"] = 1
        
        newMap := make(map[string]int)
        for k, v := range a {
            newMap[k] = v
        }
        fmt.Printf("map: %v", newMap)
       ```
   - 考虑深拷贝的问题

       ```go
        a := make(map[string]*int)
        num := 1
        a["haha"] = &num
        
        newMap := make(map[string]*int)
        for k, v := range a {
            n := *v
            newMap[k] = &n
        }
        println(&num)
        num = 2
        fmt.Printf("map: %v", newMap)
       ```
   - 使用gob编解码实现深拷贝（json.Marshal与json.UnMarshal类似）

        ```go
        func Map(m map[string]interface{}) (map[string]interface{}, error) {
            var buf bytes.Buffer
            enc := gob.NewEncoder(&buf)
            dec := gob.NewDecoder(&buf)
            err := enc.Encode(m)
            if err != nil {
                return nil, err
            }
            var mapCopy map[string]interface{}
            err = dec.Decode(&mapCopy)
            if err != nil {
                return nil, err
            }
            return mapCopy, nil
        }
        ```

3. map并发安全

4. sync.Map
## 调试与问题排查

1. pprof