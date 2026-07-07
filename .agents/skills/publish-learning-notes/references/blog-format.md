# dsxyy.github.io 内容格式

## 仓库事实

- 仓库：`dsxyy/dsxyy.github.io`
- 发布分支：当前为 `master`，实际操作时仍需读取远端默认分支确认。
- 渲染器：仓库内嵌 MDwiki 0.6.2；`index.html` 是入口，`index.md` 是首页内容，`navigation.md` 是顶部导航。
- 内容格式：纯 Markdown，不使用 YAML Front Matter。
- 现有内容：通用文章位于仓库根目录，专题文章可放在对应子目录，例如 `openstack/`。

## 新文章规则

1. 默认在仓库根目录创建 `<topic-slug>.md`；已有明确专题目录时才放入该目录。
2. 文件名优先使用简短 ASCII kebab-case，避免覆盖现有文件。
3. 首行使用一级标题：

   ```markdown
   # 标题
   ```

4. 使用相对路径引用仓库图片。图片放在文章同专题的 `pics/` 目录；没有图片时不要创建目录。
5. 不添加日期、标签等 MDwiki 不消费的 Front Matter。
6. 流程图、内存布局和普通文本代码块使用无语言标记的围栏 ```` ``` ````。禁止使用 ```` ```text ````，它会导致当前 MDwiki 0.6.2 的旧版高亮器卡死。

## 更新导航

在 `navigation.md` 中增加一个相对链接：

```markdown
[文章标题](topic-slug.md)
```

需要在首页展示文章时，再在 `index.md` 的 `Demo` 列表增加同一链接。专题文章使用包含目录的相对路径。保持周围文件原有换行风格；只增加必要条目，不重排历史链接。

## 发布检查

```bash
git status --short
git diff --check
git diff -- <article-path> index.md navigation.md
```

只暂存明确文件：

```bash
git add -- <article-path> index.md navigation.md
```
