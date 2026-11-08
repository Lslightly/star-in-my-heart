---
tags:
  - translator
  - markdown
  - pdf
  - docx
url: https://github.com/docling-project/docling
---
## 语言转文字

m4a 语音转文字 在语音时长为 8 分钟时，效果一般，时间花费比较久（大概 10 分钟？）。（还是 3090 GPU 推理）

## PDF 转 markdown

从 PDF 到 markdown 时，图片会以如下形式内嵌（embed）在 markdown 文件中。

```
![Image](data:image/png;base64,...)
```

可以通过在 `docling` cli 选项中 中添加 `--image-export-mode referenced` 选项将图片导出为 PNG 格式并引用。`--image-export-mode` 默认为 `embedded` 模式。

