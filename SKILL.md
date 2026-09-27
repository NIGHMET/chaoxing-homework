---
name: chaoxing-homework
description: 超星学习通作业自动作答。在 Playwright 内置浏览器里打开 mooc*-api.chaoxing.com 的 dowork 作业页，自动识别题型（UEditor 主观题 / CodeMirror 程序题 / 选择判断 / 填空 / 附件题），把答案写进真正的作答控件、核对是否真的存上、只点「暂时保存」绝不误点「提交」。当给出学习通作业链接并要求作答时使用。
---

# 超星学习通作业自动作答

学习通作答页最容易出的问题是**静默失败**：编辑器里看着写满了、保存返回 200、实际交上去的是空答案。截图查不出来。下面每条规则都是为了让这种失败变成能看见的。

用到的工具：`browser_navigate`、`browser_snapshot`、`browser_find`、`browser_click`、`browser_evaluate`、`browser_network_requests`、`browser_network_request`、`browser_file_upload`。

## 流程

1. `browser_navigate` 打开 dowork 链接。URL 里的 `enc` / `standardEnc` 本身就是授权参数，通常不需要登录；会话 cookie 也能跨浏览器重启存活（`https://i.chaoxing.com/` 直接进个人空间就说明已登录）。首跳偶尔落到登录页，再 `browser_snapshot` 确认一次，别急着让用户扫码。
2. **先把每题的作答控件类型探出来**，不要照上一份作业的形态假设：

   ```js
   () => Array.from(document.querySelectorAll('.questionLi')).map((li, i) => ({
     q: i + 1,
     title: (li.querySelector('.tiTitle') || li).innerText.slice(0, 30).replace(/\n/g, ' '),
     answertype: (li.querySelector('input[id^=answertype]') || {}).value,
     ueditor: li.querySelectorAll('iframe[id^=ueditor_]').length,
     codemirror: li.querySelectorAll('.CodeMirror').length,
     radios: li.querySelectorAll('input[type=radio], ul.TiTopic li').length,
     cloze: li.querySelectorAll('.clozeTextQues, .clozeTextItem').length,
     fileInputs: li.querySelectorAll('input[type=file]').length,
     runBtn: !!li.querySelector('#runBtn')
   }))
   ```

   `answertype` 17 = 程序题。附件区只有在真的存在 `input[type=file]` 时才有——**不是每题都能传附件**，别照上传那套走。
3. 读题干。纯文字题干 `innerText` 直接有；伪代码、公式题往往是 `p.ananas.chaoxing.com` 上的图片 → `references/stems.md`。
4. 按控件类型录入：
   - 主观题 / UEditor → `references/ueditor.md`
   - 程序题 / CodeMirror → `references/codeeditor.md`
   - 选择、判断、填空 → 点 `input[type=radio]` / 填 `input[type=text]`，用 `li.querySelector` 限定到该题范围内
5. 核对确实存上了 → `references/verification.md`。
6. 点「暂时保存」，报告并停下。

## 硬性规则

- **只做到「暂时保存」。** 「提交」就在旁边，很好误点，而且能不能重做由服务器端配置决定。「帮我完成作业」不等于授权提交。除非用户明确说「提交」，否则停下来报告草稿，并说清楚你点的是哪一个。
- **保存前先把控件刷进隐藏域。** UEditor 要 `UE.instants[k].sync()`，程序题要 `setProceduralAnswer()`。可见编辑器里有内容 ≠ 提交值正确；不刷的那一题会交一个空答案上去，页面毫无异常。这是全流程最主要的静默失败点。
- **逐题比对长度，不要只看一题。** `widget.getContent()` 与 `hiddenField.value` 都要读，出现 `0` 或者停留在编辑前长度的就是没刷进去。
- **别用截图下结论。** 空的隐藏域和写满的编辑器在像素上是同一个样子。
- **`id` 在多个题目间是重复的。** 一个作业页 6 道题就有 6 个 `#runBtn`、6 个 `#runstatus`。一律按题定位：`document.querySelectorAll('.questionLi')[i].querySelector('#runBtn')`。用 `getElementById` 会只命中第一题，而它照样返回成功。
- **qid 取 `textarea[data-business-id]`，不要猜 DOM id。** `.questionLi` 的 `id` 是 `question<qid>`（带前缀），而隐藏域叫 `answer<qid>`；`data-question-id` 属性并不存在。按错的那个取会匹配到 0 个元素，然后一切看起来正常。
- **上传、截图的路径必须在 Playwright MCP 允许根目录里。** 报错 `File access denied: … Allowed roots: …` 会把根列出来，照着把文件 `cp` 过去重试，别猜。
- 该站的 `browser_run_code_unsafe` 会被安全策略拦掉，页面级操作走 `browser_evaluate`。
- **吸顶工具栏会挡住点击。** UEditor 的 `autoFloat` 滚动后变成 `position:fixed` 钉在顶部，被站点自己的导航条盖住，于是「附件」等按钮点不到、`browser_click` 超时且不报错。把页面滚回该编辑器的绝对位置让工具栏回到 `position:relative`，然后重新 `browser_find` 取 ref。

## 内容本身

代码和答案该由谁写、怎么写，是普通作业工作，本技能不提供指导——它只保证「写好的东西真的进到框里、真的存到了服务器」。验证答案对不对请自己跑：程序题本地编译执行、和题面样例逐字符比对（注意 MinGW 输出是 CRLF，比对时先 `tr -d '\r'`），随机数据和边界用例对着参考实现对拍；算法分析题把伪代码逐行插桩计数，再和闭式解在若干 `n` 上互相印证。
