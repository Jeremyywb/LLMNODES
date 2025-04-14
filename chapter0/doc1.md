# GitBook自动更新指南

本指南介绍如何在Windows系统上配置GitBook环境，并通过Python脚本实现Markdown内容的自动处理与网站更新。

## 准备工作

### Node.js安装
- **安装方法**：从官网下载版本 v10.24.1 (https://nodejs.org/dist/v10.24.1/)
- **安装验证**：安装成功后使用 `node -v` 确认版本

### GitBook安装
- **安装方法**：使用命令 `npm install gitbook-cli -g` 
- **安装验证**：安装成功后 `gitbook -V` 确认输出版本号
  
> 注意：如果命令输入后还在继续安装，说明安装未成功

### 工作目录结构
- **根目录**：创建一个根目录，用于git和gitbook生成文章
- **gitbook目录**：根目录内创建gitbook目录，用于gitbook网页创建生成
- **markdown目录**：根目录内创建markdown目录，存放markdown格式文章
- **book-end目录**：用于io.book网站内容更新
  - 根目录执行：
```bash
   git clone -b gh-pages git@github.com:USER_NAME/book.git book-end
```

  - 这一步克隆了gh-pages分支，并存放在book-end目录

### git命令目录安全问题
    配置book-end目录为安全目录：
```bash
   git config --global --add safe.directory "D:/COMPETITIONS/LLMNOTES/book-end"
```

    

### 额外插件配置
1. 在gitbook目录中创建**book.json**文件，添加侧边栏可折叠插件：

```json
{
  "plugins": [
    "expandable-chapters-small",    
    "chapter-fold",
    "sidebar-style",
    "tbfed-pagefooter",
    "hide-element",
    "simple-page-toc",
    "popup",
    "page-treeview",
    "github",
    "code",
    "copy-code-button",
    "alerts",
    "page-toc-button",
    "rss"

  ],
  "styles": {
    "website": "styles/website.css"
  },

  "links": {
    "sidebar": {
      "公众号": "https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzIwODc1MDg2NQ==&scene=1&album_id=3936221539196174339&count=3#wechat_redirect"
    },
    "sharing": {
       "douban": false,
       "facebook": false,
       "google": true,
       "hatenaBookmark": false,
       "instapaper": false,
       "line": true,
       "linkedin": true,
       "messenger": false,
       "pocket": false,
       "qq": false,
       "qzone": true,
       "stumbleupon": false,
       "twitter": false,
       "viber": false,
       "vk": false,
       "weibo": true,
       "whatsapp": true,
       "all": [
           "facebook", "google", "twitter",
           "weibo", "instapaper", "linkedin",
           "pocket", "stumbleupon","whatsapp"
       ]
   }
  },

  "pluginsConfig": {
    "page-treeview": {
      "copyright": ""
    },
  
    "sidebar-style": {
            "title": "《大模型笔记》",
            "author": "游文斌"
        },
    "hide-element": {
        "elements": ["br",".gitbook-link"]      },
    "tbfed-pagefooter": {
            "copyright":"Copyright &copy 游文斌  大模型算法工程师，微信：WayneBinY",
            "modify_label": "该文件修订时间：",
            "modify_format": "YYYY-MM-DD HH:mm:ss",
            "noPowered": true
        },
    "github": {
      "url": "https://github.com/Jeremyywb"
    },
    "code": {
      "copyButtons": false
    },
    "page-toc-button": {
            "maxTocDepth": 2,
            "minTocSize": 2
       }

  }
}

```

2. 在gitbook目录中创建styles文件夹，并创建website.css，填入以下内容

```css
/* 全局设置 */
body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, "Fira Sans", "Droid Sans", "Helvetica Neue", sans-serif;
    color: #333;
    line-height: 1.6;
  }
  
  /* 整体容器 */
  .book {
    background-color: #fff;
  }
  
  /* 侧边栏 */
  .book .book-summary {
    background-color: #f7f7f7;
    border-right: 1px solid #e9e9e9;
    font-size: 0.9em;
  }
  
  /* 侧边栏链接 */
  .book .book-summary ul.summary li a {
    padding: 10px 16px;
    color: #444;
    transition: all 0.2s ease;
  }
  
  .book .book-summary ul.summary li a:hover {
    background-color: #e3f2fd;
    color: #1565c0;
    text-decoration: none;
  }
  
  /* 侧边栏活动链接 */
  .book .book-summary ul.summary li.active > a {
    color: #1565c0;
    border-left: 4px solid #1565c0;
    background-color: #e3f2fd;
    font-weight: 600;
  }
  
  /* 页面内容 */
  .book .book-body {
    color: #333;
  }
  
  .book .book-body .page-wrapper .page-inner {
    max-width: 900px;
    padding: 20px 30px;
  }
  
  /* 标题 */
  .markdown-section h1 {
    font-size: 2.2em;
    color: #1565c0;
    border-bottom: 1px solid #eaecef;
    padding-bottom: 0.3em;
    margin-bottom: 1em;
  }
  
  .markdown-section h2 {
    font-size: 1.8em;
    color: #1565c0;
    border-bottom: 1px solid #eaecef;
    padding-bottom: 0.3em;
    margin-bottom: 0.8em;
  }
  
  .markdown-section h3 {
    font-size: 1.5em;
    color: #1976d2;
    margin-top: 1.5em;
  }
  
  .markdown-section h4 {
    font-size: 1.25em;
    color: #1976d2;
  }
  
  /* 文本和段落 */
  .markdown-section p {
    margin: 1em 0;
    line-height: 1.7;
    font-size: 1em;
  }
  
  /* 链接 */
  .markdown-section a {
    color: #1976d2;
    text-decoration: none;
    transition: all 0.3s ease;
  }
  
  .markdown-section a:hover {
    color: #1565c0;
    text-decoration: underline;
  }
  
  /* 代码块 */
  .markdown-section pre {
    background-color: #f6f8fa;
    border-radius: 4px;
    padding: 1em;
    margin: 1em 0;
    overflow: auto;
    line-height: 1.45;
  }
  
  .markdown-section pre > code {
    font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
    font-size: 0.9em;
    color: #24292e;
    background-color: transparent;
    padding: 0;
  }
  
  /* 行内代码 */
  .markdown-section code {
    font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
    padding: 0.2em 0.4em;
    font-size: 0.9em;
    background-color: rgba(27, 31, 35, 0.05);
    border-radius: 3px;
    color: #24292e;
  }
  
  /* 引用块 */
  .markdown-section blockquote {
    margin: 1em 0;
    border-left: 4px solid #1976d2;
    padding: 0 1em;
    color: #6a737d;
  }
  
  /* 表格 */
  .markdown-section table {
    display: table;
    width: 100%;
    border-collapse: collapse;
    margin: 1em 0;
    overflow: auto;
  }
  
  .markdown-section table tr {
    border-top: 1px solid #dfe2e5;
  }
  
  .markdown-section table th,
  .markdown-section table td {
    padding: 0.6em 1em;
    border: 1px solid #dfe2e5;
  }
  
  .markdown-section table th {
    font-weight: 600;
    background-color: #f6f8fa;
  }
  
  .markdown-section table tr:nth-child(2n) {
    background-color: #f8f8f8;
  }
  
  /* 图片 */
  .markdown-section img {
    max-width: 100%;
    border-radius: 4px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  }
  
  /* 页面导航 */
  .navigation {
    font-size: 0.9em;
  }
  
  /* 插件：可展开章节 */
  .book .book-summary .chapter > .articles {
    transition: max-height 0.3s ease;
  }
  
  /* Treeview 修复 */
  .treeview__container {
    margin-bottom: 20px;
    border: 1px solid #e9e9e9;
    border-radius: 4px;
    padding: 15px;
    background-color: #f7f7f7;
  }
  
  /* 移动设备优化 */
  @media (max-width: 768px) {
    .book .book-body .page-wrapper .page-inner {
      padding: 15px;
    }
    
    .markdown-section h1 {
      font-size: 1.8em;
    }
    
    .markdown-section h2 {
      font-size: 1.5em;
    }
  }

```
 
3. 安装插件：

```bash
   cd gitbook
   gitbook install
```

### GitHub仓库配置
1. 登录GitHub，创建一个新仓库（例如LLMNOTES）
2. 克隆仓库到本地：`git clone git@github.com:/USERNAME/LLMNOTES.git`
3. 创建gh-pages分支：`git checkout -b gh-pages`
4. 推送分支到仓库：`git push -u origin gh-pages`
5. 切换回主分支：`git checkout master`


完成这些步骤后，GitHub会自动为你分配一个网址：`https://USERNAME.github.io/LLMNOTES`

## Markdown内容自动更新

### 自动更新Python脚本

以下是一个完整的Python脚本，用于自动处理Markdown文件并更新GitHub Pages：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import shutil
import subprocess
import datetime
import re
from git import Repo, GitCommandError


class GitbookUpdater:
    def __init__(self, root_dir=None):
        """初始化GitbookUpdater类
        
        Args:
            root_dir: 项目根目录，默认为当前目录
        """
        self.root_dir = root_dir or os.getcwd()
        self.markdown_dir = os.path.join(self.root_dir, "markdown")
        self.gitbook_dir = os.path.join(self.root_dir, "gitbook")
        self.book_end_dir = os.path.join(self.root_dir, "book-end")
        
        # 确保必要的目录存在
        for dir_path in [self.markdown_dir, self.gitbook_dir, self.book_end_dir]:
            if not os.path.exists(dir_path):
                raise FileNotFoundError(f"目录不存在: {dir_path}")
    
    def clean_directories(self):
        """清理gitbook和book-end目录，保留必要文件"""
        print("正在清理目录...")
        
        # 清理gitbook目录，保留node_modules和book.json
        for item in os.listdir(self.gitbook_dir):
            item_path = os.path.join(self.gitbook_dir, item)
            if item not in ['node_modules', 'book.json']:
                if os.path.isdir(item_path):
                    shutil.rmtree(item_path)
                else:
                    os.remove(item_path)
        
        # 清理book-end目录，保留.git文件夹
        for item in os.listdir(self.book_end_dir):
            item_path = os.path.join(self.book_end_dir, item)
            if item != '.git':
                if os.path.isdir(item_path):
                    shutil.rmtree(item_path)
                else:
                    os.remove(item_path)
    
    def generate_summary(self):
        """生成SUMMARY.md文件"""
        print("正在生成SUMMARY.md...")
        
        summary_content = ["# Summary\n", "* [简介](README.md)\n"]
        
        # 复制根目录README.md到gitbook目录
        root_readme = os.path.join(self.markdown_dir, "README.md")
        if os.path.exists(root_readme):
            shutil.copy(root_readme, os.path.join(self.gitbook_dir, "README.md"))
        else:
            # 创建默认README.md
            with open(os.path.join(self.gitbook_dir, "README.md"), 'w', encoding='utf-8') as f:
                f.write("# 项目简介\n\n这是使用GitBook自动生成的文档。\n")
        
        # 创建章节映射字典，存储原始目录名与简化目录名的映射
        chapter_mapping = {}
        
        # 遍历markdown目录生成目录结构
        for item in sorted(os.listdir(self.markdown_dir)):
            item_path = os.path.join(self.markdown_dir, item)
            
            # 跳过README.md和非目录项
            if item == "README.md" or not os.path.isdir(item_path):
                continue
            
            # 处理特殊的end目录
            if item == "end":
                chapter_mapping[item] = "end"
                summary_content.append(f"* [总结](end/README.md)\n")
                continue
            
            # 处理常规章节目录
            if item.startswith("chapter"):
                # 从目录名提取章节名称
                match = re.match(r'chapter(\d+)_(.+)', item)
                if match:
                    chapter_num = match.group(1)
                    chapter_name = match.group(2)
                    
                    # 生成简化目录名
                    simplified_dir = f"chapter{chapter_num}"
                    chapter_mapping[item] = simplified_dir
                    
                    # 添加章节标题
                    # summary_content.append(f"\n## 第{chapter_num}章 {chapter_name}\n")
                    
                    # 添加章节README到SUMMARY
                    summary_content.append(f"* [第{chapter_num}章 {chapter_name}]({simplified_dir}/README.md)\n")
                    
                    # 处理章节中的md文件
                    for md_file in sorted(os.listdir(item_path)):
                        if md_file.endswith(".md") and md_file != "README.md":
                            # 提取文件标题
                            if "_" not in md_file:
                                raise ValueError(f"文件{md_file}需要有下划线用于提取标题")
                            title = md_file.split("_")[1]
                            title = title.split(".")[0]
                            with open(os.path.join(item_path, md_file), 'r', encoding='utf-8') as f:
                                content = f.read()
                            
                            # 转换文件名，去掉前缀并使用下划线连接
                            dest_filename = md_file
                            if '_' in md_file:
                                parts = md_file.split('_', 1)
                                if parts[0].isalnum():  # 如果前缀是字母数字组合，则去掉
                                    dest_filename = parts[0] + ".md"  # 只保留前缀部分作为文件名
                            
                            # 添加到SUMMARY.md
                            summary_content.append(f"  * [{title}]({simplified_dir}/{dest_filename})\n")
        
        # 写入SUMMARY.md
        with open(os.path.join(self.gitbook_dir, "SUMMARY.md"), 'w', encoding='utf-8') as f:
            f.writelines(summary_content)
        
        # 返回章节映射，供后续使用
        return chapter_mapping
    
    def build_gitbook(self):
        """初始化GitBook项目"""
        print("正在初始化GitBook...")
        
        # 切换到gitbook目录
        os.chdir(self.gitbook_dir)
        
        # 初始化GitBook项目
        subprocess.run("gitbook init", shell=True, check=True)
        
        # 切回根目录
        os.chdir(self.root_dir)
    
    def copy_markdown_files(self, chapter_mapping):
        """复制Markdown文件到对应的GitBook目录"""
        print("正在复制Markdown文件...")
        
        # 复制特殊的end目录README
        end_dir = os.path.join(self.markdown_dir, "end")
        if os.path.exists(end_dir) and os.path.isdir(end_dir):
            end_readme = os.path.join(end_dir, "README.md")
            if os.path.exists(end_readme):
                dest_dir = os.path.join(self.gitbook_dir, "end")
                # 确保目录存在
                os.makedirs(dest_dir, exist_ok=True)
                shutil.copy(end_readme, os.path.join(dest_dir, "README.md"))
        
        # 复制各章节文件
        for orig_dir, simple_dir in chapter_mapping.items():
            if orig_dir == "end":  # 已处理过
                continue
                
            orig_path = os.path.join(self.markdown_dir, orig_dir)
            dest_path = os.path.join(self.gitbook_dir, simple_dir)
            
            # 确保目标目录存在
            os.makedirs(dest_path, exist_ok=True)
            
            # 复制README.md
            readme_src = os.path.join(orig_path, "README.md")
            if os.path.exists(readme_src):
                shutil.copy(readme_src, os.path.join(dest_path, "README.md"))
            
            # 复制其他md文件
            for md_file in os.listdir(orig_path):
                if md_file.endswith(".md") and md_file != "README.md":
                    # 生成目标文件名
                    if '_' in md_file:
                        parts = md_file.split('_', 1)
                        if parts[0].isalnum():  # 如果前缀是字母数字组合
                            dest_filename = parts[0] + ".md"  # 只保留前缀作为文件名
                        else:
                            dest_filename = md_file
                    else:
                        dest_filename = md_file
                    
                    # 复制文件
                    shutil.copy(
                        os.path.join(orig_path, md_file),
                        os.path.join(dest_path, dest_filename)
                    )
            # 复制 images 子目录中的图片文件到 img 子目录
            images_src_path = os.path.join(orig_path, "images")
            if os.path.exists(images_src_path) and os.path.isdir(images_src_path):
                img_dest_path = os.path.join(dest_path, "images")
                os.makedirs(img_dest_path, exist_ok=True)  # 确保 img 目录存在
                for img_file in os.listdir(images_src_path):
                    img_file_path = os.path.join(images_src_path, img_file)
                    if os.path.isfile(img_file_path):  # 确保是文件
                        shutil.copy(img_file_path, os.path.join(img_dest_path, img_file))
    def generate_static_site(self):
        """生成GitBook静态网站"""
        print("正在生成静态网站...")
        
        # 切换到gitbook目录
        os.chdir(self.gitbook_dir)
        
        # 构建静态网站
        subprocess.run("gitbook build", shell=True, check=True)
        
        # 切回根目录
        os.chdir(self.root_dir)
    
    def copy_to_book_end(self):
        """将生成的静态网站复制到book-end目录"""
        print("正在复制文件到book-end...")
        
        # 源目录: _book
        source_dir = os.path.join(self.gitbook_dir, "_book")
        
        # 复制所有文件和目录
        for item in os.listdir(source_dir):
            source_item = os.path.join(source_dir, item)
            target_item = os.path.join(self.book_end_dir, item)
            
            if os.path.isdir(source_item):
                if os.path.exists(target_item):
                    shutil.rmtree(target_item)
                shutil.copytree(source_item, target_item)
            else:
                shutil.copy2(source_item, target_item)


    def update_github_pages(self):
        """更新GitHub Pages"""
        print("正在更新GitHub Pages...")
        
        # 切换到book-end目录
        os.chdir(self.book_end_dir)
        
        # 添加所有更改
        subprocess.run("git add .", shell=True, check=True)
        
        # 生成commit消息
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        commit_message = f"Update at {timestamp}"
        
        # 提交更改
        try:
            subprocess.run('git config user.email "jeremy_ywb@163.com"', shell=True)
            subprocess.run('git config user.name "Auto Commit Bot"', shell=True)
            subprocess.run(f'git commit -m "{commit_message}"', shell=True, check=True)
            
            # 推送到远程仓库
            subprocess.run("git push origin gh-pages", shell=True, check=True)
            
            print(f"更新完成! 提交信息: {commit_message}")
        except subprocess.CalledProcessError:
            print("没有变更需要提交或提交过程中出现错误")
        
        # 切回根目录
        os.chdir(self.root_dir)
        
        print("请等待GitHub Actions完成部署后访问您的网站")
    
    def run(self):
        """执行完整的更新流程"""
        try:
            self.clean_directories()
            chapter_mapping = self.generate_summary()
            self.build_gitbook()
            self.copy_markdown_files(chapter_mapping)
            self.generate_static_site()
            self.copy_to_book_end()
            self.update_github_pages()
            print("全部任务已完成!")
        except Exception as e:
            print(f"发生错误: {e}")

if __name__ == "__main__":
    updater = GitbookUpdater()
    updater.run()
```

### 使用方法

1. 将上述Python脚本保存为`update_gitbook.py`
2. 在项目根目录运行脚本：`python update_gitbook.py`

### 脚本工作流程

1. **清理目录**：删除book-end与gitbook中所有内容，除了book-end中".git"文件夹以及gitbook中node_modules文件夹和book.json文件
2. **生成SUMMARY.md**：根据markdown目录结构自动生成GitBook目录文件
   - 处理文件夹结构，如：
     ```
     chapter0_资料收集
         README.md
         gitbookuseage_gitbook一键使用.md
     ```
   - 生成对应的SUMMARY.md条目：
     - 一级标题：`[第零章 资料收集](chapter0/README.md)`
     - 文章标题：`[gitbook一键使用](chapter0/gitbookuseage.md)`
3. **构建GitBook**：
   - 切换到gitbook目录
   - 执行`gitbook init`初始化网页目录
   - 拷贝所有markdown目录相关的markdown文件到gitbook对应目录中
   - 执行`gitbook build`生成静态网页，会生成一个_book目录
4. **复制静态文件**：将_book目录中所有内容拷贝到book-end目录
5. **更新GitHub仓库**：
   - 切换到book-end目录
   - 执行`git add .`
   - 执行`git commit -m "更新时间: YYYY-MM-DD HH:MM:SS"`
   - 执行`git push origin gh-pages`

### 目录结构示例

```
PROJECT_ROOT/
├── markdown/              # Markdown文章目录
│   ├── README.md          # 项目简介
│   ├── chapter0_资料收集/
│   │   ├── README.md
│   │   └── gitbookuseage_gitbook一键使用.md
│   └── chapter1_基础知识/
│       ├── README.md
│       └── python_基础语法.md
├── gitbook/               # GitBook工作目录
│   ├── book.json          # GitBook配置文件
│   └── node_modules/      # 插件目录
├── book-end/              # 静态页面输出目录(gh-pages分支)
│   └── .git/              # Git仓库信息
└── update_gitbook.py      # 自动更新脚本
```

## 常见问题解决

### GitBook安装和使用问题

如果在安装或使用GitBook时遇到`cb.apply is not a function`错误，可能是因为Node.js版本不兼容。请确保使用Node.js v10.24.1版本，这是最稳定的兼容版本。

### 其他注意事项

1. 确保GitHub仓库已正确配置，特别是gh-pages分支
2. 首次运行脚本前，确保所有目录结构已就绪
3. Windows环境下可能需要安装Git for Windows以支持命令行Git操作
4. 确保Python 3.6+已安装，并具有所需模块（如shutil等）

## 完整流程总结

1. 按照准备工作部分配置好环境
2. 创建Markdown文档并按照章节组织
3. 运行自动更新脚本
4. 等待GitHub Actions完成部署
5. 访问`https://USERNAME.github.io/REPOSITORY_NAME`查看更新的文档

通过上述步骤，你可以轻松维护一个自动更新的GitBook文档站点，而无需手动执行繁琐的构建和部署过程。

---

如有任何问题或建议，欢迎提交Issue或贡献改进。
