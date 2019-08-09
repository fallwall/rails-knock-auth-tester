# Rails JWT Authentication with Knock

## Learning Objectives

- Review authentication and authorization
- Create an ActiveRecord user model
- Use the knock Gem to implement JWT auth in Rails
- Use the `authorize_user` and `current_user` methods provided by knock

## Review

### Turn and Talk (10 minutes)

- Describe authentication, authorization, the distinction and how they relate
- Describe the utility of a [JWT](https://jwt.io/introduction/)
- Describe at a high level the process of login in from a client and making authorized requests

## Code Along

### [Create a New App](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/72c27eb13553d3682837dedca5f29016c6916485)

In the terminal, in the directory where you want to create a project, generate a new Rails project and cd in to it:

```shell
rails new scribble --api --database=postgresql
cd scribble
```

### [Add Dependencies](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/818aea0e188a240989bef0207e60f2d80a886f0e)

In your Gemfile, make sure you have:

```ruby
gem 'bcrypt', '~> 3.1.7'
gem 'rack-cors'
gem 'knock'
gem 'jwt'
```

and

```ruby
gem 'pry-rails'
```

in the development group.

In the terminal, from the root of the project, run `bundle install`.


#### Aside: Spring Hangs

Sometimes, a dependency of Rails, Spring, will get confused especially if you have recently navigated between projects with similar names (including deleting and recreating a project).

If you are trying to run a rails command and it hangs for an extended period of time (especially a generator), you can delete the `./bin` directory from the project and run `rake app:update:bin` in the terminal

### Scaffolding [User](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/e32991a05f0af818423f63d13e3c457723afcf61) and [Post](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/531848cc79affd9f37185db7be35600a2590fb87) Resources

Still in the terminal at the root of the project, run:

```shell
rails generate scaffold Post title:string body:text
```

and

```shell
rails generate scaffold User email:string password_digest:string
```

### Set up DB

In the terminal, run:

```shell
rails db:drop db:create db:migrate
```

Note: if you don't have scribble databases, the db:drop won't do anything but won't hurt either.

### Configure the [User Model](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/86ec66931a4c0bde2b83a290217c35b521dcaf01)

Update your user model to include:

1. `has_secure_password` to add an authenticate method to the user that knock will assume exists
2. Validate the presence of an email
3. Define how the payload should be written into the token

```ruby
class User < ApplicationRecord
    has_secure_password
    validates :email, presence: true

    def to_token_payload
        {
            sub: id,
            email: email
        }
    end
end
```

### [Update User Controller](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/842b6fe02b565a4eb4c38ec64f7658f3b9b9a363)

In the scaffolded User controller, update the `user_params` method to:

```ruby
def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
end
```

### [Configure knock](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/d98cdb46186d0905b0309ad20939279052dcb051)

Run `rails generate knock:install` in the terminal at the root of the project to generate a configuration file for knock at `config/initializers/knock.rb`.

### [Generate User Token Controller](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/d2c327eeb62bbe52b4ba28eb2aa9b5f7bc1d0de4)

In terminal:

```shell
rails generate knock:token_controller user
```

### [Fixing the 422 Error](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/9e1eef1f4745a4cbc69f58390a432aeeb97ecb95)

Add the following to the `UserTokenController`:

```ruby
class UserTokenController < Knock::AuthTokenController 
      skip_before_action :verify_authenticity_token, raise: false
 end
```

After this fix, the 422 error will become a 500 with a message around implicit conversion from nil to string.

To fix this, in [the Knock config](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/940e60dddba31e70baa2d39d53daed17ddbffac6) file add:

```ruby
config.token_secret_signature_key = -> { Rails.application.credentials.fetch(:secret_key_base) }
```

### Securing a Route

In a Rails controller, we can set up a before action filter which will run a controller method before running the action to handle the request.

We can use this along with `:authenticate_user` to require that a user is authenticated to access a particular end point.

First we need to add the Knock::Authenticable module to the [application controller](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/54307984e3cda0f5eb293d386562f24d462d0475):

```ruby
class ApplicationController < ActionController::API
  include Knock::Authenticable
end
```

Next we'll add the [`before_action` filter](https://guides.rubyonrails.org/action_controller_overview.html#filters) [to our `PostsController`](https://git.generalassemb.ly/wdi-nyc-octonion/scribble-lab/commit/eb53d5a1a65d93c4957aab2e7eea6e25b80d9100) to secure our post routes:

```ruby
before_action :authenticate_user, only: [:create, :update, :destroy]
```

Note: you can specify the specific actions you want to require authentication for.

## Resources

- [Knock Gem](https://github.com/nsarno/knock)
- [codebrains tutorial (smaller example)](https://codebrains.io/rails-jwt-authentication-with-knock/)
- [James Kropp tutorial](https://engineering.musefind.com/building-a-simple-token-based-authorization-api-with-rails-a5c181b83e02) (Note: some changes in Rails 5.2 are not included in this article but described here: https://medium.com/@willwoodlief/this-was-a-nice-example-to-follow-and-really-helped-me-9940888d8ed1)
