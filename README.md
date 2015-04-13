Active Record Lite - A short implementation of a useful tool
-----
# Building ActiveRecordLite

In this project, we build our own lite version of ActiveRecord. The
purpose of this project is for you to understand how ActiveRecord
actually works: how your ActiveRecord world is translated into SQL.

## Setup

We will email you the skeleton project to work from - check your email!

There are specs in it which
will guide you through the project.

## Phase 0: Implement `my_attr_accessor`

This phase will serve as a (relatively) easy warm up to
metaprogramming. You already know what the standard Ruby method
`attr_accessor` does. What if Ruby didn't provide this convenient
method for you?

In the `lib/00_attr_accessor_object.rb` file, implement a
`::my_attr_accessor` macro, which should do exactly the same thing as
the real `attr_accessor`: it should define setter/getter methods.

To do this, use `define_method` inside `::my_attr_accessor` to define
getter and setter instance methods. You'll want to investigate and use
the `instance_variable_get` and `instance_variable_set` methods
described [here][ivar-get].

[ivar-get]: http://ruby-doc.org/core-2.0.0/Object.html#method-i-instance_variable_get

There is a corresponding `spec/00_attr_accessor_object_spec.rb` spec
file. Run it using `bundle exec rspec` to check your work.

**NB:** This phase is just a warm up. Don't use `my_attr_accessor` in
the next phase, as the mode of data storage will be an attributes hash
instead of instance variables.

## Phase I: `SQLObject`: Overview

Our job is to write a class, `SQLObject`, that will interact with the
database. By the **end** of this phase, we will have written the
following methods, just like the real `ActiveRecord::Base`:

* `::all`: return an array of all the records in the DB
* `::find`: look up a single record by primary key
* `#insert`: insert a new row into the table to represent the
  `SQLObject`.
* `#update`: update the row with the `id` of this `SQLObject`
* `#save`: convenience method that either calls `insert`/`update`
  depending on whether or not the `SQLObject` already exists in the table.

## Phase Ia: `::table_name` and `::table_name=`

Before we begin writing methods that interact with the DB, we need to
be able to figure out which table the records should be fetched from,
inserted into, etc. We should write a class getter method
`::table_name` which will get the name of the table for the class. We
should also write a `::table_name=` setter to set the table. You
definitely want to use a class instance variable to store this.

Example:

```ruby
class Human < SQLObject
  self.table_name = "humans"
end

Human.table_name # => "humans"
```

It would also be nice if, in the absence of an explicitly set table
name, we would have `::table_name` by default convert the class name
to snake\_case and pluralize:

```ruby
class BigDog < SQLObject
end

BigDog.table_name # => "big_dogs"
```

ActiveSupport (part of Rails) has an inflector library that adds
methods to `String` to help you do this. In particular, look at the
`String#tableize` method. You can require
the inflector with `require 'active_support/inflector'`.

**NB**: you cannot always infer the name of the table. For example:
the `inflector` library will, by default, pluralize `human` into
`humen`, not `humans`. [WAT][wat]. That's what your `::table_name=` is
for: so users of `SQLObject` can override the default, inferred table
name.

Make sure the `::table_name` specs in `spec/01_sql_object_spec` are
working and move onward!

[wat]: http://codropspz.tympanus.netdna-cdn.com/codrops/wp-content/uploads/2013/07/wat.jpg

## Phase Ib: Listing Columns

In our sample database, the `cats` table has `id`, `name`, and
`owner_id` columns. When we define a model class `Cat < SQLObject`, it
should automatically have setter and getter methods for each of the
columns. For instance, we want to be able to write:

```ruby
class Cat < SQLObject
  # We'll explain finalize! later!
  self.finalize!
end

c = Cat.new
c.name = "Gizmo"
c.owner_id = 123

c.name #=> "Gizmo"
c.owner_id #=> 123
```

We'll get there eventually, but let's start by writing a `SQLObject`
**class method** `::columns`, which should return an array with the
names of the table's columns.  We want `Cat.columns == [:id, :name,
:owner_id]`. To do this, we can query the database using
`DBConnection.execute2` to ask it what the columns are for a table.
Let's see how we might do this:

```ruby
DBConnection.execute2(<<-SQL)
  SELECT
    *
  FROM
    cats
SQL
# => [
#   ["id", "name", "owner_id"],
#   {"id"=>1, "name"=>"Breakfast", "owner_id"=>1},
#   {"id"=>2, "name"=>"Earl", "owner_id"=>2},
#   {"id"=>3, "name"=>"Haskell", "owner_id"=>3},
#   {"id"=>4, "name"=>"Markov", "owner_id"=>3}
# ]
```

The `DBConnection::execute2` method returns an array; the first is
a list of the names of columns, while the rest of the items represent
individual records in the DB. (**Note:** We're using `execute2` to
easily retrieve the column names. You'll want to stick with the
regular `execute` method for all other queries.)

Now that you've seen how to use `DBConnection::execute2`, write a
`::columns` method which queries the DB (interpolating the class's
`::table_name`), and returns the array of columns **as symbols**.

## Phase Ic: Getters and Setters

Now that we can list columns, we'll write a class method `::finalize!`
that automatically adds getter and setter methods for each columns.
Here is how it's intended to be used:

```ruby
class Cat < SQLObject
  # Finalize is called at the end of the subclass definition to
  # add the getters/setters.
  self.finalize!
end

cat = Cat.new
cat.name = "Gizmo"
cat.name #=> "Gizmo"
```

The `::finalize!` class method should call `::columns` and iterate
through them, using `define_method` (twice) to define a getter and
setter instance method for each column.

When we say `cat.name = "Gizmo"`, where should we store the value
"Gizmo"? We *could* save it in an instance variable `@name`, but I don't
want you to do that. Instead, your setter methods should store all the
record data in an attributes hash. It should work like so:

```ruby
cat = Cat.new
cat.attributes #=> {}
cat.name = "Gizmo"
cat.attributes #=> { name: "Gizmo" }
```

**Define an `#attributes` instance method.** It should initialize
`@attributes` to an empty hash and store any new values added to it.

**NB**: it's important that the user of `SQLObject` call `finalize!`
at the end of their subclass definition, otherwise the getter/setter
methods don't get defined. That's hacky, but it will have to do. :-)

Make sure the `::columns` and setter/getter specs now pass.

## Phase Id: `#initialize`

Write an `#initialize` method for `SQLObject`. It should take in a
single `params` hash. We want:

```ruby
cat = Cat.new(name: "Gizmo", owner_id: 123)
cat.name #=> "Gizmo"
cat.owner_id #=> 123
```

Your `#initialize` method should iterate through each of the `attr_name,
value` pairs. For each `attr_name`, it should first convert the name to
a symbol, and then check whether the `attr_name` is among the `columns`.
If it is not, raise an error:

    unknown attribute '#{attr_name}'

Set the attribute by calling the setter method. Use `#send`; avoid
using `@attributes` or `#attributes` inside `#initialize`.

**Hint**: we need to call `::columns` on a class object, not the
instance. For example, we can call `Dog::columns` but not
`dog.columns`.

Note that `dog.class == Dog`. How can we use the `Object#class` method
to access the `::columns` **class method** from inside the
`#initialize` **instance method**?

Run the specs, Luke!

## Phase Ie: `::all`, `::parse_all`

We now want to write a method `::all` that will fetch all the records
from the database. The first thing to do is to try to generate the
necessary SQL query to issue. Generate SQL and print it out so you can
view and verify it. Use the heredoc syntax to define your query.

Example:

```ruby
class Cat < SQLObject
  finalize!
end

Cat.all
# SELECT
#   cats.*
# FROM
#   cats

class Human < SQLObject
  self.table_name = "humans"

  finalize!
end

Human.all
# SELECT
#   humans.*
# FROM
#   humans
```

Notice that the SQL is formulaic except for the table name, which we
need to insert. Use ordinary Ruby string interpolation (`#{whatevs}`) for
this; SQL will only let you use `?` to interpolate **values**, not
table or column names.

Once we've got our query looking good, it's time to execute it. Use
the provided `DBConnection` class. You can use
`DBConnection.execute(<<-SQL, arg1, arg2, ...)` in the usual manner.

Calling `DBConnection` will return an array of raw `Hash` objects
where the keys are column names and the values are column values. We
want to turn these into Ruby objects:

```ruby
class Human < SQLObject
  self.table_name = "humans"

  finalize!
end

Human.all
=> [#<Human:0x007fa409ceee38
  @attributes={:id=>1, :fname=>"Devon", :lname=>"Watts", :house_id=>1}>,
 #<Human:0x007fa409cee988
  @attributes={:id=>2, :fname=>"Matt", :lname=>"Rubens", :house_id=>1}>,
 #<Human:0x007fa409cee528
  @attributes={:id=>3, :fname=>"Ned", :lname=>"Ruggeri", :house_id=>2}>]
```

To turn each of the `Hash`es into `Human`s, write a
`SQLObject::parse_all` method. Iterate through the results, using
`new` to create a new instance for each.

`new` what? `SQLObject.new`? That's not right, we want `Human.all` to
return `Human` objects, and `Cat.all` to return `Cat`
objects. **Hint**: inside the `::parse_all` class method, what is
`self`?

Run the `::parse_all` and `::all` specs! Then carry on!

## Phase If: `::find`

Write a `SQLObject::find(id)` method to return a single object with
the given id. You could write `::find` using `::all` and `Array#find`
like so:

```ruby
class SQLObject
  def self.find(id)
    self.all.find { |obj| obj.id == id }
  end
end
```

That would be inefficient: we'd fetch all the records from the DB.
Instead, write a new SQL query that will fetch at most one record.

Yo dawg, I heard you like specs, so I spent a lot of time writing
them. Please run them again. :-)

## Phase Ih: `#insert`

Write a `SQLObject#insert` instance method. It should build and
execute a SQL query like this:

```sql
INSERT INTO
  table_name (col1, col2, col3)
VALUES
  (?, ?, ?)
```

To simplify building this query, I made two local variables:

* `col_names`: I took the array of `::columns` of the class and
  joined it with commas.
* `question_marks`: I built an array of question marks (`["?"] * n`)
  and joined it with commas. What determines the number of question marks?

Lastly, when you call `DBConnection.execute`, you'll need to pass in
the values of the columns. Two hints:

* I wrote a `SQLObject#attribute_values` method that returns an array
  of the values for each attribute. I did this by calling `Array#map`
  on `SQLObject::columns`, calling `send` on the instance to get
  the value.
* Once you have the `#attribute_values` method working, I passed this
  into `DBConnection.execute` using the splat operator.

When the DB inserts the record, it will assign the record an ID.
After the `INSERT` query is run, we want to update our `SQLObject`
instance with the assigned ID. Check out the `DBConnection` file for a
helpful method.

Again with the specs please.

## Phase Ii: `#update`

Next we'll write a `SQLObject#update` method to update a record's
attributes. Here's a reminder of what the resulting SQL should look
like:

```sql
UPDATE
  table_name
SET
  col1 = ?, col2 = ?, col3 = ?
WHERE
  id = ?
```

This is very similar to the `#insert` method. To produce the
"SET line", I mapped `::columns` to `#{attr_name} = ?` and joined with
commas.

I again used the `#attribute_values` trick. I additionally passed in
the `id` of the object (for the last `?` in the `WHERE` clause).

Every day I'm testing.

## Phase Ij: `#save`

Finally, write an instance method `SQLObject#save`. This should call
`#insert` or `#update` depending on whether `id.nil?`. It is not
intended that the user call `#insert` or `#update` directly (leave
them public so the specs can call them :-)).

You did it! Good work!

## Phase II: `Searchable`

Let's write a module named `Searchable` which will add the ability to
search using `::where`. By using `extend`, we can mix in `Searchable`
to our `SQLObject` class, adding all the module methods as class
methods.

So let's write `Searchable#where(params)`. Here's an example:

```ruby
haskell_cats = Cat.where(:name => "Haskell", :color => "calico")
# SELECT
#   *
# FROM
#   cats
# WHERE
#   name = ? AND color = ?
```

I used a local variable `where_line` where I mapped the `keys` of the
`params` to `"#{key} = ?"` and joined with `AND`.

To fill in the question marks, I used the `values` of the `params`
object.

## Phase III+: Associations

Phase III: Associatable

It's time to move into defining belongs_to and has_many. We're going to add these methods to an Associatable module that we'll mixin to SQLObject.

Phase IIIa: AssocOptions

The first step is to build classes that will store the essential information needed to define the belongs_to and has_many associations:

#foreign_key
#class_name
#primary_key
We will write BelongsToOptions and HasManyOptions classes. Both will extend AssocOptions. The main responsibility of these classes is to provide default values for the three important attributes:

options = BelongsToOptions.new(:owner)
options.foreign_key # => :owner_id
options.primary_key # => :id
# this is not the class name...
options.class_name # => "Owner"

# override defaults
options = BelongsToOptions.new(:owner, :class_name => "Human")
options.class_name # => "Human"
Use the inflector's String#camelcase,String#singularize, String#underscore to aid you in your quest.

After you have these basic defaults working, write #model_class, which should use String#constantize to go from a class name to the class object. Likewise, write #table_name to give the name of the table:

options = BelongsToOptions.new(:owner, :class_name => "Human")
options.model_class # => Human
# should call `Human::table_name`
options.table_name # => "humans"


Phase IIIb: belongs_to, has_many

Begin writing a belongs_to method for Associatable. This method should take in the association name and an options hash. It should build a BelongsToOptions object; save this in a local variable named options.

Within belongs_to, call define_method to create a new method to access the association. Within this method:

Use send to get the value of the foreign key.
Use model_class to get the target model class.
Use where to select those models where the primary_key column is equal to the foreign key value.
Call first (since there should be only one such item).
Throughout this method definition, use the options object so that defaults are used appropriately.

Do likewise for has_many.

Phase IV: has_one_through

This is the last phase! We want to write a has_one_through method that will combine two belongs_to associations. For example:

class Cat < SQLObject
  belongs_to :human, :foreign_key => :owner_id
  has_one_through :home, :human, :house

  finalize!
end

class Human < SQLObject
  self.table_name = "humans"

  belongs_to :house

  finalize!
end

class House < SQLObject
  finalize!
end

cat.home # => house the cat's owner lives in.
Our goal is to generate SQL that looks like this:

SELECT
  houses.*
FROM
  humans
JOIN
  houses ON humans.house_id = houses.id
WHERE
  humans.id = ?
  
Phase IVa: storing AssocOptions

has_one_through is going to need to make a join query that uses and combines the options (table_name, foreign_key, primary_key) of the two constituent associations. This requires us to store the options of a belongs_to association so that has_one_through can later reference these to build a query.

Modify your 03_associatiable.rb file and implement the ::assoc_options class method. It lazily-initializes a class instance variable with a blank hash. Modify your belongs_to method to save the BelongsToOptions in the assoc_options hash, setting the options as the value for the key name:

class Cat < SQLObject
  belongs_to :human, :foreign_key => :owner_id

  finalize!
end

human_options = Cat.assoc_options[:human]
human_options.foreign_key # => :owner_id
human_options.class_name # => "Human"
human_options.primary_key # => :id

Part IVb: writing has_one_through

Okay, now that we are saving the BelongsToOptions, we can access them later to build the has_one_through(name, through_name, source_name) query.

As before, use define_method to define a method that will fetch the associated object. To get the necessary options objects:

Lookup through_name in assoc_options; call this through_options.
Using through_options.model_class, lookup source_name in assoc_options; call this source_options.

Why can we not lookup source_name in self.class.assoc_options?
Once you have these two sets of options, it's time to write the query. Look at the above sample query to inspire your building of the query from the constituent association options.

Unlike when you used where in the belongs_to/has_many, you'll have to ::parse_all yourself.

A Common Mistake

There's a common mistake that most people make:

module Associatable
  def has_one_through(name, through_name, source_name)
    through_options = self.class.assoc_options[through_name]
    # no! too early!
    source_options =
      through_options.model_class.assoc_options[source_name]

    define_method(name) do
      # ...
    end
  end
end

Why is this bad? Let's see:

class Cat < SQLObject
  belongs_to :human, :foreign_key => :owner_id
  # `Human` class not defined yet!
  has_one_through :home, :human, :house

  finalize!
end

class Human < SQLObject
  # ...
end

The problem is that at the time we call has_one_through in Cat, we haven't' yet defined the Human class. But has_one_through calls through_options.model_class, which is going to try to call "Human".constantize. This will fail, because Human is not defined yet.

The solution to this problem is to move the fetching of the options inside the defined method. Presumably someone will only call cat.house after the declaration of both Human and House classes. That means that at the time #house is called, has_one_through will be able to constantize "Human" and "House" successfully.

Extension Ideas

Write where so that it is lazy and stackable. Implement a Relation class.
Write an includes method that does pre-fetching.
has_many :through
This should handle both belongs_to => has_many and has_many => belongs_to.
Validation methods/validator classes
joins
