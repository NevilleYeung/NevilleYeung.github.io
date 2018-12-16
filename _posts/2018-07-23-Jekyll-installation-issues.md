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
    

### 问题2       

<font size="3" style="font-weight:bold">【现象】</font>  
安装jekyll一段时间后，突然在cmd里找不到jekyll命令和ruby命令。以管理员身份启动cmd后，能正常找到ruby命令，但依然找不到jekyll。  
1、输入“jekyll -v”，报“'jekyll' 不是内部或外部命令，也不是可运行的程序”。  
2、执行命令“gem install jekyll”，显示找不到gem命令，报错与jekyll类似。  
3、重新安装gem和jekyll命令后，执行jekyll命令却报错如下。  
```     
E:\Code\Github\NevilleYeung.github.io>jekyll s
Traceback (most recent call last):
        10: from D:/programs/Ruby25-x64/bin/jekyll:23:in `<main>'
         9: from D:/programs/Ruby25-x64/bin/jekyll:23:in `load'
         8: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/jekyll-3.8.5/exe/jekyll:11:in `<top (required)>'
         7: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/jekyll-3.8.5/lib/jekyll/plugin_manager.rb:50:in `require_from_bundler'
         6: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler.rb:107:in `setup'
         5: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:26:in `setup'
         4: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:26:in `map'
         3: from D:/programs/Ruby25-x64/lib/ruby/2.5.0/forwardable.rb:229:in `each'
         2: from D:/programs/Ruby25-x64/lib/ruby/2.5.0/forwardable.rb:229:in `each'
         1: from D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:31:in `block in setup'
D:/programs/Ruby25-x64/lib/ruby/gems/2.5.0/gems/bundler-1.16.3/lib/bundler/runtime.rb:313:in `check_for_activated_spec!': You have already activated jekyll 3.8.5, but your Gemfile requires jekyll 3.8.3. Prepending `bundle exec` to your command may solve this. (Gem::LoadError)

``` 

<font size="3" style="font-weight:bold">【原因】</font>  
NAN  

<font size="3" style="font-weight:bold">【解决方案】</font>  
1、重新安装gem  
在我本地的ruby安装目录下找到了安装文件setup.rb。  
D:\programs\Ruby25-x64\lib\ruby\gems\2.5.0\gems\kramdown-1.17.0  
执行命令【ruby setup.rb】，安装gem。  
```     
D:\programs\Ruby25-x64\lib\ruby\gems\2.5.0\gems\kramdown-1.17.0>ruby setup.rb
setup.rb:283: warning: key "bin-dir" is duplicated and overwritten on line 284
---> bin
<--- bin
---> lib
---> lib/kramdown
---> lib/kramdown/converter
……
略

``` 
【注意】最好下载最新的gem版本，此处偷了个懒。  

2、安装jekyll  
执行命令【gem install jekyll】，安装jekyll。  

3、问题3的报错很明显，当前jekyll的版本是3.8.5，而Gemfile需要的版本是3.8.3。  
用它提示的命令即可解决。  
```     
E:\Code\Github\NevilleYeung.github.io>bundle exec jekyll s
Configuration file: E:/Code/Github/NevilleYeung.github.io/_config.yml
            Source: E:/Code/Github/NevilleYeung.github.io
       Destination: E:/Code/Github/NevilleYeung.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 2.245 seconds.
  Please add the following to your Gemfile to avoid polling for changes:
    gem 'wdm', '>= 0.1.0' if Gem.win_platform?
 Auto-regeneration: enabled for 'E:/Code/Github/NevilleYeung.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
……
略

``` 
【注意】最好重新安装下gem或者更换jekyll的版本。  
  

<br>

转载请注明：[蚊帐的博客](https://nevilleyeung.github.io) » [点击阅读原文](https://nevilleyeung.github.io/2018/07/Jekyll-installation-issues/) 

 



