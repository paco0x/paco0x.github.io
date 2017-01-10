---
layout:     post
title:      "我的 Sublime Text 常用插件"
subtitle:   "The text editor you'll fall in love with"
date:       2014-11-13 24:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /sublime-text-intro/
tags:
    - Tools
    - Python
---

> Sublime Text: The text editor you will fall in love with


sublime text 是一款跨平台的编辑器，安装插件方便，界面相当美观，同时有大量的主题和配色可供选择。

废话不多少，具体可以去 [Sublime Text][ST] 官网了解，下面介绍我所使用的插件和配置，注意我的环境是windows环境。

## 配置文件

配置文件在 Preferences -->  Settings--User, 下面是我的配置

	{
		// Theme & Color
		"theme": "Soda Dark 3.sublime-theme",
		"color_scheme": "Packages/Monokai Gray/MonokaiGray.tmTheme",
		"drag_text": false,

		// Font
		"font_face": "Consolas",
		"font_size": 11,
		"font_options":
		[
			"subpixel_antialias"
		],
		"line_padding_bottom": 1,
		"line_padding_top": 1,
		"default_line_ending": "unix",
		"default_encoding": "UTF-8",

		// Indentation
		"auto_indent": true,
		"smart_indent": true,
		"tab_size": 4,
		"translate_tabs_to_spaces": true,
		"trim_trailing_white_space_on_save": true,
		"indent_to_bracket": true,
		"ensure_newline_at_eof_on_save": true,
		"trim_automatic_white_space": true,

		// Editor View
		"show_full_path": true,
		"show_minimap": false,
		"highlight_line": true,
		"fold_buttons": false,
		"word_wrap": true,

		// Editor Behavior
		"find_selected_text": true,
		"scroll_past_end": false,
		"highlight_modified_tabs": true,

		// Sidebar
		"file_exclude_patterns":
		[
		    ".DS_Store",
		    "*.pid",
		    "*.pyc",
		    "desktop.ini",
		    "*.lnk",
			"*.pdf",
		],
		"folder_exclude_patterns":
		[
			".git",
			"__pycache__"
		],

		// Package Control
		"ignored_packages":
		[
			"Vintage"
		]
	}


上面的配置使用了`Monokai-Gray`这个配色，这个配色对`Monokai`进行了一些扩展。

开启了自动缩进，将tab缩进转换为空格，且为4个空格长度，转换换行符为unix格式，字符集默认保存为utf-8


## Package Control

刚装好的Sublime是没插件包管理器的，因此首先需要安装一个包管理器，方便我们安装和管理插件。

在Sublime中按<code>ctrl + `</code>，复制下面的代码，回车，即可安装 Package Control, 这里我的Sublime版本是 ST3，如果是ST2，可以去 [Package Control][PC] 这个网站获取安装代码。

```python
import urllib.request,os,hashlib; h = 'eb2297e1a458f27d836c04bb0cbaf282' + 'd0e7a3098092775ccb37ca9d6b2e4b7d'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

安装完成后，使用`ctrl+shift+p`可以调出命令面板，输入ip就可以看到安装包的指令了。


## Git

安装这个包可以让你在命令面板中使用`git add`, `git commit` 等命令，就不需要再去调用命令行了。前提是系统需要先安装好Git, windows环境下可以安装 [msysgit][git]，并将git的执行路径加入到系统环境变量中。


## GitGutter

这个包安装后，sublime可以自动识别git管理的文件，如果某行进行了修改，GitGutter会对修改的行进行标识，如增加，删除或修改。效果是下面这个样子

![](/img/in-post/sublime-text-intro/gitgutter.png)


## SideBarEnhancements

对侧边栏进行了一些增强，支持更多的文件操作，还可以定义快捷键。


## BracketHighlighter

对括号和引号标识，妈妈再也不用担心找不到对应的括号了。


## ChangeQuotes

快键转换单双引号，强迫症患者必备，在**Key Bindings --- User**中加入快捷键

`{ "keys": ["ctrl+shift+'"], "command": "change_quotes" }`

使用方法：将光标移动到引号中间，调用快捷键，自动在单双引号之间进行转换。


## Color Highlighter

能够识别类似于 `#ff0000`这样的颜色代码，前端必备，类似的插件还有[ColorPicker](http://weslly.github.io/ColorPicker/), [GutterColor](https://github.com/ggordan/GutterColor)

## Alignment

能够定义快捷键对代码的等号进行自动对齐，默认的快捷键是`ctrl+alt+a`，与QQ的快捷键冲突，我定义的快捷键为`ctrl+shift+a`，在**Key Bindings --- User**加入

`{ "keys": ["ctrl+shift+a"], "command": "alignment" }`

使用的方法是，选中代码，调用快捷键即可。


## ConvertToUTF8

windows下经常有GBK编码的文本，这个插件可以转为UTF8识别


## IMESupport

使用输入法时，会有输入框不会跟随光标的问题，安装这个插件可以解决。


## Markdown Preview

快捷的预览**Markdown**文本，先绑定快捷键，我的配置是`alt+m`

`{ "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} }`

选中markdown文字，调用快捷键，就会打开浏览器，可以在浏览器中预览了。


## Modific

在打开的文件标签中，高亮标识修改过但未保存的文件。


## Tag

格式化 HTML/XML 文本，选中要格式化的文本，使用快捷键`ctrl+alt+f`就可以格式化了。


## nginx, Puppet, SaltStack-related, Ansible

这三个插件是语法高亮插件，做运维的可能会需要。


## SublimeCodeIntel

使sublime具有类似IDE的补全功能


## Python PEP8 Autoformat

格式化Python文件使之符合**PEP8**标准。快捷键是`ctrl+shift+r`，使用后自动格式化当前文件。

*PEP8* 中定义一行不要超过79个字符，这个应该是由于为了适应以前的终端设备而定义的，而现在的显示设备早已不是古老的终端了，因此忽略这一彪悍尊，在此插件的配置中定义`"ignore": ["E501"]`即可。

</br>
参考：


- [The Best Plugins for Sublime Text](http://ipestov.com/the-best-plugins-for-sublime-text/)
- [使用Sublime Text 3做Python开发](http://sw897.github.io/2014/02/13/sublime-text-3-for-python/)
- [配置sublime打造python编辑器](http://opslinux.com/sublime_python.html)

</br></br>

[ST]: http://www.sublimetext.com/
[PC]: https://sublime.wbond.net/installation
[git]: http://msysgit.github.io/


