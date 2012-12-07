RORM
====
The intention of this project is to offer persistence support for domain objects written purely in Ruby.
The intention is to let the persistence concern to be separated from domain logic concern.
Motivation
==========
Ruby is increasingly being used in many different systems. 
Many of those systems use an ORM mechanism based on ActiveRecord idea where the model objects have their 
persistence information mixed with business logic. Although this is fine for small to medium system, this 
is not the best approach for large systems where domain objects are recommended.

 * Isolate business logic from persistence

In large systems it is important to have an isolated business logic from the persistence logic.  
Thus a class User is focused on represent the business logic of users without concern to the persistence mechanism. 

 * Flexible persistence mechanism 

A good ORM should allow the developers to have some flexibility between the object model and the persistence model. 
It is not always good to have one table per class. Class hierarchies can be represented in different ways. Besides in other cases we would like to use value objects to represent some data types inside entities.

 * Work with legacy database

It is not very common to have systems written entirely from zero. 
ORM should allow developer to write entities that can be mapped to legacy database infrastructure.

 * Faster to test
Isolated business objects are faster to test since they don´t depend on the database to check their logic. 
As a consequence, automated tests take shorter time.

Pure Ruby Classes Approach
==========================

Plain Old Ruby Objects are declared only with standard ruby language constructs.

Take the class below as an example.

``` ruby
class User
  attr_accessor :name  
  attr_accessor :email
  attr_accessor :address
  attr_accessor :status
  attr_accessor :tasks
  attr_accessor :groups
end
```

Although clean, the class User above provides insufficient information about its usage. 
Because ruby is a dynamic language, all the properties declared above can be used as a string, 
number, date time, reference to an object, etc. 
Nothing tell us, for example, that the address attribute is actually an association to an address object 
or if it is simply a string field.

Virtus Approach
===============

Virtus is a ruby gem that can be found in https://github.com/solnic/virtus.

It enriches a standard ruby class with extra meta-data by providing a way to declare attributes with types. 
See the difference from the class above with the new class User below:

``` ruby
class User
  include Virtus
  attribute :name, String
  attribute :email, String
  attribute :address, Address
  attribute :status, Status
  attribute :tasks, Array[Task]
  attribute :groups, Array[Group]
end
```

Associations are now clearly represented as attributes of a defined type. 

ORM Proposal for Ruby Domain Objects (RORM)
===========================================

No matter with the domain objects were built as plain ruby objects or with virtus. 
The ORM design proposed here should work both ways. 
The design makes use of Repository patterns [http://martinfowler.com/eaaCatalog/repository.html] to separate persistence from domain logic.

Usage Examples:

* Finding Objects

``` ruby
user = RORM::Repository.Users.where(:name => "Jones")
user_name = Repository.Users
.where(:email => "jones@att.com")
.order_by(:email)
.select(:name).first
```

* Save 

``` ruby
user = User.new( :name = "Jones", :email = "jones@corp.com")
RORM::Repository.save(user)
```

* Update

``` ruby
user = User.find(23)
user.name = "James"
RORM::Repository.save(user)
```

* Remove

``` ruby
RORM::Repository.delete(user)
```

Mapping Strategy Options
========================

Mapping strategy can be different depending on the situation and the software engineer's objectives. 
A good ORM could support at least two of the most used strategies listed below.

* Hash-based Mapping

Domain objects are mapped to the database by using a hash that describes their representation in the database. 
For relational databases, it means tables and columns and for document databases it means collections and properties.

Example for User mapping to a relational database:

``` ruby
class UserMap < RORM::Mapper
  def map	
  {			
    :table => "User_Table",
    :fields => [
	    { :name => "name", :column => "name", :type => String },
	    { :name => "email", :column => "email", :type => String },	
    ],
    :components=> [
	    { 
        :name => "address", 
        fields => [
          { :name => "number", :column => "address_number", :type => String },
          { :name => "street", :column => "address_street", :type => String },
          { :name => "city", :column => "address_city", :type => String },
          { :name => "country", :column => "address_country", :type => String }
        ]
			}
    ],
    :many-to-one-associations => [
	    { :name => "status", column => "status_id", type=> Status }
    ],
    :one-to-many-associations => [
	    { :name => "tasks", :type=> Task, :reverse_column => "task_id"  }
    ],
    :many-to-many-associations => [
      { :name => "groups", :type=> "Group", :table=> "User_Group",      
        :origin_column=>"user_id", :target_column=>"group_id" }
    ]
  }
  end
end
``` 

Although very verbose, the advantage of hash mapping is flexibility what is important to offer database 
modeling freedom and also work with legacy databases.
The mapping strategy describe above could work for both Virtus domain objects as well as plain ruby objects 
since the missing type information is included in the hash.

* Persistence Convention Mapping

This strategy consists of analysing the domain objects attributes so that it can automatically (re)generate 
the database and the object-relational mapping by using a mapping convention that can be customized. 
Thus it does not need any kind of manual mapping from the developer.
However it depends on the attribute type metadata to be described in domain objects to generate a database 
and mappings, as a consequence this approach can not work in plain ruby objects but it can work with Virtus 
domain objects and take advantage over the type information for each attribute.
It is still important to tell the persistence infrastructure which classes are to be persisted automatically.

``` ruby
RORM::Repository.automatic_mapper(User)
RORM::Repository.automatic_mapper(Item)
RORM::Repository.automatic_mapper(Product)
```

* Mixed Mapping Strategies

The ORM should allow both strategies above to coexist so that the developer always have two options during 
the domain object persistence construction.