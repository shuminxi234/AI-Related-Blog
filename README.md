# AI Related Blog 🚀

AI 技术博客 — 聚焦 AI Agent、LLM 应用、自动化工程实践与技术思考。

## 技术栈

- **框架**: [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题
- **托管**: GitHub Pages（自动部署）
- **写作**: Markdown + YAML frontmatter

## 本地开发

```bash
# 克隆
git clone --recurse-submodules https://github.com/shuminxi234/AI-Related-Blog.git
cd AI-Related-Blog

# 启动开发服务器
hugo server -D

# 构建静态文件
hugo
```

## 写作规范

- 文章放在 `content/posts/` 目录
- frontmatter 必须包含 `title`、`date`、`tags`、`author`
- 提交 PR 到 `main` 分支，合并后自动部署

## 写作流程

本博客采用多智能体协作流水线：

1. **选题编辑** → 筛选技术热点
2. **研究员** → 深度调研
3. **撰稿人** → 撰写文章
4. **审核人** → 技术审查
5. **发布者** → 推送发布
