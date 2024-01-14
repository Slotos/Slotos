# Webrick
##### In memory of VCR



## Every villain needs a story

"I used VCR once, it was terrific."

_Or a story of a client auth and a capricious client_



## What we mean to ask

- Does my code reach to the remote server correctly?
- Does my code react to remote errors correctly? <!-- .element: class="fragment" data-fragment-index="1" -->
- Does my code process the data correctly? <!-- .element: class="fragment" data-fragment-index="2" -->
- Does my code deal with data errors correctly? <!-- .element: class="fragment" data-fragment-index="2" -->


## What we are really asking

_Does VCR intercept the network library call?_



## Serve

```ruby
class TestServer
  include Singleton

  def initialize
    @server = ::WEBrick::HTTPServer.new(
      BindAddress: "127.0.0.1",
      Port: 3001,
      StartCallback: -> { @started = true }
    )

    Thread.new { @server.start }
    Timeout.timeout(1) { :wait until @started }
  end
end
```



## Be real

```ruby [6-8]
  def initialize
    @server = ::WEBrick::HTTPServer.new(
      BindAddress: "127.0.0.1",
      Port: 3001,
      StartCallback: -> { @started = true }
      SSLEnable: true,
      SSLCertificate: ssl_options[:web_cert],
      SSLPrivateKey: ssl_options[:web_key]
    )
  end
```


## Be real

- Generate certificate authority and issue certificates under said authority.
- Use `SSL_CERT_DIR` to tell your processes where to find trusted certificates.
- Name certifiate files as `#{hex_cert_hash}.0`


## Be*ware*

`::Net::HTTP` reads SSL info _once_. If you generate certificates in place, you're definitely late.

You can hack your way in, if you want to repeat my mistakes:

```ruby
::Net::HTTP.prepend(Module.new do
  define_method :use_ssl= do |*args|
    @ca_path ||= your_ca_path
    super(*args)
  end
end)
```


## Be pragmatic

Use https://github.com/square/certstrap or openssl cli tools. Start your tests with a script that prepares certificates for you.



## Utility methods

```ruby
  def url(path)
    schema = @server.config[:SSLEnable] ? 'https' : 'http'
    address = @server.config[:BindAddress=]
    port = @server.config[:Port]
    Pathname("#{schema}://#{address}:#{port}")
  end

  def mount(callable = nil, &block)
    raise "Needs block" unless block
    raise "Needs callable" unless callable

    path = SecureRandom.uuid
    path_url = url(path)

    @server.mount_proc("/#{path}", &proc)
    yield(path_url)
    umount("/#{path}")
  end

  def mount_echo(echo, &block)
    raise "Needs block" unless block
    mount(->(request, response) { response.body = echo }, &block)
  end
```


## Utility methods

```ruby
  def umount(path)
    @server.umount(path)
  end

  def shutdown
    @server.shutdown
  end
```



## Simple echo

```ruby
TestWebserver.instance.mount_echo('{"key": "value", "success": true}') do |url|
  assert_eq(Client.new(url:).fetch_value, "value")
end
```



# Complex example

```ruby
test_responder = Class.new do
  attr_accessor :req

  def initialize(response)
    @response = response
  end

  def callable
    ->(req, res) {
      @req = req
      res["Content-Type"] = "application/json;charset=utf-8"
      res.body = @response
    }
  end
end

responder = test_responder.new('{"key": "value", "success": true}')
TestWebserver.instance.mount(responder.callable) do |url|
  assert_eq(Client.new(url:).fetch_value, "value")
  assert_eq(responder.req["Accept"], "application/json")
end
```



## RSpec integration

```ruby
RSpec.configure do |config|
  # This way web server only gets initialized if it's actually called.
  def config.web_server
    @_test_web_server ||= TestWebserver.instance
  end

  def config.shutdown_web_server
    @_test_web_server&.shutdown
  end

  config.after(:suite) { config.shutdown_web_server }
end
```



## VCR feature: Record and Replay

`req` object contains everything one needs to make a request to remote server or match against a local repository.



## VCR feature: Detecting unrecognized network requests

Block network access with a firewall when not recording new requests. Block network access on CI by default.



## Thank you

Time for questions and critique.
