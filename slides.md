title: Easing into Node
author:
  name: Matt McKegg
  twitter: MattMcKegg
  url: https://github.com/mmckegg
style: styles.css
output: index.html

--

# Easing into Node

## the slippery slope

--

### Why Use Node?

- **One less language** to think about
- potential for **sharing client and server code**
- the **abstactions** are in the right place
- **best module system** for writing modular apps
- **npm** - lego for your codes
- **everything is async** - great for the emerging realtime web
--

**We have an existing RoR app,** but we want:
- Realtime push updating
- Client-side JS code to be more modular and use packages from npm.
- Rendering of our templates on both server and client
- Posting data to cloud services but avoid waiting for the request to finish

**Rewrite in Node.js?**

Probably a bad idea. But can we augment our existing app using some Node hackons?

--

# The Event Pusher

## **Existing rails app:** We want realtime updates

--

## Broadcast changes to records

```ruby
class Post < ActiveRecord::Base
  has_many :comments
  after_save :notify_change

  def notify_change
    EventPusher.push("post_#{self.id}", self)
  end

end
```

```ruby
class Comment < ActiveRecord::Base
  belongs_to :post
  after_save :notify_change

  def notify_change
    EventPusher.push("post_#{self.post_id}", self)
  end

end
```

--

## The Event Pusher

```ruby
class EventPusher
  
  ENDPOINT = URI.parse('http://localhost:9999/push-event')

  def self.push(obj)
    data = { 
      'object' => obj.attributes, 
      'type' => obj.class.name, 
      'channel' => "post_#{self.post_id}" 
    }

    http = Net::HTTP::Persistent.new('event_pusher')
    http.open_timeout = 1
    http.read_timeout = 10

    req = Net::HTTP::Post.new(ENDPOINT.path, 'content-type' => 'application/json')
    req.body = data.to_json

    http.request(ENDPOINT, req)
  end

end
```

--

## node/server.js

```javascript
var http = require('http')
var createPusher = require('./event-pusher')
var authorize = require('./authorize')

var socketServer = http.createServer()
var pushEvent = createPusher(socketServer, authorize)

var jsonBody = require("body/json")

var privateServer = require('http').createServer(function(req,res){
  if (req.url === '/push-event'){

    jsonBody(req, function(err, body){
      res.writeHead(200)
      res.end()
      pushEvent(JSON.parse(body))
    })

  }
})

privateServer.listen(9999, 'localhost')
socketServer.listen(8080) // public port
```

--

## node/authorize.js

```javascript

module.exports = function(handshakeData, cb){

  // insert custom authentication logic here!

  cb(null, { 
    channel: handshakeData.channel 
  })

}
```

--

## node/event-pusher.js

```javascript
var shoe = require('shoe')
var handshake = require('./handshake')

module.exports = function(server, authorize){
  
  var channels = {}

  shoe(function(socket){

    socket.once('data', function(data){
      handshake(channels, socket, data, authorize)
    })

  }).install(server, '/')

  return function push(event){
    if (channels[event.channel]){
      channels[event.channel].forEach(function(socket){
        socket.write(JSON.stringify(event))
      })
    }
  }

}

```
--

## node/handshake.js

```javascript
module.exports = function(channels, socket, data, authorize){

  var data = JSON.parse(data)

  authorize(data, function(err, handshakeData){

    if (!err && handshakeData){
      
      var channel = handshakeData.channel
      socket.handshakeData = handshakeData
      channels[channel] = channels[channel] || []
      channels[channel].push(socket)

      socket.on('end', function(){
        var index = channels[channel].indexOf(socket)
        channels[channel].splice(index, 1)
      })

    } else { // reject connection
      socket.close()
    }

  })

}
```

--

## client.js

```javascript
var shoe = require('shoe')
var ws = shoe('http://localhost:8080/')

var handshake = {
  channel: 'post_' + window.context.postId
}

ws.write(JSON.stringify(handshake))

ws.on('data', function(data){
  var event = JSON.parse(data)

  // insert page update code here!
  updatePage(event)
})
```

```bash
# bundle this file up with browserify and send to client!
$ browserify client.js -o client-bundled.js
```
--

# ⟚

--

# Modules in the Browser

## **Existing client-side JavaScript:** We want future code to be more modular and use packages from npm.

--

**Node is just JavaScript** with a cool module loader and some useful built-ins.

**Our client side code is (probably) JavaScript.** Maybe we can't have Node on our server right now, but what if we could **share** some of the Node niceties with the browser...

--

**Browserify** (http://browserify.org) lets you require node modules in the browser by bundling up all of your dependencies.

- use many of the tens of thousands of modules on [npm](http://npmjs.org) in the browser
- `process.nextTick()`, `__dirname`, and `__filename` node-isms work
- get browser versions of the node core libraries `events`, `stream`, `path`, `url`, `assert`, `buffer`, `util`, `querystring`, `http`, `vm`, and `crypto` when you `require()` them

--

**You can use browserify along side your existing client-side code.**

```xml
<script src='/javascripts/jquery.js'></script>
<script src='/javascripts/application.js'></script>
<script src='/javascripts/backbone.js'></script>
<script src='/javascripts/browserify-bundle.js'></script>
```

But things are a lot nicer if all of your code is bundled with browserify. 

--

Create a `clientside/global/index.js` that requires all of your old client side code. 

```
- MyApp
| - application
| - public
| - lib
| - clientside
  | - global <- move our old code in here
    | - models.js 
    | - application.js
    | - jquery.js
  | - index.js <- browserify entry
```

--

## clientside/global/index.js

```javascript
require('./jquery.js')
require('./application.js')
require('./models.js')
```

**Specify all of the files you want to include** "old-school-style".

--

## clientside/index.js

```javascript

// require legacy global code
require('./global')

// our new stuff
var behave = require('dom-behavior')

behave({
  'link': function(element){
    element.onclick = function(){
      window.location = element.getAttribute('data-href')
    }
  } 
})

```

**We'll bundle this file using browserify.** All required files will also be recursively included in the bundle.

```bash
$ browserify clientside -o public/bundle.js
```

--

## clientside/global/application.js

```javascript
var deepEqual = require('deep-equal')

window.Application = {
  fancyHelperFunctions: function(){
    // nasty, nasty old code!
  },
  another: function(a, b){
    return deepEqual(a, b)
  }
}
```

**Browserify wraps every file in a closure** so any global variables will have to be on `window` otherwise they won't be accessible anymore. 

But once you've done this, you'll be able to **require modules straight from npm** in your code.
--

## _clientside/application.js_

```javascript
var deepEqual = require('deep-equal')

module.exports = { // <-- FOR EXTRA CREDIT: export instead of using globals
  fancyHelperFunctions: function(thing, whatsit){
    // nasty, nasty old code!
  },
  another: function(a, b){
    return deepEqual(a, b)
  }
}
```

Depending on how your client side code works, it may be easy enough to rewrite parts of it to use `module.exports` and `var mod = require('module')`.

--

## clientside/global/models.js

```javascript
var Application = require('../application.js') // <-- instead of globals

Application.registerModel('Post', {
  // ...
})

Application.registerModel('Comment', {
  // ...
})
```

Once you have the initial closure issues out of the way, the rest of the rewriting can just happen as it's needed. 

--

**Some more tips**

Use [`watchify`](https://www.npmjs.org/package/watchify) instead of the `browserify` command to automatically **rebuild** the bundle when you change any of the code.

Use the `-d` flag when developing to enable **source map** generation. Web Inspector will now show you the original file line numbers and code.

```bash
$ watchify clientside -d -o public/bundle-debug.js
```
--

# ⟚

--

# Sharing Code

## **Existing rails app:** We want to render our templates on both server and client

--

## **We'll need to generate our views in JavaScript.**

## ⟱

##  **Rails:** Handle request (controller/model) - generate JSON Context

## ⇣

##  **JS:** Render the view

## ⇣

## **Rails:** Serve to client

--

## **How should we pass the data to JavaScript?**

We could run a Node server in the background and fire off http requests (containing JSON) which respond with HTML, which we then send to client.

But that may be asking for trouble. **What if the Node process goes down?**

--

## **Another fun option:** Embed V8 in Ruby

```ruby
gem 'therubyracer', '~> 0.12.1'
```

**therubyracer** (https://github.com/cowboyd/therubyracer) provides [libv8](https://code.google.com/p/v8/) bindings for Ruby and lets you eval JS code from inside your Rails app.

We can use [browserify](https://browserify.org) to generate a bundle specifically for our server, containing code needed to render templates.

--

## app/controllers/post_controller.rb

```ruby
class PostController < ApplicationController

  def index
    posts = Post.all
    html = ViewRenderer.render(
      :posts => posts.map(&:public_attributes),
      :channel => 'posts',
      :view => 'home'
    )

    render :text => html, :layout => true
  end
  
  def show
    post = Post.find(params[:id])
    html = ViewRenderer.render(
      :post => post.public_attributes,
      :comments => post.comments.map(&:public_attributes),
      :channel => "post_#{post.id}",
      :view => 'post'
    )

    render :text => html, :layout => true
  end

end
```
--

## lib/view_renderer.rb

```ruby
class ViewRenderer
  
  @js = V8::Context.new
  @js.load("#{Rails.root}/build/serverside.js")


  def self.render(data)
    json = Oj.dump(data, :mode => :compat)
    html = @js.eval("render(#{json})")

    # add a script tag containing the data needed to render page
    "<script type='application/json' id='bindingData'>#{json}</script>" + html
  end

end
```

```ruby
gem 'oj', '~> 2.6.1'
```
--

## serverside/index.js

```javascript
var renderView = require('./views')

global.render = function(json){
  var data = JSON.parse(json)
  return renderView(data)
}
```
--

## serverside/views/index.js

```javascript
var View = require('rincewind')
var get = require('onward')

var views = {
  'home': View(__dirname + '/home.html'),
  'post': View(__dirname + '/post.html')
}

module.exports = function renderView(data, viewName){
  var render = views[viewName]
  if (render){
    return render({
      data: data, 
      get: function(query){
        return get(query, this)
      }
    })
  }
}
```

--

## serverside/views/post.html

```xml
<? require './markdown.js' as markdown ?>
<? require './_comment.html' as comment ?>
<? require './formatters/date.js' as formatDate ?>

<article class='Post'>
  <header>
    <h1 t:bind='post.title' />
    <p>Posted on <span t:bind='post.date:formatDate(full)' /></p>
  </header>
  <div t:bind='post.description' t:view='markdown' />

  <aside>
    <section class='Comment' t:repeat='comments' t:view='comment' />
  </aside>
</article>

```
--

## severside/views/_comment.html

```xml
<? require './markdown.js' as markdown ?>

<header>
  <h1><span t:bind='.name' /> says:</h1>
</header>

<div t:bind='.body' t:view='markdown' />
```

--


## serverside/views/markdown.js

```javascript
var marked = require('marked')
module.exports = function(context){
  // return some html
  return marked(context.source)
}
```

--

## serverside/views/formatters/date.js

```javascript
var strftime = require('strftime')
var relativeDate = require('relative-date')

module.exports = function formatDate(input, format){
  if (typeof input === 'string'){
    input = new Date(input)
  }

  if (format === 'full'){
    return strftime('%d %B %y at %H:%M:%S', input)
  } else if (format === 'relative'){
    return relativeDate(input)
  } else {
    return strftime('%d %B %y', input)
  }
}
```
--

## clientside/index.js

```javascript
var domready = require('domready')
var bind = require('./bind')

domready(function(){
  var dataElement = document.getElementById('bindingData')
  var rootElement = document.getElementById('root')

  var data = JSON.parse(dataElement.innerHTML)
  bind(rootElement, data)
})
```
--

## clientside/bind.js

```javascript
var subscribe = require('./subscribe')
var renderView = require('../serverside/views')
var become = require('become')

module.exports = function bind(element, data){
  if (data.channel){
    subscribe({
      channel: data.channel
    }, function(event){
      if (data.post && event.type === 'Post'){
        data.post = event.object
      } else if (event.type === 'Comment'){
        data.comments.push(event.object)
      }
      var newHtml = renderView(data)
      become(element, newHtml, {inner: true})
    })
  }
}
```
--

## clientside/subscribe.js

```javascript
var shoe = require('shoe')

module.exports = function(options, cb){
  var ws = shoe('http://localhost:8080/')

  ws.write(JSON.stringify(options))

  ws.on('data', function(data){
    var event = JSON.parse(data)
    cb(event)
  })

}
```

--

## app/layouts/application.html.erb

```xml
<html>
<head>
  <title>Blog - <%= @title %></title>
  <link rel='stylesheet' href='/bundle.css' />
  <script src='/bundle.js'></script>
</head>
<body>
  <header class='SiteHeader'>
    <h1>My Blog</h1>
    <div class='Page' id='root'>
      <%= yield %>
    </div>
  </header>
</body>
</html>
```
--

## package.json

```json
{
  "name": "my-blog",
  "scripts": {
    "watch": "npm run watch-serverside & npm run watch-clientside",
    "watch-serverside": "watchify serverside -d -o build/serverside.js",
    "watch-clientside": "watchify clientside -d -o public/javascripts/bundle.js",
  },
  "browserify": {
    "transform": ["rincwind-precompile-transform"]
  },
  "private": true,
  "devDependencies": {
    "browserify": "^3.33.0",
    "watchify": "^0.6.3",
    "rincewind-precompile-transform": "^1.1.0"
  },
  "dependencies": {
    "rincewind": "^1.4.0",
    "become": "^1.2.2",
    "onward": "^0.1.0",
    "domready": "^1.0.4",
    "...": "..."
  }
}
```
```bash
$ npm run watch
```
--

# ⟛

--

# Simple Event Queue

## **Existing rails app:** We want to upload data to S3 but don't want client waiting for request to finish

--

## app/controllers/attachment_controller.rb

```ruby
class AttachmentController < ApplicationController
  def create
    @attachment = Attachment.new(:filename => params[:file].original_filename)
    @attachment.save!
    Queue.upload(params[:file], @attachment.s3_key)
    redirect_to :back
  end
end
```

--

## lib/queue.rb

```ruby
class Queue
  
  ENDPOINT = URI.parse('http://localhost:9999/queue')

  def self.upload(file, s3_key)
    data = { 
      'action' => 'upload',
      'local_path' => file.path, 
      'content_type' => file.content_type,
      's3_key' => s3_key
    }

    http = Net::HTTP::Persistent.new('node_queue')
    http.open_timeout = 1
    http.read_timeout = 10

    req = Net::HTTP::Post.new(ENDPOINT.path, 'content-type' => 'application/json')
    req.body = data.to_json

    http.request(ENDPOINT, req)
  end

end
```
--

## node/server.js

```javascript
var http = require('http')

var queue = require('./queue')
var jsonBody = require("body/json")

var privateServer = require('http').createServer(function(req,res){
  if (req.url === '/queue'){

    jsonBody(req, function(err, body){
      res.writeHead(200) // return immediately
      res.end()
      queue(JSON.parse(body))
    })

  }
})

privateServer.listen(9999, 'localhost')
```
--

## node/queue.js

```javascript
var fs = require('fs')
var knox = require('knox')

var s3 = knox.createClient({
  key: '<api-key-here>', secret: '<secret-here>', bucket: 'my-blog-uploads'
})

module.exports = function queue(action){

  if (item.action === 'upload'){
    fs.stat(item.local_path, function(err, stat){

      var req = s3.put(item.s3_key, {
        'Content-Length': stat.size,
        'Content-Type': item.content_type
      })

      fs.createReadStream(item.local_path).pipe(req)

    })
  }

}
```
--

# ⚒

## Hopefully this helps.

