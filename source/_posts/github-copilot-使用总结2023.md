---
title: github-copilot-使用总结-2023
abbrlink: 74d092af
mermaid: true
date: 2023-06-24 23:01:10
tags:
  - ai
  - github-copilot
categories: ai
index_img: /img/post/04.png
---
## 介绍
GitHub Copilot 是一个ai辅助编程工具（ide 插件）， 底层由 OpenAI Codex(通过对 github 数十亿行公共代码进行训练而成) 驱动。

Copilot 通过读取注释跟代码中来形成上下文，通过上下文来生成单行或整个函数的代码。  

具体功能就是会根据你的输入提供代码补全，你可以选择接受或拒绝这些自动生成的代码。

### 它是如何工作的？

整体来看，Copilot 由 底层的 Codex 模型（用于生成代码）， 中间的服务层（用于将用户输入转换为 Codex 模型的输入，以及将 Codex 模型的输出转换为用户可用的代码）还有上层的 ide 插件也就是用户交互层组成。

```mermaid
graph LR
A[用户] --> B[ide 插件]
B --> C[服务层]
C --> D[Codex 模型]
```
## 使用场景与技巧演示

Copilot 的主要功能只有一个，就是根据上下文推测接下来的文本，跟 chatgpt 一样，属于万能的模型，可以认为是专为编程微调的 GPT-3 模型。其他领域具体玩法各位应该可以各自探索。

在聊编程三件套，**代码&注释**，**测试**，**文档** 这些之前，先来看代理配置与快捷键配置。

### 配置代理

为什么要把这个放在最前面，因为代理的响应速度会极大的影响 coding 体验。<sup><a href="#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99:~:4.&text=GitHub%20Copilot%EF%BC%9A%E5%81%9A%E5%87%BA%E4%B8%80%E4%B8%AA%E5%88%92%E6%97%B6%E4%BB%A3%E7%9A%84%E4%BA%A7%E5%93%81%EF%BC%8C%E5%8F%AA%E9%9C%80%E8%A6%81%206%20%E4%B8%AA%E4%BA%BA" id="ref1">[^4]</a></sup>    

vscode 配置
![/img/github-copilot-proxy-01](/img/github-copilot-proxy-01.png)

idea 配置
![/img/github-copilot-proxy-02](/img/github-copilot-proxy-02.png)

### idea 中的快捷键

- 触发行内 `Alt/Option + \` 手动触发 copilot 建议
- 下一条建议 `Alt/Option + ]`
- 上一条建议 `Alt/Option + [`
- 接受建议：`Tab `
- 拒绝建议：`Esc`

好了你学会了，可以去试试了，下面是一些使用场景与技巧演示。

### 代码&注释

#### 注释&变量命名
这个是最常见的，比如我有一个 status 有 5个 状态,我需要定义5个常量，这时候我可以这样写
```php
// status 有 5个 状态 ，正常，异常，警告，错误，未知
const STATUS_NORMAL = 1; # 正常

```
![/img/github-copilot-01](/img/github-copilot-01.png)
根据注释，Copilot 会自动帮你补全常量的定义。
![/img/github-copilot-02](/img/github-copilot-02.png)
根据注释，Copilot 会自动帮你补全注释。

- **重点 copilot 会学习你的代码风格，会读取你光标所在位置跟 ide 打开页面的上下文，所以你要写的代码跟这些上下文关联越紧密，copilot 生成的代码就越准确。** <sup><a href="#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99:~:text=2023%2D05%2D17%2Dhow%2Dgithub%2Dcopilot%2Dis%2Dgetting%2Dbetter%2Dat%2Dunderstanding%2Dyour%2Dcode" id="ref1">[^3]</a></sup>


#### 代码转换
```javascript
// $str = strtoupper("hello world");
// ---- translate to javascript 以下 为 copilot 生成的代码 ----
let str = "hello world".toUpperCase();
```
- 技巧：
    - 当你需要使用其他语言，但你不知道api时，你可以先用自己熟悉的语言写出来，然后让 copilot 帮你转换成其他语言的代码，这样你就知道怎么使用了
    - 其实用伪代码也可以，只要 copilot 能 get 你的意思就行

#### 短逻辑补全
比如我想要判断当前时间是否大于创建时间 1 小时，如果大于就抛出异常，那么我可以这样写
```php
// current time must > created time 1 hour  throw exception 中文提示
if (time() > $createdTime + 3600) {
    throw new Exception('当前时间必须大于创建时间 1 小时');
}
```
![/img/github-copilot-03](/img/github-copilot-03.png)

需要定义异常中响应的格式

```php
// current time must > created time 1 hour else tips 系统处理中，请在 n 分钟 后操作。n = 60 - (当前时间 - 创建时间) / 60
if (time() > $createdTime + 3600) {
    throw new Exception('系统处理中，请在 ' . (60 - (time() - $createdTime) / 60) . ' 分钟 后操作');
}
```
![/img/github-copilot-04](/img/github-copilot-04.png)

- 小技巧：
	- copilot 能读取你的上下文，但是 对于提示文案是无能为力的，如果你强制格式，就请把文案写出来
	- prompt 用 伪代码跟结构化的数据，输入少 响应反而更加精确

#### 错误修复
```php
$header = ["id", "name", "other"];  
$mapData = [
    ["1", "张三", "其他"],
    ["2", "李四", "其他"],
];
// 导出到本地
$fileName = "list" . date("YmdHis") . ".csv";  
$filePath = storage_path("app/" . $fileName);

// ----------- 以下为 copilot 生成的代码 ------------
$fp = fopen($filePath, "w");
fputcsv($fp, $header);
foreach ($mapData as $item) {
    fputcsv($fp, $item);
}
fclose($fp);
```
生成的代码，看起来没有什么问题，但是实际上是有问题的，比如导致导出的文件是乱码的，这时候我们需要让 copilot 帮我们修复这个问题，我们可以这样写

```php
// $fp = fopen($filePath, "w");
// fputcsv($fp, $header);  
// foreach ($exportData as $item) {  
// fputcsv($fp, $item);  
// }  
// fclose($fp); 

// 以上注释代码会导致 中文乱码，
// 请使用下面的代码进行修复
$fp = fopen($filePath, "w");
fwrite($fp, chr(0xEF) . chr(0xBB) . chr(0xBF));
fputcsv($fp, $header);
foreach ($exportData as $item) {
    fputcsv($fp, $item);
}
fclose($fp);
```
![/img/github-copilot-05](/img/github-copilot-05.png)
- 技巧：
	- copilot 生成的代码，有会出现一些问题，你要去检验它的正确性并通过反馈的方式进行提问，它才能更好的响应

#### 写好具体的上下文

```golang
// 写一个函数 UserSig 
// 输入参数: sdkappid, userId, expire, userbuf, key 
// 输出签名: token 
// 处理过程: 输入参数包装成 map , 然后对 map 倒序排列，最后进行 md5 计算 
// 每行代码都写上中文注释

// ---- 以下为 copilot 生成的代码 ----
func UserSig(sdkappid, userId, expire, userbuf, key string) (token string) {
    m := map[string]string{
        "sdkappid": sdkappid,
        "userId":   userId,
        "expire":   expire,
        "userbuf":  userbuf,
        "key":      key,
    }
    var keys []string
    for k := range m {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    var str string
    for _, k := range keys {
        str += k + "=" + m[k] + "&"
    }
    str += "appkey=" + key
    return fmt.Sprintf("%x", md5.Sum([]byte(str)))
}
```
- 以上就是一个很典型的 prompt `[输入，处理过程，输出]` 的例子, 因为 copilot 受到 code 的影响最深，所以你把代码的思路写出来，它负责生成代码

#### Google 的平替

```txt
# 提出问题，如何在 markdown 中引入上标与下标，并与参考链接一起使用？
# --- 以下为 copilot 生成的代码 ---
<sup><a href="#fn1" id="ref1">1</a></sup>

# 并给出解释
# --- 以下为 copilot 生成的代码 ---
<sup><a href="#fn1" id="ref1">1</a></sup> 这是一个上标，用于引用参考链接，如 [^1]
```

以上部分提问之后，copilot 会给出一个答案，这样就避免了你去查阅文档的过程，保证了开发过程中的专注力。

### 单元测试

#### golang 中实践
- 在 golang 中写测试其实是一个比较简单的事情 配合 goland 的快捷键，写起来也是非常的方便，
![/img/github-copilot-06](/img/github-copilot-06.png)

```golang
	tests := []struct {
		name     string
		fields   fields
		args     args
		wantResp *types.UserLoginReply
		wantErr  bool
	}{
		// TODO: Add test cases.
        //------ 以下代码为 copilot 生成的代码 ------//
		{
			name: "fail",
			fields: fields{
				Logger: logx.WithContext(context.Background()),
				ctx:    context.Background(),
				svcCtx: &svc.ServiceContext{},
			},
			args: args{
				req: &types.UserLoginReq{
					Email:    "",
					Password: "",
				},
			},
			wantResp: nil,
			wantErr:  true,
		},
	}
```

#### php 中实践
- 同样的，php 中，配合 phpstorm 的快捷键，写起来也是非常的方便
![/img/github-copilot-07](/img/github-copilot-07.png)
```php
    public function testCreate(): void
    {
       $response = $this->json(
                method: 'POST',
                uri: '/api/v1/users',
                data: [
                    'email' => '',
                    'password' => '',
                ],
                headers: ['Authorization' => 'Bearer ' . $this->token,],
         );

        $response->assertStatus(200);
    }
```

- 在两种语言中，copilot 都能帮我们生成一些用例，但是这些用例都是比较简单的，所以我们需要自己去补充一些复杂的用例，比如边界值，异常值等等

### 生成文档

#### 非结构化文档
写文档，写blog，这部分其实是 copilot 的弱项，因为 copilot 本身是一个模型，它在生成代码的部分确实还不错，
但是它不知道你想要什么，而且它生成的文档受到上下文的影响很深，需要有逻辑性. 所以这部分需要我们自己去补充。

```markdown
# 写一个详细的大纲 关于介绍 github copilot 的使用
--- 以下为 copilot 生成的文档 ---
## 介绍
github copilot 是一个基于机器学习的代码提示工具，它可以帮助开发者更快的编写代码
## 安装
github copilot 是一个插件，需要安装到编辑器中
## 使用
github copilot 会根据上下文，生成代码提示，然后你可以选择是否使用
## 优点
github copilot 可以帮助开发者更快的编写代码
## 缺点
github copilot 生成的代码有时候不太准确，需要开发者自己去检查
## 总结
github copilot 是一个非常有意思的工具，它可以帮助开发者更快的编写代码
```
#### 结构化文档
- 不过如果是结构化的文档，比如 openapi ,那么 copilot 就能很好的生成文档了
```yaml
# 写一个swagger文档 api/v1/users
#--- 以下为 copilot 生成的文档 ---
openapi: 3.0.0
info:
  title: "api/v1/users"
  description: "api/v1/users"
  version: "1.0.0"
servers:
    - url: "http://xxx.com"
paths:
    /api/v1/users:
        post:
        tags:
            - "api/v1/users"
        summary: "创建用户"
        description: "创建用户"
        operationId: "CreateUser"
        requestBody:
            content:
            application/json:
                schema:
                $ref: "#/components/schemas/CreateUserReq"
        responses:
            "200":
            description: "OK"
            content:
                application/json:
                schema:
                    $ref: "#/components/schemas/CreateUserReply"
components:
    schemas:
        CreateUserReq:
        type: "object"
        properties:
            email:
            type: "string"
            password:
            type: "string"
        CreateUserReply:
        type: "object"
        properties:
            id:
            type: "string"
            email:
            type: "string"
            password:
            type: "string"
```

## 总结

这就是我在使用 copilot 快一年的一些感受，copilot 在 写注释，函数，单元测试，生成文档，帮助优化代码，处理错误异常等等方面都有一定的帮助，除了价格有一点点偏贵，在偏向于业务开发的场景是非常好用。  

本篇 blog 的内容部分也是 copilot 生成，在未来，类似的专业工具会越来越多，我们需要做的就是学会如何使用它们，你要面对的竞争对手不仅仅是人类，还有机器，学会如何与机器协作，而不是与机器竞争。

## 参考资料
1. [github copilot](https://copilot.github.com/)
2. [2023-06-20-how-to-write-better-prompts-for-github-copilot](https://github.blog/2023-06-20-how-to-write-better-prompts-for-github-copilot/)
3. [2023-05-17-how-github-copilot-is-getting-better-at-understanding-your-code](https://github.blog/2023-05-17-how-github-copilot-is-getting-better-at-understanding-your-code)
4. [GitHub Copilot：做出一个划时代的产品，只需要 6 个人](https://mp.weixin.qq.com/s/dxowVu3BIbNG70C8Yn-D_A#:~:text=Copilot%20%E5%9B%A2%E9%98%9F%E6%94%B6%E9%9B%86%E4%BA%86%E4%B8%80%E5%A4%A7%E5%A0%86%E7%BB%9F%E8%AE%A1%E6%95%B0%E6%8D%AE%EF%BC%8C%E5%B9%B6%E6%84%8F%E8%AF%86%E5%88%B0%E9%80%9F%E5%BA%A6%E5%9C%A8%E4%BB%BB%E4%BD%95%E7%BE%A4%E4%BD%93%E4%B8%AD%E9%83%BD%E6%98%AF%E6%9C%80%E9%87%8D%E8%A6%81%E7%9A%84%E6%8C%87%E6%A0%87%E3%80%82%E2%80%9C%E6%88%91%E4%BB%AC%E5%8F%91%E7%8E%B0%E5%BB%B6%E8%BF%9F%E6%AF%8F%E5%A2%9E%E5%8A%A0%2010%20%E6%AF%AB%E7%A7%92%EF%BC%8C%E5%B0%B1%E4%BC%9A%E6%9C%89%201%25%20%E7%9A%84%E7%94%A8%E6%88%B7%E6%94%BE%E5%BC%83%E8%BF%99%E9%A1%B9%E5%8A%9F%E8%83%BD%E3%80%82%E5%8F%A6%E5%A4%96%E5%9C%A8%E6%96%B0%E5%8A%9F%E8%83%BD%E5%85%AC%E5%BC%80%E5%8F%91%E5%B8%83%E7%9A%84%E5%A4%B4%E5%87%A0%E4%B8%AA%E6%9C%88%EF%BC%8C%E5%8D%B0%E5%BA%A6%E7%9A%84%E4%BD%BF%E7%94%A8%E5%AE%8C%E6%88%90%E7%8E%87%E6%98%AF%E6%9C%80%E4%BD%8E%E7%9A%84%E2%80%94%E2%80%94%E4%B8%8D%E7%A1%AE%E5%AE%9A%E4%B8%BA%E4%BB%80%E4%B9%88%EF%BC%8C%E4%BD%86%E5%AE%8C%E6%88%90%E7%8E%87%E7%A1%AE%E5%AE%9E%E6%98%8E%E6%98%BE%E4%BD%8E%E4%BA%8E%E6%AC%A7%E6%B4%B2%E3%80%82%E2%80%9D)