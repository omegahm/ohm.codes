---
layout: post
title: "Service Object Generators"
subtitle: "Let's create a generator for service objects"
date: 2024-01-27 21:23:49 +0100
summary: |
  What are service object and how do we create and configure custom generators for them in Ruby on Rails.
---

Let's start easy.
With Rails we have access to generators.
You've probably used already them to generate models, controllers, etc.
Usually you'd write something like this:

```bash
$ bin/rails generate model User name:string email:string
```

We can create our own generators as well, if we have some patterns that we use often.
One such pattern is the Service Object pattern.

What we want to achieve is to have a generator that will create a service object for us.
What is a service object?
It's a good way to extract some logic from, for example, the controller or the model.

Let's take a look at an example.
First let's look at a `Cart` model:

```ruby
class Cart < ApplicationRecord
  def shipping_cost(destination, shipping_method)
    # calculate shipping cost based on destination and shipping method
  end
end
```

It's not really the `Cart` model's responsibility to calculate the shipping cost.
We should extract that logic to a separate class.
But where?
We could create a `ShippingCostCalculator` class, which would be our newly created service object.
It could look something like this:

```ruby
class ShippingCostCalculator
  def initialize(cart, destination, shipping_method)
    @cart = cart
    @destination = destination
    @shipping_method = shipping_method
  end

  def calculate
    # calculate shipping cost based on destination and shipping method
  end
end
```

We can then use it in our controller, instead of calling the `#shipping_cost` method on `Cart`:

```ruby
ShippingCostCalculator.new(cart, destination, shipping_method).calculate
```

When a service object only have one method, I tend to make a class method instead:

```ruby
class ShippingCostCalculator
  class << self
    def calculate(cart, destination, shipping_method)
      new(cart, destination, shipping_method).calculate
    end
  end

  ...
end
```

That way, we don't need to instantiate the class, we can just call the class method:

```ruby
ShippingCostCalculator.calculate(cart, destination, shipping_method)
```

<acronym title="In My Humble Opinion">IMHO</acronym> this looks cleaner.

With this service object we've extracted the logic away from the `Cart` model and stored it somewhere central, that all other classes can benefit from.

---

Okay, we've seen a service object in action.
Let's see if we can generate them with a generator.
Rails has a generator for generating generators. (Try saying that 10 times fast.)

```bash
$ bin/rails generate generator service_object
      create  lib/generators/service_object
      create  lib/generators/service_object/service_object_generator.rb
      create  lib/generators/service_object/USAGE
      create  lib/generators/service_object/templates
      invoke  rspec
      create    spec/generator/service_objects_generator_spec.rb
```

This generates the actual generator, a template for the generator and a spec file.
The `USAGE` file describes how to use the generator.
Inside the `service_object_generator.rb`-file we can put our logic for generating the service object.

```ruby
# frozen_string_literal: true

class ServiceObjectGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  argument :method_name, type: :string, default: "call", desc: "Name of the method to generate"
  argument :arguments, type: :array, default: [], desc: "Arguments to the method"

  def create_service_object
    template "service_object.rb.erb", File.join("app/services", class_path, "#{file_name}.rb")
  end
end
```

Here we have two arguments, `method_name` and `arguments`.
They can be used when generating the service object like this:

```bash
$ bin/rails generate service_object shipping_cost calculate cart destination shipping_method
```

In the `templates` directory we now have a template file, `service_object.rb.erb`.
Here we can specify how our template for a service object should look like.
This is just a plain old ERB-file, so we can use Ruby in it:

```ruby
# frozen_string_literal: true

class <%= class_name %>
  class << self
    def <%= method_name %><%= arguments.any? ? "(#{arguments.join(", ")})" : "" %>
      new<%= arguments.any? ? "(#{arguments.join(", ")})" : "" %>.<%= method_name %>
    end
  end
  <%- if arguments.any? -%>

  def initialize(<%= arguments.join(", ") %>)
    <%- arguments.each do |arg| -%>
    @<%= arg %> = <%= arg %>
    <%- end -%>
  end
  <%- end -%>

  def <%= method_name %>
    # ... your logic goes here
  end
end
```

We see that our arguments from before can be used as Ruby variables in the ERB-file.

Finally, if we now invoke the bash commend from before, we'll get a new service object:

```bash
$ bin/rails generate service_object shipping_cost calculate cart destination shipping_method
      create  app/services/shipping_cost.rb
```

```ruby
# frozen_string_literal: true

class ShippingCost
  class << self
    def calculate(cart, destination, shipping_method)
      new(cart, destination, shipping_method).cart
    end
  end

  def initialize(cart, destination, shipping_method)
    @cart = cart
    @destination = destination
    @shipping_method = shipping_method
  end

  def calculate
    # ... your logic goes here
  end
end
```

There's a lot more here that can be done and I will probably cover some in future posts:

- You can customize the template to your liking.
- We haven't talked about the spec file, but that can be customized as well, and you can add your own tests to it.
- The `USAGE` file will be shown when you run the generator with the `--help` flag.
