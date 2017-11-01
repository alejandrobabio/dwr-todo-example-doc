# Add signup action

We will do a `GET` for the page with the sign up form, and a `POST` with the new user information to be added. It needs to pass through several operations that ends with adding a new user to the system, and returning the response to the user. If you are new to dry-rb this may not be an easy topic. Anyway, it has the ordered steps to build the action and some references to where you should search for detailed information.

## Add test for the signup action

Let's add the file `spec/features/main/signup_spec.rb` with a simple test for sign up.

```ruby
# frozen_string_literal: true

require 'web_spec_helper'

RSpec.feature 'Sign Up' do
  scenario 'sign up as new user' do
    visit '/signup'
    fill_in 'user[email]', with: 'user@example.com'
    fill_in 'user[password]', with: 'secret'
    click_on 'Sign Up'

    expect(page).to have_content('Welcome!')
  end
end
```

If you run the test the message will be `Capybara::ElementNotFound ... user[email]`, as the page does not exist you can check the `page.status_code` is 404, which is correct.

We will use the `multi_route` plugin. So, you must activate it in the line 24 of the program `apps/main/system/todo/main/web.rb` that has the `r.multi_route` call.

Then you can add routes to the `config.routes` path. Next, add the routes to `/signup` in `apps/main/web/routes/signup.rb`

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Web
      route 'signup' do |r|
        r.get do
          r.view 'users.signup'
        end
      end
    end
  end
end
```

Now our test show the error ` Dry::Container::Error: Nothing registered with the key "views.users.signup"` that is because `r.view 'users.signup'` expect a `User::Signup` class registered under `Views` namespace, this class should be a `View::Controller`. 

Visit the official site to learn more about the [dry-view](http://dry-rb.org/gems/dry-view) gem.

Add a view for sign up: `apps/main/lib/todo/main/views/users/signup.rb`

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Users
        class Signup < View::Controller
          configure do |config|
            config.template = 'users/signup'
          end
        end
      end
    end
  end
end
```

And in `apps/main/web/templates/users/signup.html.slim` the template:

```slim
h1 Sign Up

form action='/signup' method='post'
  == csrf_tag
  label
    | Email
    input name='user[email]' type='text'
  label
    | Password
    input name='user[password]' type='password'
  input type='submit' value='Sign Up'
```

If we run the test we'll get an expected 404 response. Because we did not write the response to `POST /signup`.

## Routing the signup post

First, we need to add the route, in the file `apps/main/web/routes/signup.rb`:

```ruby
module Todo
  module Main
    class Application
      route 'signup' do |r|
      ...
        r.post do
          r.resolve 'transactions.signup' do |signup|
            signup.(r[:user]) do |m|
              m.success do
                flash[:notice] = 'Welcome!'
                r.redirect '/'
              end
              m.failure do |validation|
                r.view 'users.signup', validation: validation
              end
            end
          end
        end
      end
    end
  end
end
```

The `r.resolve` call is part of the Roda plugin [flow](https://github.com/AMHOL/roda-flow), that delegates to the application [`#resolve`](https://github.com/dry-rb/dry-web-roda/blob/v0.9.0/lib/dry/web/roda/application.rb#L22) method, which calls the container `#[]` (alias of the [dry-container](http://dry-rb.org/gems/dry-container/)'s `#resolve` method).

Then it searches in the container the class `Transactions::Signup`, instantiates a new object and calls the `call` method with `r[:user]` as a parameter, and pass a block for the process the result.

## Signup transaction and its operations

For our `Signup` class we will use the [dry-transaction](http://dry-rb.org/gems/dry-transaction) gem, and each `step` will be an `Operation` in our `lib` folder.

A base class for transactions `apps/main/lib/todo/main/transaction.rb` only for avoid include `Dry::Transaction` in each one.

```ruby
# frozen_string_literal: true

# auto_register: false

require 'dry/transaction'

module Todo
  module Main
    class Transaction
      include Dry::Transaction(container: Container)
    end
  end
end
```

In transactions folder, I'll put the actions that require something more than retrieve data and show to the user. Usually, post, put, patch and delete methods. These will define my business rules and usually, will be the ones that update my DB.
 
In a transaction for signup `apps/main/lib/todo/main/transactions/signup.rb` I need to validate the input data, encrypt the password before store it, and create the user record in my DB. Then this could be my `Signup` transaction:

```ruby
# frozen_string_literal: true

require 'todo/main/transaction'

module Todo
  module Main
    module Transactions
      class Signup < Transaction
        step :validate,         with: 'operations.users.validate'
        step :encrypt_password, with: 'operations.users.encrypt_password'
        step :persist,          with: 'operations.users.create'
      end
    end
  end
end
```

If you did not read the [dry-transaction](http://dry-rb.org/gems/dry-transaction) documentation, please do it. 

In a dry-transaction the steps are executed in sequence, the first receive the input of the transaction, and if all goes well, the output of each step is the input of the next one, and the last one output becomes the output of the transaction. But, if one of the steps decides that it cannot continue its output becomes the output of the transaction and the pending steps are never executed. The implementation requires that all operations return a `Right` object if all is correct, and a `Left` object if it wants to finish the transaction. And each operation should respond to `call` (that's the method called in each step).

Each operation should inherit from `Todo::Operation` that give access to `Right` and `Left` methods and prepends a `#call` method that accepts parameters and a block, with the parameters it runs the call method that you write in the operation class, and then it processes the block. 

The validation operation that I located in `apps/main/lib/todo/main/operations/users/validate.rb` uses [dry-validation Form](http://dry-rb.org/gems/dry-validation/forms) to define the `Schema` that will test the data. Its `call` method returns a `Right` object with the parameters for the next step if validation pass, otherwise it returns a `Left` with the result of the validation.

```ruby
# frozen_string_literal: true

require 'todo/operation'
require 'dry/validation'

module Todo
  module Main
    module Operations
      module Users
        class Validate < Todo::Operation
          EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i

          Schema = Dry::Validation.Form do
            required(:email).filled(format?: EMAIL_REGEX)
            required(:password).filled(:str?, min_size?: 6)
          end.freeze
          private_constant :Schema

          def call(*args)
            validation = Schema.(*args)

            if validation.success?
              Right(validation.output)
            else
              Left(validation)
            end
          end
        end
      end
    end
  end
end
```

The next operation is `EncryptPassword` and it's located in `apps/main/lib/todo/main/operations/users/encrypt_password.rb`
In order to encrypt the user password you need to add to your `Gemfile` the gem `bcrypt`. 
This operation removes the `password` from args and includes the `password_digest` with the encrypted value before returning a `Right` object.

```ruby
# frozen_string_literal: true

require 'todo/operation'
require 'dry/validation'
require 'bcrypt'

module Todo
  module Main
    module Operations
      module Users
        class EncryptPassword < Todo::Operation
          def call(args)
            password = args.delete(:password)
            output = args.merge(
              password_digest: BCrypt::Password.create(password)
            )

            Right(output)
          end
        end
      end
    end
  end
end
```

The **persist** step calls the create operation with the key `operations.users.create`  so the file `apps/main/lib/todo/main/operations/users/create.rb` should be in place. 

This `Create` class include `Import['core.repositories.users_repo']` that is a [dry-auto_inject](http://dry-rb.org/gems/dry-auto_inject), it injects the `users_repo` instance that give you access to the DB. It returns a `Right` object because data is valid, the password was encrypted and at this point, there is no reason to cancel the transaction.

```ruby
# frozen_string_literal: true

require 'todo/operation'

module Todo
  module Main
    module Operations
      module Users
        class Create < Todo::Operation
          include Import['core.repositories.users_repo']

          def call(attrs)
            user = users_repo.create(attrs)
            Right(user)
          end
        end
      end
    end
  end
end
```

Now is needed to add the flash to application layout  `apps/main/web/templates/layouts/application.html.slim`

```slim
html
  body
    == flash[:notice]
    == flash[:alert]
    == yield
```

And the test should pass. This was the happy path. Some minor changes should be added to process an invalid input.

## Adding tests for invalid data

First include in `spec/features/main/signup_spec.rb` these new tests:

```ruby
RSpec.feature 'Sign Up' do
  ...
  scenario 'show error with invalid email' do
    visit '/signup'
    fill_in 'user[email]', with: 'user'
    fill_in 'user[password]', with: 'secret'
    click_on 'Sign Up'

    expect(find('[name="user[email]"]').value).to eq 'user'
    expect(page).to have_content('Email: is in invalid format')
  end

  scenario 'show error with short password' do
    visit '/signup'
    fill_in 'user[email]', with: 'user@example.com'
    fill_in 'user[password]', with: 'nope'
    click_on 'Sign Up'

    expect(page).to have_content('Password: size cannot be less than 6')
  end
end
```

When data is invalid our route will send the validation object to the `Views::Users::Signup` instance in this line `r.view 'users.signup', validation: validation`. This data can be sent to the template with `expose :validation`. It is expected that the view shows the invalid data in error, then I added a `new_user` method that receives the input and builds a `Struct` to share the values with the template. So this is the final status of the Signup view controller and its template.

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Users
        class Signup < View::Controller
          configure do |config|
            config.template = 'users/signup'
          end

          expose :new_user, :validation

          private

          def new_user(validation: {})
            Struct.new(:email, :password).new(validation[:email], nil)
          end
        end
      end
    end
  end
end
```

Now, show `validation` errorrs and use the `new_user` data in the template: `apps/main/web/templates/users/signup.html.slim`
```slim
h1 Sign Up

div
  - if validation
    - validation.messages.each do |key, all_messages|
      b = key.capitalize
      = ": #{all_messages.join(', ')}"
      | &nbsp;
br

form action='/signup' method='post'
  == csrf_tag
  label
    | Email
    input name='user[email]' type='text' value=new_user.email
  label
    | Password
    input name='user[password]' type='password'
  input type='submit' value='Sign Up'
```

All tests should pass.