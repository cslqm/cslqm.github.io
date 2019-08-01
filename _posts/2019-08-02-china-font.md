---
layout: post
title: "ubuntu自动更新系统软件后中文出现乱码"
subtitle: 'ubuntu auto update system software'
author: "cslqm"
header-style: text
tags:
  - Linux
---

今天系统提示有软件可以更新了，随手点了更新。然后就出现了中文乱码。一开始只有雷鸟mail中有，我还以为是软件更新删除了微软雅黑字体。最后发现原来是字体显示优先级的配置文件被替换掉了。

在/etc/fonts/conf.avail/目录下多了一个64-language-selector-prefer.conf.dpkg-dist文件顶替了原来配置文件64-language-selector-prefer.conf的功能。

如下是64-language-selector-prefer.conf.dpkg-dist的内容，可以看到日文JP优先与中文SC/TC。
``` xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
	<alias>
		<family>sans-serif</family>
		<prefer>
			<family>Noto Sans CJK JP</family>
			<family>Noto Sans CJK KR</family>
			<family>Noto Sans CJK SC</family>
			<family>Noto Sans CJK TC</family>
			<family>Noto Sans CJK HK</family>
		</prefer>
	</alias>
	<alias>
		<family>serif</family>
		<prefer>
			<family>Noto Serif CJK JP</family>
			<family>Noto Serif CJK KR</family>
			<family>Noto Serif CJK SC</family>
			<family>Noto Serif CJK TC</family>
		</prefer>
	</alias>
	<alias>
		<family>monospace</family>
		<prefer>
			<family>Noto Sans Mono CJK JP</family>
			<family>Noto Sans Mono CJK KR</family>
			<family>Noto Sans Mono CJK SC</family>
			<family>Noto Sans Mono CJK TC</family>
			<family>Noto Sans Mono CJK HK</family>
		</prefer>
	</alias>
</fontconfig>

```

修改顺序后
``` xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
	<alias>
		<family>sans-serif</family>
		<prefer>
			<family>Noto Sans CJK SC</family>
			<family>Noto Sans CJK TC</family>
			<family>Noto Sans CJK HK</family>
			<family>Noto Sans CJK JP</family>
			<family>Noto Sans CJK KR</family>
		</prefer>
	</alias>
	<alias>
		<family>serif</family>
		<prefer>
			<family>Noto Serif CJK SC</family>
			<family>Noto Serif CJK TC</family>
			<family>Noto Serif CJK HK</family>
			<family>Noto Serif CJK JP</family>
			<family>Noto Serif CJK KR</family>
		</prefer>
	</alias>
	<alias>
		<family>monospace</family>
		<prefer>
			<family>Noto Sans Mono CJK SC</family>
			<family>Noto Sans Mono CJK TC</family>
			<family>Noto Sans Mono CJK HK</family>
			<family>Noto Sans Mono CJK JP</family>
			<family>Noto Sans Mono CJK KR</family>
		</prefer>
	</alias>
</fontconfig>

```

重启雷鸟之后，发现字体显示正常。

看来ubuntu的软件还是不能随便升级啊。