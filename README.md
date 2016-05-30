# Devise and Pundit

## Learning Objectives

  1. Explain what a policy is, and how it helps manage permissions
  2. Implement Pundit policies for a message board.

## Overview

[CanCanCan](https://github.com/CanCanCommunity/cancancan) is pretty nice when you have a simple permissions model. For that matter, performing authorization checks in controllers is perfectly fine for simple applications.

What happens when, inevitably, your application is no longer simple?

You'll quickly find that your controllers are littered with authorization checks. Or, in CanCan, your `Ability` class will grow until it starts to be unmanageable.

[Pundit] offers a way for us to organize our authorization code. With Pundit, you create `Policy` classes for your models, which define what users can do to those models. This gives you a straightforward, modular way to separate the concern of authorization from your controllers and your model logic.

## Setup
```ruby
    # Gemfile
    gem "pundit"

    # app/controllers/application_controller.rb
    class ApplicationController < ActionController::Base
      include Pundit  # add this line
      protect_from_forgery
    end
```
Finally, `bundle` and run the generator:
```ruby
    rails g pundit:install
```
You'll have to restart Rails at this point for it to pick up the changes.

## Writing simple policies

Consider a message board with these roles:

   * guests can only read posts
   * normal users can do everything guests can do. They can also create posts and edit their own posts.
   * moderators can do all that, and edit and delete the posts of other users.
   * administrators can do anything.

Let's implement this in Pundit with policies.

In Pundit, you keep your authorization rules in `Policy` classes. A policy is a class named like `<model>Policy`. For example, `PostPolicy` describes the policy for a the `Post` model. You keep these in `app/policies`. For example,
```ruby
   # app/policies/post_policy.rb
   class PostPolicy < ApplicationPolicy
     def update?
       user.admin? || user.moderator? || record.try(:user) == user
     end
   end
```
This policy says that admins and moderators can update any post, and users who are neither of those things can update only posts they own.

`record` is the name for the model object. It is set in the `ApplicationPolicy` generated by the installer. If you totally hate this, you can change it in your Policy classes by overriding the initializer:
```ruby
   class PostPolicy < ApplicationPolicy
     attr_reader :post

     def initialize(user, post)
       super(user, post)
       @post = record
     end

     def update?
       user.admin? || user.moderator? || post.try(:user) == user
     end
   end
```
Though this seems not quite worth it to me. Saying `record` is fine.

To use this policy, we call `authorize` within our controller,
```ruby
    class PostsController
      def update
        @post = Post.find(params[:id])
        authorize @post
	# perform an update
      end
    end
```
`authorize` is added to your controllers by adding `include Pundit` to your `ApplicationController`. By default, `authorize` calls the policy method of the same name as your controller action with a question mark after it. We're in the `update` route, and `@post` is a `Post` object, so `authorize` does the equivalent of this:
```ruby
    PostPolicy.new(current_user, @post).update?
```
Within your controllers, Pundit offers a helper for constructing the policy of a model:
```ruby
    policy(@post)
    #which is equivalent to:
    PostPolicy.new(current_user, @post)
```
This is particularly useful in views:
```erb
    <% if policy(@post).update? %>
      <%= link_to "Edit post", edit_post_path(@post) %>
    <% end %>
```
## Testing

Pundit's data model makes testing your authorization code very easy. Where before you needed to write controller tests (which need to understand sessions and make requests to routes and so on), with Pundit, you can write simple unit tests that send in model objects and assert that the right thing happens.
```ruby
    class PostPolicyTest
      test "users can't update others posts" do
        amethyst = users(:amethyst)
        post = posts(:garnet_private)
        expect(post.user).not_to eq(amethyst)
        expect(PostPolicy.new(amethyst, post).update?).to be false
      end
    end
```
## Scopes

Let's add a feature to our message board: drafts. `Posts` now have a `published?` method, and only `admins` can see unpublished posts.

In our `Post#index` route, we want to show the user all the posts they have access to. Pundit uses a `Scope` to give us easy access to that information, particularly in our views.

To use it, you define a class named `Scope`, which inherits from `Scope`, inside your policy:
```ruby
   class PostPolicy < ApplicationPolicy
     class Scope < Scope
       def resolve
         if user.admin?
           scope.all
         else
           scope.where(:published => true)
         end
       end
     end
     # ...
   end
```
The `Scope` class initializes with a `user` and a `scope`, which is an `ActiveRecord::Relation`—that is, exactly the kind of thing you get out of a `where` query. This lets you chain the policy with your queries, so for example I can see all the posts on a particular topic which only I can see.

In your views, you use it like this:
```erb
    <% policy_scope(@user.posts).each do |post| %>
      <p><%= link_to post.title, post_path(post) %></p>
    <% end %>
```
# Updating parameters

Pundit also gives you tools for controlling what attributes of a model a user can update. This is *fabulous*.

To use it, add a `permitted_attributes` method to your Policy:
```ruby
    class PostPolicy < ApplicationPolicy
      def permitted_attributes
        if user.admin? || user.owner_of?(post)
          [:title, :body, :tag_list]
        else
          [:tag_list]
        end
      end
    end
```
Here we've added a feature where users can update the tags of any post, and edit the title and body of their own.

In your controllers, you get a helper, `permitted_attributes`, which takes a model and returns which attributes are writable.
```ruby
    class PostsController
      def update
        @post = Post.find(params[:id])
        if @post.update_attributes(permitted_attributes(@post))
          redirect_to @post
        else
          render :edit
        end
      end
    end
```
You will be surprised at how many of your `params.permit` headaches this will resolve.

## Conclusion

[Pundit] is a useful gem with a simple design. As the docs say, Pundit doesn't do anything you couldn't have easily done yourself. That said, that's true of a lot of programming, and it's nice that someone has made the design decisions already.

The documentation is extremely useful, and I recommend reading through it. As you do, consider how Pundit is completing the whole picture. [Devise] gives us all the tools to figure out who someone is, and [Pundit] is a neat, expressive way to say what they can do. Together, you can build a lot with them.

## Resources
  * [Pundit]
  * [Devise]

[Devise]: https://github.com/plataformatec/devise
[Pundit]: https://github.com/elabs/pundit

<p data-visibility='hidden'>View <a href='https://learn.co/lessons/devise_pundit_readme'>Devise and Pundit</a> on Learn.co and start learning to code for free.</p>
