# Piperator

Pipelines for streaming large collections. The pipeline enables composition of streaming pipelines with lazy enumerables.

The library is heavily inspired by [Elixir pipe operator](https://elixirschool.com/lessons/basics/pipe-operator/) and [Node.js Stream](https://nodejs.org/api/stream.html).

[![Build Status](https://travis-ci.org/lautis/piperator.svg?branch=master)](https://travis-ci.org/lautis/piperator)


## Installation

Piperator is distributed as a ruby gem and can be installed with

```
$ gem install piperator
```

## Usage

Start by requiring the gem

```ruby
require 'piperator'
```

As an appetiser, here's a pipeline that triples all input values and then sums the values.

```ruby
Piperator.
  pipe(->(values) { values.lazy.map { |i| i * 3 } }).
  pipe(->(values) { values.sum }).
  call([1, 2, 3])
# => 18
```

If desired, the input enumerable can also be given as the first pipe.

```ruby
Piperator.
  pipe([1, 2, 3]).
  pipe(->(values) { values.lazy.map { |i| i * 3 } }).
  pipe(->(values) { values.sum }).
  call
# => 18
```

There is, of course, a much more idiomatic alternative in Ruby:

```ruby
[1, 2, 3].map { |i| i * 3 }.sum
```

So why bother?

To run code before the stream processing start and after processing has ended. Let's use the same pattern to calculate the decompressed length of a GZip file fetched over HTTP with streaming.

```ruby
require 'piperator'
require 'uri'
require 'em-http-request'
require 'net/http'

module HTTPFetch
  def self.call(url)
    uri = URI(url)
    Enumerator.new do |yielder|
      Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == 'https') do |http|
        request = Net::HTTP::Get.new(uri.request_uri)
        http.request request do |response|
          response.read_body { |chunk| yielder << chunk }
        end
      end
    end
  end
end

module GZipDecoder
  def self.call(enumerable)
    Enumerator.new do |yielder|
      decoder = EventMachine::HttpDecoders::GZip.new do |chunk|
        yielder << chunk
      end

      enumerable.each { |chunk| decoder << chunk }
      yielder << decoder.finalize.to_s
    end
  end
end

length = proc do |enumerable|
  enumerable.lazy.reduce(0) { |aggregate, chunk| aggregate + chunk.length }
end

Piperator.
  pipe(HTTPFetch).
  pipe(GZipDecoder).
  pipe(length).
  call('http://ftp.gnu.org/gnu/gzip/gzip-1.2.4.tar.gz')
```

At no point is it necessary to keep the full response or decompressed content in memory. This is a huge win when the file sizes grow beyond the 780kB seen in the example.

Pipelines themselves respond to `#call`. This enables using pipelines as pipes in other pipelines.

```ruby
append_end = proc do |enumerator|
  Enumerator.new do |yielder|
    enumerator.each { |item| yielder << item }
    yielder << 'end'
  end
end

prepend_start = proc do |enumerator|
  Enumerator.new do |yielder|
    yielder << 'start'
    enumerator.each { |item| yielder << item }
  end
end

double = ->(enumerator) { enumerator.lazy.map { |i| i * 2 } }

prepend_append = Piperator.pipe(prepend_start).pipe(append_end)
Piperator.pipe(double).pipe(prepend_append).call([1, 2, 3]).to_a
# => ['start', 2, 4, 6, 'end']
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/lautis/piperator.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
