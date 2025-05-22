# SQLITEM

根据这个[想法](https://x.com/linexjlin/status/1924723978180165779)实现的一个玩具项目。 这个项目实现了用自然语言写了一个用 web 管理 sqlite3 数据库的想法。 

## 主要实现思路就是： 

`写出想法 -> 让 AI 给出思路及项目框架 -> 检查 AI 给的思路 -> 实现具体的代码`

可以看看自然语言生成后端，前端代码的 prompt 过程： [backend.prompt.md](./backend.prompt.md), [frontend.prompt.md](./frontend.prompt.md)  重点就看最开始的请求就行，就能理解这个思路了，后面是 AI 的规划。 

## 实现效果：

![](images/screenshot.png)

## 一些经验

- 最好，找个可以编辑 AI 回复的 Chat UI，可以直接对 AI 的思路做调整。（通常第三方的 UI 都可以， gemini studio 也可以）
- 后端实现的统一接口很重要（主要就是这行文字： `给出 API 请求例子`）。  有了统一的接口前端就能跟后端独立生成了，互不干扰。 
- 后端代码，我每次都是重新生成。 
- 前端代码我在 gemini 用了 canvas 修改了几个版本，最后，再更新文档。 
- 我用 gemini 2.5 pro 每次生成的代码基本直接可用，不能用的不要花时间 debug 直接重新生成。
- 这个项目比较简单，还要很多功能可以做， 功能都做完整了，还要花不少时间。