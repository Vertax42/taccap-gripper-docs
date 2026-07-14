# XTac-UMI G1 数采快速使用手册(文档站)

基于 [mkdocs-material](https://squidfunk.github.io/mkdocs-material/) 的
**XTac-UMI G1 手持触觉数采夹爪** lerobot 数据采集快速上手文档。中英双语,
发布到 GitHub Pages。

## 本地预览

```bash
# 1. 创建虚拟环境(任选 venv / conda)
python3 -m venv .venv && source .venv/bin/activate

# 2. 安装依赖
pip install -r requirements.txt

# 3. 本地热更新预览(默认 http://127.0.0.1:8000)
mkdocs serve

# 4. 严格构建(CI 用同一条命令,可提前发现死链/告警)
mkdocs build --strict
```

## 目录结构

```
.
├── mkdocs.yml                # 站点配置(主题/i18n/导航/扩展)
├── requirements.txt          # 构建依赖
├── docs/
│   ├── index.md              # 首页(中文默认)
│   ├── index.en.md           # 首页(English)
│   ├── 01-overview.md        # 1 概述
│   ├── 02-environment.md     # 2 环境部署
│   ├── 03-host-hardware.md   # 3 主机与硬件配置
│   ├── 04-calibration.md     # 4 标定与自检
│   ├── 05-data-collection.md # 5 数据采集
│   ├── 06-dataset.md         # 6 数据集与示例
│   ├── 07-faq-reference.md   # 7 常见问题与参考
│   └── assets/               # logo / favicon / 截图 / 图示
└── .github/workflows/deploy.yml   # GitHub Pages 自动发布
```

## 双语约定

- 中文为默认语言,文件名不带后缀(`01-overview.md`)。
- 英文译版加 `.en` 后缀(`01-overview.en.md`),由
  `mkdocs-static-i18n` 自动挂到 `/en/` 路径。
- 缺失的英文页会**自动回落**到中文版(`fallback_to_default: true`),
  不会导致构建失败——可以逐页翻译。

## 素材待补

- Pico4 Ultra 企业版 APP 使用截图 / 数采启动流程截图 → `docs/assets/`
- 站点 logo 与 favicon → `docs/assets/logo.png` / `docs/assets/favicon.png`

## 发布

推送到 `main` 分支后,GitHub Actions 自动 `mkdocs build --strict` 并发布到
GitHub Pages。首次需在仓库 **Settings → Pages → Source** 选择
**GitHub Actions**。
