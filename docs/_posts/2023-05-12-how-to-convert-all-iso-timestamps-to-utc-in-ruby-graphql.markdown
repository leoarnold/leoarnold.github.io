---
layout: post
title:  "How to convert all ISO timestamps to UTC in Ruby GraphQL"
date:   "2023-05-12T16:43:14+02:00"
tags:
  - Ruby
  - GraphQL 
---

I recently broke a mobile application by changing the timezone[^timezone] of the backend application.
This came as a surprise because we were using [ISO8601][ISO8601] timestamps with UTC offset in the API,
so every client _should_ still have been able to decode the timestamp correctly _even though_ the offset had changed.

It turned out that the mobile application was using a [parser][ISO_INSTANT] which _only_ supports ISO timestamps
with the [Z suffix][ISO8601_Z] (i.e UTC)[^omegastar] and the backend was now serving them in local time.

Since one can never be sure that an update to a mobile application reaches all users, we had to adapt the backend
to continue serving the timestamps in UTC, but now on purpose rather than by accident.

# I couldn't quite follow. Could you please give some examples?

Sure. Let's see look at it in [IRB][IRB].

```ruby
irb> require 'time'
 => false
irb> my_time = Time.now
 => 2023-05-12 16:43:14.080126602 +0200 
```

An ISO timestamp looks this:

```ruby
irb> my_time.iso8601
 => "2023-05-12T16:43:14+02:00" 
```

Note the `+02:00` indicating that my machine is currently running on [CEST][CEST].
And this is also the format which the mobile application was not able to parse. It was expecting:

```ruby
irb> my_time.utc.iso8601
 => "2023-05-12T14:43:14Z" 
```

which denotes the exact same moment in time, just in a slightly different format. 

# So what does the GraphQL specification say?

Nothing. Except for [this example](https://spec.graphql.org/October2021/#example-8b658) maybe[^go_parse_yourself]

```
"timestamp": "Fri Feb 9 14:33:09 UTC 2018"
```

The GraphQL specification makes sure that all [basic scalar types][GraphQL_Scalars] work the same way
in any implementation, e.g. an `Int` must always be understood as a 32-bit signed integer.
But whether you communicate a timestamp as an `Int` or use any kind of `String` representation,
that is not specified and totally up to you[^y2k38].

In my case, the API builds on top of the [Ruby GraphQL][graphql-ruby] library which offers
a custom `ISO8061DateTime` type right out of the box. Let's look at its implementation:

```ruby
# Abridged version of https://github.com/rmosolgo/graphql-ruby/blob/a048682e6d8468d16947ee1c80946dd23c5d91f9/lib/graphql/types/iso_8601_date_time.rb

require 'time'

module GraphQL
  module Types
    class ISO8601DateTime < GraphQL::Schema::Scalar
      # ...

      def self.coerce_result(value, _ctx)
        case value
        when Date
          return value.to_time.iso8601(time_precision)
        when ::String
          return Time.parse(value).iso8601(time_precision)
        else
          # Time, DateTime or compatible is given:
          return value.iso8601(time_precision)
        end
      rescue StandardError => error
        raise GraphQL::Error, "An incompatible object (#{value.class}) was given to #{self}. Make sure that only Times, Dates, DateTimes, and well-formatted Strings are used with this type. (#{error.message})"
      end

      # ...
```

The class method `.coerce_result` takes the value we provided to the GraphQL field and (depending on its type)
tries to convert it to an instance of `Time` in order to apply `Time#iso8601` just like we did
[above](#i-couldnt-quite-follow-could-you-please-give-some-examples). Let's try it out:

```ruby
irb> require 'graphql'
 => true
irb> GraphQL::Types::ISO8601DateTime.coerce_result(my_time, nil)
 => "2023-05-12T16:43:14+02:00" 
```

If we could only tweak that method to somehow enforce UTC, then we would be done ...

# Scope reopening to the rescue

We're trying to solve a seemingly simple problem: Just convert `Time` values to UTC before generating the ISO format.
It should be solved in one place and automatically apply everywhere an ISO timestamp is generated in an API response.
We don't want to fork the `graphql-ruby` gem for this. And luckily we don't need to.

One of the most polarizing features of the Ruby programming language is _scope reopening_[^scope_reopening].
This means that in Ruby you add, change, and remove methods from objects and classes
form any place at any time (even dynamically at runtime[^runtime]).
And that's exactly what we're about to do ...

## The basic idea

The `ISO8601DateTime` type already takes care of generating a valid ISO timestamp for us.
To arrive at the UTC version of it, we just need to apply some "post-processing":

```ruby
irb> iso_timestamp = GraphQL::Types::ISO8601DateTime.coerce_result(my_time, nil)
 => "2023-05-12T16:43:14+02:00" 
irb> Time.parse(iso_timestamp).utc.iso8601
 => "2023-05-12T14:43:14Z"
```

Let's try to redefine the `.coerce_result` to do just that.

## Redefining methods: A common pitfall

Once `graphql-ruby` is loaded, we can start to redefine its classes.
And it might seem plausible to do it like this:

```ruby
module GraphQL
  module Types
    class ISO8601DateTime < GraphQL::Schema::Scalar
      def self.coerce_result(value, _ctx)
        iso_timestamp = super

        Time.parse(iso_timestamp).utc.iso8601
      end
    end
  end
end
```

But this just results in a type error

```ruby
irb> iso_timestamp = GraphQL::Types::ISO8601DateTime.coerce_result(my_time, nil)
/.../lib/ruby/3.2.0/time.rb:380:in `_parse': no implicit conversion of Time into String (TypeError)
```

The gotcha here is that `super` is _not_ pointing to the previous implementation in `ISO8601DateTime`
but rather the implementation of `.coerce_result` in its superclass `GraphQL::Types::Scalar`:

```ruby
irb> GraphQL::Types::ISO8601DateTime.method(:coerce_result).super_method.source_location
=> ["/.../graphql-2.0.21/lib/graphql/schema/scalar.rb", 12]
```

That's where the _switcheroo_ comes into play ...

##  Let's do a switcheroo

Instead of redefining the original `.coerce_result`, we're just gonna "swap it out":

1. Alias the original implementation of `.coerce_result` to another method name called `.old_coerce_result`
2. Add our custom implementation called `.new_coerce_result` which uses `.old_coerce_result` internally
3. Alias `.new_coerce_result` to be the new implementation of `.coerce_result`

Here's the piece of cargo cult you were looking for:

```ruby
# Load this at the start of your application
# e.g. as a Rails initializer in `config/initializers/graphql_types_iso_8601_date_time.rb`

# Make sure original implementation is already loaded
require 'graphql'

module GraphQL
  module Types
    class ISO8601DateTime < GraphQL::Schema::Scalar
      class << self # Redefine class methods
        
        # New implementation, built around original implementation
        def new_coerce_result(value, ctx)
          iso_timestamp = old_coerce_result(value, ctx)

          Time.parse(iso_timestamp).utc.iso8601(time_precision)
        end
        
        # Switcheroo
        alias old_coerce_result coerce_result
        alias coerce_result new_coerce_result
      end
    end
  end
end
```

... and it "just works" as if that was how `graphql-ruby` always worked:

```ruby
irb> GraphQL::Types::ISO8601DateTime.coerce_result(my_time, nil)
 => "2023-05-12T14:43:14Z" 
```

# Footnotes

[^timezone]: You can spell it _timezone_, _time-zone_, or _time zone_ according to [StackExchange](https://english.stackexchange.com/a/3937).
[^omegastar]: I blame the [OmegaStar](https://youtu.be/y8OnoxKotPQ?t=80) team for this.
[^go_parse_yourself]: I'm sure you're itching to implement a parser for this if you're a [Golang](https://youtu.be/vcFBwt1nu2U?t=2980) programmer.
[^y2k38]: If you plan to use your system in the [year 2038 or beyond](https://en.wikipedia.org/wiki/Year_2038_problem), please don't use `Int` for Unix timestamps. 
[^scope_reopening]: I'm not sure whether _scope reopening_ is the correct computer science term for this. I got it from Obie's [Introduction to Rails](https://ruby-doc.org/docs/Introduction%20to%20Ruby%20and%20RoR%20given%20to%20the%20Agile%20Atlanta%20User%20Group/Ruby%20On%20Rails.ppt).
[^runtime]: You just had to come here after trying to debate me on _timezone_, didn't you? Here, help yourself to some more [StackExchange](https://english.stackexchange.com/a/67029).

[CEST]: https://en.wikipedia.org/wiki/Central_European_Summer_Time
[graphql-ruby]: https://github.com/rmosolgo/graphql-ruby
[GraphQL_Scalars]: https://spec.graphql.org/October2021/#sec-Scalars
[IRB]: https://de.wikipedia.org/wiki/Interactive_Ruby_Shell
[ISO8601]: https://en.wikipedia.org/wiki/ISO_8601
[ISO8601_Z]: https://en.wikipedia.org/wiki/ISO_8601#Coordinated_Universal_Time_(UTC)
[ISO_INSTANT]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/format/DateTimeFormatter.html#ISO_INSTANT
