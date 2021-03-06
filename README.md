# Gores
[![Build Status](https://travis-ci.com/wang502/gores.svg?token=KeHkjMsksZ2RWDDg6h5k&branch=master)](https://travis-ci.org/wang502/gores)
An asynchronous job execution system based on Redis

## Installation
Get the package
```
$ go get github.com/wang502/gores/gores
```
Import the package
```go
import "github.com/wang502/gores/gores"
```

## Usage
### Configuration
Add a config.json in your project folder
```json
{
  "REDISURL": "127.0.0.1:6379",
  "REDIS_PW": "mypassword",
  "BLPOP_MAX_BLOCK_TIME" : 1,
  "MAX_WORKERS": 2,
  "Queues": ["queue1", "queue2"]
}
```
- ***REDISURL***: Redis server address. If you run in a local Redis, the dafault host is ```127.0.0.1:6379```
- ***REDIS_PW***: Redis password. If the password is not set, then password can be any string.
- ***BLPOP_MAX_BLOCK_TIME***: Blocking time when calling BLPOP command in Redis.
- ***MAX_WORKERS***: Maximum number of concurrent workers, each worker is a separate goroutine that execute specific task on the fetched item.
- ***Queues***: Array of queue names on Redis message broker

### Enqueue item to message broker
An item is a Go map. It is required to have several keys:
- ***Name***, name of the item to enqueue, items with different names are mapped to different tasks.
- ***Queue***, name of the queue you want to put the item in.
- ***Args***, the required arguments that you need in order for the workers to execute those tasks.
- ***Enqueue_timestamp***, the timestamp of when the item is enqueued, which is a Unix timestamp.

```go

configPath := flag.String("c", "config.json", "path to configuration file")
flag.Parse()
config, err := gores.InitConfig(*configPath)

resq := gores.NewResQ(config)
item := map[string]interface{}{
  "Name": "Rectangle",
  "Queue": "TestJob",
  "Args": map[string]interface{}{
                "Length": 10,
                "Width": 10,
          },
  "Enqueue_timestamp": time.Now().Unix(),
}

err = resq.Enqueue(item)
if err != nil {
	log.Fatalf("ERROR Enqueue item to ResQ")
}
```

```
$ go run main.go -c ./config.json -o produce
```

### Define tasks
```go
package tasks

// task for item with 'Name' = 'Rectangle'
// calculating the area of an rectangle by multiplying Length with Width
func CalculateArea(args map[string]interface{}) error {
    var err error

    length := args["Length"]
    width := args["Width"]
    if length == nil || width == nil {
        err = errors.New("Map has no required attributes")
        return err
    }
    fmt.Printf("The area is %d\n", int(length.(float64)) * int(width.(float64)))
    return err
}
```

### Launch workers to consume items
```go

flag.Parse()
config, err := gores.InitConfig(*configPath)

tasks := map[string]interface{}{
              "Item": tasks.PrintItem,
              "Rectangle": tasks.CalculateArea,
         }
gores.Launch(config, &tasks)
```

```
$ go run main.go -c ./config.json -o consume
```

### Output
```
The rectangle area is 100
```
