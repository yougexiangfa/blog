---
title: Webpacker 最佳实践
category: rails
layout: post
---

自从Rails 6 开始，官方默认推荐通过 webpacker 来处理 Javascript 静态资源，保留 Assets Pipeline(Sprockets) 作为 CSS 处理方案。

webpack 的引入，显然解决了一些 assets pipiline 很难解决的痛点，比如对编写下一代 JS 的支持。

虽然 webpacker 的生态也还在不断完善之中，但是从 assets pipeline 切换到 webpacker 也并非无痛的，最典型的场景就是对 Rails engine 中 assets 自动加载变得很难。

好在开源的代码，给了我们再次加工的发挥空间。最佳实践的目的是尽可能减少迁移过程的痛苦，在此分享下我的实践。

## 配置 JS 处理方式

1. 移除 assets pipeline 对 js 处理的配置

`app/assets/config/manifest.js` 文件默认配置了 assets pipiline 需要处理的静态资源文件，把 js 相关的内容移除；

如果 `app/config/application.rb` 中定义了 assets.js_compressor，也一并移除；

万恶的 `uglifier`, `coffee-rails` gem 也可以永远拜拜了，再也不会为配置 execjs 的 runtime 而烦恼。

2. 修改 webpacker 配置

webpacker 默认配置中，js的路径是 `app/javascript/packs`，为了兼容老项目，我们将其路径更改为 `app/assets/javascripts`, 修改 `app/config/webpacker.yml` 如下配置：
```yaml
source_path: app/assets
source_entry_path: javascripts
```

## 兼容 Rails engine 

由于我们的Rails项目采用了组件化开发，引入了多个 engine， 大量的 js 代码散布在 engine 的 `app/assets/javascripts` 目录下。 

如何才能让 webpack 在编译的时候能够加载到 engine 下的 js 代码呢，webpack 的工作目录是 Rails主项目，最大的难点在于如何让 webpack 知道各个engine 在文件系统中的具体位置，也就是涉及到 ruby 和 js 之间分享数据。

我采用了一个比较粗暴的办法，在rails 项目启动完成的时候，将 engine 的位置信息更新到 config/webpacker.yml 文件当中，然后在 config/webpack 的配置文件中去解析这个文件，获取 engine 的路径信息。

### 1. 在 rails 项目中导出 engine 路径信息

先写个 util，方便把 ruby 中的对象（数据）存到 json文件中，

```ruby
# https://github.com/work-design/rails_com/blob/master/lib/rails_com/webpacker/yaml_helper.rb

module Webpacker
  class YamlHelper

    def initialize(path: 'config/webpacker.yml', export: 'config/webpacker.yml')
      real_path = Rails.root + path
      real_export = Rails.root + export
      
      @yaml = YAML.parse_stream File.read(real_path)
      @content = @yaml.children[0].children[0].children
      @parsed = @yaml.to_ruby[0]
      @io = File.new(real_export, 'a+')
    end
    
    def dump
      @io.truncate(0)
      @yaml.yaml @io
      @io.fsync
      @io.close
    end
    
    def append(env = 'default', key, value)
      return if Array(@parsed.dig(env, key)).include? value
      env_index = @content.find_index { |i| i.scalar? && i.value == env }

      env_content = @content[env_index + 1].children
      key_index = env_content.find_index { |i| i.scalar? && i.value == key }
      
      value_content = env_content[key_index + 1]
      if value_content.sequence?
        value_content.style = 1  # block style
        value_content.children << Psych::Nodes::Scalar.new(value)
      end

      value_content
    end
    
  end
end
```
然后定义了一个 rails 初始化过程中的回调，如果engine 下存在 app/assets/javascripts 文件夹，则将这个路径存到 tmp/share_object.json 文件。

```ruby
# https://github.com/work-design/rails_com/blob/master/lib/rails_com/engine.rb#L30
config.after_initialize do |app|
  webpack = Webpacker::YamlHelper.new
  Rails::Engine.subclasses.each do |engine|
    engine.paths['app/assets'].existent_directories.select(&->(i){ i.end_with?('javascripts') }).each do |path|
      webpack.append 'resolved_paths', path
    end
  end
  webpack.dump
end
```

### 2. js 通过数据文件 或许相关的路径信息；

```js
// https://github.com/work-design/rails_com/blob/master/package/index.js

const { basename, dirname, join, relative, resolve } = require('path')
const { sync } = require('glob')
const extname = require('path-complete-extname')
const config = require('@rails/webpacker/package/config')

const paths = () => {
  const { extensions } = config
  let glob = extensions.length === 1 ? `**/*${extensions[0]}` : `**/*{${extensions.join(',')}}`
  let result = {}

  config.resolved_paths.forEach((rootPath) => {
    const ab_paths = sync(join(rootPath, glob))

    ab_paths.forEach((path) => {
      const namespace = relative(join(rootPath), dirname(path))
      const name = join(namespace, basename(path, extname(path)))
      result[name] = resolve(path)
    })
  })

  return result
};

module.exports = paths
```

这里我们导出了两个数据：

* paths：所有的js文件路径，用于webpack 的 entry 配置；
* resolved_roots: 所有存在js文件的路径，用于配置 webpack 的 resolve.modules，告诉webpack 在解析代码时需要搜索哪些路径，与 assets pipeline 的 assets.paths 配置功能一致；

### 3. 暴露 jquery, rails_ujs 等库

一般的项目都使用了 这两个js库，为了能够在代码里使用 `$('body')`, `Rails.ajax` 这样的代码，我们需要增加一点配置；

```javascript
module.exports = {
  test: require.resolve('jquery'),
  use: [
    {
      loader: 'expose-loader',
      options: 'jQuery'
    },
    {
      loader: 'expose-loader',
      options: '$'
    }
  ]
}
```

### 4. 接下来修改下 config/webpack 下的 配置文件

```js
// config/webpack/environment.js
const share_path = require('rails_com/package/index')

const jquery = require('rails_com/package/loaders/jquery')
const rails_ujs = require('rails_com/package/loaders/rails_ujs')

environment.loaders.append('jquery', jquery)
environment.loaders.append('rails_ujs', rails_ujs)

const env = environment.toWebpackConfig()
env.entry = Object.assign(share_path.paths(), env.entry)
env.resolve.modules = env.resolve.modules.concat(share_path.resolved_roots)
```

至此，我们就可以直接编译和使用 engine 下所有的js文件了。

## 改写 assets pipiline 中的 require 语法

```javascript
//= require channels  // assets pipeline
import 'channels' // webpacker 
```

## 其他提示

1. webpack 相关配置文件是为 nodejs 使用的，所以使用 nodejs 的模块语法：`module.exports/require`，前端 js 代码会经过 babel 编译，虽然webpack能理解 CommonJS 等多种模块体系，但是推荐使用 ES6 的 `export/import` 语法。

2. 在Rails开发模式下，如果没有启动 webpack-dev-server, rails会将前端代码编译到 public 目录下，此时修改js代码是不能立即生效的。所以推荐在开发js时，同时启动`bin/webpack-dev-server`。

3. 当新增或删除了 js 文件之后，entry 改变之后，需要重启 bin/webpack-dev-server;

4. 由于 config/webpacker.yml 会根据项目的实际路径进行更新，建议git update-index --assume-unchanged config/webpacker.yml；