#### 基本用法

```shell
# 使用前准备
npm install hexo-cli hexo-server -g	# 安装hexo
cd shzhxh.github.io
npm install	# 安装依赖

# 创建文章
hexo new <title>	# 新建文章
hexo server	# 本地看文章的效果
```

#### 格式约束

1. 每篇文章里categories首字母应大写。目前使用的分类：
   - Command：命令的用法归纳或源码分析。
   - Hardware：硬件的原理或设计实现。
   - Language：编程语言的用法或设计实现。
   - Network：网络原理与协议的分析。
   - OS：操作系统的原理或设计实现。
   - Other：一些技术无关的内容。

1. 太长的文章应用`<!-- more -->`控制预览的长度。

