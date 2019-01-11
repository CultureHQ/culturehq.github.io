---
heading: Delegating ActiveRecord scopes
subheading: 2019-01-11 - @kddeisz
---

In `Ruby on Rails`, access to the database is by default controlled through the `ActiveRecord` ORM. `ActiveRecord` operates on `Relation` objects that contain the configuration for an SQL query that will be executed at a later time. `Relation` objects have the ability to copy their internal configuration over to a new `Relation` object with new options in order to further customize those queries.

Let's look at an example to see what all of that actually looks like:

```ruby
ActiveRecord::Schema.define do
  create_table :users, force: true do |t|
    t.string :name
    t.boolean :admin
    t.timestamps
  end
end

class User < ActiveRecord::Base
end

users = User.all
# => users is an ActiveRecord::Relation

users.to_sql
# => SELECT "users".* FROM "users"

admins = users.where(admin: true)
# => admins is a new relation, and not equal to users

admins.to_sql
# => SELECT "users".* FROM "users" WHERE "users"."admin" = TRUE
```

This allows us to construct various relations in our apps without having to manually write out SQL queries. Because these kinds of relation mutations (e.g. `where(admin: true)`) tend to get repeated throughout the application, `Rails` provides a mechanism called scopes to name them and define the relation mutation.

For example, in order to define what consistutes an "admin" user in one location without having to use `where` everywhere, you can define a scope like:

```ruby
class User < ActiveRecord::Base
  scope :admin, -> { where(admin: true) }
end

User.admin.to_sql
# => SELECT "users".* FROM "users" WHERE "users"."admin" = TRUE
```

In addition to these scopes making our code less repetitive, they have the ability to be combined. This allows us to express queries quite simply that would otherwise be extremely verbose if we were restricted to `ActiveRecord` primitives (as in `where`, `order`, `group`, `having`, etc.).

## Associations

Scopes really start to shine when associations are involved. Let's add to our example application the concept of posts that belong to users, as in:

```ruby
ActiveRecord::Schema.define do
  create_table :users, force: true do |t|
    t.string :name
    t.boolean :admin
    t.timestamps
  end

  create_table :posts, force: true do |t|
    t.references :user
    t.string :title
    t.text :body
    t.timestamps
  end
end

class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy

  scope :admin, -> { where(admin: true) }
end

class Post < ActiveRecord::Base
  belongs_to :user
end
```

This parent-child relationship between two tables is very common, and often results in the requirement of mutating child queries based on some criteria about the parent table. For example, say we want to find only the posts from admin users. In this case we can construct a query like:

```ruby
Post.joins(:user).where(users: { admin: true }).to_sql
# => SELECT "posts".* FROM "posts" INNER JOIN "users" ON "users"."id" = "posts"."user_id" WHERE "users"."admin" = TRUE
```

This works, but fails to take advantage of the named scope that we defined to abstract away what makes an admin user. Fortunately, `ActiveRecord` has a way to take the mutations already performed on one `Relation`, and merge them into another, aptly called `merge`:

```ruby
Post.joins(:user).merge(User.admin).to_sql
# => SELECT "posts".* FROM "posts" INNER JOIN "users" ON "users"."id" = "posts"."user_id" WHERE "users"."admin" = TRUE
```

Now we're able to reuse the named scope logic throughout our application wherever admin users need to be found, resulting in code that is easier to read and maintain. We can go one step further in readability and make the `merge` call itself a scope on the `Post` class, as in:

```ruby
class Post < ActiveRecord::Base
  belongs_to :user

  scope :by_admins, -> { joins(:user).merge(User.admin) }
end
```

At this point the logic of the query is fully abstracted, and we can call `Post.by_admins` to generate the same query we've been working with.

## Delegating

A common pattern within object-oriented languages is method delegation (also known as method forwarding). Basically, when a message is received by an object, it can either handle it itself or delegate it to an associated object that contains the internal data necessary to process it.

Applying this pattern to our exercise in scopes above, we've effectively delegated the decision of what constitutes an admin user to the `User` class (where it naturally should live). `User` has knowledge of its own internal data structure (or in this case column names) that `Post` should not have to know about in order to request admin users.

The pattern of delegating scopes is common enough, in fact, that we can abstract the logic of merging scopes through another class macro to even further improve readability, as in the following code:

```ruby
module Ext
  module DelegateScope
    def delegate_scope(*scope_names, to:, source: :name)
      klass = reflect_on_association(to).klass

      scope_names.each do |scope_name|
        name = source == :name ? scope_name : source
        scope scope_name, delegate_scope_for(to, klass, name)
      end
    end

    private

    def delegate_scope_for(to, klass, name)
      ->(*args) { joins(to).merge(klass.public_send(name, *args)) }
    end
  end
end

ActiveRecord::Base.extend(Ext::DelegateScope)
```

A couple of metaprogramming things are happening here worth discussing:

* [`reflect_on_assocation`](https://api.rubyonrails.org/classes/ActiveRecord/Reflection/ClassMethods.html#method-i-reflect_on_association) is a method that comes from `ActiveRecord` that returns an `ActiveRecord::Reflection::AssociationReflection` object containing metadata about an association. In this context we're using it to determine the class to which we're delegating.
* The second argument to [`ActiveRecord::Base::scope`](https://api.rubyonrails.org/classes/ActiveRecord/Scoping/Named/ClassMethods.html#method-i-scope) is required to be a callable (in this case a lambda), which we are constructing using the `delegate_scope_for` method.
* [`public_send`](https://apidock.com/ruby/Object/public_send) is a method for dynamically dispatching a public method to an object that isn't known at "compile" time. ("compile" is in quotes because `Ruby` is not a traditional compiled language even though it has a VM bytecode compile step.)

The code example above allows us to refine the `by_admins` scope in the `Post` class to look like the following:

```ruby
class Post < ActiveRecord::Base
  belongs_to :user

  delegate_scope :by_admins, to: :user, source: :admin
end
```

While we haven't changed the overall functionality of the `by_admins` scope, we've arguably improved the readability. As scopes proliferate throughout a codebase as it grows, it helps to be able to use `delegate_scope` to cut down on the `joins().merge()` repetition and to give a name to what it is the scope is trying to achieve.

In this way we can delineate between scopes that are defined for the purpose of modifying the query for the table defined in the current class versus scopes that are used to modify the query for associated tables.

## tl;dr

The `ActiveRecord` ORM provides named scopes that contain logic determining how to refine SQL queries. These can be combined across associations using the `merge` method. Furthermore, the `merge` call can be defined in its own named scope. Finally, we can give a name to these kinds of `merge` scopes (`delegate_scope` is proposed above) to indicate that the named scope modifies associated tables.
