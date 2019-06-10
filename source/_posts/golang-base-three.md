---
title: golang base (three)
date: 2019-06-09 21:43:37
categories: golang
tags: 
- golang
---
## error 处理

- 没有异常机制
- error 类型实现了 error 接口
  
  ```go
  type error interface{
      Error string
  }
  ```

- 可以通过 errors.New 来快速创建错误实例
  
  ```go
    errors.New(" error info")
    var NumberLess2Error =  errors.New("n must be not less than 2")
    var NumberLarger100Error =  errors.New("n must be not larger than 100")
    func GetFibonacci(n int) (c []int, err error) {
        // 及早失败
        if n < 2  {
            err = NumberLess2Error
            return
        }
        if n > 100  {
            err = NumberLarger100Error
            return
        }
        c = append(c, 1,1)
        a, b := 1, 1
        for i := 0; i < n; i++ {
            a, b = b, a+b
            c = append(c, b)
        }
        return c, nil
    }

    func TestGetFibonacci(t *testing.T) {
        if n,err := GetFibonacci(-10);err!=nil{
            if err == NumberLess2Error{
                fmt.Printf("less ")
            }
            t.Error(err)
        }else {
            t.Log(n)
        }
    }
  ```

## `panic` 与 `recover`

- `panic` 与 `os.Exit`
  - `os.Exit` 退出时不会调用 defer 指定的函数
  - `os.Exit` 退出时不输出当前调用栈信息

- `recover` 错误恢复

- recover 成为恶魔
  - 形成僵尸服务进程，导致  health check 失效
  - "Let it Crash!" 往往是我们恢复不确定性的最好方法 简称实在不行就重启
