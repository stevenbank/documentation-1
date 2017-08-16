# PayPal

Shopr has a module for taking payments with PayPal. This small tutorial
will guide you through the steps in order to integrate it to your Rails app.

To begin, add the `shopr-paypal` gem to your Gemfile

```
::Gemfile
gem "shopr-paypal"
```

and install the gem

```
bundle install
```

You'll need to create a new REST application on the [PayPal Developer](https://developer.paypal.com/) site
if you haven't already. Once you've created an app, you'll be given a Client ID & Secret ID.

The PayPal module defaults to the Sandbox environment in development and live in production.

These details need to be entered into the settings page in the Shopr admin.
Go to http://localhost:3000/shopr/settings and enter them in the required fields. You will also need
to enter a currency code. This defaults to USD.

In this guide we will create a "Pay by PayPal" button which will redirect the customer to
PayPal to login & approve a payment. Once approved, they will be redirected back to your
Rails app and can complete the order.

Add a link to the `payment.html.erb` file if you've been following the other tutorials.
PayPal recommends you use their [buttons](https://www.paypal.com/us/webapps/mpp/logos-buttons) but
for this tutorial I'm just using a text link.

```rhtml
::app/views/orders/payment.html.erb
<%= link_to "Pay with PayPal", checkout_paypal_path %>
```

In your `routes.rb` file, add the path for the link we just created.

```rb
::config/routes.rb
get "checkout/paypal", to: "orders#paypal"
```

Before we can add the controller method for PayPal, the PayPal setup method
needs to be initialised for the specific controller methods. Add the
following code to the top of your `orders_controller.rb`. The
`Shopr::Paypal.setup_paypal` method needs to be ran / initialised before
anything that accesses the PayPal API because it sets the API keys
that were entered on the settings page earlier.

```ruby
::app/controllers/orders_controller.rb
before_filter(only: [:paypal, :payment]) { Shopr::Paypal.setup_paypal }
```

Now we can add the controller method to the orders controller. The
`redirect_to_paypal` method requires two paramters, a success & cancel URL.
These are for PayPal to redirect to when the payment has been approved or
cancelled. Notice I'm using `_url` instead of `_path`. This shows the full
URL. e.g. http://localhost:3000/checkout/paypal?success=true

```ruby
::app/controllers/orders_controller.rb
def paypal
  @order = Shopr::Order.find(session[:current_order_id])
  url = @order.redirect_to_paypal(checkout_payment_url(success: true), checkout_payment_url(success: false))
  redirect_to url
end
```

Clicking the "Pay with PayPal" button will redirect you to PayPal with the correct
price. Once confirmed, you will be redirected to the success URL we set earlier. In
this tutorial it is `http://localhost:3000/checkout/paypal?success=true`. When redirected
there will be 3 extra parameters added to the URL. These are as follows:

+ `paymentId`
+ `token`
+ `PayerID`

These all need to be stored alongside the order and a payment should be created. Back in
the orders controller, add the following code to the payment method

```ruby
::app/controllers/orders_controller.rb
if params[:success] == "true" && params[:PayerID].present?
  @order.accept_paypal_payment(params[:paymentId], params[:token], params[:PayerID])
end
```    

The `accept_paypal_payment` method will store the 3 properties to the order called:

+ `paymentId` = `paypal_payment_id`
+ `token` = `paypal_payment_token`
+ `PayerID` = `paypal_payer_id`

It will also create a Payment for that order.

Because the user has approved a payment by PayPal, the credit card form isn't required.
Assuming you also have the Stripe module, replace the code in `payment.html.erb`
with the following:

```rhtml
::app/views/orders/payment.html.erb
<%= shopr_stripe_javascript %>

<h2>Make your payment</h2>

<% if params[:success] == "true" && params[:PayerID].present? %>

  <p>Payment will be processed by PayPal</p>
  <%= link_to "Continue", checkout_confirmation_path %>

<% else %>

  <%= form_tag nil, class: "stripeForm" do %>
    <%= hidden_field_tag "stripe_token", nil, "data-stripe" => "token" %>
    <dl>
      <dt><%= label_tag 'card_number' %></dt>
      <dd><%= text_field_tag 'card_number', nil, name: nil, "data-stripe" => "number" %></dd>
      <dt><%= label_tag 'expiry_month' %></dt>
      <dd><%= text_field_tag 'expiry_month', nil, name: nil, "data-stripe" => "exp_month" %></dd>
      <dt><%= label_tag 'expiry_year' %></dt>
      <dd><%= text_field_tag 'expiry_year', nil, name: nil, "data-stripe" => "exp_year" %></dd>
      <dt><%= label_tag 'security_code' %></dt>
      <dd><%= text_field_tag 'security_code', nil, name: nil, "data-stripe" => "cvc" %></dd>
      <dd><%= submit_tag "Continue" %></dd>
    </dl>
  <% end %>

  <hr>

  <%= link_to "Pay with PayPal", checkout_paypal_path %>

<% end %>
```

Ideally the PayPal PayerID should be verified but this is out of the scope
of this tutorial for the moment.

When the order is accepted in the Shopr admin area, the PayPal payment is
"executed" and the funds will be deducted from their account.

Your PayPal integration is now complete!

More information about the `shopr-paypal` module can be found on [GitHub](https://github.com/deanperry/shopr-paypal).
