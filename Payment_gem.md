#Payment gem

##How to Integrate Stripe Payments in a Rails Application - Charges

This guides walks you through how to integrate a basic implementation of Stripe payments into your application.

In this guide we'll walk through how to integrate a payment feature into a Rails application using the [Stripe](http://stripe.com) payment service. In other guides we'll go through more advanced features, such as **having a payment form**, **dynamically creating plans**, and **creating subscriptions**. For this lesson we're going to build:

- The API connector between the application and `Stripe`

- The ability for users to enter their payment information

- A number of custom features, such as how to accept Bitcoin, configuring default values sent to Stripe, and much more

We'll be starting out by following the instructions from Stripe, however after we have the system working I'll show you how we can refactor the code so it's more flexible.


## Starting application

We'll be starting out with an application that has the `Devise` integration and `Post` scaffold already built out (you can go through the [Devise Guide](http://rails.devcamp.com/trails/ruby-gem-walkthroughs/campsites/authentication/guides/devise)  here), you can pull down the starter code [here](https://github.com/rails-camp/stripe-guide)


## Configuring Stripe

Now let's start the process of integrating **Stripe**. Start by adding the gem to the `Gemfile`

```ruby
# Gemfile

gem 'stripe'
```

After running `bundle` we'll have access to the gem's modules. Let's create an initializer file that will run each time the server starts up and sets our Stripe API connectors:

```
touch config/initializers/stripe.rb
```

In that file you can add the following configuration code:

```ruby
# config/initializers/stripe.rb

Rails.configuration.stripe = {
  :publishable_key => ENV['PUBLISHABLE_KEY'],
  :secret_key      => ENV['SECRET_KEY']
}

Stripe.api_key = Rails.configuration.stripe[:secret_key]
```

We'll need to include these environment variables securely, let's install the `dot-env` gem to handle this for us. Add the gem to the `Gemfile` and run `bundle`:

```ruby
# Gemfile

gem 'dotenv-rails', :groups => [:development, :test]
```

Now let's create the file that will store our environment variables:

```
touch .env
```

Add this file to your `.gitignore` file since you wouldn't want to share your secret keys with the world:

```
# .gitignore

# Ignore environment variables
.env
```

You can verify that this is working by going to the terminal and typing `git status`, it shouldn't list that the `.env` file at all. Now add in the secret keys to the `.env` file with the syntax:

```
PUBLISHABLE_KEY="Your publishable key from stripe here"
SECRET_KEY="Your secret key from stripe here"
```

You can get your keys by signing up for a `Stripe` account. That's it for setting up the `Stripe` configuration, nice and easy, right?


## Creating a charges controller

Following the Stripe prescribed pattern, let's create a `charges` controller that will let us take advantage of RESTful routes and data flow and let us work with the Stripe API, run the following generator:

```
rails g controller charges new create
```

This will generate the controller and give us the controller actions of `new` and `create`. Let's update the routes file like below:

```ruby
# config/routes.rb

Rails.application.routes.draw do
  resources :charges, only: [:new, :create]
  devise_for :users
  resources :posts
  root to: 'posts#index'
end
```

These two `resource` routes for `charges` will map directly to the actions that were generated with the controller. Now let's implement a basic version of the code in the controller:

```ruby
# app/controllers/charges_controller.rb

def new
end

def create
  @amount = 500

  customer = Stripe::Customer.create(
    email: params[:stripeEmail],
    source: params[:stripeToken]
  )

  charge = Stripe::Charge.create(
    customer: customer.id,
    amount: @amount,
    description: 'Rails Stripe customer',
    currency: 'usd'
  )

rescue Stripe::CardError => e
  flash[:error] = e.message
  redirect_to new_charge_path
end
```

This is way too much code for the controller's `create` method, however our first priority is to get this working, we'll refactor this code into its own class at the end of the guide. So what is this code doing? Let's walk through it step by step:

1. It's setting a default amount that will be charged

2. It's creating a stripe customer, the `Stripe::Customer.create` code is a call to the `stripe` gem. Here it's passing the customer's email address and the `stripeToken` in as arguments.

3. Then it calls the `Charge` module, passing in a number of attributes that we want to pass to Stripe's API.

4. Lastly it catches any errors and redirect the user to the `new` path if any errors occur.


## Creating the payment form view

With the controller and routes taken care of to handle the data flow, let's implement the payment form. First create a few new pages:

```
touch app/views/charges/new.html.erb
touch app/views/charges/create.html.erb
```

The `new` view will show the form and the `create` view will be our **Thank you** page.

And add in the following code that connects to the Stripe `JavaScript` code:

```ERB
<!-- app/views/charges/new.html.erb -->

<%= form_tag charges_path do %>
  <article>
    <% if flash[:error].present? %>
      <div id="error_explanation">
        <p><%= flash[:error] %></p>
      </div>
    <% end %>
    <label class="amount">
      <span>Amount: $5</span>
    </label>
  </article>

  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<%= Rails.configuration.stripe[:publishable_key] %>"
          data-description="A month's subscription"
          data-amount="500"
          data-locale="auto"></script>
<% end %>
```

This form communicates with the Stripe `JavaScript` code, which handles the process of rendering the form and it passes in some key values, such as:

1. The `publishable_key`

2. The product description

3. The amount

4. And the `locale`, which dictates the currency that will be rendered in the form.

It's important to realize that none of these items have anything to do with the code in our controller, these are simply arguments passed into the form so it knows what to render to the user. For example, if you change the `data-amount` value it won't override the value that will be charged to the user, that's handled in the controller, during our refactor stage I'll show you how you can make sure these items always match.

In the `create.html.erb` file let's add in some code for our thank you page:

```ERB
<!-- app/views/charges/create.html.erb -->

<h2>Thanks, you paid <strong>$5.00</strong>!</h2>
```

## Creating a Charge

With all that setup let's test it out, startup the rails server and navigate to `localhost:3000/charges/new` and you'll see:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-1.png)

If you click the `Pay with Card` button it will pop up a modal where the user can enter in their payment information (great UI for such a small amount of code written on our side):

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-2.png)

You can use test data to fill out the information, Stripe has a few requirements for their test data:

* **Email** - this should be your email address

* **CC Number** - for a working test card use `4242424242424242`

* **Expiration Date** - any valid date in the future

* **CVV code** - any three digit code

Entering that in and you'll see that it processed the charge and redirected us to the thank you page.

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-3.png)

We can confirm that this worked by opening up the stripe dashboard and seeing if it picked up the charge:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-4.png)

Nice, we're in the money!

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-5.jpg)


## Refactor

Now that we have that working, let's refactor the implementation so it's more flexible and follows best practices. Let's list out what we want accomplished in the refactor:

1. Move the logic out of the controller into a custom class for the charge feature.

2. Make the amount charged and product description dynamic and shared throughout the charge process

3. Create a true thank you page so that users aren't forwarded to a URL called `create`

4. Force users to be signed in before purchasing and pre-populating the email address sent to stripe instead of having to force the users to type it in manually

5. Add in the ability for customers to pay with **Bitcoin** instead of a credit card


With this list in mind, let's implement each of these refactors, starting with the largest.

### Move the logic out of the controller into a custom class for the charge feature

In Ruby development, it's considered a best practice for methods to only have a single goal (take in data, output data, run a process, etc). If you take a look at our `create` method it's performing multiple tasks, some of which are pretty important features. Let's create a new module that will handle all of this logic and then this `charges` controller can simply call the module's methods:

```
touch app/models/concerns/stripe_tool.rb
```

Now let's list out what methods should be in this module, there are two key features that should be split out namely:

- Creating a new Stripe customer

- Creating a new charge

That's a pretty good start, let's simply add some empty methods for these two tasks in the `StripeTool` module:

```ruby
# app/models/concerns/stripe_tool.rb

module StripeTool
  def self.create_customer
  end

  def self.create_charge
  end
end
```

I added `self` to each method because I want to be able to call them from the controller with the syntax `StripeTool.create_charge` instead of having to instantiate the module separately. So what do we do next? Let's pull out the functionality from the controller's `create` action and split it into these two methods. Keep in mind our module won't know what params the view is passing the controller with the `params` method, so we'll need to add arguments to the module methods:

```ruby
# app/models/concerns/stripe_tool.rb

module StripeTool
  def self.create_customer(email: email, stripe_token: stripe_token)
    Stripe::Customer.create(
      email: email,
      source: stripe_token
    )
  end

  def self.create_charge(customer_id: customer_id, amount: amount, description: description)
    Stripe::Charge.create(
      customer: customer_id,
      amount: amount,
      description: description,
      currency: 'usd'
    )
  end
end
```

This organizes our code much better than it was before, these two processes are separate and now can be called separately. Imagine that we need to create customers at another spot in our application, now we can simply call this new `create_customer` method instead of copying and pasting code. Now let's update the controller:

```ruby
# app/controllers/charges_controller.rb

class ChargesController < ApplicationController
  def new
  end

  def create
    @amount = 500

    customer = StripeTool.create_customer(email: params[:stripeEmail],
                                          stripe_token: params[:stripeToken])

    charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: 'Rails Stripe customer')

  rescue Stripe::CardError => e
    flash[:error] = e.message
    redirect_to new_charge_path
  end
end
```

Now if you run the application you'll see it's still working properly, so the refactor didn't break anything and now our code is much more flexible, nice work!


### Make the amount charged and product description dynamic and shared throughout the charge process

All of these hard coded amounts and descriptions are making me reach for my Tums. Right now if you want to change the `amount` you'll need to change 4 different spots of the application across three files, that's not how we're going to choose to live our lives. When it comes to values like the amount to charge, we should only have to set it at one place, any more than that would be waste and would guarantee that at some point we'd forget to change one of the values and then we'd have a big problem.

Let's sit back and think about the best way to do this refactor:

1. Let's create a method where we set the amount value and store it in an instance variable

2. Let's add a `before_action` that runs this method prior to the `new` and `create` actions so they have access to the value

3. Then the views can render the value of the instance variable instead of hard coding the amounts

4. Since the amount needs to be in `cents`, let's also create a view helper method that let's us have properly formatted amount values

Let's update the controller first:

```ruby
# app/controllers/charges_controller.rb

class ChargesController < ApplicationController
  before_action :amount_to_be_charged

  def new
  end

  def create
    customer = StripeTool.create_customer(email: params[:stripeEmail],
                                          stripe_token: params[:stripeToken])

    charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: 'Rails Stripe customer')

  rescue Stripe::CardError => e
    flash[:error] = e.message
    redirect_to new_charge_path
  end

  private

    def amount_to_be_charged
      @amount = 500
    end
end

```

This cleans up the controller nicely, and our `amount` now only has to be set in one place and is available to the other two methods, this is a much better way to handle this flow. Depending on your app's needs you may not need this since you may be setting the values dynamically, such as a customer adding items to a shopping cart and having the total amount summed up, however this is a good way of isolating where the amount is set.

Now let's create the view helper so our views can render the amount properly:

```ruby
# app/helpers/charges_helper.rb

module ChargesHelper
  def pretty_amount(amount_in_cents)
    number_to_currency(amount_in_cents / 100)
  end
end
```

This view helper method takes in the amount as an argument (notice how I also used **Duck Typing** to show that this amount should always in in cents?). Then it divides that amount by `100` to get the dollar value. Lastly it passes that value to the Rails `number_to_currency` method that will convert the value to a financial data format. As an example, this view helper method will take in a value such as `500` and output `$5.00`.

Now we can update the views so they're no longer hard coded values.

```ERB
# app/views/charges/new.html.erb

<%= form_tag charges_path do %>
  <article>
    <% if flash[:error].present? %>
      <div id="error_explanation">
        <p><%= flash[:error] %></p>
      </div>
    <% end %>
    <label class="amount">
      <span>Amount: <%= pretty_amount(@amount) %></span>
    </label>
  </article>

  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<%= Rails.configuration.stripe[:publishable_key] %>"
          data-description="A month's subscription"
          data-amount="<%= @amount %>"
          data-locale="auto"></script>
<% end %>
```

```ERB
# app/views/charges/create.html.erb

<h2>Thanks, you paid <strong><%= pretty_amount(@amount) %></strong>!</h2>
```

Much better looking code, right? If you run this in the browser you'll see that everything is working. If you change the amount in the controller you'll see it populates everywhere. Nice work!

Now you probably already know how to do the same for the description since it follows the a similar pattern as above (except it doesn't require a view helper method), let's start by updating the controller with another private method:

```ruby
# app/controllers/charges_controller.rb

  # at the top of the page
  before_action :set_description

    # in the create method
    charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: @description)

    # as a private method
    def description
      @description = "Some amazing product"
    end
```

Now you can update the view:

```ERB
<!-- app/views/charges/new.html.erb -->

data-description="<%= @description %>"
```

If you test this in the browser you'll see that this is all working, very nicely done!


### Create a true thank you page so that users aren't forwarded to a URL called `create`

I'm not a fan of sending users to a page with the URL of `/create`. I also don't like that we're using a template mapped to the `create` method. If we're following RESTful principles the `create` action shouldn't render a view, it should only `create` a resource. So let's fix this. Draw a new route for our `charges` controller under the `resources` call:

```ruby
# config/routes.rb

get 'thanks', to: 'charges#thanks', as: 'thanks'
```

Now create a new method in the `ChargesController`:

```ruby
# app/controllers/charges_controller.rb

  def thanks
  end
```

This method will automatically have access to the `@amount` and  `@description` instance variables. Now let's update the `create` action so it redirects the user instead of trying to render the `create.html.erb` template:

```ruby
# app/controllers/charges_controller.rb

  def create
    customer = StripeTool.create_customer(email: params[:stripeEmail],
                                          stripe_token: params[:stripeToken])

    charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: @description)

    redirect_to thanks_path
  rescue Stripe::CardError => e
    flash[:error] = e.message
    redirect_to new_charge_path
  end
```

Now let's create the template file by copying the `create` file over and then deleting the `create` file:

```
cp app/views/charges/create.html.erb app/views/charges/thanks.html.erb
rm -rf app/views/charges/create.html.erb
```

Now if you startup the rails server and test it out you'll see that everything is working and now we're redirected to the new `Thanks` page.


### Force users to be signed in before purchasing and pre-populating the email address sent to stripe instead of having to force the users to type it in manually

I'm not a huge fan of forcing users to type their email address each time they want to purchase a product. I also don't want users that aren't logged into the application to be able to purchase anything (this is a pretty standard practice). So let's implement this feature. First let's block users from accessing any of the payment pages unless they are logged in. We can do this by adding another `before_action` to the charges controller, this time calling the `Devise` method `authenticate_user!` which will automatically redirect a user to the login page if they try to access any of the `charges` routes:

```ruby
# app/controllers/charges_controller.rb

before_action :authenticate_user!
```

Now if users try to go to the `charges/new` path they'll be redirected to the login page and you can test this in the browser. That was easy, now that we know that a user will be signed in whenever they access one of the charge pages we can confidently call the `current_user` method and grab their email address to send to Stripe. To implement this feature, update the `JavaScript` code on the view by passing in another argument `data-email="<%= current_user.email %>"`:

```ERB
<!-- app/views/charges/new.html.erb -->

  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<%= Rails.configuration.stripe[:publishable_key] %>"
          data-description="<%= @description %>"
          data-amount="<%= @amount %>"
          data-email="<%= current_user.email %>"
          data-locale="auto"></script>
```

Now if you startup the application, register for an account and go to `charges/new` you'll see that your email address is automatically added to the modal.

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-6.png)


### Add in the ability for customers to pay with Bitcoin instead of a credit card

The last refactor in this guide is going to be a walkthrough on how to integrate the ability to accept Bitcoin payments, don't blink or you might miss it. To make this happen update the script on the view again, this time passing in the attribute `data-bitcoin="true"`

```ERB
<!-- app/views/charges/new.html.erb -->

  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<%= Rails.configuration.stripe[:publishable_key] %>"
          data-description="<%= @description %>"
          data-amount="<%= @amount %>"
          data-email="<%= current_user.email %>"
          data-bitcoin="true"
          data-locale="auto"></script>
```

That's it, you're application now accepts bitcoin! You can see it for yourself here:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe-guide/stripe-guide-7.png)


You should now have a solid idea on how to implement the ability to accept payments in a Ruby on Rails application using the Stripe gem.


## Resources

- [Final code for this guide](https://github.com/rails-camp/stripe-guide/tree/8aadfe6c538b2ba449ea19180cf0e5aa8ab2da47)

##How to Implement Subscriptions in Stripe

Learn how to use the Stripe payment system API to create subscriptions and implement recurring charges for your application.

Imagine that you have an application that needs to charge users for a monthly or annual membership. You wouldn't want to force the user to login and create a new charge every month, instead you would want them to be automatically billed. In this lesson we're going to extend the work performed in our [Stripe Implementation Guide](http://rails.devcamp.com/ruby-gem-walkthroughs/payment/how-to-integrate-stripe-payments-in-a-rails-application-charges) and see what we need to do to implement recurring payments.

You can see the code that we'll be starting out with [here](https://github.com/rails-camp/stripe-guide/tree/8aadfe6c538b2ba449ea19180cf0e5aa8ab2da47).

### Difference Between Charges and Subscriptions in Stripe

When building on an API, I find it helpful to visualize how the data works, especially the relationship between different API calls. Below is a diagram that clarifies the difference between `Charges` and `Subscriptions`.

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe+subscriptions/charges-vs-subscriptions.png)

As you can see:

- A `charge` can be created on the fly for a single payment, such as we did during the [Stripe implementation guide tutorial](http://rails.devcamp.com/ruby-gem-walkthroughs/payment/how-to-integrate-stripe-payments-in-a-rails-application-charges)

- A `subscription` needs to be associated with a `plan`, this is so Stripe knows how much to charge the customer.

This may seem odd if you're new to the Stripe API, so I like to think about it in real world terms. A `subscription` is like your mobile phone subscription, it is simply a connector between the user (you) and the monthly wireless plan that you pay for. By itself the subscription doesn't know anything about the how much you're going to pay or any details related to the product. Whereas the `plan` is the product itself. If you sign up for a 10GB monthly wireless plan with your cellular carrier, the plan stores the information, such as the monthly fee, the currency type, etc. The plan doesn't know anything about the user, it is a high level object that simply holds the information about itself. Here is a diagram showing how the relationships works (at a high level):

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe+subscriptions/stripe-subscription-relationship.png)

This isn't exactly the way the data is modeled in Stripe since Stripe combines the customer creation process with the subscription process, however it's the way I like to think about it since I feel it's important to decouple the customer from the subscription. The reason for this is because it's pretty standard for applications to have the requirement to charge customers a monthly fee, while simultaneously allowing the same customer to purchase one-off type of products. An example of this would be Amazon. Amazon charges a monthly subscription to customers for their Amazon Prime membership, while still letting the users purchase single items anytime they want.

### Creating a Plan

Stripe gives a few different ways to create a plan:

1. Creating a new plan in the dashboard by going to `dashboard > plans > new` and entering in the plan details.

2. Using the API to dynamically creating plans.

Which one should you choose? It's completely based on your application's requirements. If the website has a simple membership structure and won't change regularly it would make the most sense to create the plan in the dashboard. If the application needs to give users the ability to create plans on the fly it will be necessary to build the `plan` generator directly into the application. Let's take two applications as case studies:

- **Hitting.com** - this eCommerce application has a simple membership structure that doesn't change regularly, so there was no need to spend the time to implement the API integration.

- **DevCamp** - the DevCamp platform gives instructors the ability to dynamically create and customize their own packages that they sell to students, therefore it was VERY necessary to integrate the plan generator API for this use case.

In this lesson we'll create the plan using the dashboard since to properly create a comprehensive plan/package management system would distract from building the Stripe subscription feature.

To get started, navigate to Stripe and go to `dashboard > plans > new`, on the `new` page I'm going to enter in the following information:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe+subscriptions/plan-creator.png)

You can see that you have a few different parameters that you can associate with a `plan`:

- `id` - this is a unique identifier

- `name` - this is the short description of the `plan`

- `currency` - dictates what type of currency the `plan` is going to use

- `amount` - how much should be charged each time the subscription runs

- `interval` - here you can select how often the subscription should run (daily, weekly, monthly, etc)

- `trial period` - if you want to let users sign up for a service but not charge their card for a period of time you can place that optional parameter here

- `statement description` - allows you to customize what is shown on the customer's bill

Now that we have a plan created we can have users sign up for subscriptions.


### Implementing Subscriptions

Following our the **Amazon** example we discussed above, we're going to build in a membership feature into the application. We can easily extend the functionality of our `charge` process since creating `subscriptions` is very similar to creating `charges`. Some of the items we'll be putting in place are simply to mock a normal checkout process, but they aren't things that would be implemented in a production site, I'll make a note when this occurs so you can know what you can implement in a real site and what features are simply being put in place for the sake of making this example work.

Let's begin by creating a new method in our `StripeTool` module to managing memberships/subscriptions:

```ruby
# app/models/concerns/stripe_tool.rb

  def self.create_membership(email: email, stripe_token: stripe_token, plan: plan)
    Stripe::Customer.create(
      email: email,
      source: stripe_token,
      plan: plan
    )
  end
```
This is very similar to the `create_customer` method, except this code will generate a charge and subscription, which is why it needs to be separate from the `create_customer` method that is passed to the `create_charge` method.

Now let's add in a `before_action` and private method to see the `plan`, this is mocking the behavior of setting the plan ID, usually this value would be stored in a shopping cart cookie or session value.

```ruby
# app/controllers/charges_controller.rb

before_action :set_plan

private

    def set_plan
      @plan = 9999
    end
```

I'm hard coding in the `id` value for the plan we created in the Stripe dashboard, you can fill in this information with whatever ID you used.

Now let's update the view. This is also going to mock a shopping cart feature, we'll simply place a hidden field tag looking for a subscription parameter. Essentially what this will do is store a value passed to the URL:

```ERB
<!-- app/views/charges/new.html.erb -->

<%= form_tag charges_path do %>
  <article>
    <% if flash[:error].present? %>
      <div id="error_explanation">
        <p><%= flash[:error] %></p>
      </div>
    <% end %>
    <label class="amount">
      <span>Amount: <%= pretty_amount(@amount) %></span>
    </label>
  </article>

  <%= hidden_field_tag :subscription, value: params[:subscription] %>

  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<%= Rails.configuration.stripe[:publishable_key] %>"
          data-description="<%= @description %>"
          data-amount="<%= @amount %>"
          data-email="<%= current_user.email %>"
          data-bitcoin="true"
          data-locale="auto"></script>
<% end %>
```

Ugly yes, but it will let us test between charges and subscriptions without having to duplicate code. Lastly let's update the controller so it's looking for this new `hidden_field` and deciding between creating a `charge` or a `subscription`. Update the `create` action as I have below:

```ruby
# app/controllers/charges_controller.rb

  def create
    if params[:subscription].include? 'yes'
      StripeTool.create_membership(email: params[:stripeEmail],
                                   stripe_token: params[:stripeToken],
                                   plan: @plan)
    else
      customer = StripeTool.create_customer(email: params[:stripeEmail],
                                            stripe_token: params[:stripeToken])
      charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: @description)
    end

    redirect_to thanks_path
  rescue Stripe::CardError => e
    flash[:error] = e.message
    redirect_to new_charge_path
  end
```

I added the conditional `if params[:subscription].include? 'yes'` which is checking for our hidden field value and if it is set to `yes` than it will call the `create_membership` method and pass in the `@plan` value, if not it creates a normal `charge`. You can test this out by starting up the rails application and navigating to `localhost:3000/charges/new` and going through the ordering process like normal. If you check the stripe dashboard you'll see that everything worked like before. Now if you navigate to `localhost:3000/charges/new?subscription=yes` and go through the same steps you'll see that instead of creating a single charge it creates a charge and a subscription. You can confirm this by going to the Stripe dashboard and clicking on the subscription that you created earlier. You can see this below:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe+subscriptions/subscription-creator.png)

I could have made this easy and simply created a different controller or method that managed subscriptions (I was tempted to do so), however I wanted to give you the closest thing to a real world implementation. And in a production app you wouldn't want to have one checkout page for subscriptions and one for products. Hopefully this also illustrates how important it is to pull out methods like we did with our `StripeTool` concern. With only a few lines of code we were able to enable our form to dynamically create subscriptions or charges based on parameters we supplied it with, with very little code duplication.


### Resources

* [Code for this guide](https://github.com/rails-camp/stripe-guide/tree/ed45328bd010b2b03483dd975046fc39db88eaad)

##How to Use a Custom Form for Stripe

In this guide we'll walk through how to implement a custom form for a Stripe payment feature.

In past Gem guides we've walked through [how to integrate Stripe charges](http://rails.devcamp.com/ruby-gem-walkthroughs/payment/how-to-integrate-stripe-payments-in-a-rails-application-charges) and [how to implement subscriptions in Stripe](http://rails.devcamp.com/ruby-gem-walkthroughs/payment/how-to-implement-subscriptions-in-stripe). Now let's walk through the process for using a custom form to collect the credit card information and pass it to Stripe. In the [charge guide](http://rails.devcamp.com/ruby-gem-walkthroughs/payment/how-to-integrate-stripe-payments-in-a-rails-application-charges) you saw how the Stripe JavaScript implementation supplied a popup form that communicated directly with the Stripe API. While the JavaScript integration works well for many applications, if your application needs a more custom payment page you'll need a different process.

If you want to follow along with where the application is at this stage you can get the code from the [project repo](https://github.com/rails-camp/stripe-guide/tree/ed45328bd010b2b03483dd975046fc39db88eaad).

Since this is a completely different process than the JavaScript implementation we'll create a different controller to manage this process. Let's create a controller:

```
rails g controller Payments new thanks
```

Let's update the routes file to use the standard RESTful routes and also make room for our `thanks` page (very similar to what we did in the previous tutorials).

```ruby
# config/routes.rb

Rails.application.routes.draw do
  resources :payments, only: [:new, :create]
  get 'payment-thanks', to: 'payments#thanks', as: 'payment_thanks'
  resources :charges, only: [:new, :create]
  get 'thanks', to: 'charges#thanks', as: 'thanks'
  devise_for :users
  resources :posts
  root to: 'posts#index'
end
```

In a real world application we wouldn't have duplicate code like this, however in a real world application we also wouldn't have two different checkout pages. I'm not replacing the `charges` process because I felt like it would be helpful for you to see the differences side by side in the same application.

The first thing we need to do is call the Stripe API, we'll do this in the master application layout file because it's recommended that the script tags be placed inside of the `<head>` tags for cross browser compatibility.

```ERB
<!-- app/views/layouts/application.html.erb -->
  <script type="text/javascript" src="https://js.stripe.com/v2/"></script>

  <script type="text/javascript">
  	Stripe.setPublishableKey('<%= Rails.configuration.stripe[:publishable_key] %>');
  </script>
```

This is calling the Stripe API and passing in the API keys.

Now let's add in the form into the `new` template, here we're going to use the Rails `form_tag` method:

```ERB
<!-- app/views/payments/new.html.erb -->

<%= form_tag payments_path, method: 'post', id: 'payment-form' do %>
  <span class="payment-errors"></span>

  <div class="form-row">
    <label>
      <span>Card Number</span>
      <input type="text" size="20" data-stripe="number"/>
    </label>
  </div>

  <div class="form-row">
    <label>
      <span>CVC</span>
      <input type="text" size="4" data-stripe="cvc"/>
    </label>
  </div>

  <div class="form-row">
    <label>
      <span>Expiration (MM/YYYY)</span>
      <input type="text" size="2" data-stripe="exp-month"/>
    </label>
    <span> / </span>
    <input type="text" size="4" data-stripe="exp-year"/>
  </div>

  <button type="submit">Submit Payment</button>
<% end %>
```

There are a couple items to notice here:

- We're calling the `payments` `create` action by passing in the path `payments_path` to the `form_tag` method. This is possible because of how we setup our route file.

- The form itself doesn't use any Rails form elements, this is because the form is simply going to communicate with the Stripe API

Now let's add in some JavaScript code to the `payments.coffee` file:

```js
jQuery ($) ->
  $('#payment-form').submit (event) ->
    $form = $(this)
    # Disable the submit button to prevent repeated clicks
    $form.find('button').prop 'disabled', true
    Stripe.card.createToken $form, stripeResponseHandler
    # Prevent the form from submitting with the default action
    false
  return

stripeResponseHandler = (status, response) ->
  $form = $('#payment-form')
  if response.error
    # Show the errors on the form
    $form.find('.payment-errors').text response.error.message
    $form.find('button').prop 'disabled', false
  else
    # response contains id and card, which contains additional card details
    token = response.id
    # Insert the token into the form so it gets submitted to the server
    $form.append $('<input type="hidden" name="stripeToken" />').val(token)
    # and submit
    $form.get(0).submit()
  return
```

This Coffeescript code performs a few actions:

1. It creates a secure token for sending the data to Stripe

2. It handles errors

3. It offers validation for the form

Now let's implement the `payments` controller. If you went through the previous lessons this is very similar to the `ChargesController` with a few changes, you can copy and paste this code into the controller. If you didn't go through the other Stripe guides, I'm using a custom class we created called `StripeTool` that is managing the Stripe API calls.

```ruby
# app/controllers/payments_controller.rb

class PaymentsController < ApplicationController
  before_action :amount_to_be_charged
  before_action :set_description
  before_action :set_plan
  before_action :authenticate_user!

  def new
  end

  def create
    customer = StripeTool.create_customer(email: params[:stripeEmail],
                                            stripe_token: params[:stripeToken])
    charge = StripeTool.create_charge(customer_id: customer.id,
                                      amount: @amount,
                                      description: @description)

    redirect_to payment_thanks_path
  rescue Stripe::CardError => e
    flash[:error] = e.message
    redirect_to new_payment_path
  end

  def thanks
  end

  private

    def set_description
      @description = "Basic Membership"
    end

    def amount_to_be_charged
      @amount = 2999
    end

    def set_plan
      @plan = 9999
    end
end
```

The most important part of this code is the `create` action, this is where the customer creation and charge occurs. If you compare the `ChargesController` with the `PaymentsController` in the project you'll see both actions look for the same parameter. I like that Stripe made both implementations so similar, which makes it easier to switch between the JavaScript implementation and the custom form version.

Let's add some text to the `thanks` page:

```ERB
<!-- app/views/payments/thanks.html.erb -->

<h1>You were successfully charged from the custom form!</h1>
```

Now let's test it out and see if this all works. Startup the Rails server and navigate to `localhost:3000/payments/new`, you should see the custom credit card form here, such as in the image below:

![](https://s3.amazonaws.com/rails-camp-tutorials/gems/stripe+custom+form/stripe-custom-form.png)

If you fill out the form and submit you'll see that everything is working properly, very nice work, you now know how to implement both the built in and custom forms for Stripe in Rails.

One thing to keep in mind, it's strongly recommended that you get a SSL certificate if you're using the custom form implementation.


## Resources

- [Project code for the completed feature](https://github.com/rails-camp/stripe-guide/tree/f755dfcb6bfe131afe3409add1e49243262ebb80)
