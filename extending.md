# Extending Shopr

## Order Model

The Order model in Shopr has 3 different callbacks that can be ran. There are
currently `confirmation`, `acceptance` and `rejection`.

You can see all 3 of these callbacks being used in the [shopr-stripe gem](https://github.com/tryshopr/shopr-stripe)
at different parts of an order.

## Building a Staff Order Notification Module

In this example, we will build a gem which will send staff an email saying
a new order has been created. This will use the `confirmation` callback.

```bash
$ bundle gem shopr-notification
```

Open the newly created directory `shopr-notification` in your favourite 
text editor and open the `shopr-notification.gemspec` file. Edit the required
information such as summary, description and homepage (if required) and 
add the Shopr gem as a dependency.

```ruby
::shopr-notification.gemspec
# coding: utf-8
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)
require 'shopr/notification/version'

Gem::Specification.new do |spec|
  spec.name          = "shopr-notification"
  spec.version       = Shopr::Notification::VERSION
  spec.authors       = ["Dean Perry"]
  spec.email         = ["dean@voupe.com"]

  spec.summary       = "A Shopr module which emails staff on new orders"
  spec.description   = "A Shopr module which emails staff on new orders"
  spec.homepage      = "http://tryshopr.com"

  spec.files         = `git ls-files -z`.split("\x0").reject { |f| f.match(%r{^(test|spec|features)/}) }
  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  spec.add_dependency "shopr"

  spec.add_development_dependency "bundler", "~> 1.8"
  spec.add_development_dependency "rake", "~> 10.0"
end
```

Once the gemspec file has been edited, we need to add a `setup` method
which will link to the `confirmation` callback. Add the following to 
the `lib/shopr/notification.rb` file.

```ruby
::lib/shopr/notification.rb
def self.setup
  Shopr::Order.before_confirmation do
    Shopr::NotificationMailer.order_received(self).deliver_now
  end
end
```

Create a new file at `lib/shopr/notification/engine.rb` and enter the following

```ruby
::lib/shopr/notification/engine.rb
module Shopr
  module Notification
    class Engine < Rails::Engine

      initializer "shopr.notification.initializer" do
        Shopr::Notification.setup
      end

    end
  end
end
```

and add this to the top of `notification.rb`, the file we edited earlier

```ruby
require "shopr/notification/engine"
```

Create the Mailer at `app/mailers/shopr/notification_mailer.rb` with the following code

```ruby
::app/mailers/shopr/notification_mailer.rb
module Shopr
  class NotificationMailer < ActionMailer::Base
  
    def order_received(order)
      @order = order
      staff = Shopr::User.all.map(&:email_address)
      mail from: Shopr.settings.outbound_email_address, to: staff, subject: "New Order Received"
    end

  end
end
```

The `Shopr::User.all.map(&:email_address)` code collects all the email addresses for 
users who can login to Shopr. This doesn't include any email addresses for orders.

Once the mailer has been created, you can now create the email text file that will
be rendered and sent to the staff.

```
::app/views/shopr/notification_mailer/order_received.text.erb
Hello Team!

Re: Order #<%= @order.number %>

We're pleased to let you know that we have just received a new order! You might want to accept and ship it.

Many thanks,

<%= Shopr.settings.store_name %>
<%= Shopr.settings.email_address %>
```

The main part of the Gem has been completed. All you need to do now is push the gem to
Rubygems (if you want) and add it to the Gemfile of your project and that's it! When a new
order has been received, staff will receive an email saying so.
