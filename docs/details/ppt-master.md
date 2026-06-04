---
tags:
  - ppt
  - pdf
  - llm
---
## 下载

examples 目录居然有 581M 样例。

![[attachments/Pasted image 20260604151833.png]]

期望之后会有精简版吧（太离谱了）

## 大致流程（印象版）

1. 将 PDF 文件（这里没有尝试其他类型的文件）放到 `projects` 目录下
2. 开始生成 PPT 时，先创建 `projects/<title>` 目录
3. agent 会生成 PPT 的页面大小等信息（一般会填写在 `spec_lock.md` 中），并且会给出大纲，一般为 `design_spec.md`
4. `image_analysis.csv` 会存放解析出来的图片信息
5. 然后会逐渐在 `svg_output` 中生成 svg 文件，（看上去）每一个 svg 对应一页 slide。
6. 最后会在 `exports` 目录输出 ppt

> 比较奇怪的点是，输入原文件会被 **移动（不是拷贝）** 到 `sources` 目录下，万一 agent 把 `sources` 目录也删了就完蛋了。

生成 12 页 PPT，总共花费 26M tokens。

![[attachments/Pasted image 20260604152418.png]]

> v4-flash 之前也用，混杂了之前的数据，不能看,v4-pro 的 In/Out 可以参考

