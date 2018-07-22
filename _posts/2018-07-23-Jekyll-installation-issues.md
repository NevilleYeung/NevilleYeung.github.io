---
layout: post
title: "Jekyll安装启动问题集"
date: 2018-07-23 
tags: Jekyll
---


### 问题1       

<font size="3" style="font-weight:bold">【现象】</font>  
在Windows上安装了Jekyll后，通过在cmd执行命令【jekyll s】，却报错，如下。  
```     
E:\Code\Github\NevilleYeung.github.io>jekyll s
Traceback (most recent call last):
        15: from D:/programs/Ruby25-x64/bin/jekyll:23:in `<main>'
        14: from D:/programs/Ruby25-x64/bin/jekyll:23:in `load'
        13: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/jekyll-3.8.3/exe/jekyll:11:in `<top (required)>'
        12: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/jekyll-3.8.3/lib/jekyll/plugin_manager.rb:50:in `require_from_bundler'
        11: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler.rb:107:in `setup'
        10: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:20:in `setup'
         9: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:108:in `block in definition_method'
         8: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/definition.rb:227:in `requested_specs'
         7: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/definition.rb:238:in `specs_for'
         6: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/definition.rb:171:in `specs'
         5: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/definition.rb:258:in `resolve'
         4: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/resolver.rb:22:in `resolve'
         3: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/resolver.rb:48:in `start'
         2: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/resolver.rb:257:in `verify_gemfile_dependencies_are_found!'
         1: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/resolver.rb:257:in `each'
D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/resolver.rb:289:in `block in verify_gemfile_dependencies_are_found!': Could not find gem 'jekyll-paginate x64-mingw32' in any of the gem sources listed in your Gemfile. (Bundler::GemNotFound)

``` 
<font size="3" style="font-weight:bold">【原因】</font>  
没有安装jekyll-paginate，安装即可。  


<font size="3" style="font-weight:bold">【解决方案】</font>  
执行命令【gem install jekyll-paginate】，安装jekyll-paginate。  
```     
E:\Code\Github\NevilleYeung.github.io>gem install jekyll-paginate
Fetching: jekyll-paginate-1.1.0.gem (100%)
Successfully installed jekyll-paginate-1.1.0
Parsing documentation for jekyll-paginate-1.1.0
Installing ri documentation for jekyll-paginate-1.1.0
Done installing documentation for jekyll-paginate after 0 seconds
1 gem installed

``` 

后续又相继遇到了类似的问题，均是安装相关组件即可解决。  

  

<br>

转载请注明：[蚊帐的博客](https://nevilleyeung.github.io) » [点击阅读原文](https://nevilleyeung.github.io/2018/07/Jekyll-installation-issues/) 

 



