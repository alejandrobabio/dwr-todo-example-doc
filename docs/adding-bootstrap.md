# Adding Bootstrap

## Layouts

Update application template and add partials for flash and navigation.

`apps/main/web/templates/layouts/application.html.slim`:

```slim
doctype html
html lang="en"
  head
    /! Mobile first
    meta name="viewport" content="width=device-width, initial-scale=1"

    /! Latest compiled and minified CSS
    link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous"

    /! Optional theme
    link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css" integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" crossorigin="anonymous"

    script src="https://code.jquery.com/jquery-3.2.1.min.js"

    /! Latest compiled and minified JavaScript
    script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js" integrity="sha384-Tc5IQib027qvyjSMfHjOMaLkfuWVxZxUPnCJA7l2mCWNIpG9mGCD8wGNIcPD7Txa" crossorigin="anonymous"

  body style="padding-top: 60px"

    == render 'navigation'

    .container-fluid
      .row
        .col-md-8.col-md-offset-2

          == render 'flash'

          .action
            == yield
```

`apps/main/web/templates/layouts/_flash.html.slim`:

```slim
- if flash[:notice]
  .notice.alert.alert-success role='alert'
    = flash[:notice]
    button.close type="button" data-dismiss="alert" aria-label="Close"
      span aria-hidden="true" &times;

- if flash[:alert]
  .alert.alert.alert-danger role='alert'
    = flash[:alert]
    button.close type="button" data-dismiss="alert" aria-label="Close"
      span aria-hidden="true" &times;
```

`apps/main/web/templates/layouts/_navigation.html.slim`:

```slim
.navbar.navbar-fixed-top.navbar-inverse
  .navbar-inner
    .container-fluid
      .navbar-header
      button.navbar-toggle.collapsed type="button" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1" aria-expanded="false"
        span.sr-only Toggle navigation
        span.icon-bar
        span.icon-bar
        span.icon-bar
      a.navbar-brand href="/" Todo

      ul.nav.navbar-nav
        li
          a href="/tasks"
            | Tasks

      ul.nav.navbar-nav.navbar-right
        - if current_user
          li
            a == current_user.email
          li
            a href="/logout"
              | Log Out
        - else
          li
            a href="/login"
              | Log In
          li
            a href="/signup"
              | Sign Up
```

## Improve the action's pages

`apps/main/web/templates/sessions/login.html.slim`:

```slim
h1 Log In

div
  -if error
    = error
br

form action='/login' method='post'
  == csrf_tag
  .form-group
    label for='user_email'
      | Email
    input.form-control id='user_email' name='user[email]' type='text'
  .form-group
    label for='user_password'
      | Password
    input.form-control id='user_password' name='user[password]' type='password'
  input.btn.btn-default type='submit' value='Log in'
```

`apps/main/web/templates/tasks/index.html.slim`: 

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
  input.form-control name='task[description]' placeholder='Add a new task' value=new_task.description

br

ul.list-group.tasks
  - tasks.each do |task|
    - contextual_type = task.completed ? 'info' : 'warning'
    li.list-group-item.task class="list-group-item-#{contextual_type}"
      span = task.description
      - if task.completed
        .pull-right
          | Completed
      - else
        form.pull-right action="/tasks/#{task.id}" method='post'
          == csrf_tag
          input type='hidden' name='_method' value='patch'
          input.btn.btn-default.btn-xs type='submit' value='Complete'
```

`apps/main/web/templates/users/signup.html.slim`:

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
  .form-group
    label for='user_email'
      | Email
    input.form-control id='user_email' name='user[email]' type='text' value=new_user.email
  .form-group
    label for='user_password'
      | Password
    input.form-control id='user_password' name='user[password]' type='password'
  input.btn.btn-default type='submit' value='Sign Up'
```