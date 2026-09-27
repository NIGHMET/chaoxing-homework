# 读题干：文字在 snapshot 里，公式和伪代码在图片里

`browser_snapshot` 能拿到题型、分值、作答要求和纯文字题干，但**算法题的伪代码、含公式的题干通常是 `p.ananas.chaoxing.com` 上的图片**，snapshot 里只有空壳。不要把图片链接直接抛给用户，也不要凭题号猜内容。

## 找到图片并归题

```js
() => Array.from(document.images).map(im => ({
  src: im.src,
  w: im.naturalWidth, h: im.naturalHeight,
  near: (im.closest('.questborder, .answerPer, .tiTitle') || document.body).innerText
        .slice(0, 60)
}))
```

按 `near` 把图归到对应题目；宽高偏大、`src` 里带 `readno`/`uuid` 的通常是题干截图。

## 下载必须带 Referer

`p.ananas.chaoxing.com/star3/origin/…` 直连返回 **403**，看起来像链接坏了，其实是防盗链。补上站点 Referer 和一个浏览器 UA：

```bash
curl -sL -e https://mooc1-api.chaoxing.com/ \
  -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/126.0 Safari/537.36' \
  -o q1_full.png '<image-url>'
```

## 看清小字：本地放大再读

整页截图有两个坑：滚动时 UEditor 的吸顶工具栏和 `.subNav` 会盖住题干中下部（曾因此漏看一行 `sorted ← true`，而那一行恰好决定最好情况的下限）；原图字号也偏小。下载原图后用 PIL 裁切放大再 `Read`：

```python
from PIL import Image
im = Image.open('q1_full.png')
im.resize((im.width * 3, im.height * 3), Image.LANCZOS).save('q1_3x.png')
im.crop((0, im.height // 3, im.width, im.height // 2)) \
  .resize((im.width * 3, im.height // 6 * 3), Image.LANCZOS).save('q1_band.png')
```

逐行抄出伪代码，连行号、缩进、赋值与比较符号一起抄准——行号抄错会让后面所有计数都错。抄完对照原图再核一遍，不要拿截图复核。
