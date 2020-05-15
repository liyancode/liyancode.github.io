---  
layout: post  
title:  "Ruby Bundler"  
date:   2014-07-21 09:16:58  
category: ruby
tags: [ruby,tool]
author: liyan  
image: image/larg.png  
--- 

# Ruby_Bundler（中文参考）[bundler.io](http://www.bundler.io)
![](image/bundlerlogo.jpeg)

## Bundler 是什么？
> Bundler能够跟踪并安装所需的特定版本的gem，以此来为Ruby项目提供一致的运行环境。  
> Bundler是Ruby依赖管理的一根救命稻草，它可以保证你所要依赖的gem如你所愿地出现在开发、测试和生产环境中。利用Bundler启动项目简单到只用一条命令： ``` bundle install ```  


## 快速上手
Bundler使用起来非常简单。  

**第一步：打开命令行输入如下命令：**  
```
$ gem install bundler
```  
<!--break-->
**第二步：在Ruby项目的根目录下新建文件 _Gemfile_（[详细](#gemfile)）**  
文件内容像下面这样：  
> **source 'https://rubygems.org'  \# gem源地址  
gem 'nokogiri'                     \# 被依赖的gem，不指定版本默认安装最新版本  
gem 'rack', '~>1.1'                \# 指定版本  
gem 'rspec', :require => 'spec'    \# 主文件名和gem名称不一致**  

**第三步：利用Bundler从你指定的gem源地址下载安装gem**  
```
$ bundle install  
$ git add Gemfile Gemfile.lock
```  

第一条 **[bundle install命令](#bundle-install)** 执行完成之后，所有在Gemfile中指定的依赖gem会被安装到当前的环境中，并且在Gemfile的同级目录下会生成一个新的文件： **_Gemfile.lock_**,这个文件里会详细列出Gem,Platform以及Dependency的信息。  
第二条git命令将Gemfile和Gemfile.lock两个文件添加到你的git版本库中，这样做的好处是，可以确保其他开发者以及在不同的开发环境下，你的项目会依赖到完全相同的第三方库。  

## 更详细的介绍

#### Gemfile  

Gemfile应该包含至少一个gem源地址，源地址以URL的形式指定：  
> **source 'https://rubygems.org'**  

用 ```bundle init``` 命令可以生成Gemfile，缺省的gem源地址是rubygems.org  
**_如果包含不止一个gem源地址，Bundler查找的顺序是从最后一行源地址往第一行查。_**  
有的gem源需要用户名和密码才可以连接，对于这种源可以使用 ```bundle config``` 命令设置名户名和密码，并且每次换新环境在执行install之前都要设置一次。 
```
$ bundle config https://gems.example.com/ user:password
```  

有些公司配置的gem源，身份信息可以直接写在源地址的URL中：  
> **source "https://user:password@gems.example.com"**  

**_直接写在URL中的身份信息优先级高于_** ```config```  
跟在source后面的就是你要依赖的gem的声明，包括gem名称和版本号，例如：
> **gem 'nokogiri'  
    gem 'rails', '3.0.0.beta3'  
    gem 'rack',  '>=1.0'  
    gem 'thin',  '~>1.1'**  

大部分指定标记例如 ```>=``` 都是本身的意思（大于或等于）， ```~>``` 意思比较特殊：  
```~>2.0.3``` 表示 ```>=2.0.3 and <2.1```  
```~>2.1``` 表示 ```>=2.1 and <3.0```  
```~>2.2.beta``` 表示匹配预发布版本，例如 ```2.2.beta.12```  
如果一个gem的主文件名和gem名称不一致，这时候需要指定被require的主文件名，例如：  
> **gem 'rspec', :require => 'spec' \# 主文件名和gem名称不一致  
    gem 'sqlite3'**  
    
如果指定 ```:require=>false``` Bundler就不会require这个gem，但是依然会安装和维护这个依赖，例如：  
> **gem 'rspec', :require => false  
    gem 'sqlite3'**  
    
**通过调用 ```Bundler.require``` 来把Gemfile中的gem require到你的项目中，关于 ```Bundler.require``` 详见后文。**  
如果部分gem需要从私有服务器下载，可以通过指定私有地址覆盖缺省的源地址。例如一个私有gem服务器只有一个gem被你引用，可以通过最简单的 ```:source``` 选项指定到对应的gem上：  
> **gem 'my_gem', '1.0', :source => 'https://gems.example.com'**  

如果有超过一个的gem指向同一个私有地址，可以通过 ```source``` 块将这些gem组织到一起：  
> **source 'https://gems.example.com' do  
       &nbsp;&nbsp;&nbsp;&nbsp;gem 'my_gem', '1.0'  
       &nbsp;&nbsp;&nbsp;&nbsp;gem 'another_gem', '1.2.1'  
    end**  
    
私有地址的身份信息可以配置在URL中也可以通过 ```bundle config``` 来配置，具体参考前文描述。  
Git仓库也可以作为有效的gem源地址，只要这个仓库包含可用的gem，可以通过 ```:tag```，```:branch``` 或 ```:ref``` 指定拉取的分支。缺省是 ```master``` 分支：  
> **gem 'nokogiri', :git => 'https://github.com/tenderlove/nokogiri.git', :branch => '1.4'**  

**如果指定的Git仓库不包含 ```.gemspec``` 文件，bundler会自动创建一个简单的，不包含任何的依赖、不可执行并且没有C扩展。对于简单的gem来说有效，但是复杂一些的就不行。
如果Git仓库不包含 ```.gemspec``` 文件，最好就不要用这个仓库。***  
如果想直接使用文件系统中解压好的gem，用 ```:path``` 选项指定包含gem文件的路径：  
> **gem 'extracted_library', :path => './vendor/extracted_library'**  

如果想直接从文件系统使用多个gem，可以设置全局‘path’选项指定到包含这些gem的路径，gemspec文件会被自动的从子文件夹加载：  
> **path 'components' do  
      &nbsp;&nbsp;&nbsp;&nbsp;gem 'admin_ui'  
      &nbsp;&nbsp;&nbsp;&nbsp;gem 'public_ui'  
    end**  

依赖还可以被组织成group，具体的group用法参见 ```Bundler.require```  
Gemfile中还可以指定ruby的版本，如果Gemfile被加载在不同版本的ruby环境中，Bundler会报异常并告诉你ruby的版本不对：  
> **ruby '2.1.0'**  

#### bundle install
在执行 ```bundle install``` 之前，请确保Gemfile中所有依赖对于你的项目来说都是可用的。  
```bundle install``` 执行时可以加以下这些可选的选项：  
```
$ bundle install [--binstubs=PATH] [--clean] [--deployment] [--frozen]  
                 [--full-index] [--gemfile=FILE] [--local] [--no-cache]  
                 [--no-prune] [--path=PATH] [--quiet] [--shebang=STRING]  
                 [--standalone[=GROUP [GROUP...]] [--system]  
                 [--without=GROUP[ GROUP...]] [--retry=NUMBER]  
                 [--trust-policy=SECURITYLEVEL]  
```  
可选选项：  
```--binstubs```： 生成被bundle过的gem的存根记录到 ./bin文件夹下  
```--clean```： 安装完依赖自动执行clean操作  
```--deployment```： 开发环境和集成环境使用的缺省设置  
```--frozen```： 安装完成后防止Gemfile.lock文件被更新  
```--full-index```： Use the rubygems modern index instead of the API endpoint  
```--gemfile```： Use the specified gemfile instead of Gemfile  
```--jobs```：  Install gems using parallel workers.  
```--local```： Do not attempt to fetch gems remotely and use the gem cache instead  
```--no-cache```： Don't update the existing gem cache.  
```--no-prune```： Don't remove stale gems from the cache.  
```--path```： Specify a different path than the system default ($BUNDLE_PATH or $GEM_HOME). Bundler will remember this value for future installs on this machine  
```--quiet```： Only output warnings and errors.  
```--retry```： Retry network and git requests that have failed.  
```--shebang```： Specify a different shebang executable name than the default (usually 'ruby')  
```--standalone```： Make a bundle that can work without the Bundler runtime  
```--system```： Install to the system location ($BUNDLE_PATH or $GEM_HOME) even if the bundle was previously installed somewhere else for this application  
```--trust-policy```： Sets level of security when dealing with signed gems. Accepts `LowSecurity`, `MediumSecurity` and `HighSecurity` as values.  
```--without```： Exclude gems that are part of the specified named group.  


