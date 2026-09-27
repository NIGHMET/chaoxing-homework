# 程序题（CodeMirror 编辑区）

`answertype = 17`、题面 `.questionLi` 里有 `.CodeMirror`、通常还有 `#runBtn`。这类题**一般没有附件上传区**（先确认 `li.querySelector('input[type=file]')` 在不在，别照主观题那套去上传）。

## 定位实例

页面把每个编辑器实例存在全局 `codeEditors` 里，key 是 qid。qid 只能从 `li.querySelector('textarea[data-business-id]').dataset.businessId` 取：

```js
() => Array.from(document.querySelectorAll('.questionLi')).map((li, i) => ({
  i,
  domId: li.id,                                   // "question<qid>"，带前缀
  qid: li.querySelector('textarea[data-business-id]').dataset.businessId,  // 裸 qid
  hiddenId: 'answer' + li.querySelector('textarea[data-business-id]').dataset.businessId,
  cm: li.querySelectorAll('.CodeMirror').length
}))
```

`data-question-id` 属性并不存在，`li.id` 也带着 `question` 前缀——用错的那个取实例会得到 `undefined`，写进去变成空操作，却不报错。

## 录入

```js
() => {
  codeEditors[qid].setValue(code);   // 纯文本源码
  setProceduralAnswer();             // 关键：把所有编辑器刷进各自的隐藏答案域
  const raw = document.getElementById('answer' + qid).value;
  return { editorLen: codeEditors[qid].getValue().length, storedLen: raw.length, raw };
}
```

- **不要用 `browser_type` / 键盘逐字输入，也不要 `document.execCommand`**——CodeMirror 不读 contenteditable 那套。
- `setProceduralAnswer()` 会把 `#answer<qid>` 重写成一个 JSON 数组 `[{language, answer, answerTxt}]`，其中 `answer` 是源码 HTML 转义后包在 `<pre class="line-numbers …"><code class="language-plain">…` 里的字符串。**键叫 `answer`，不是 `code`**——`JSON.parse(v).code` 会静默给出 `undefined`。不刷的话隐藏域还是空模板，页面看上去一切正常。
- 多行源码含 `"%d\n"` 这类转义时，本地 `base64 -w0 文件`，页面内解码，避免三层转义出错：
  `const code = decodeURIComponent(escape(atob(b64)));`
- 语言由该题的 `.languageSelect` 决定（答案 JSON 的 `language` 字段就是它，1 = C）。注意它常是自定义下拉，`.options` / `.innerText` 可能取不到值，要读隐藏 value 或答案 JSON。题面给什么语言就按什么语言写，`bits/stdc++.h` 在 C 下不可用。

## 用站点自带的运行结果核对

`#runBtn` 和 `#runstatus` 在多个题目间是重复 id，必须按题定位。点击是普通 AJAX，不需要 user activation：

```js
async () => {
  const li = document.querySelectorAll('.questionLi')[1];
  const st = li.querySelector('#runstatus');
  li.querySelector('#runBtn').click();
  for (let t = 0; t < 150; t++) {
    await new Promise(r => setTimeout(r, 400));
    if (getComputedStyle(st).display === 'none') break;   // 隐藏即本轮结束
  }
  return Array.from(li.querySelectorAll('textarea.runResult')).map(x => x.value.trim()).join(' / ');
}
```

通过时读到 `Accepted（运行通过） / 第1/1条 运行结果 / 运行结果正确`。用 `#runstatus` 的 `display` 作结束信号，别用结果文本——它可能还留着上一轮的值。

如果本地跑对、站上却 WA：判题响应通常只给状态码一类字段，**不给期望输出和实际输出**，所以题面印的「输入用例」不能当作判题真正喂进去的东西。先把 `browser_network_requests` / `browser_network_request` 里那一对请求响应对读出来，再分别检验两个互相独立的假设（读入格式、输出格式），不要一次改两样。
