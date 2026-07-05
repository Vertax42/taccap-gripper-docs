# docs/assets/ — 发布用素材

放这里的图片会被 mkdocs 打包进站点、随文档一起发布。**只有 `docs/` 内的文件才会进站点**
(项目根的 `materials/` 是原始资料,不发布)。

> 本文件本身通过 `mkdocs.yml` 的 `exclude_docs` 排除,不会生成页面。

## 目录约定

| 子目录 | 用途 | 对应章节 |
|---|---|---|
| `brand/` | 站点 logo、favicon | 主题 |
| `pico4/` | Pico4 企业版 APP 使用截图 | §3.4 |
| `bringup/` | 数采启动流程截图 | §3.6 |
| `hardware/` | 设备/串口/接线/发现规则示意 | §3 |
| `record/` | 录制终端/界面、Rerun 截图 | §4 / §5 |
| `dataset/` | 数据集结构、校验、可视化 | §6 |

## 命名约定

- 全小写、连字符分隔、带章节前缀,便于定位:
  `pico4/3-4-pair-tracker.png`、`bringup/3-6-step2-unity-client.png`
- 优先 `.png`(截图)/`.svg`(矢量图示);大图控制在合理体积(建议 < 500 KB)。

## 在文档里引用

Markdown 里用**相对 docs 根**的路径。例如在 `docs/03-host-hardware.md` 中:

```markdown
![Pico4 配对追踪器](assets/pico4/3-4-pair-tracker.png)
```

加说明/宽度(需要 `attr_list`,已启用):

```markdown
![启动 Unity 客户端](assets/bringup/3-6-step3-unity.png){ width="480" }
```

点击放大(`glightbox` 已启用,默认对内容区图片生效,无需额外语法)。

## 启用自定义 logo / favicon

把文件放到 `brand/` 后,取消 `mkdocs.yml` 里这两行的注释:

```yaml
theme:
  logo: assets/brand/logo.png
  favicon: assets/brand/favicon.png
```
