# Add login and logout 

## Routes for sessions

Add routes in `apps/main/web/routes/sessions.rb`. It needs the same process of sign up. The get method renders the form via the view controller `sessions.login`. The post resolve with a transaction `transactions.login` with the form data and a block to process the result. Additionally the post method stores in `session` the user id. The `logout` clears the `session` and redirects. 

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Web
      route 'login' do |r|
        r.get do
          r.view 'sessions.login'
        end

        r.post do
          r.resolve 'transactions.login' do |login|
            login.(r[:user]) do |m|
              m.success do |user|
                flash[:notice] = "Welcome back #{user.email}"
                session[:user_id] = user.id
                r.redirect '/'
              end
              m.failure do |error|
                r.view 'sessions.login', error: error
              end
            end
          end
        end
      end

      route 'logout' do |r|
        flash[:notice] = 'User logged out'
        session.clear
        r.redirect '/'
      end
    end
  end
end
```

## Login action

Add the login view controller `apps/main/lib/todo/main/views/sessions/login.rb` that exposes the `:error` of failed login.

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Sessions
        class Login < View::Controller
          configure do |config|
            config.template = 'sessions/login'
          end

          expose :error
        end
      end
    end
  end
end
```

And the template: `apps/main/web/templates/sessions/login.html.slim`.

```slim
h1 Log In

div
  -if error
    = error
br

form action='/login' method='post'
  == csrf_tag
  label
    | Email
    input name='user[email]' type='text'
  label
    | Password
    input name='user[password]' type='password'
  input type='submit' value='Log in'
```

The transaction that resolves the post request `apps/main/lib/todo/main/transactions/login.rb`, it has only one step, even it's possible to call the operation from the route, I prefer to call transactions from routes, and use operation from the transactions.  

```ruby
# frozen_string_literal: true

require 'todo/main/transaction'

module Todo
  module Main
    module Transactions
      class Login < Transaction
        step :validate, with: 'operations.sessions.validate'
      end
    end
  end
end
```

Add validate session operation: `apps/main/lib/todo/main/operations/sessions/validate.rb`

```ruby
# frozen_string_literal: true

require 'todo/operation'
require 'todo/main/import'
require 'bcrypt'

module Todo
  module Main
    module Operations
      module Sessions
        class Validate < Todo::Operation
          include Import['core.repositories.users_repo']

          def call(attrs)
            email, password = attrs.values_at('email', 'password')
            user = users_repo.users.where(email: email).one

            if user && BCrypt::Password.new(user.password_digest) == password
              Right(user)
            else
              Left('Invalid Credentials')
            end
          end
        end
      end
    end
  end
end
```

Then log in and logout actions should work (in GitHub you can find the tests).

## Show current user in layout

In order to show the current user in the templates, we need a method `current_user` in the view context: `apps/main/lib/todo/main/view/context.rb`

```ruby
module Todo
  module Main
    module View
      class Context < Todo::View::Context
        def current_user
          self[:current_user]
        end
      end
    end
  end
end
```

Within the application (the `Web` class here), the view context is instantiated with the attributes that are set in `view_context_options` and we need to add the `current_user` to this hash. The current user is retrieved from the user's repository with the `user.id` stored in the session.

Adding to `Todo::Main::Web` (`apps/main/system/todo/main/web.rb` ) a `current_user` method and including this value in the `view_context_options` hash, it will work.

```ruby
module Todo
  module Main
    class Web < Dry::Web::Roda::Application
      # ...
      def current_user
        return unless session[:user_id]
        @current_user ||= self.class['core.repositories.users_repo']
          .users.by_pk(session[:user_id]).one
      end

      def view_context_options
        {
          flash:        flash,
          csrf_token:   Rack::Csrf.token(request.env),
          csrf_metatag: Rack::Csrf.metatag(request.env),
          csrf_tag:     Rack::Csrf.tag(request.env),
          current_user: current_user
        }
      end
      # ...
    end
  end
end
```

Update layout `apps/main/web/templates/layouts/application.html.slim` to show current user and links for login and logout:

```slim
html
  body
    - if current_user
      == "Logged as #{current_user.email}"
      | &nbsp;
      a href='/logout'
        | Logout
    - else
      a href='/login'
        | Login
    br
    == flash[:notice]
    == flash[:alert]
    br
    == yield
```