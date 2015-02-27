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
  attr_accessor :birth_year
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

ORM Proposal for Ruby Domain Objects (RORM)
===========================================

The ORM design proposed here should work with plain ruby objects even if they do not contain enough metadata.
The design makes use of Repository patterns [http://martinfowler.com/eaaCatalog/repository.html] to separate 
persistence from domain logic.

Usage Examples:

* Finding Objects

``` ruby
users = RORM::Repository.Users.where(:name => "Jones")

user_name = RORM::Repository.Users
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
user = RORM::Repository.Users.find(23)
user.name = "James"
RORM::Repository.save(user)
```

* Remove

``` ruby
RORM::Repository.delete(user)
```

Mapping Strategy Options
========================

* Hash-based Mapping

Domain objects are mapped to the database by using a hash that describes their representation in the database. 
For relational databases, it means tables and columns and for document databases it means collections and properties.

Example for User mapping to a relational database:

``` ruby
class UserMapper < RORM:Mapper
  map_class(:User).to_table(:User)
  map_string_prop(:name).to_column(:name)
  map_string_prop(:email).to_column(:email)
  map_integer_prop(:birth_year).to_column(:birth_year)
  map_component(:address) do
    map_integer_prop(:number).to_column(:address_number)
	map_string_prop(:street).to_column(:address_street)
	map_string_prop(:city).to_column(:address_city)
	map_string_prop(:country).to_column(:address_country)
  end
  map_many_to_one(:status).to_column(:status_id).return_type(:Status).with(:lazy, true)
  map_one_to_many(:tasks).return_type(Task).inverse_column(:task_id).with(:lazy, true)
  map_many_to_many(:groups).return_type(Group).table(:User_Group).origin_id(:user_id).target_id(:group_id).with(:lazy, true)	
end
``` 

To make it less verbose, the developer can omit some database configurations such as tables and columns if they 
match their corresponding object configurations. Besides, fields are not lazy by default while associations are lazy.

With this idea in mind, User mapping above can be implemented as:

``` ruby
class UserMapper < RORM:Mapper # By convention it will assume a class User and a table User
  map_string_props :name, :email # columns name, email in User table will be assumed
  map_integer_props :birth_year # column birth_year in User table will be assumed
  map_component(:address) do
    map_integer_props(:number)  # column address_number in user table is assumed
	map_string_props(:street, :city, :country) # columns address_street, address_city and address_country are assumed in user table
  end
  map_many_to_one(:status) # status_id column will be assumed in user table as convention with lazy return type Status
  map_one_to_many(:tasks)  # task_id reverse column will be assumed in task class table as convention with lazy return type as collection of Tasks
  map_many_to_many(:groups) # collection of Group objects will be assumed with a association table User_Group with columns user_id and group_id
end
``` 

Although very verbose, the advantage of hash mapping is flexibility what is important to offer database 
modeling freedom and also work with legacy databases.
The mapping strategy describe above could work for both Virtus domain objects as well as plain ruby objects 
since the missing type information is included in the hash.

