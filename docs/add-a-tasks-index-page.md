# Add a tasks index page

## Routing tasks

In order to access to the `/tasks` path you need to add a `tasks.rb` file in `apps/main/web/routes/` that calls the view controller that respond to this request:

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Web
      route 'tasks' do |r|
        r.is do
          r.get do
            r.view 'tasks.index'
          end
        end
      end
    end
  end
end
```

## A view controller for tasks index

Then the view controller for `Tasks::Index` should be in the file: `apps/main/lib/todo/main/views/tasks/index.rb`.  It needs to include the `Todo::Repositories::TasksRepo` that gives access to the tasks table content. It needs to expose the tasks array to the template.

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Tasks
        class Index < Main::View::Controller
          include Import['core.repositories.tasks_repo']

          configure do |config|
            config.template = 'tasks/index'
          end

          expose :tasks

          private

          def tasks
            tasks_repo.tasks.to_a
          end
        end
      end
    end
  end
end
```

## The view template

And the view `apps/main/web/templates/tasks/index.html.slim` builds a html list with the tasks array.

```slim
h1 Tasks

ul.tasks
  - tasks.each do |task|
    li.task
      span = task.description
```

## Adding tasks from console

Let's add some tasks in the console in order to see it working. 

```
$ bin/console
> user = Todo::Container['repositories.users_repo'].users.first
> tasks_repo = Todo::Container['repositories.tasks_repo']
> tasks_repo.create(description: 'My first task', user_id: user.id)
```

The tasks are not user-scoped yet. If you inspect the file `log/development.log` you will find the SQL used to respond the `get /tasks` request.
```
Started GET "/tasks" for 127.0.0.1 at 2017-10-08 15:37:39 -0300
  Loaded :users in 1.55ms SELECT "id", "email", "password_digest", "created_at", "updated_at" FROM "users" WHERE ("users"."id" = 1) ORDER BY "users"."id"
  Loaded :tasks in 0.4ms SELECT "id", "description", "completed", "user_id", "created_at", "updated_at" FROM "tasks" ORDER BY "tasks"."id"
Finished GET "/tasks" for 127.0.0.1 in 33.67ms [Status: 200]
```

There is not a `WHERE` condition when reading tasks. So, next to do is:

## Scoping tasks for current user

I want a user can see its own tasks, but he/she should not see a task that is not own. Then the view controller needs to know the `current_user`, so it will be passed as a parameter.

And I want to put all the logic of access for each resource in a common place, the folder `apps/main/lib/todo/main/policies/`. The policy for tasks will be located in the file `tasks_scope.rb`.

The `apps/main/lib/todo/main/policies/tasks_scope.rb` will import the `TasksRepo` and will be called with a `user`, then it will return the relation of tasks that belong to the user received.

```ruby
# frozen_string_literal: true

require 'todo/main/import'

module Todo
  module Main
    module Policies
      class TasksScope
        include Import['core.repositories.tasks_repo']

        def call(user)
          tasks_repo.tasks.where(user_id: user.id)
        end
      end
    end
  end
end
```

The view controller: `apps/main/lib/todo/main/views/tasks/index.rb` does not need any more the `TasksRepo` because it will ask to the `Policies::TaskScope` for the `current_user`'s tasks. And `current_user` will be received as a `private_expose`.

```ruby
# frozen_string_literal: true

require 'todo/main/view/controller'
require 'todo/main/import'

module Todo
  module Main
    module Views
      module Tasks
        class Index < Main::View::Controller
          include Import['policies.tasks_scope']

          configure do |config|
            config.template = 'tasks/index'
          end

          private_expose :current_user

          expose :tasks do |current_user|
            tasks_scope.(current_user).to_a
          end
        end
      end
    end
  end
end
```

Last to make it work, is to add the `current_user` as a parameter for the view call in `apps/main/web/routes/tasks.rb`.

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Application
      route 'tasks' do |r|
        r.is do
          r.get do
            r.view 'tasks.index', current_user: current_user
          end
        end
      end
    end
  end
end
```

And the user will see his/her own tasks exclusively. And the SQL at the log reflects this behavior.
```
Started GET "/tasks" for 127.0.0.1 at 2017-10-08 15:44:47 -0300
  Loaded :users in 1.7ms SELECT "id", "email", "password_digest", "created_at", "updated_at" FROM "users" WHERE ("users"."id" = 1) ORDER BY "users"."id"
  Loaded :tasks in 0.61ms SELECT "id", "description", "completed", "user_id", "created_at", "updated_at" FROM "tasks" WHERE ("user_id" = 1) ORDER BY "tasks"."id"
Finished GET "/tasks" for 127.0.0.1 in 36.16ms [Status: 200]
```