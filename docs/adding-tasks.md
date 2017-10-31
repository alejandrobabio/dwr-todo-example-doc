# Adding tasks

In order to add tasks I need:

* Add a form that posts a new task
* Add a route that process the post `/tasks` and call a dry-transaction to create the task
* The transaction should have a step for validating the input data and one step for persisting the data in the database
* The view and the template should show the validation errors
* The created task should be assigned to the logged user

## Add a form to index tasks page

In `apps/main/web/templates/tasks/index.html.slim` add form and show validation errors.

```slim
h1 Tasks

div
  -if validation
    - validation.messages.each do |key, all_messages|
      b = key.capitalize
      = ": #{all_messages.join(', ')}"
br

form action='/tasks' method='post'
  == csrf_tag
  input name='task[description]' placeholder='Add a new task' value=new_task.description

br

ul.tasks
  - tasks.each do |task|
    li.task
      span = task.description
```


`apps/main/lib/todo/main/views/tasks/index.rb` add expose of `validation` and `new_task`

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Tasks
        class Index < Main::View::Controller
          include Main::Import['policies.tasks_scope']

          configure do |config|
            config.template = 'tasks/index'
          end

          private_expose :current_user

          expose :tasks do |current_user|
            tasks_scope.(current_user).to_a
          end

          expose :validation

          expose :new_task do |validation|
            description = validation.to_h[:description]
            Struct.new(:description).new(description)
          end
        end
      end
    end
  end
end
```

## Add a route to process the post call

In `apps/main/web/routes/tasks.rb` add the post route processing that calls the `create_task` transaction with the form data (`r[:task]`) and the `current_user`.  

Also, I added `hash_with_user` to the get processing, in order to send `current_user` to `tasks.index` view controller.

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Application
      route 'tasks' do |r|
        r.is do
          r.get do
            r.view 'tasks.index', **hash_with_user
          end
 
          r.post do
            r.resolve 'transactions.create_task' do |create_task|
              create_task.(params_with_user(r[:task])) do |m|
                m.success do |_task|
                  flash[:notice] = 'Task created!'
                  r.redirect '/tasks'
                end
                m.failure do |validation|
                  r.view 'tasks.index', **hash_with_user(validation: validation)
                end
              end
            end
          end
        end
      end
    end
  end
end
```

In `apps/main/system/todo/main/web.rb` add methods `params_with_user` and `hash_with_user` to be used in routes. And memorize current user with memoizable gem (that you need to add to Gemfile and install).

```ruby
# frozen_string_literal: true

require 'dry/web/roda/application'
require_relative 'container'
require 'memoizable'

module Todo
  module Main
    class Web < Dry::Web::Roda::Application
      include Memoizable
      ...
      def current_user
        return unless session[:user_id]
        self.class['core.repositories.users_repo']
          .users.by_pk(session[:user_id]).one
      end
      memoize :current_user
    
      def params_with_user(params)
        { params: params, current_user: current_user }
      end

      def hash_with_user(hash = {})
        hash.merge(current_user: current_user)
      end
      ...
    end
  end
end
```

## Add transaction to resolve create task action

The process of update data is quite standard: route / transaction / operation. And that's what I'm doing here. In `apps/main/lib/todo/main/transactions/create_task.rb` we have the transaction that triggers its operations:

```ruby
# frozen_string_literal: true

require 'todo/main/transaction'

module Todo
  module Main
    module Transactions
      class CreateTask < Transaction
        step :validate, with: 'operations.tasks.validate'
        step :persist,  with: 'operations.tasks.create'
      end
    end
  end
end
```

In `apps/main/lib/todo/main/operations/tasks/validate.rb` Task schema expect a task description that should have a length of at least 3, and a `user_id` informed and numeric. The `user_id` is formatted from the `current_user` received as parameter.

```ruby
# frozen_string_literal: true

require 'todo/operation'
require 'dry/validation'

module Todo
  module Main
    module Operations
      module Tasks
        class Validate < Todo::Operation
          Schema = Dry::Validation.Form do
            required(:description).filled(:str?, min_size?: 3)
            required(:user_id).filled(:int?)
          end.freeze
          private_constant :Schema

          def call(hash)
            attrs = hash[:params].merge(user_id: hash[:current_user].id)
            validation = Schema.(attrs)

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

And in `apps/main/lib/todo/main/operations/tasks/create.rb` the data is pushed to the DB.

```ruby
# frozen_string_literal: true

require 'todo/operation'

module Todo
  module Main
    module Operations
      module Tasks
        class Create < Todo::Operation
          include Import['core.repositories.tasks_repo']

          def call(attrs)
            task = tasks_repo.create(attrs)
            Right(task)
          end
        end
      end
    end
  end
end
```