# UEditor 主观题：录入与同步

每道主观题下面是一个百度 UEditor 实例，内容在**同名 iframe**里，按 DOM 顺序为 `#ueditor_0`、`#ueditor_1` …。iframe 内是 `document.body`（contenteditable）；页面另有一个真正会被提交的黑盒 `textarea[name^=answer]`（如 `answer<qid>`），保存时 POST 的是它。

## 只能用 execCommand，不能用 browser_type

`browser_type` / Playwright 的 `pressSequentially` 会把焦点打在 iframe 外层元素上，从 contenteditable 里拉走——结果一个字都没进去，而且不报错。先用一次真实点击 + 单次按键确认通路（会出现 `<p>a</p>`），再走 `browser_evaluate` 批量写入：

```js
() => {
  const lines = [/* 逐行答案，不带首尾空行 */];
  const doc = document.getElementById('ueditor_0').contentDocument;
  const win = doc.defaultView;
  win.focus(); doc.body.focus();
  doc.execCommand('selectAll', false, null);
  doc.execCommand('delete', false, null);
  for (let i = 0; i < lines.length; i++) {
    if (i > 0) doc.execCommand('insertParagraph', false, null);
    if (lines[i].length) doc.execCommand('insertText', false, lines[i]);
  }
  doc.dispatchEvent(new Event('input', { bubbles: true }));
  return doc.body.innerText.length;
}
```

一次调用只写一个编辑器实例。文本含引号、换行或 `%` 转义时，本地 `base64 -w0 文件` 生成、页面内 `decodeURIComponent(escape(atob(b64)))` 解码后再 `insertText`，比在 JS 字符串字面量里逐层转义可靠。`× ≤ ≥ ∑ ∅` 这类字符 UEditor 不过滤，直接 `insertText` 即可。

## 保存前必须 sync()

**最近编辑过的那个实例不会自动写回自己的 textarea**，于是保存会带着空答案发给服务器，页面上看不出任何异常。

```js
() => {
  const out = {};
  Object.keys(UE.instants).forEach(k => { try { UE.instants[k].sync(); } catch (e) {} });
  Object.keys(UE.instants).forEach(k => { out[k] = { ue: UE.instants[k].getContent().length }; });
  document.querySelectorAll('textarea[name^=answer]').forEach(t => { out[t.name] = { field: t.value.length }; });
  return out;
}
```

逐题比两边长度。`getContent()` 带 `<p>` 标签，会比 textarea 长几十字符，这正常；要找的是**任何一题是 0**，或者长度还停在编辑前。

## 工具栏吸顶挡住点击

`autoFloat` 在滚动后把工具栏变成 `position:fixed` 钉在 y≈7，而页面自己的 `.subNav`（bottom≈40）盖在它上面，于是「附件」等按钮点不到。恢复办法是把页面滚到该编辑器的绝对位置附近，让工具栏回到 `position:relative`：

```js
() => {
  const fr = document.getElementById('ueditor_1');
  window.scrollTo(0, fr.getBoundingClientRect().top + window.scrollY - 420);
  return { scrolledTo: window.scrollY };
}
```

之后重新 `browser_find` 取 ref（几何变了旧 ref 就失效），再点击。

## 附件上传

弹窗是 `eduiNNN_iframe` 里加载的 `dialogs/attachment_new/attachment.html`，用 WebUploader，真正的 `<input type=file multiple>` 是隐藏的。

1. `browser_find` 搜「附件」拿到工具栏按钮 ref，**真实 `browser_click`** 打开弹窗。必须真实点击——`#pickfiles` 用 JS 触发没有 user activation，文件选择器不响应，随后 `browser_file_upload` 会报 "can only be used when there is related modal state present"。
2. 再 `browser_find` 搜「上传文件」/「Choose File」，点那个**链接**。不要直接点 `input[type=file]`：`.popLocal_text` 覆盖层会 intercept pointer events，导致超时。
3. 立刻调 `browser_file_upload`（只在弹窗状态存在时有效），路径必须在允许根目录内。`File access denied: … Allowed roots: …` 会列出根目录，照它说的把文件 `cp` 过去再传。多题共用一个目录时给文件加题号前缀，便于在页面上核对。
4. 等弹窗里出现文件名再关闭，然后重跑一遍上面的 sync + 长度核对。
