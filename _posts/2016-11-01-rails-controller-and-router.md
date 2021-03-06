---
layout: post
title: Rails 里的 Controller 和 Router
---

功能是在 Action Pack 里实现的。

Router定义了外部访问时URL的样子，通过匹配筛选后，交给对应的Controller处理。

玩转了它们，就可以构造自己想要的url，并对相应的请求指定 Controller 进行处理。

其实，一个 Controller 不一定需要 Model， 因为它只是处理 Route 传来的请求，它也不需要 View，因为
它可以 render 任何 views 目录的模板。

Rails 是很灵活的，它里面的 MVC 可以单独活动，只是秉着“惯例胜过配制”的原则：

> CONVENTIONS OVER CONFIGURATIONS

默认了很多根据名字的方便，比如 PostsController 的 view 就在 view/posts 目录里，而它有个
Model 是 Post，数据库建的表是 posts。然后在 config/routes.rb 里：

```ruby
resources :posts
```

就会建好指向 PostController 的 CURD 操作，并定义好 RESTful 的 url。

是的，很方便，也符合信息论理论，用短的编码实现常用的操作。非常用的就用参数配制嘛！

# Router

## Resourceful Routes

就是我们常见的，Rails 默认的规则。

其实也应该尽量用这些 Resourceful 路由：

- Route Map 里把访问的东西看成资源，不出现动词（都在 HTTP 的方法里了）
- Controller 里用七种方法：index, new, create, show, edit, update, delete
- 用规范的 HTTP 方法：GET, POST, PUT/PATCH, DELETE

[devise](https://github.com/plataformatec/devise) 就是一个很好的例子：

```ruby
devise_for :users, controllers: {
  registrations: 'users/registrations',
  sessions: 'users/sessions',
  passwords: 'users/passwords'
}
```

看它把 sessions 从 registrations 里分离处理，登录就是 create 一个 session，登出就是
destroy 当前的 session，而不出写两个 action：sign_in 和 sing_out。这样路由表里就出现了
动词，而我们需要的只是名词，因为名词才能被抽象成资源。

如果用 resource 的话，是没有 index 的，而且 member 里也不用指明 id 了，因为就那一个资源。


## Non-Resourceful Routes

而往往我们构造的页面，不会像脚手架生成的那样死板。

### 定义 url

在 url 定义里，冒号 —— ":" 后面的就是传到 Controller params 里的 key，所有有：

- :controller 对应app里的一个Controller
- :action 对应Controller里的Action
- :其他 可以自定义，会传到params里
- () 表示这是可选的

自定义参数的 url 需要明确指明 Controller 和 Action，否则它不知道参数传到哪里。

比如：

```ruby
get ':controller(/:action(/:id))'
```

- 路径： /photos/show/1?user_id=2
- 得到： params = { controller: 'photos', action: 'show', id: '1', user_id: '2' }

```ruby
get ':controller/:action/:id/:user_id'
```

- 路径： /photos/show/1/2
- 得到： params = { controller: 'photos', action: 'show', id: '1', user_id: '2' }

```ruby
get ':controller/:action/:id/with_user/:user_id'
```

- 路径： /photos/show/1/with_user/2
- 得到： params = { controller: 'photos', action: 'show', id: '1', user_id: '2' }

自定义路径时记得指定 action，要不路由不知道指到 Controller 的哪个方法：

```ruby
# /photos/:id/:index(.:format), preview_photo_path(@photo, 1)
get ':index', to: "photos#show", as: :preview
```

- 路径：/photos/1/1
- 得到：params = {controller: 'photos', action: 'show', id: '1', index: '1' }

注： as 指明了 url helper 的名字，否则不会生成 url helper 的。

### 改变 url 的样子或对应的 Controller

- as:    定义 url helper 的名字
- path:  定义 url 前缀的名字

块：
- 默认没有：     就是在当前 resource 后面
- collection:  resource 集合
- member:      当前 resource

```ruby
# 关于 photes 的处理，就会交给 ImagesController 来处理，但是 path helper 还是 photos。
resources :photos, controller: 'images'

# 改变 path helper 的名字
resources :photos, as: 'images'

# 那么这个意思就是 —— 只改变 url 为 /photos/xxxx
# Controller 还是 ImagesController，helper 也是 images_path
resources :photos, controller: 'images', as: 'images'

# 上面情况的相反，url 是 /images/xxxx，其他是 photos
resources :photos, path: 'images'

# 所以上上可以写作
resources :images, path: 'photos'

# 这么表示 /:id 直接传到 PhotosController
resources :photos, path: ''

# 只有符合条件的 id 才能传给 Controller
resources :photos, constraints: { id: /[A-Z][A-Z][0-9]+/ }

# 覆盖 Actions
resources :photos, path_names: { new: 'make', edit: 'change' }

# 默认就是用 name 代替 id，比如 show action 就成了 、photos/:name
resources :photos, param: :name
# 当然相应要 通知 model，咱不用 id 了，用 name 了。就是需要覆盖 to_param 方法
class Photo < ApplicationRecode
  def to_param
    name
  end
end

```

namespace 和 scope，它们都是给 url 加给前缀，而 namespace 也给 helper 加个前缀，
scope就只是给 url 加前缀。


# Controller
Controller 根据 Router 传来的请求，操作 Model，并生成 View。

但这里注意了，生成的 View 是由模板和 Model 一起生成的。

传到 Router 的请求，也是用户在 View 上点击传来的。

因此，这是一个蛇咬尾巴的故事。

Controller 是执行操作的地方，比如 render，redirect_to 等，而不是处理逻辑的地方。
Model 才是处理逻辑的地方，还方便在其他地方使用。所有如果在这里出现了复杂的逻辑，就应该考虑重构代码
换个地方了。

## ActionController::Parameters

为了安全，一般不在Controller里直接使用params这个变量，而是通过permit筛选一遍后只提取许可的参数。
从而保证入口参数可控可靠。

而且只有permitted的参数才可以进行mass assign，防止改变不想改变的数据，比如is_admin.

- require 提取一个或多个key的值
- permit 授权一个多个参数

```ruby
params = ActionController::Parameters.new(person: { name: 'Francesco' })

# require后还是一个Parameters，并且没有permitted，只是将person提取出来。
params.require(:person)
=> <ActionController::Parameters {"name"=>"Francesco"} permitted: false>

# 现在params又嵌了一层
params
=> <ActionController::Parameters {"person"=><ActionController::Parameters {"name"=>"Francesco"} permitted: false>} permitted: false>

# 只有使用permit后，才能permitted，然后可以使用
person_params = params.require(:person).permit(:name)
=> <ActionController::Parameters {"name"=>"Francesco"} permitted: true>

person_params[:name]
=> "Francesco"

# 或一开始就只用permit，得到一个hash参数
permitted_params = params.permit(person:[:name])
=> <ActionController::Parameters {"person"=><ActionController::Parameters {"name"=>"Francesco"} permitted: true>} permitted: true>

permitted_params[:person][:name]
=> "Francesco"
```

## callback —— 回调
Controller 里的回调有 before、after和around xxxx_action，然后它们又可以 prepend_xxxx_action 和 skip_xxxx_action。

xxxx_action 其实是在相应的 action 里，比如 show 里插入代码，所以它在逻辑上属于 action 里的代码，通过 paras[:action] 就可以看出来。知道这个，就可以在使用 pundit 时很方便的在 before_action 里 authorize @record 了。

prepend_xxxx_action 就是将当前回调 prepend 到回调链里，因为默认是 append，排在后面的。

在 Controller 里用下面方法定义：

```ruby
# rails/actionpack/lib/abstract_controller/callback.rb
[:before, :after, :around].each do |callback|
  define_method "#{callback}_action" do |*names, &blk|
    _insert_callbacks(names, blk) do |name, options|
      set_callback(:process_action, callback, name, options)
    end
  end

  define_method "prepend_#{callback}_action" do |*names, &blk|
    _insert_callbacks(names, blk) do |name, options|
      set_callback(:process_action, callback, name, options.merge(prepend: true))
    end
  end

  # Skip a before, after or around callback. See _insert_callbacks
  # for details on the allowed parameters.
  define_method "skip_#{callback}_action" do |*names|
    _insert_callbacks(names) do |name, options|
      skip_callback(:process_action, callback, name, options)
    end
  end

  # *_action is the same as append_*_action
  alias_method :"append_#{callback}_action", :"#{callback}_action"
end
```

从上面的代码可以看到，Controller 用的是 process_action 事件，当然还有其他事件，比如：

active_job 有 perform, enqueue。
active_model 更多了，有 validation, commit, save, rollback 等等。

是的，所有的回调都是通过 set_callback 方法实现的：

```ruby
# rails/activesupport/lib/active_support/callback.rb
# Install a callback for the given event.
#
#   set_callback :save, :before, :before_method
#   set_callback :save, :after,  :after_method, if: :condition
#   set_callback :save, :around, ->(r, block) { stuff; result = block.call; stuff }
#
# The second argument indicates whether the callback is to be run +:before+,
# +:after+, or +:around+ the event. If omitted, +:before+ is assumed. This
# means the first example above can also be written as:
#
#   set_callback :save, :before_method
#   ......
def set_callback(name, *filter_list, &block)
  type, filters, options = normalize_callback_params(filter_list, block)
  self_chain = get_callbacks name
  mapped = filters.map do |filter|
    Callback.build(self_chain, filter, type, options)
  end

  __update_callbacks(name) do |target, chain|
    options[:prepend] ? chain.prepend(*mapped) : chain.append(*mapped)
    target.set_callbacks name, chain
  end
end
```

### 如何定义一个可以回调的事件
定义在 activesupport 里的 callback 模块可以很方便的拿来使用。

> Callbacks are code hooks that are run at key points in an object's life cycle.

一个常用的情景是在子类继承父类时，不需要覆盖或重写父类的方法就能修改增强或修改父类的功能。

使用步骤：

1. 使用 define_callbacks 指明支持 callback 的事件(们)。这样会为这个事情定义相关的方法。
2. 在一个方法里用 run_callbacks 来触发事件，并执行相应功能，这个方法一般和事件同名。
3. 使用 set_callback 来定义 callback 方法。

```ruby
class Record
  include ActiveSupport::Callbacks
  define_callbacks :save

  def save
    run_callbacks :save do
      puts "- save"
    end
  end
end

class PersonRecord < Record
  set_callback :save, :before, :saving_message
  def saving_message
    puts "saving..."
  end

  set_callback :save, :after do |object|
    puts "saved"
  end
end

person = PersonRecord.new
person.save

# Output:
# saving...
# - save
# saved
```


## 其他

[helper_method(*meths)](http://api.rubyonrails.org/classes/AbstractController/Helpers/ClassMethods.html#method-i-helper_method) 可以让 Controller 里方法在 View 也可以使用。

```ruby
# File actionpack/lib/abstract_controller/helpers.rb, line 61
def helper_method(*meths)
  meths.flatten!
  self._helper_methods += meths

  meths.each do |meth|
    _helpers.class_eval "def #{meth}(*args, &blk)          # def current_user(*args, &blk)
      controller.send(%(#{meth}), *args, &blk)             #   controller.send(:current_user, *args, &blk)
    end                                                    # end
  ", __FILE__, __LINE__ + 1
  end
end
```


# 参考
- http://guides.rubyonrails.org/routing.html
- http://api.rubyonrails.org/classes/ActionController/Parameters.html
- http://www.nascenia.com/ruby-on-rails-development-principles/
- http://stackoverflow.com/questions/2837102/changing-the-id-parameter-in-rails-routing
- https://www.youtube.com/watch?v=MjHO-LjhuL4
