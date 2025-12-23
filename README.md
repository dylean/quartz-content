# Quartz Content Repository

这是 Quartz 博客的内容仓库，使用 git submodule 方式与主仓库（quartz-dean）集成。

## 自动部署设置

当你在 content 仓库中推送更改时，会自动触发主仓库的 GitHub Pages 部署。

### 设置步骤

1. **创建 GitHub Personal Access Token (PAT)**
   - 访问：https://github.com/settings/tokens
   - 点击 "Generate new token" > "Generate new token (classic)"
   - 设置名称：`Quartz Deploy Token`
   - 选择权限：
     - `repo` (完整仓库访问权限)
   - 点击 "Generate token"
   - **重要**：复制生成的 token（只显示一次）

2. **在 content 仓库中添加 Secret**
   - 访问：https://github.com/dylean/quartz-content/settings/secrets/actions
   - 点击 "New repository secret"
   - Name: `QUARTZ_DEPLOY_TOKEN`
   - Value: 粘贴刚才复制的 token
   - 点击 "Add secret"

3. **完成！**
   - 现在当你推送内容到 content 仓库时，会自动：
     1. 更新主仓库（quartz-dean）中的 submodule 引用
     2. 触发主仓库的 GitHub Actions workflow
     3. 自动部署到 GitHub Pages

## 使用方法

1. 编辑 content 目录中的 Markdown 文件
2. 提交并推送：
   ```bash
   git add .
   git commit -m "更新内容"
   git push origin main
   ```
3. 部署会自动触发，无需手动操作！

## 手动更新 submodule（可选）

如果你想手动更新主仓库的 submodule 引用：

```bash
cd /path/to/quartz-dean
git submodule update --remote content
git add content
git commit -m "Update content submodule"
git push origin v4
```

