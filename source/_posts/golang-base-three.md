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

- `recover` 错误恢复,类似于java与php 的 try{}catch(){}
  ```go
  func TestPanicVxExit(t *testing.T) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("recover from:", err)
		}
	}()
	fmt.Println("Start")
	panic(errors.New("Something is Wrong"))
  }
  ```
- recover 成为恶魔
  - 形成僵尸服务进程，导致  health check 失效
  - "Let it Crash!" 往往是我们恢复不确定性的最好方法 简称实在不行就重启

## package 构建可复用的模块

- 基本复用单元
  - 以首字母大写来表明可被包外当代码访问
- 代码的package 可以和所在目录不一致
- 同一目录里的 Go 代码的 package 要保持一致
- init 方法
  - 在main被执行前，所有依赖的 package 的init 方法 都会被执行
  - 不同包的init 函数按照包的依赖关系决定执行顺序
  - 每个包可以有多个init 函数
  - 包的每个源文件也可以有多个 init 函数，这点很特殊
- 通过go get 来获取远程依赖
  - go get -u 每次都去拉取最新的包
  - 包是从GOPATH/src/ 开始算的 
- 注意代码在Github 上的组织形式，以适应 go get
  - 直接以代码路径开始，不要有src

## 依赖管理

- Go未解决的依赖问题
  - 同一环境下，不同项目使用同一包的不同版本
  - 无法管理对包的特定版本的依赖 
- vendor 路径
  - 随着 Go 1.5 版本的发布，vendor目录被添加到除了GOPATH 与 GOROOT 之外的依赖目录查找方案，在go 1.6以前，需要手动的设置环境变量
- 查找依赖包路径的解决方案如下
  - 1. 在当前包下的 vendor 目录
  - 2. 向上级目录查找，直到找到src下的 vendor 目录
  - 3. 在 GOPATH 下面查找
  - 4. 在 GOROOT 下面查找
- 第三方依赖管理工具
  - godep 早期 https://github.com/tools/godep
  - glide https://github.com/Masterminds/glide
  - dep 最新 https://github.com/golang/dep
  - 都是充分利用了 vendor 路径

## 协程机制

- 线程与携程 Thead vs Groutine
  - JDK5 以后 java thread stack 默认为 1M
  - Groutine 的 Stack 初始化大小为 2K
- 和 KSE（Kernel Speace Entity）的对应关系
  -  java thread 是 1:1
  -  Groutine 是 M:N 多对多
  -  系统线程切换的消耗 比较大 go与系统线程是多对多关系，无需系统的线程切换，自己去调度，尽可能的去利用系统线程的能力