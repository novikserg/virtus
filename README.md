Virtus
======

[![Gem Version](https://badge.fury.io/rb/virtus.png)][gem]
[![Build Status](https://secure.travis-ci.org/solnic/virtus.png?branch=master)][travis]
[![Dependency Status](https://gemnasium.com/solnic/virtus.png)][gemnasium]
[![Code Climate](https://codeclimate.com/github/solnic/virtus.png)][codeclimate]
[![Coverage Status](https://coveralls.io/repos/solnic/virtus/badge.png?branch=master)][coveralls]

This is a partial extraction of the DataMapper [Property
API](http://rubydoc.info/github/datamapper/dm-core/master/DataMapper/Property)
with various modifications and improvements. The goal is to provide a common API
for defining attributes on a model so all ORMs/ODMs could use it instead of
reinventing the wheel all over again. It is also suitable for any other
usecase where you need to extend your ruby objects with attributes that require
data type coercions.

Installation
------------

``` terminal
$ gem install virtus
```

or in your **Gemfile**

``` ruby
gem 'virtus'
```

Coercions
---------

Virtus uses [Coercible](https://github.com/solnic/coercible) for coercions. This
feature is turned on by default. You can turn it off for all attributes like that:

```ruby
# Turn coercions off globally
Virtus::Attribute.coerce(false)

# ...or you can turn it off for a single attribute
class User
  include Virtus

  attribute :name, String, :coerce => false
end
```

You can configure coercers too:

```ruby
Virtus.coercer do |config|
  config.string.boolean_map = { true => 'yup', false => 'nope' }
end

# Virtus.coercer instance is used by default for all attributes.
# You *can* override it for a single attribute if you want:

my_cool_coercer = Coercible::Coercer.new do |config|
  # some customization
end

class User
  include Virtus

  attribute :name, String, :coercer => my_cool_coercer
end
```

Please check out [Coercible README](https://github.com/solnic/coercible/blob/master/README.md)
for more information.

Examples
--------

### Using Virtus with Classes

You can create classes extended with virtus and define attributes:

``` ruby
class User
  include Virtus

  attribute :name, String
  attribute :age, Integer
  attribute :birthday, DateTime
end

user = User.new(:name => 'Piotr', :age => 29)
user.attributes # => { :name => "Piotr", :age => 29 }

user.name # => "Piotr"

user.age = '29' # => 29
user.age.class # => Fixnum

user.birthday = 'November 18th, 1983' # => #<DateTime: 1983-11-18T00:00:00+00:00 (4891313/2,0/1,2299161)>

# mass-assignment
user.attributes = { :name => 'Jane', :age => 21 }
user.name # => "Jane"
user.age  # => 21
```

### Using Virtus with Modules

You can create modules extended with virtus and define attributes for later
inclusion in your classes:

```ruby
module Name
  include Virtus

  attribute :name, String
end

module Age
  include Virtus

  attribute :age, Integer
end

class User
  include Name, Age
end

user = User.new(:name => 'John', :age => '30')
```

### Dynamically Extending Instances

It's also possible to dynamically extend an object with Virtus:

```ruby
class User
  # nothing here
end

user = User.new
user.extend(Virtus)
user.attribute :name, String
user.name = 'John'
user.name # => 'John'
```

### Default Values

``` ruby
class Page
  include Virtus

  attribute :title, String

  # default from a singleton value (integer in this case)
  attribute :views, Integer, :default => 0

  # default from a singleton value (boolean in this case)
  attribute :published, Boolean, :default => false

  # default from a callable object (proc in this case)
  attribute :slug, String, :default => lambda { |page, attribute| page.title.downcase.gsub(' ', '-') }

  # default from a method name as symbol
  attribute :editor_title, String,  :default => :default_editor_title

  def default_editor_title
    published? ? title : "UNPUBLISHED: #{title}"
  end
end

page = Page.new(:title => 'Virtus README')
page.slug         # => 'virtus-readme'
page.views        # => 0
page.published    # => false
page.editor_title # => "UNPUBLISHED: Virtus README"
```

### Embedded Value

``` ruby
class City
  include Virtus

  attribute :name, String
end

class Address
  include Virtus

  attribute :street,  String
  attribute :zipcode, String
  attribute :city,    City
end

class User
  include Virtus

  attribute :name,    String
  attribute :address, Address
end

user = User.new(:address => {
  :street => 'Street 1/2', :zipcode => '12345', :city => { :name => 'NYC' } })

user.address.street # => "Street 1/2"
user.address.city.name # => "NYC"
```

### Collection Member Coercions

``` ruby
# Support "primitive" classes
class Book
  include Virtus

  attribute :page_numbers, Array[Integer]
end

book = Book.new(:page_numbers => %w[1 2 3])
book.page_numbers # => [1, 2, 3]

# Support EmbeddedValues, too!
class Address
  include Virtus

  attribute :address,     String
  attribute :locality,    String
  attribute :region,      String
  attribute :postal_code, String
end

class PhoneNumber
  include Virtus

  attribute :number, String
end

class User
  include Virtus

  attribute :phone_numbers, Array[PhoneNumber]
  attribute :addresses,     Set[Address]
end

user = User.new(
  :phone_numbers => [
    { :number => '212-555-1212' },
    { :number => '919-444-3265' } ],
  :addresses => [
    { :address => '1234 Any St.', :locality => 'Anytown', :region => "DC", :postal_code => "21234" } ])

user.phone_numbers # => [#<PhoneNumber:0x007fdb2d3bef88 @number="212-555-1212">, #<PhoneNumber:0x007fdb2d3beb00 @number="919-444-3265">]

user.addresses # => #<Set: {#<Address:0x007fdb2d3be448 @address="1234 Any St.", @locality="Anytown", @region="DC", @postal_code="21234">}>
```

### Hash attributes coercion

``` ruby
class Package
  include Virtus

  attribute :dimensions, Hash[Symbol => Float]
end

package = Package.new(:dimensions => { 'width' => "2.2", :height => 2, "length" => 4.5 })
package.dimensions # => { :width => 2.2, :height => 2.0, :length => 4.5 }
```

### IMPORTANT note about member coercions

Virtus performs coercions only when a value is being assigned. If you mutate the value later on using its own
interfaces then coercion won't be triggered.

Here's an example:

``` ruby
class Book
  include Virtus

  attribute :title, String
end

class Library
  include Virtus

  attribute :books, Array[Book]
end

library = Library.new

# This will coerce Hash to a Book instance
library.books = [ { :title => 'Introduction to Virtus' } ]

# This WILL NOT COERCE the value because you mutate the books array with Array#<<
library.books << { :title => 'Another Introduction to Virtus' }
```

A suggested solution to this problem would be to introduce your own class instead of using Array and implement
mutation methods that perform coercions. For example:

``` ruby
class Book
  include Virtus

  attribute :title, String
end

class BookCollection < Array
  def <<(book)
   if book.kind_of?(Hash)
    super(Book.new(book))
   else
     super
   end
  end
end

class Library
  include Virtus

  attribute :books, BookCollection[Book]
end

library = Library.new
library.books << { :title => 'Another Introduction to Virtus' }
```

### Value Objects

``` ruby
class GeoLocation
  include Virtus::ValueObject

  attribute :latitude,  Float
  attribute :longitude, Float
end

class Venue
  include Virtus

  attribute :name,     String
  attribute :location, GeoLocation
end

venue = Venue.new(
  :name     => 'Pub',
  :location => { :latitude => 37.160317, :longitude => -98.437500 })

venue.location.latitude # => 37.160317
venue.location.longitude # => -98.4375

# Supports object's equality

venue_other = Venue.new(
  :name     => 'Other Pub',
  :location => { :latitude => 37.160317, :longitude => -98.437500 })

venue.location === venue_other.location # => true
```

### Custom Coercions

``` ruby
require 'json'

# With a custom writer class
class JsonWriter < Virtus::Attribute::Writer::Coercible
  def coerce(value)
    value.is_a?(Hash) ? value : JSON.parse(value)
  end
end

class User
  include Virtus

  attribute :info, Hash, :writer_class => JsonWriter
end

user = User.new
user.info = '{"email":"john@domain.com"}' # => {"email"=>"john@domain.com"}
user.info.class # => Hash

# With a custom attribute encapsulating coercion-specific configuration
class NoisyString < Virtus::Attribute::String
  class UpperCase < Virtus::Attribute::Writer::Coercible
    def coerce(value)
      super.upcase
    end
  end

  def self.writer_class(*)
    UpperCase
  end
end

class User
  include Virtus

  attribute :scream, NoisyString
end

user = User.new(:scream => 'hello world!')
user.scream # => "HELLO WORLD!"
```

### Private Attributes

``` ruby
class User
  include Virtus

  attribute :unique_id, String, :writer => :private

  def set_unique_id(id)
    self.unique_id = id
  end
end

user = User.new(:unique_id => '1234-1234')
user.unique_id # => nil

user.unique_id = '1234-1234' # => NoMethodError: private method `unique_id='

user.set_unique_id('1234-1234')
user.unique_id # => '1234-1234'
```

Credits
-------

* Dan Kubb ([dkubb](https://github.com/dkubb))
* Chris Corbyn ([d11wtq](https://github.com/d11wtq))
* Emmanuel Gomez ([emmanuel](https://github.com/emmanuel))
* Fabio Rehm ([fgrehm](https://github.com/fgrehm))
* Ryan Closner ([rclosner](https://github.com/rclosner))
* Markus Schirp ([mbj](https://github.com/mbj))
* Yves Senn ([senny](https://github.com/senny))

Contributing
-------------

* Fork the project.
* Make your feature addition or bug fix.
* Add tests for it. This is important so I don't break it in a
  future version unintentionally.
* Commit, do not mess with Rakefile or version
  (if you want to have your own version, that is fine but bump version in a commit by itself I can ignore when I pull)
* Send me a pull request. Bonus points for topic branches.

License
-------

Copyright (c) 2011-2013 Piotr Solnica

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
