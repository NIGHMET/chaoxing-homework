# 确认答案真的存上了

三件事是分开成立的，页面可以前两件都对、第三件全错：编辑器里显示有内容 · 保存返回 200 · 服务器存的是对的值。

## 1. 保存前：控件 → 隐藏域

逐题读，不要抽查：

```js
() => Array.from(document.querySelectorAll('.questionLi')).map(li => {
  const qid = li.querySelector('textarea[data-business-id]').dataset.businessId;
  const f = document.getElementById('answer' + qid);
  return { qid, len: f ? f.value.length : -1 };
})
```

出现 `0`、`-1`（说明域 id 拼错了）、或值还等于未编辑的模板，就是没刷进去。

## 2. 保存请求本身

```
browser_network_requests({ filter: 'addStudentWorkNewWeb', static: false })
browser_network_request({ index: <n>, part: 'request-body' })
browser_network_request({ index: <n>, part: 'response-body' })
```

两半都要读：

- **request-body** — 确认各 `answer<qid>` 非空，并确认那个区分草稿/正式提交的参数是你想要的值。「暂时保存」和「提交」往往打的是同一个接口，只差这一个参数（草稿是 `saveStatus=1`），所以 200 完全不能告诉你刚点的是哪一个。
- **response-body** — 成功形如 `{"msg":"保存成功！","status":true}`。注意 HTTP 200 里塞 `{"status":false}` 或把错误写在 `msg` 里是常态，状态码不是成功信号。

请求体和 URL 里带着会话 token 和签名参数（`token=`、`enc=`、`standardEnc=`）。**不要把它们在聊天、issue、截图或提交里贴出来**——拿到的人不需要密码就能进你的作答页。

## 3. 重新加载后比对

用同一个 URL 再 `browser_navigate` 一次，重读各题内容。刷新后还在才算真的存住；只有客户端状态的东西会消失。

```js
async () => {
  await new Promise(r => setTimeout(r, 2500));
  return Array.from(document.querySelectorAll('.questionLi')).map(li => {
    const qid = li.querySelector('textarea[data-business-id]').dataset.businessId;
    const code = window.codeEditors[qid] ? codeEditors[qid].getValue() : '';
    return { qid, len: code.length, head: code.slice(0, 20) };
  });
}
```

长度允许有系统性小偏差：服务端往返常会去掉结尾换行、把 `\r\n` 归一、或给纯文本重新包一层标签，所以重载后比源文件少一两个字符是正常的。要比就去比较**解码后的内容**（剥掉标签、还原实体），不要直接比序列化出来的长度。空值或模板形状的值不是偏差，是没存上。

## 报告

逐题说明：写了什么、上面三项里哪一项确认了它、以及明确说出点的是「暂时保存」还是「提交」——这两个词不是同一件事，而且只有一个还能改。
