---
heading: Dynamic ActiveRecord columns
subheading: 2019-01-18 - @kddeisz
---

When dealing with any kind of data transmission, the problem almost always boils down to the amount of data being sent. Engineers can optimize the speed of the wire and the compression of the data, but at the end of the day sending fewer things is almost always going to yield the biggest gains.

We see this discussed in web programming all the time with regard to the frontend of the application. Tools like [webpack](https://webpack.js.org/) and [rollup](https://rollupjs.org/) are pushing more and more toward code splitting as an effective way of improving initial page load times (i.e., sending less JavaScript down to the client until they actually need it for execution).

This post, however, is about backend technology. Specifically, it's about sending less data over the wire when selecting data from the application database. Even more specifically, in `Ruby on Rails` applications the `ActiveRecord` [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping) allows us to generate SQL queries. This tool is a [sharp knife](https://m.signalvnoise.com/provide-sharp-knives-cc0a22bf7934) that can very easily end up selecting way more data than we need, resulting in large numbers of allocations, fragmentation, and overall memory bloat. One way to combat this is to use a feature of `ActiveRecord` that involves dynamically creating columns in result hashes.

## Joining strings

As a place to start looking at this technique, let's say we have a `User` class with `first_name` and `last_name` columns. We want a list of the full names of every user in the system, by combining the first and last names of every user into a `full_name` string. We can perform this in `ruby` just fine by running:

```ruby
User.all.map do |user|
  "#{user.first_name} #{user.last_name}"
end
```

This works well and achieves our result, but at the cost of way more memory than we needed. The problem is, `User.all` by default will generate a query that looks like:

```sql
SELECT "users".* FROM "users"
```

That `*` is the dangerous part. It tells the database to return every column that exists on the `users` table. Each of those columns is then returned to `ActiveRecord`, which dutifully converts each of them into `ruby` objects that each require their own allocations. `ActiveRecord` then returns the resulting allocated `User` instances back to us as an array. Once we have it back, we then only use a tiny amount of the allocated memory to perform the task that we actually need to perform.

Remember what we said at the beginning:

> sending fewer things is almost always going to yield the biggest gains

Let's rewrite the query to send fewer things back to us and then work backward from there. Since we only need the `first_name` and `last_name` columns, let's select only those columns:

```sql
SELECT "users"."first_name", "users"."last_name" FROM "users"
```

We can perform the exact same mapping as before, and this time we've required less data (proportional to the number of columns on the `users` table) to be sent back from the database.

One small caveat to the above approach: it works in this example because we're only referencing the `first_name` and `last_name` columns out of the resulting `User` objects. If we were to instead reference a column that we didn't select (e.g., `admin`) then it would raise an error complaining about:

```ruby
ActiveModel::MissingAttributeError (missing attribute: admin)
```

## Dynamic columns

We can do better than the above query though, by performing the string joining in the SQL query itself. One nice feature of `ActiveRecord` is that when it encounters a column in a response from a database query that is not contained in the table that maps to the object performing the query, it treats it as it would any other column. This means we can do the following:

```ruby
users = User.arel_table
concat =
  Arel::Nodes::NamedFunction.new(
    'CONCAT',
    [users[:first_name], Arel::Nodes.build_quoted(' '), users[:last_name]]
  )

User.select(concat.as('full_name')).map(&:full_name)
# => SELECT CONCAT("users"."first_name", ' ', "users"."last_name") AS full_name FROM "users"
```

A couple of things are going on here worth discussing:

* [arel_table](https://apidock.com/rails/ActiveRecord/Core/ClassMethods/arel_table) is a method that gives access to the underlying `Arel` DSL for query genereation. It was introduced in Rails 3 and in the latest versions of Rails is bundled with `ActiveRecord`.
* `Arel::Nodes::NamedFunction` is a way of generating the string for SQL functions that are eventually passed into `ActiveRecord::Relation` objects.
* The real trick here happens with the `as` call, which names the result of the concatenation as a column, which results in the `User` objects having a dynamically-defined `full_name` method, which we can then call as `user.full_name`. In this case we're mapping over the list using symbol's `to_proc` syntax (`&:full_name`).

As a best practice, I like to write the actual definition of the method on the class as well so it looks less like magic. In order to mirror the behavior of missing attributes above, I tend to write methods like the following:

```ruby
class User < ActiveRecord::Base
  def full_name
    return read_attribute(:full_name) if has_attribute?(:full_name)

    raise ActiveModel::MissingAttributeError, 'missing attribute: full_name'
  end
end
```

This can be further simplified with a little metaprogramming, as in the following:

```ruby
module Ext
  module AttrDynamic
    def attr_dynamic(*names)
      names.each do |name|
        define_attr_dynamic(name)
      end
    end

    private

    def define_attr_dynamic(name)
      define_method(name) do
        return read_attribute(name) if has_attribute?(name)

        raise ActiveModel::MissingAttributeError, "missing attribute: #{name}"
      end
    end
  end
end

ActiveRecord::Base.extend(Ext::AttrDynamic)
```

Now I can go back into my `User` class and just call the `attr_dynamic` method, as in:

```ruby
class User < ActiveRecord::Base
  attr_dynamic :full_name
end
```

This indicates to future developers that this column is a dynamically-defined column based on the other columns of the table, and so they should not be surprised when the `full_name` method is referenced later in the code.

## Counting

This approach really shines when getting counts of a parent-child table relationship. Let's take as an example an app with `users` and `posts`:

```ruby
class User < ActiveRecord::Base
  has_many :posts
end

class Post < ActiveRecord::Base
  belongs_to :user
end
```

Let's say we want to know the number of posts by each user within a list of users, for instance on the admin page. In this case we could go and perform a count for each user:

```ruby
User.all.map do |user|
  user.posts.size
end
```

Immediately some combination of your tooling (for instance the [bullet](https://github.com/flyerhzm/bullet) gem) or your coworkers should point out that this will result in an N+1 query problem (in that you make one query for the users and then one more for every users' posts). You can see it in your logs as:

```sql
User Load (13.8ms)  SELECT "users".* FROM "users"
 (36.8ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."user_id" = $1  [["user_id", 1]]
 (0.8ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."user_id" = $1  [["user_id", 2]]
 (0.4ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."user_id" = $1  [["user_id", 3]]
 ...
```

So, you'll go into your code and add the appropriate eager-loading, just as your tooling and coworkers are telling you to:

```ruby
User.includes(:posts).map do |user|
  user.posts.size
end
```

Now when you rerun the code you'll only get two generated queries, one for the `users` table and one for the `posts` table:

```sql
User Load (3.0ms)  SELECT "users".* FROM "users"
  Post Load (1.7ms)  SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN ($1, $2, $3, $4, $5, ...)  [["user_id", 1], ["user_id", 2], ["user_id", 3], ["user_id", 4], ["user_id", 5], ...]
```

This works, but think about all of that data that you didn't need to send back. You selected `"posts".*`, which likely includes a full text column, and you only needed a count. Let's apply the same technique we did above to cut down on all that waste:

```ruby
class User < ActiveRecord::Base
  has_many :posts

  # Using `attr_dynamic` here from the above code example
  attr_dynamic :posts_count

  scope :with_posts_count, lambda {
    select(arel_table[Arel.star], Post.arel_table[:id].count.as('posts_count'))
      .left_joins(:posts).group(arel_table[:id])
  }
end

class Post < ActiveRecord::Base
  belongs_to :user
end
```

Now we can run achieve the same effect as above using both more readable and performant code, as in:

```ruby
User.with_posts_count.map(&:posts_count)
# => SELECT "users".*, COUNT("posts"."id") AS posts_count FROM "users" LEFT OUTER JOIN "posts" ON "posts"."user_id" = "users"."id" GROUP BY "users"."id"
```

This results in an understandable named scope returning a column that is read from a method, all with minimal code and maximal performance.

## Caveats

It would be a little misleading to end the post here, within attaching a slight bit of warning. There are times when dynamic columns are appropriate and times when they aren't, and those times are almost exclusively going to determined through measurement. I would check out [memory_profiler](https://github.com/SamSaffron/memory_profiler) and [ruby-prof](https://github.com/ruby-prof/ruby-prof), both are excellent tools for measuring the impact of code on memory and performance.

Additionally, you can and should measure the performance of your database. This post has discussed sending too much information back, which is a measurable quantity. To find out how, check your respective databases' documentation. There are great ecosystems around all of the popular options (`MySQL`, `PostgreSQL`, `Oracle`, etc.).

Also as an aside, most of the examples above could be even further optimized with some combinations of `select_values`/`pluck`. I'm assuming you have other things to do with the objects in your view layer besides just getting counts.

## tl;dr

`ActiveRecord` is a powerful tool that in inexperienced hands can select far more data than necessary. You can tell it to create method names from dynamic column definitions using SQL strings or `Arel`. These dynamic methods can drastically cut down on the amount of memory allocated both by `ruby` and by your database.
