---
title: Webpacker 最佳实践
category: rails
layout: post
---

自从 Rails 6 开始，官方默认采用 webpacker 来处理 Javascript，保留 Assets Pipeline(Sprockets) 作为 CSS、图片等静态资源处理方案。

webpack 的引入显然解决了一些 Assets Pipiline 很难解决的痛点，比如对编写下一代 JS 的支持。

虽然 webpacker 的生态也还在不断完善之中，但是从 assets pipeline 切换到 webpacker 也并非无痛的，最典型的场景就是对 Rails engine 中 assets 自动加载变得很难。

不过我们可以自己动手，丰衣足食，尽可能减少迁移过程的痛苦，在此分享下我的实践。

## 配置 JS 处理方式

### 移除 assets pipeline 对 js 处理的配置

`app/assets/config/manifest.js` 文件默认配置了 assets pipiline 需要处理的静态资源文件，把 js 相关的内容移除；

如果 `app/config/application.rb` 中定义了 assets.js_compressor，也一并移除；

曾经的 `uglifier`, `coffee-rails` 等 gem 也可以永远拜拜了，再也不会为配置 execjs 的 runtime 而烦恼。

### 修改 webpacker 配置

webpacker 默认配置中，js的路径是 `app/javascript/packs`，为了兼容老项目，我们将其路径更改为 `app/assets/javascripts`, 修改 `app/config/webpacker.yml` 中的如下配置：
```yaml
source_path: app/assets
source_entry_path: javascripts
```

## 兼容 Rails engine 

由于我们的Rails项目采用了组件化开发，引入了多个 engine， 大量的 js 代码散布在 engine 的 `app/assets/javascripts` 目录下。 

如何才能让 webpack 在编译的时候能够加载 engine 下的 js 代码呢，webpack 的工作目录是 Rails主项目，关键点就是如何让 webpack 知道各个 engine 在文件系统中的具体位置，也就是 ruby 和 js 之间分享数据。

我采用了一个比较粗暴的办法，在 rails 项目启动完成的时候，将 engine 的位置信息更新到 config/webpacker.yml 文件当中，然后在 config/webpack 的配置文件中去解析这个文件，获取 engine 的路径信息。

### 1. 在 rails 项目中导出 engine 路径信息

先实现对 webpacker.yml 文件的读写，代码如下：

```ruby
# https://github.com/work-design/rails_com/blob/master/lib/rails_com/webpacker/yaml_helper.rb

module Webpacker
  class YamlHelper
    
    # uses config/webpacker_template.yml in rails_com engine as default,
    # config/webpacker_template.yml in Rails project will override this.
    def initialize(template: 'config/webpacker_template.yml', export: 'config/webpacker.yml')
      template_path = (Rails.root + template).existence || RailsCom::Engine.root + template
      export_path = Rails.root + export
      
      @yaml = YAML.parse_stream File.read(template_path)
      @content = @yaml.children[0].children[0].children
      @parsed = @yaml.to_ruby[0]
      @io = File.new(export_path, 'w+')
    end
    
    def dump
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

然后在 rails 初始化过程中增加一个回调，如果相应的 engine 下存在 app/assets/javascripts 文件夹，则将这个路径写入到`config/webpacker.yml`文件。

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
}

module.exports = paths
```

这里我们导出了所有的 js 文件路径，用于 webpack 的 entry 配置。

### 3. 暴露 jquery, rails_ujs 等库

一般的项目都使用了这两个js库，为了能够在代码里使用 `$('body')`, `Rails.ajax` 这样的代码，我们需要增加一点配置；

```javascript
# https://github.com/work-design/rails_com/tree/master/package/loaders
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
我没有对 webpack 和 expose-loader 的代码做深入阅读，不过我认为在 import jquery 的时候直接赋值给 windows.$ 也就解决问题了，不知道 expose-loader 是否还有其他的效果，知晓的朋友可以留言告知下。 

### 4. 接下来修改下 config/webpack 下的 配置文件

```js
// config/webpack/environment.js
const { environment } = require('@rails/webpacker')
const { resolve } = require('path')
const paths = require('rails_com')

const jquery = require('rails_com/package/loaders/jquery')
environment.loaders.append('jquery', jquery)

const env = environment.toWebpackConfig()
env.entry = Object.assign(paths(), env.entry)
env.resolve.modules = env.resolve.modules.concat(resolve('node_modules'))

module.exports = env
```

config/webpacker.yml 中的 resolved_paths 配置了所有存在js文件的路径，用于配置 webpack 的 resolve.modules，告诉webpack 在解析代码时需要搜索哪些路径，与 assets pipeline 的 assets.paths 配置功能一致；

同时也配置了 babel-loader 中的生效路径。 

至此，我们就可以直接编译和使用 engine 下所有的js文件了。
 
## 改写 assets pipiline 中的 require 语法

```javascript
//= require channels  // assets pipeline
import 'channels' // webpacker 
```

## 其他提示

1. webpack 相关配置文件是为 nodejs 使用的，所以使用 nodejs 的模块语法：`module.exports/require`，前端 js 代码会经过 babel 编译，虽然 webpack能理解 CommonJS 等多种模块体系，但是推荐使用 ES6 的 `export/import` 语法。

2. 在Rails开发模式下，如果没有启动 webpack-dev-server, rails会将前端代码编译到 public 目录下，此时修改js代码是不能立即生效的。所以推荐在开发js时，同时启动`bin/webpack-dev-server`。

3. 当新增或删除了 js 文件之后，entry 改变之后，需要重启 bin/webpack-dev-server。

4. 由于 config/webpacker.yml 会根据项目的实际路径进行更新，建议将其在 git 中忽略。

5. 更新 Gemfile 配置： gem 'webpacker', require: File.exist?('config/webpacker.yml')，这样可以杜绝 config/webpacker.yml 未生成时的报错。

6. config.webpacker.xxx = xx if config.respond_to?(:webpacker) 这个配置主要是解决上述第5条配置的副作用。