# Gisture

[![Build Status](https://travis-ci.org/markrebec/gisture.png)](https://travis-ci.org/markrebec/gisture)
[![Coverage Status](https://coveralls.io/repos/markrebec/gisture/badge.svg)](https://coveralls.io/r/markrebec/gisture)
[![Code Climate](https://codeclimate.com/github/markrebec/gisture.png)](https://codeclimate.com/github/markrebec/gisture)
[![Gem Version](https://badge.fury.io/rb/gisture.png?1=1)](http://badge.fury.io/rb/gisture)
[![Dependency Status](https://gemnasium.com/markrebec/gisture.png)](https://gemnasium.com/markrebec/gisture)

Execute one-off gists inline or in the background.

```ruby
Gisture.run('c3b478ef0592eacad361')
```

The above will run [this gist](https://gist.github.com/markrebec/c3b478ef0592eacad361) and print "You are using Gisture version 0.0.2"

## Getting Started

Add the gem to your project's Gemfile:

    gem 'gisture'

Then run `bundle install`. Or install gisture system wide with `gem install gisture`.

### Configuration

Gisture uses the [github_api](http://peter-murach.github.io/github/) gem to load gists, and passes through a couple configuration options (mostly around authentication). You can configure these options and any other gisture options using `Gisture.configure` with a block wherever appropriate for your project (for example in a rails config initializer). Below is an example along with list of all gisture config options and their defaults.

```ruby
Gisture.configure do |config|
  # config options for the github_api gem
  config.basic_auth     = nil # user:password string
  config.oauth_token    = nil # oauth authorization token
  config.client_id      = nil # oauth client id
  config.client_secret  = nil # oauth client secret
  config.user           = nil # global user used in requets if none provided
  config.org            = nil # global organization used in request if none provided

  config.tmpdir         = Dir.tmpdir  # location to store gist tempfiles when using the require or load strategies
end
```

## Usage

There are a couple ways to load and run a gist. You can use the `Gisture::Gist` class directly, or use shortcut methods directly on the `Gisture` module. The only required argument is the ID of the gist with which you'd like to work.

```ruby
Gisture.run(gist_id)
Gisture.new(gist_id).run!
gist = Gisture::Gist.new(gist_id: gist_id)
gist.run!
```

You can also pass an execution strategy (outlined further below) and optional filename (if your gist contains more than one file).

```ruby
Gisture.run(gist_id, :require, 'my_file.rb')
gist = Gisture::Gist.new(gist_id: gist_id, strategy: :require, filename: 'my_file.rb')
gist.run!
```

### Callbacks

You can provide an optional callback block to execute after your gist has been loaded and run. This can be handy if, for example, your gist defines a few classes and/or methods, but does not do anything with them directly or perform any inline functionality. In that case, you might want to make calls to those methods or classes after your gist has been loaded to perform some action.

```ruby
# gist 123
def do_thing_from_gist
  puts "Doing the thing from the gist"
end
```

```ruby
Gisture.run('123') do
  do_thing_from_gist # this would call the method that was defined in your gist to actually perform the action
end
```

Without the provided block, all that would happen is your method would be defined and available for use. Since the method is never actually called anywhere in the gist, we need to trigger it manually in the block if we want it to be executed.

### Execution Strategies

There are three execution strategies available for running your gists. These are `:eval`, `:load` and `:require`. The strategy you use will depend heavily on what exactly you're trying to do and how the code in your gist is structured. For example, if you're defining modules/classes for use elsewhere in your project, then you probably want to `load` or `require` the gist file, then utilize those classes. If you are performing a lot of inline functional operations that you want to be executed inline or in a background task you'd probably lean towards `eval` or `load`.

**:eval**

As the name implies, the eval strategy will evaluate the contents of your gist file (as a string) using ruby's `eval` method. This happens inside a sanitary object called `Gisture::Evaluator`, which is returned and can be manipulated further or used to call any defined methods, etc.

The eval strategy can be a handy way to take advantage of metaprogramming to create one-off objects with custom functionality. For example, if you were to define the following method in a gist, and then use the eval strategy, the `Gisture::Evaluator` object that was returned would have the method defined as an instance method.

```ruby
# gist 123
def do_thing_from_gist
  puts "Doing the thing from the gist"
end
```

```ruby
e = Gisture.run('123', :eval) # returns a Gisture::Evaluator
e.do_thing_from_gist # prints "Doing the thing from the gist"
```

**:require**

The require strategy puts the contents of the gist into a tempfile and uses ruby's built-in `require` to include it. This means that all the caveats around how the code is included at runtime apply. If we were to take the same gist from the eval example above, we'd end up with the method defined at the toplevel binding.

```ruby
# gist 123
def do_thing_from_gist
  puts "Doing the thing from the gist"
end
```

```ruby
Gisture.run('123', :require) # returns true (the result of the call to `require`)
do_thing_from_gist # prints "Doing the thing from the gist"
```

**:load**

The load strategy puts the contents of the gist into a tempfile and uses ruby's built-in `load` to include it. This means that like require, all the caveats around how the code is included at runtime apply. If we were to take the same gist from the eval example above, it would behave the same way as our require example.

### Multiple Files

If your gist contains more than one file, you'll need to tell gisture which file you'd like to run. You can do this by passing the appropriate argument when creating a gist.

```ruby
Gisture.new('123', :load, 'myfile.rb')
Gisture::Gist.new(gist_id: '123', filename: 'myfile.rb')
```

### Rake Task

Gisture also provides a built-in rake task, named `gisture:run`, to easily run one-off tasks in the background. If you're using rails, the gisture railtie will automatically include the task for you, otherwise you can include the tasks found in `lib/tasks/gisture.rake` however is appropriate for your project.

You can call the rake task with your gist ID as well as an optional strategy, filename and callback string (which will be eval'd as a callback block).

```
rake gisture:run[abc123]
rake gisture:run[abc123,load]
rake gisture:run[abc123,eval,my_file.rb]
rake gisture:run[abc123,load,my_method.rb,'my_method(whatever)']
```

## TODO

* Specs
* Support for specifying a version of a gist
* Better support for gists with multiple files
* Allow sources other than gists, like repos containing one-off scripts, generic github blobs, local files, etc.

## Contributing
1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
