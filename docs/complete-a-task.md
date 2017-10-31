# Complete a task

The process is always the same: add a form that triggers the action, add a route that captures the request and process it, a transaction with its steps or operations, and render the response through a view controller. 

## Add the form

In order to allow mark tasks as completed. In tasks index page we will add a button to set it up. Here I added a form for each uncompleted task or show the `Completed` status.

`apps/main/web/templates/tasks/index.html.slim`

```slim
...
ul.tasks
  - tasks.each do |task|
    li.task
      span = task.description
      | &nbsp;
      - if task.completed
          | [Completed]
      - else
        form action="/tasks/#{task.id}" method='post'
          == csrf_tag
          input type='hidden' name='_method' value='patch'
          input type='submit' value='Complete'
```

## Add a route to process the action

The `patch` method is not accepted by default by Roda, but it can be activated by adding the plugin `all_verbs` and using the `Rack::MethodOverride` middleware. So in `Main::Application` (`apps/main/system/todo/main/web.rb`) add these lines:

```ruby
...
module Todo
  module Main
    class Web < Dry::Web::Roda::Application
      ...
      use Rack::MethodOverride

      plugin :all_verbs
      ...
    end
  end
end
```

An update to the `tasks` routes to process `patch '/tasks/:id` calling to the `complete_task` transaction and rendering the response.

`apps/main/web/routes/tasks.rb`

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Application
      route 'tasks' do |r|
        r.is do
          ...
        end

        r.is :id do |id|
          r.patch do
            r.resolve 'transactions.complete_task' do |complete_task|
              complete_task.(params_with_user(id: id)) do |m|
                m.success do |_task|
                  flash[:notice] = "task id: #{id}, completed"
                  r.redirect '/tasks'
                end
                m.failure do
                  flash[:alert] = 'Unexpected error'
                  r.view 'tasks.index', **hash_with_user
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

## Add the transaction and its operations

The actions to do are: first find the task in the `current_user` scope and return its `id`. And mark the task as completed with the provided `id`.

`apps/main/lib/todo/main/transactions/complete_task.rb`

```ruby
# frozen_string_literal: true

require 'todo/main/transaction'

module Todo
  module Main
    module Transactions
      class CompleteTask < Transaction
        step :find_task, with: 'operations.tasks.find_task'
        step :complete,  with: 'operations.tasks.complete'
      end
    end
  end
end
```

In `apps/main/lib/todo/main/operations/tasks/find_task.rb` search the task `id` in the `current_user` scope of tasks. 

```ruby
# frozen_string_literal: true

require 'todo/operation'

module Todo
  module Main
    module Operations
      module Tasks
        class FindTask < Todo::Operation
          include Import['policies.tasks_scope']

          def call(hash)
            id = tasks_scope.(hash[:current_user])
              .where(id: hash[:params][:id]).pluck(:id).first

            if id
              Right(id)
            else
              Left(:error)
            end
          end
        end
      end
    end
  end
end
```

And in `apps/main/lib/todo/main/operations/tasks/complete.rb` update its `complete` status.

```ruby
# frozen_string_literal: true

require 'todo/operation'

module Todo
  module Main
    module Operations
      module Tasks
        class Complete < Todo::Operation
          include Import['core.repositories.tasks_repo']

          def call(id)
            result = tasks_repo.update(id, completed: true)
            Right(result)
          end
        end
      end
    end
  end
end
```

And that's all.