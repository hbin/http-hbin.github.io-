---
layout: post
title: How to set default value for serialize JSON attribute in migration
date: 2016-04-29 14:20:57 +0800
comments: true
categories: [Web]
tags: [Rails]
---
If you have an attribute that needs to be saved to the database as an
object, and retrieved as the same object, Ruby on Rails offer a `serialize`
class method inside an ActiveRecord model which we can specify what
kind of data are stored in a column and Rails would automatically take
care of converting/parsing the actual values.

Here is a great post showing how to make use of the `serialize` in
Ruby on Rails:
http://thelazylog.com/using-serialize-option-in-ruby-on-rails/

To set a default value, Array, and Hash type works as normal, but the
JSON is a little bit different. It took me a while to figure out the
right way to set the default for JSON type. Here is the answer:

``` ruby
class CreateShippingProfiles < ActiveRecord::Migration
  def change
    create_table :shipping_profiles do |t|
      t.string :dimensions, default: ActiveRecord::Coders::JSON.dump(width: 0, height: 0, depth: 0 )

      t.timestamps null: false
    end
  end
end
```

For a deep understanding, let's take a look under the hood.

``` ruby
# File activerecord/lib/active_record/attribute_methods/serialization.rb, line 38
def serialize(attr_name, class_name_or_coder = Object)
  # When ::JSON is used, force it to go through the Active Support JSON encoder
  # to ensure special objects (e.g. Active Record models) are dumped correctly
  # using the #as_json hook.
  coder = if class_name_or_coder == ::JSON
            Coders::JSON
          elsif [:load, :dump].all? { |x| class_name_or_coder.respond_to?(x) }
            class_name_or_coder
          else
            Coders::YAMLColumn.new(class_name_or_coder)
          end

  decorate_attribute_type(attr_name, :serialize) do |type|
    Type::Serialized.new(type, coder)
  end
end
```

## Note:

A serialized attribute will always be updated during save, even if it was not changed.
