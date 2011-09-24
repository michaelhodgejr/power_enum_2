# Power Enum

Enumerations for Rails 3.X Done Right.

## What is this?:

Power Enum allows you to treat instances of your ActiveRecord models as though they were an enumeration of values.
It allows you to cleanly solve many of the problems that the traditional Rails alternatives handler poorly if at all.
It is particularly suitable for scenarios where your Rails application is not the only user of the database, such as
when it's used for analytics or reporting.

Power Enum is built on top of the Rails 3 modernization made by the fine folks at Protocool https://github.com/protocool/enumerations\_mixin
to the original plugin by Trevor Squires located at https://github.com/protocool/enumerations\_mixin.  While many of the core ideas remain,
it has been reworked and a full test suite written to facilitate further development.

At it's most basic level, it allows you to say things along the lines of:

    booking = Booking.new(:status => BookingStatus[:provisional])
    booking.status = :confirmed

    Booking.find :first,
                 :conditions => ['status_id = ?', BookingStatus[:provisional].id]

    BookingStatus.all.collect {|status|, [status.name, status.id]}

See "How to use it" below for more information.

## Installation

Add the gem to your Gemfile

    gem 'power_enum'

then run

    bundle install

## Gem Contents

This package adds two mixins and a helper to Rails' ActiveRecord, as well as methods to migrations to simplify the creation of backing tables.

`acts_as_enumerated` provides capabilities to treat your model and its records as an enumeration.
At a minimum, the database table for an acts\_as\_enumerated must contain an 'id' column and a column
to hold the value of the enum ('name' by default).  It is strongly recommended that there be
a NOT NULL constraint on the 'name' column.  All instances for the `acts_as_enumerated` model
are cached in memory.  If the table has an 'active' column, the value of that attribute
will be used to determine which enum instances are active.
Otherwise, all values are considered active.

`has_enumerated` adds methods to your ActiveRecord model for setting and retrieving enumerated values using an associated acts\_as\_enumerated model.

There is also an `ActiveRecord::VirtualEnumerations` helper module to create 'virtual' acts\_as\_enumerated models which helps to avoid
cluttering up your models directory with acts\_as\_enumerated classes.

## How to use it

In the following example, we'll look at a Booking that can have several types of statuses.

### migration

    create_enum :booking_status, :name_limit => 50
    # The above is equivalent to saying
    # create_table :booking_statuses do |t|
    #   t.string :name, :limit => 50, :null => false

    #   t.timestamps
    # end

    create_table :bookings do |t|
      t.integer :status_id

      t.timestamps
    end

    # Ideally, you would use a gem of some sort to handle foreign keys.
    execute "ALTER TABLE bookings ADD 'bookings_bookings_status_id_fk' FOREIGN KEY (status_id) REFERENCES booking_statuses (id);"
    
There are two methods added to Rails migrations:

##### `create_enum(enum_name, options = {})`

Creates a new enum table.  `enum_name` will be automatically pluralized.  The following options are supported:

- [:name_column]  Specify the column name for name of the enum.  By default it's :name.  This can be a String or a Symbol
- [:description]  Set this to `true` to have a 'description' column generated.
- [:name_limit]  Set this define the limit of the name column.
- [:desc_limit]  Set this to define the limit of the description column
- [:active]  Set this to `true` to have a boolean 'active' column generated.  The 'active' column will have the options of NOT NULL and DEFAULT TRUE.

Example:

    create_enum :booking_status

is the equivalent of

    create_table :booking_statuses do |t|
      t.string :name
      t.timestamps
    end

In a more complex case:

    create_enum :booking_status, :name_column => :booking_name,
                                 :name_limit  => 50,
                                 :description => true,
                                 :desc_limit  => 100,
                                 :active      => true

is the equivalent of
    
    create_table :booking_statuses do |t|
      t.string :booking_name, :limit => 50, :null => false
      t.string :description, :limit => 100
      t.boolean :active, :null => false, :default => true
      t.timestamps
    end

##### `remove_enum(enum_name)`

Drops the enum table.  `enum_name` will be automatically pluralized.

Example:

    remove_enum :booking_status
    
is the equivalent of
    
    drop_table :booking_statuses

### acts\_as\_enumerated

    class BookingStatus < ActiveRecord::Base
      acts_as_enumerated  :conditions        => 'optional_sql_conditions',
                          :order             => 'optional_sql_orderby',
                          :on_lookup_failure => :optional_class_method,
                          :name_column       => 'optional_name_column'  #If required, may override the default name column
    end

With that, your BookingStatus class will have the following methods defined:

#### Class Methods

##### `[](arg)`

`BookingStatus[arg]` performs a lookup for the BookingStatus instance for the given arg.  The arg value can be a 'string' or a :symbol,
in which case the lookup will be against the BookingStatus.name field.  Alternatively arg can be a Fixnum,
in which case the lookup will be against the BookingStatus.id field.

The `:on_lookup_failure` option specifies the name of a *class* method to invoke when the `[]` method is unable to locate a BookingStatus record for arg.
The default is the built-in `:enforce_none` which returns nil. There are also built-ins for `:enforce_strict` (raise and exception regardless of the type for arg),
`:enforce_strict_literals` (raises an exception if the arg is a Fixnum or Symbol), `:enforce_strict_ids` (raises and exception if the arg is a Fixnum)
and `:enforce_strict_symbols` (raises an exception if the arg is a Symbol).

The purpose of the `:on_lookup_failure` option is that a) under some circumstances a lookup failure is a Bad Thing and action should be taken,
therefore b) a fallback action should be easily configurable.

##### `all`

`BookingStatus.all` returns an array of all BookingStatus records that match the `:conditions` specified in `acts_as_enumerated`, in the order specified by `:order`.

##### `active`

`BookingStatus.active` returns an array of all BookingStatus records that are marked active.  See the `active?` instance method.

##### `inactive`

`BookingStatus.inactive` returns an array of all BookingStatus records that are inactive.  See the `inactive?` instance method.

#### Instance Methods

Each enumeration model gets the following instance methods.

##### `===(arg)`

`BookingStatus[:foo] === arg` returns true if `BookingStatus[:foo] === BookingStatus[arg]` returns true if arg is Fixnum, String,
or Symbol.  If arg is an Array, will compare every element of the array and return true if any element return true for `===`.

You should note that defining an `:on_lookup_failure` method that raises an exception will cause `===` to also raise an exception for any lookup failure of `BookingStatus[arg]`.

`like?` is aliased to `===`

##### `in?(*list)`

Returns true if any element in the list returns true for `===(arg)`, false otherwise.

##### `name`

Returns the 'name' of the enum, i.e. the value in the `:name_column` attribute of the enumeration model.

##### `name_sym`

Returns the symbol representation of the name of the enum.  `BookingStatus[:foo].name_sym` returns :foo.

##### `active?`

Returns true if the instance is active, false otherwise.  If it has an attribute 'active',
returns the attribute cast to a boolean, otherwise returns true.  This method is used by the `active`
class method to select active enums.

##### `inactive?`

Returns true if the instance is inactive, false otherwise.  Default implementations returns `!active?`
This method is used by the `inactive` class method to select inactive enums.

#### Notes

`acts_as_enumerated` records are considered immutable. By default you cannot create/alter/destroy instances because they are cached in memory.
Because of Rails' process-based model it is not safe to allow updating acts\_as\_enumerated records as the caches will get out of sync.

However, one instance where updating the models *should* be allowed is if you are using seeds.rb to seed initial values into the database.

Using the above example you would do the following:

    BookingStatus.enumeration_model_updates_permitted = true
    ['pending', 'confirmed', 'canceled'].each do | status_name |
        BookingStatus.create( :name => status_name )
    end

Note that a `:presence` and `:uniqueness` validation is automatically defined on each model for the name column.

### has\_enumerated

First of all, note that you *could* specify the relationship to an `acts_as_enumerated` class using the belongs_to association.
However, `has_enumerated` is preferable because you aren't really associated to the enumerated value, you are *aggregating* it. As such,
the `has_enumerated` macro behaves more like an aggregation than an association.

    class Booking < ActiveRecord::Base
      has_enumerated  :status, :class_name        => 'BookingStatus',
                               :foreign_key       => 'status_id',
                               :on_lookup_failure => :optional_instance_method
    end

By default, the foreign key is interpreted to be the name of your has\_enumerated field (in this case 'status') plus '\_id'.  Additionally,
the default value for `:class_name` is the camel-ized version of the name for your has\_enumerated field. `:on_lookup_failure` is explained below.

With that, your Booking class will have the following methods defined:

#### `status`

Returns the BookingStatus with an id that matches the value in the Booking.status_id.

#### `status=(arg)`

Sets the value for Booking.status_id using the id of the BookingStatus instance passed as an argument.  As a
short-hand, you can also pass it the 'name' of a BookingStatus instance, either as a 'string' or :symbol, or pass in the id directly.

example:

    mybooking.status = :confirmed
    
this is equivalent to:

    mybooking.status = 'confirmed'

or:

    mybooking.status = BookingStatus[:confirmed]

The `:on_lookup_failure` option in has_enumerated is there because you may want to create an error handler for situations
where the argument passed to `status=(arg)` is invalid.  By default, an invalid value will cause an ArgumentError to be raised.  

Of course, this may not be optimal in your situation.  In this case you can specify an *instance* method to be called in the case of a lookup failure. The method signature is as follows:

    your_lookup_handler(operation, name, name_foreign_key, acts_enumerated_class_name, lookup_value)

The 'operation' arg will be either `:read` or `:write`.  In the case of `:read` you are expected to return something or raise an exception,
while in the case of a `:write` you don't have to return anything.

Note that there's enough information in the method signature that you can specify one method to handle all lookup failures
for all has\_enumerated fields if you happen to have more than one defined in your model.

NOTE: A `nil` is always considered to be a valid value for `status=(arg)` since it's assumed you're trying to null out the foreign key.
The `:on_lookup_failure` will be bypassed.

### ActiveRecord::VirtualEnumerations

In many instances, your `acts_as_enumerated` classes will do nothing more than just act as enumerated.

In that case there isn't much point cluttering up your models directory with those class files. You can use ActiveRecord::VirtualEnumerations to reduce that clutter.

Copy virtual\_enumerations\_sample.rb to Rails.root/config/initializers/virtual\_enumerations.rb and configure it accordingly.

See virtual\_enumerations\_sample.rb in the examples directory of this gem for a full description.


## How to run tests

Go to the 'dummy' project:
    
    cd ./spec/dummy

Run migrations for test environment:

    RAILS_ENV=test rake db:migrate

If you're using Rails 3.1, you should do this instead:

    RAILS_ENV=test bundle exec rake db:migrate

Go back to gem root directory:

    cd ../../

And finally run tests:

    rake spec

## Copyrights and License

* Initial Version Copyright (c) 2005 Trevor Squires
* Rails 3 Updates Copyright (c) 2010 Pivotal Labs
* Initial Test Suite Copyright (c) 2011 Sergey Potapov
* Subsequent Updates Copyright (c) 2011 Arthur Shagall

Released under the MIT License.  See the LICENSE file for more details.
