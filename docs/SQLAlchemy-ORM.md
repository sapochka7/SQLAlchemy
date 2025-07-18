SQLAlchemy ORM
Here, the Object Relational Mapper is introduced and fully described. If you want to work with higher-level SQL which is constructed automatically for you, as well as automated persistence of Python objects, proceed first to the tutorial.
• ORM Quick Start
o Declare Models
o Create an Engine
o Emit CREATE TABLE DDL
o Create Objects and Persist
o Simple SELECT
o SELECT with JOIN
o Make Changes
o Some Deletes
o Learn the above concepts in depth
• ORM Mapped Class Configuration
o ORM Mapped Class Overview
o Mapping Classes with Declarative
o Integration with dataclasses and attrs
o SQL Expressions as Mapped Attributes
o Changing Attribute Behavior
o Composite Column Types
o Mapping Class Inheritance Hierarchies
o Non-Traditional Mappings
o Configuring a Version Counter
o Class Mapping API
• Relationship Configuration
o Basic Relationship Patterns
o Adjacency List Relationships
o Configuring how Relationship Joins
o Working with Large Collections
o Collection Customization and API Details
o Special Relationship Persistence Patterns
o Using the legacy ‘backref’ relationship parameter
o Relationships API
• ORM Querying Guide
o Writing SELECT statements for ORM Mapped Classes
o Writing SELECT statements for Inheritance Mappings
o ORM-Enabled INSERT, UPDATE, and DELETE statements
o Column Loading Options
o Relationship Loading Techniques
o ORM API Features for Querying
o Legacy Query API
• Using the Session
o Session Basics
o State Management
o Cascades
o Transactions and Connection Management
o Additional Persistence Techniques
o Contextual/Thread-local Sessions
o Tracking queries, object and Session Changes with Events
o Session API
• Events and Internals
o ORM Events
o ORM Internals
o ORM Exceptions
• ORM Extensions
o Asynchronous I/O (asyncio)
o Association Proxy
o Automap
o Baked Queries
o Declarative Extensions
o Mypy / Pep-484 Support for ORM Mappings
o Mutation Tracking
o Ordering List
o Horizontal Sharding
o Hybrid Attributes
o Indexable
o Alternate Class Instrumentation
• ORM Examples
o Mapping Recipes
o Inheritance Mapping Recipes
o Special APIs
o Extending the ORM


ORM Quick Start
For new users who want to quickly see what basic ORM use looks like, here’s an abbreviated form of the mappings and examples used in the SQLAlchemy Unified Tutorial. The code here is fully runnable from a clean command line.
As the descriptions in this section are intentionally very short, please proceed to the full SQLAlchemy Unified Tutorial for a much more in-depth description of each of the concepts being illustrated here.
Changed in version 2.0: The ORM Quickstart is updated for the latest PEP 484-aware features using new constructs including mapped_column(). See the section ORM Declarative Models for migration information.
Declare Models
Here, we define module-level constructs that will form the structures which we will be querying from the database. This structure, known as a Declarative Mapping, defines at once both a Python object model, as well as database metadata that describes real SQL tables that exist, or will exist, in a particular database:
 from typing import List
 from typing import Optional
 from sqlalchemy import ForeignKey
 from sqlalchemy import String
 from sqlalchemy.orm import DeclarativeBase
 from sqlalchemy.orm import Mapped
 from sqlalchemy.orm import mapped_column
 from sqlalchemy.orm import relationship

 class Base(DeclarativeBase):
    pass

 class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]

    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

 class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))

    user: Mapped["User"] = relationship(back_populates="addresses")

    def __repr__(self) -> str:
        return f"Address(id={self.id!r}, email_address={self.email_address!r})"
The mapping starts with a base class, which above is called Base, and is created by making a simple subclass against the DeclarativeBase class.
Individual mapped classes are then created by making subclasses of Base. A mapped class typically refers to a single particular database table, the name of which is indicated by using the __tablename__ class-level attribute.
Next, columns that are part of the table are declared, by adding attributes that include a special typing annotation called Mapped. The name of each attribute corresponds to the column that is to be part of the database table. The datatype of each column is taken first from the Python datatype that’s associated with each Mapped annotation; int for INTEGER, str for VARCHAR, etc. Nullability derives from whether or not the Optional[] (or its equivalent) type modifier is used. More specific typing information may be indicated using SQLAlchemy type objects in the right side mapped_column() directive, such as the String datatype used above in the User.name column. The association between Python types and SQL types can be customized using the type annotation map.
The mapped_column() directive is used for all column-based attributes that require more specific customization. Besides typing information, this directive accepts a wide variety of arguments that indicate specific details about a database column, including server defaults and constraint information, such as membership within the primary key and foreign keys. The mapped_column() directive accepts a superset of arguments that are accepted by the SQLAlchemy Column class, which is used by SQLAlchemy Core to represent database columns.
All ORM mapped classes require at least one column be declared as part of the primary key, typically by using the Column.primary_key parameter on those mapped_column() objects that should be part of the key. In the above example, the User.id and Address.id columns are marked as primary key.
Taken together, the combination of a string table name as well as a list of column declarations is known in SQLAlchemy as table metadata. Setting up table metadata using both Core and ORM approaches is introduced in the SQLAlchemy Unified Tutorial at Working with Database Metadata. The above mapping is an example of what’s known as Annotated Declarative Table configuration.
Other variants of Mapped are available, most commonly the relationship() construct indicated above. In contrast to the column-based attributes, relationship() denotes a linkage between two ORM classes. In the above example, User.addresses links User to Address, and Address.user links Address to User. The relationship() construct is introduced in the SQLAlchemy Unified Tutorial at Working with ORM Related Objects.
Finally, the above example classes include a __repr__() method, which is not required but is useful for debugging. Mapped classes can be created with methods such as __repr__() generated automatically, using dataclasses. More on dataclass mapping at Declarative Dataclass Mapping.
Create an Engine
The Engine is a factory that can create new database connections for us, which also holds onto connections inside of a Connection Pool for fast reuse. For learning purposes, we normally use a SQLite memory-only database for convenience:
 from sqlalchemy import create_engine
 engine = create_engine("sqlite://", echo=True)
Tip
The echo=True parameter indicates that SQL emitted by connections will be logged to standard out.
A full intro to the Engine starts at Establishing Connectivity - the Engine.
Emit CREATE TABLE DDL
Using our table metadata and our engine, we can generate our schema at once in our target SQLite database, using a method called MetaData.create_all():
 Base.metadata.create_all(engine)
BEGIN (implicit)
PRAGMA main.table_info("user_account")

PRAGMA main.table_info("address")

CREATE TABLE user_account (
    id INTEGER NOT NULL,
    name VARCHAR(30) NOT NULL,
    fullname VARCHAR,
    PRIMARY KEY (id)
)

CREATE TABLE address (
    id INTEGER NOT NULL,
    email_address VARCHAR NOT NULL,
    user_id INTEGER NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY(user_id) REFERENCES user_account (id)
)

COMMIT
A lot just happened from that bit of Python code we wrote. For a complete overview of what’s going on on with Table metadata, proceed in the Tutorial at Working with Database Metadata.
Create Objects and Persist
We are now ready to insert data in the database. We accomplish this by creating instances of User and Address classes, which have an __init__() method already as established automatically by the declarative mapping process. We then pass them to the database using an object called a Session, which makes use of the Engine to interact with the database. The Session.add_all() method is used here to add multiple objects at once, and the Session.commit() method will be used to flush any pending changes to the database and then commit the current database transaction, which is always in progress whenever the Session is used:
 from sqlalchemy.orm import Session

 with Session(engine) as session:
    spongebob = User(
        name="spongebob",
        fullname="Spongebob Squarepants",
        addresses=[Address(email_address="spongebob@sqlalchemy.org")],
    )
    sandy = User(
        name="sandy",
        fullname="Sandy Cheeks",
        addresses=[
            Address(email_address="sandy@sqlalchemy.org"),
            Address(email_address="sandy@squirrelpower.org"),
        ],
    )
    patrick = User(name="patrick", fullname="Patrick Star")

    session.add_all([spongebob, sandy, patrick])

    session.commit()
BEGIN (implicit)
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[] ('spongebob', 'Spongebob Squarepants')
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[] ('sandy', 'Sandy Cheeks')
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[] ('patrick', 'Patrick Star')
INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
[] ('spongebob@sqlalchemy.org', 1)
INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
[] ('sandy@sqlalchemy.org', 2)
INSERT INTO address (email_address, user_id) VALUES (?, ?) RETURNING id
[] ('sandy@squirrelpower.org', 2)
COMMIT
Tip
It’s recommended that the Session be used in context manager style as above, that is, using the Python with: statement. The Session object represents active database resources so it’s good to make sure it’s closed out when a series of operations are completed. In the next section, we’ll keep a Session opened just for illustration purposes.
Basics on creating a Session are at Executing with an ORM Session and more at Basics of Using a Session.
Then, some varieties of basic persistence operations are introduced at Inserting Rows using the ORM Unit of Work pattern.
Simple SELECT
With some rows in the database, here’s the simplest form of emitting a SELECT statement to load some objects. To create SELECT statements, we use the select() function to create a new Select object, which we then invoke using a Session. The method that is often useful when querying for ORM objects is the Session.scalars() method, which will return a ScalarResult object that will iterate through the ORM objects we’ve selected:
 from sqlalchemy import select

 session = Session(engine)

 stmt = select(User).where(User.name.in_(["spongebob", "sandy"]))

 for user in session.scalars(stmt):
    print(user)
BEGIN (implicit)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.name IN (?, ?)
[] ('spongebob', 'sandy')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
The above query also made use of the Select.where() method to add WHERE criteria, and also used the ColumnOperators.in_() method that’s part of all SQLAlchemy column-like constructs to use the SQL IN operator.
More detail on how to select objects and individual columns is at Selecting ORM Entities and Columns.
SELECT with JOIN
It’s very common to query amongst multiple tables at once, and in SQL the JOIN keyword is the primary way this happens. The Select construct creates joins using the Select.join() method:
 stmt = (
    select(Address)
    .join(Address.user)
    .where(User.name == "sandy")
    .where(Address.email_address == "sandy@sqlalchemy.org")
)
 sandy_address = session.scalars(stmt).one()
SELECT address.id, address.email_address, address.user_id
FROM address JOIN user_account ON user_account.id = address.user_id
WHERE user_account.name = ? AND address.email_address = ?
[] ('sandy', 'sandy@sqlalchemy.org')
 sandy_address
Address(id=2, email_address='sandy@sqlalchemy.org')
The above query illustrates multiple WHERE criteria which are automatically chained together using AND, as well as how to use SQLAlchemy column-like objects to create “equality” comparisons, which uses the overridden Python method ColumnOperators.__eq__() to produce a SQL criteria object.
Some more background on the concepts above are at The WHERE clause and Explicit FROM clauses and JOINs.
Make Changes
The Session object, in conjunction with our ORM-mapped classes User and Address, automatically track changes to the objects as they are made, which result in SQL statements that will be emitted the next time the Session flushes. Below, we change one email address associated with “sandy”, and also add a new email address to “patrick”, after emitting a SELECT to retrieve the row for “patrick”:
 stmt = select(User).where(User.name == "patrick")
 patrick = session.scalars(stmt).one()
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.name = ?
[] ('patrick',)
 patrick.addresses.append(Address(email_address="patrickstar@sqlalchemy.org"))
SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
FROM address
WHERE ? = address.user_id
[] (3,)
 sandy_address.email_address = "sandy_cheeks@sqlalchemy.org"

 session.commit()
UPDATE address SET email_address=? WHERE address.id = ?
[] ('sandy_cheeks@sqlalchemy.org', 2)
INSERT INTO address (email_address, user_id) VALUES (?, ?)
[] ('patrickstar@sqlalchemy.org', 3)
COMMIT
Notice when we accessed patrick.addresses, a SELECT was emitted. This is called a lazy load. Background on different ways to access related items using more or less SQL is introduced at Loader Strategies.
A detailed walkthrough on ORM data manipulation starts at Data Manipulation with the ORM.
Some Deletes
All things must come to an end, as is the case for some of our database rows - here’s a quick demonstration of two different forms of deletion, both of which are important based on the specific use case.
First we will remove one of the Address objects from the “sandy” user. When the Session next flushes, this will result in the row being deleted. This behavior is something that we configured in our mapping called the delete cascade. We can get a handle to the sandy object by primary key using Session.get(), then work with the object:
 sandy = session.get(User, 2)
BEGIN (implicit)
SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
FROM user_account
WHERE user_account.id = ?
[] (2,)
 sandy.addresses.remove(sandy_address)
SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
FROM address
WHERE ? = address.user_id
[] (2,)
The last SELECT above was the lazy load operation proceeding so that the sandy.addresses collection could be loaded, so that we could remove the sandy_address member. There are other ways to go about this series of operations that won’t emit as much SQL.
We can choose to emit the DELETE SQL for what’s set to be changed so far, without committing the transaction, using the Session.flush() method:
 session.flush()
DELETE FROM address WHERE address.id = ?
[] (2,)
Next, we will delete the “patrick” user entirely. For a top-level delete of an object by itself, we use the Session.delete() method; this method doesn’t actually perform the deletion, but sets up the object to be deleted on the next flush. The operation will also cascade to related objects based on the cascade options that we configured, in this case, onto the related Address objects:
 session.delete(patrick)
SELECT user_account.id AS user_account_id, user_account.name AS user_account_name, user_account.fullname AS user_account_fullname
FROM user_account
WHERE user_account.id = ?
[] (3,)
SELECT address.id AS address_id, address.email_address AS address_email_address, address.user_id AS address_user_id
FROM address
WHERE ? = address.user_id
[] (3,)
The Session.delete() method in this particular case emitted two SELECT statements, even though it didn’t emit a DELETE, which might seem surprising. This is because when the method went to inspect the object, it turns out the patrick object was expired, which happened when we last called upon Session.commit(), and the SQL emitted was to re-load the rows from the new transaction. This expiration is optional, and in normal use we will often be turning it off for situations where it doesn’t apply well.
To illustrate the rows being deleted, here’s the commit:
 session.commit()
DELETE FROM address WHERE address.id = ?
[] (4,)
DELETE FROM user_account WHERE user_account.id = ?
[] (3,)
COMMIT
The Tutorial discusses ORM deletion at Deleting ORM Objects using the Unit of Work pattern. Background on object expiration is at Expiring / Refreshing; cascades are discussed in depth at Cascades.
Learn the above concepts in depth
For a new user, the above sections were likely a whirlwind tour. There’s a lot of important concepts in each step above that weren’t covered. With a quick overview of what things look like, it’s recommended to work through the SQLAlchemy Unified Tutorial to gain a solid working knowledge of what’s really going on above. Good luck!


ORM Mapped Class Configuration
Detailed reference for ORM configuration, not including relationships, which are detailed at Relationship Configuration.
For a quick look at a typical ORM configuration, start with ORM Quick Start.
For an introduction to the concept of object relational mapping as implemented in SQLAlchemy, it’s first introduced in the SQLAlchemy Unified Tutorial at Using ORM Declarative Forms to Define Table Metadata.
• ORM Mapped Class Overview
o ORM Mapping Styles
• Declarative Mapping
• Imperative Mapping
o Mapped Class Essential Components
• The class to be mapped
• The table, or other from clause object
• The properties dictionary
• Other mapper configuration parameters
o Mapped Class Behavior
• Default Constructor
• Maintaining Non-Mapped State Across Loads
• Runtime Introspection of Mapped classes, Instances and Mappers
• Inspection of Mapper objects
• Inspection of Mapped Instances
• Mapping Classes with Declarative
o Declarative Mapping Styles
• Using a Declarative Base Class
• Declarative Mapping using a Decorator (no declarative base)
o Table Configuration with Declarative
• Declarative Table with mapped_column()
• ORM Annotated Declarative - Automated Mapping with Type Annotations
• Dataclass features in mapped_column()
• Accessing Table and Metadata
• Declarative Table Configuration
• Explicit Schema Name with Declarative Table
• Setting Load and Persistence Options for Declarative Mapped Columns
• Naming Declarative Mapped Columns Explicitly
• Appending additional columns to an existing Declarative mapped class
• ORM Annotated Declarative - Complete Guide
• mapped_column() derives the datatype and nullability from the Mapped annotation
• Customizing the Type Map
• Union types inside the Type Map
• Support for Type Alias Types (defined by PEP 695) and NewType
• Mapping Multiple Type Configurations to Python Types
• Mapping Whole Column Declarations to Python Types
• Using Python Enum or pep-586 Literal types in the type map
• Declarative with Imperative Table (a.k.a. Hybrid Declarative)
• Alternate Attribute Names for Mapping Table Columns
• Applying Load, Persistence and Mapping Options for Imperative Table Columns
• Mapping Declaratively with Reflected Tables
• Using DeferredReflection
• Using Automap
• Automating Column Naming Schemes from Reflected Tables
• Mapping to an Explicit Set of Primary Key Columns
• Mapping a Subset of Table Columns
o Mapper Configuration with Declarative
• Defining Mapped Properties with Declarative
• Mapper Configuration Options with Declarative
• Constructing mapper arguments dynamically
• Other Declarative Mapping Directives
• __declare_last__()
• __declare_first__()
• metadata
• __abstract__
• __table_cls__
o Composing Mapped Hierarchies with Mixins
• Augmenting the Base
• Mixing in Columns
• Mixing in Relationships
• Mixing in _orm.column_property() and other _orm.MapperProperty classes
• Using Mixins and Base Classes with Mapped Inheritance Patterns
• Using _orm.declared_attr() with inheriting Table and Mapper arguments
• Using _orm.declared_attr() to generate table-specific inheriting columns
• Combining Table/Mapper Arguments from Multiple Mixins
• Creating Indexes and Constraints with Naming Conventions on Mixins
• Integration with dataclasses and attrs
o Declarative Dataclass Mapping
• Class level feature configuration
• Attribute Configuration
• Column Defaults
• Integration with Annotated
• Using mixins and abstract superclasses
• Relationship Configuration
• Using Non-Mapped Dataclass Fields
• Integrating with Alternate Dataclass Providers such as Pydantic
o Applying ORM Mappings to an existing dataclass (legacy dataclass use)
• Mapping pre-existing dataclasses using Declarative With Imperative Table
• Mapping pre-existing dataclasses using Declarative-style fields
• Using Declarative Mixins with pre-existing dataclasses
• Mapping pre-existing dataclasses using Imperative Mapping
o Applying ORM mappings to an existing attrs class
• SQL Expressions as Mapped Attributes
o Using a Hybrid
o Using column_property
• Adding column_property() to an existing Declarative mapped class
• Composing from Column Properties at Mapping Time
• Using Column Deferral with column_property()
o Using a plain descriptor
o Query-time SQL expressions as mapped attributes
• Changing Attribute Behavior
o Simple Validators
• validates()
o Using Custom Datatypes at the Core Level
o Using Descriptors and Hybrids
o Synonyms
• synonym()
o Operator Customization
• Composite Column Types
o Working with Mapped Composite Column Types
o Other mapping forms for composites
• Map columns directly, then pass to composite
• Map columns directly, pass attribute names to composite
• Imperative mapping and imperative table
o Using Legacy Non-Dataclasses
o Tracking In-Place Mutations on Composites
o Redefining Comparison Operations for Composites
o Nesting Composites
o Composite API
• composite()
• Mapping Class Inheritance Hierarchies
o Joined Table Inheritance
• Relationships with Joined Inheritance
• Loading Joined Inheritance Mappings
o Single Table Inheritance
• Resolving Column Conflicts with use_existing_column
• Relationships with Single Table Inheritance
• Building Deeper Hierarchies with polymorphic_abstract
• Loading Single Inheritance Mappings
o Concrete Table Inheritance
• Concrete Polymorphic Loading Configuration
• Abstract Concrete Classes
• Classical and Semi-Classical Concrete Polymorphic Configuration
• Relationships with Concrete Inheritance
• Loading Concrete Inheritance Mappings
• Non-Traditional Mappings
o Mapping a Class against Multiple Tables
o Mapping a Class against Arbitrary Subqueries
o Multiple Mappers for One Class
• Configuring a Version Counter
o Simple Version Counting
o Custom Version Counters / Types
o Server Side Version Counters
o Programmatic or Conditional Version Counters
• Class Mapping API
o registry
• registry.__init__()
• registry.as_declarative_base()
• registry.configure()
• registry.dispose()
• registry.generate_base()
• registry.map_declaratively()
• registry.map_imperatively()
• registry.mapped()
• registry.mapped_as_dataclass()
• registry.mappers
• registry.update_type_annotation_map()
o add_mapped_attribute()
o column_property()
o declarative_base()
o declarative_mixin()
o as_declarative()
o mapped_column()
o declared_attr
• declared_attr.cascading
• declared_attr.directive
o DeclarativeBase
• DeclarativeBase.__mapper__
• DeclarativeBase.__mapper_args__
• DeclarativeBase.__table__
• DeclarativeBase.__table_args__
• DeclarativeBase.__tablename__
• DeclarativeBase.metadata
• DeclarativeBase.registry
o DeclarativeBaseNoMeta
• DeclarativeBaseNoMeta.__mapper__
• DeclarativeBaseNoMeta.__mapper_args__
• DeclarativeBaseNoMeta.__table__
• DeclarativeBaseNoMeta.__table_args__
• DeclarativeBaseNoMeta.__tablename__
• DeclarativeBaseNoMeta.metadata
• DeclarativeBaseNoMeta.registry
o has_inherited_table()
o synonym_for()
o object_mapper()
o class_mapper()
o configure_mappers()
o clear_mappers()
o identity_key()
o polymorphic_union()
o orm_insert_sentinel()
o reconstructor()
o Mapper
• Mapper.__init__()
• Mapper.add_properties()
• Mapper.add_property()
• Mapper.all_orm_descriptors
• Mapper.attrs
• Mapper.base_mapper
• Mapper.c
• Mapper.cascade_iterator()
• Mapper.class_
• Mapper.class_manager
• Mapper.column_attrs
• Mapper.columns
• Mapper.common_parent()
• Mapper.composites
• Mapper.concrete
• Mapper.configured
• Mapper.entity
• Mapper.get_property()
• Mapper.get_property_by_column()
• Mapper.identity_key_from_instance()
• Mapper.identity_key_from_primary_key()
• Mapper.identity_key_from_row()
• Mapper.inherits
• Mapper.is_mapper
• Mapper.is_sibling()
• Mapper.isa()
• Mapper.iterate_properties
• Mapper.local_table
• Mapper.mapped_table
• Mapper.mapper
• Mapper.non_primary
• Mapper.persist_selectable
• Mapper.polymorphic_identity
• Mapper.polymorphic_iterator()
• Mapper.polymorphic_map
• Mapper.polymorphic_on
• Mapper.primary_key
• Mapper.primary_key_from_instance()
• Mapper.primary_mapper()
• Mapper.relationships
• Mapper.selectable
• Mapper.self_and_descendants
• Mapper.single
• Mapper.synonyms
• Mapper.tables
• Mapper.validators
• Mapper.with_polymorphic_mappers
o MappedAsDataclass
o MappedClassProtocol


ORM Mapped Class Overview
Overview of ORM class mapping configuration.
For readers new to the SQLAlchemy ORM and/or new to Python in general, it’s recommended to browse through the ORM Quick Start and preferably to work through the SQLAlchemy Unified Tutorial, where ORM configuration is first introduced at Using ORM Declarative Forms to Define Table Metadata.
ORM Mapping Styles
SQLAlchemy features two distinct styles of mapper configuration, which then feature further sub-options for how they are set up. The variability in mapper styles is present to suit a varied list of developer preferences, including the degree of abstraction of a user-defined class from how it is to be mapped to relational schema tables and columns, what kinds of class hierarchies are in use, including whether or not custom metaclass schemes are present, and finally if there are other class-instrumentation approaches present such as if Python dataclasses are in use simultaneously.
In modern SQLAlchemy, the difference between these styles is mostly superficial; when a particular SQLAlchemy configurational style is used to express the intent to map a class, the internal process of mapping the class proceeds in mostly the same way for each, where the end result is always a user-defined class that has a Mapper configured against a selectable unit, typically represented by a Table object, and the class itself has been instrumented to include behaviors linked to relational operations both at the level of the class as well as on instances of that class. As the process is basically the same in all cases, classes mapped from different styles are always fully interoperable with each other. The protocol MappedClassProtocol can be used to indicate a mapped class when using type checkers such as mypy.
The original mapping API is commonly referred to as “classical” style, whereas the more automated style of mapping is known as “declarative” style. SQLAlchemy now refers to these two mapping styles as imperative mapping and declarative mapping.
Regardless of what style of mapping used, all ORM mappings as of SQLAlchemy 1.4 originate from a single object known as registry, which is a registry of mapped classes. Using this registry, a set of mapper configurations can be finalized as a group, and classes within a particular registry may refer to each other by name within the configurational process.
Changed in version 1.4: Declarative and classical mapping are now referred to as “declarative” and “imperative” mapping, and are unified internally, all originating from the registry construct that represents a collection of related mappings.
Declarative Mapping
The Declarative Mapping is the typical way that mappings are constructed in modern SQLAlchemy. The most common pattern is to first construct a base class using the DeclarativeBase superclass. The resulting base class, when subclassed will apply the declarative mapping process to all subclasses that derive from it, relative to a particular registry that is local to the new base by default. The example below illustrates the use of a declarative base which is then used in a declarative table mapping:
from sqlalchemy import Integer, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


# declarative base class
class Base(DeclarativeBase):
    pass


# an example mapping using the base
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(String(30))
    nickname: Mapped[Optional[str]]
Above, the DeclarativeBase class is used to generate a new base class (within SQLAlchemy’s documentation it’s typically referred to as Base, however can have any desired name) from which new classes to be mapped may inherit from, as above a new mapped class User is constructed.
Changed in version 2.0: The DeclarativeBase superclass supersedes the use of the declarative_base() function and registry.generate_base() methods; the superclass approach integrates with PEP 484 tools without the use of plugins. See ORM Declarative Models for migration notes.
The base class refers to a registry object that maintains a collection of related mapped classes. as well as to a MetaData object that retains a collection of Table objects to which the classes are mapped.
The major Declarative mapping styles are further detailed in the following sections:
• Using a Declarative Base Class - declarative mapping using a base class.
• Declarative Mapping using a Decorator (no declarative base) - declarative mapping using a decorator, rather than a base class.
Within the scope of a Declarative mapped class, there are also two varieties of how the Table metadata may be declared. These include:
• Declarative Table with mapped_column() - table columns are declared inline within the mapped class using the mapped_column() directive (or in legacy form, using the Column object directly). The mapped_column() directive may also be optionally combined with type annotations using the Mapped class which can provide some details about the mapped columns directly. The column directives, in combination with the __tablename__ and optional __table_args__ class level directives will allow the Declarative mapping process to construct a Table object to be mapped.
• Declarative with Imperative Table (a.k.a. Hybrid Declarative) - Instead of specifying table name and attributes separately, an explicitly constructed Table object is associated with a class that is otherwise mapped declaratively. This style of mapping is a hybrid of “declarative” and “imperative” mapping, and applies to techniques such as mapping classes to reflected Table objects, as well as mapping classes to existing Core constructs such as joins and subqueries.
Documentation for Declarative mapping continues at Mapping Classes with Declarative.
Imperative Mapping
An imperative or classical mapping refers to the configuration of a mapped class using the registry.map_imperatively() method, where the target class does not include any declarative class attributes.
Tip
The imperative mapping form is a lesser-used form of mapping that originates from the very first releases of SQLAlchemy in 2006. It’s essentially a means of bypassing the Declarative system to provide a more “barebones” system of mapping, and does not offer modern features such as PEP 484 support. As such, most documentation examples use Declarative forms, and it’s recommended that new users start with Declarative Table configuration.
Changed in version 2.0: The registry.map_imperatively() method is now used to create classical mappings. The sqlalchemy.orm.mapper() standalone function is effectively removed.
In “classical” form, the table metadata is created separately with the Table construct, then associated with the User class via the registry.map_imperatively() method, after establishing a registry instance. Normally, a single instance of registry shared for all mapped classes that are related to each other:
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)


class User:
    pass


mapper_registry.map_imperatively(User, user_table)
Information about mapped attributes, such as relationships to other classes, are provided via the properties dictionary. The example below illustrates a second Table object, mapped to a class called Address, then linked to User via relationship():
address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id)
    },
)

mapper_registry.map_imperatively(Address, address)
Note that classes which are mapped with the Imperative approach are fully interchangeable with those mapped with the Declarative approach. Both systems ultimately create the same configuration, consisting of a Table, user-defined class, linked together with a Mapper object. When we talk about “the behavior of Mapper”, this includes when using the Declarative system as well - it’s still used, just behind the scenes.
Mapped Class Essential Components
With all mapping forms, the mapping of the class can be configured in many ways by passing construction arguments that ultimately become part of the Mapper object via its constructor. The parameters that are delivered to Mapper originate from the given mapping form, including parameters passed to registry.map_imperatively() for an Imperative mapping, or when using the Declarative system, from a combination of the table columns, SQL expressions and relationships being mapped along with that of attributes such as __mapper_args__.
There are four general classes of configuration information that the Mapper class looks for:
The class to be mapped
This is a class that we construct in our application. There are generally no restrictions on the structure of this class. [1] When a Python class is mapped, there can only be one Mapper object for the class. [2]
When mapping with the declarative mapping style, the class to be mapped is either a subclass of the declarative base class, or is handled by a decorator or function such as registry.mapped().
When mapping with the imperative style, the class is passed directly as the map_imperatively.class_ argument.
The table, or other from clause object
In the vast majority of common cases this is an instance of Table. For more advanced use cases, it may also refer to any kind of FromClause object, the most common alternative objects being the Subquery and Join object.
When mapping with the declarative mapping style, the subject table is either generated by the declarative system based on the __tablename__ attribute and the Column objects presented, or it is established via the __table__ attribute. These two styles of configuration are presented at Declarative Table with mapped_column() and Declarative with Imperative Table (a.k.a. Hybrid Declarative).
When mapping with the imperative style, the subject table is passed positionally as the map_imperatively.local_table argument.
In contrast to the “one mapper per class” requirement of a mapped class, the Table or other FromClause object that is the subject of the mapping may be associated with any number of mappings. The Mapper applies modifications directly to the user-defined class, but does not modify the given Table or other FromClause in any way.
The properties dictionary
This is a dictionary of all of the attributes that will be associated with the mapped class. By default, the Mapper generates entries for this dictionary derived from the given Table, in the form of ColumnProperty objects which each refer to an individual Column of the mapped table. The properties dictionary will also contain all the other kinds of MapperProperty objects to be configured, most commonly instances generated by the relationship() construct.
When mapping with the declarative mapping style, the properties dictionary is generated by the declarative system by scanning the class to be mapped for appropriate attributes. See the section Defining Mapped Properties with Declarative for notes on this process.
When mapping with the imperative style, the properties dictionary is passed directly as the properties parameter to registry.map_imperatively(), which will pass it along to the Mapper.properties parameter.
Other mapper configuration parameters
When mapping with the declarative mapping style, additional mapper configuration arguments are configured via the __mapper_args__ class attribute. Examples of use are available at Mapper Configuration Options with Declarative.
When mapping with the imperative style, keyword arguments are passed to the to registry.map_imperatively() method which passes them along to the Mapper class.
The full range of parameters accepted are documented at Mapper.
Mapped Class Behavior
Across all styles of mapping using the registry object, the following behaviors are common:
Default Constructor
The registry applies a default constructor, i.e. __init__ method, to all mapped classes that don’t explicitly have their own __init__ method. The behavior of this method is such that it provides a convenient keyword constructor that will accept as optional keyword arguments all the attributes that are named. E.g.:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str]
An object of type User above will have a constructor which allows User objects to be created as:
u1 = User(name="some name", fullname="some fullname")
Tip
The Declarative Dataclass Mapping feature provides an alternate means of generating a default __init__() method by using Python dataclasses, and allows for a highly configurable constructor form.
Warning
The __init__() method of the class is called only when the object is constructed in Python code, and not when an object is loaded or refreshed from the database. See the next section Maintaining Non-Mapped State Across Loads for a primer on how to invoke special logic when objects are loaded.
A class that includes an explicit __init__() method will maintain that method, and no default constructor will be applied.
To change the default constructor used, a user-defined Python callable may be provided to the registry.constructor parameter which will be used as the default constructor.
The constructor also applies to imperative mappings:
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)


class User:
    pass


mapper_registry.map_imperatively(User, user_table)
The above class, mapped imperatively as described at Imperative Mapping, will also feature the default constructor associated with the registry.
Added in version 1.4: classical mappings now support a standard configuration-level constructor when they are mapped via the registry.map_imperatively() method.
Maintaining Non-Mapped State Across Loads
The __init__() method of the mapped class is invoked when the object is constructed directly in Python code:
u1 = User(name="some name", fullname="some fullname")
However, when an object is loaded using the ORM Session, the __init__() method is not called:
u1 = session.scalars(select(User).where(User.name == "some name")).first()
The reason for this is that when loaded from the database, the operation used to construct the object, in the above example the User, is more analogous to deserialization, such as unpickling, rather than initial construction. The majority of the object’s important state is not being assembled for the first time, it’s being re-loaded from database rows.
Therefore to maintain state within the object that is not part of the data that’s stored to the database, such that this state is present when objects are loaded as well as constructed, there are two general approaches detailed below.
1. Use Python descriptors like @property, rather than state, to dynamically compute attributes as needed.
For simple attributes, this is the simplest approach and the least error prone. For example if an object Point with Point.x and Point.y wanted an attribute with the sum of these attributes:
class Point(Base):
    __tablename__ = "point"
    id: Mapped[int] = mapped_column(primary_key=True)
    x: Mapped[int]
    y: Mapped[int]

    @property
    def x_plus_y(self):
        return self.x + self.y
•  An advantage of using dynamic descriptors is that the value is computed every time, meaning it maintains the correct value as the underlying attributes (x and y in this case) might change.
Other forms of the above pattern include Python standard library cached_property decorator (which is cached, and not re-computed each time), as well as SQLAlchemy’s hybrid_property decorator which allows for attributes that can work for SQL querying as well.
•  Establish state on-load using InstanceEvents.load(), and optionally supplemental methods InstanceEvents.refresh() and InstanceEvents.refresh_flush().
These are event hooks that are invoked whenever the object is loaded from the database, or when it is refreshed after being expired. Typically only the InstanceEvents.load() is needed, since non-mapped local object state is not affected by expiration operations. To revise the Point example above looks like:
from sqlalchemy import event


class Point(Base):
    __tablename__ = "point"
    id: Mapped[int] = mapped_column(primary_key=True)
    x: Mapped[int]
    y: Mapped[int]

    def __init__(self, x, y, **kw):
        super().__init__(x=x, y=y, **kw)
        self.x_plus_y = x + y


@event.listens_for(Point, "load")
def receive_load(target, context):
    target.x_plus_y = target.x + target.y
If using the refresh events as well, the event hooks can be stacked on top of one callable if needed, as:
@event.listens_for(Point, "load")
@event.listens_for(Point, "refresh")
@event.listens_for(Point, "refresh_flush")
def receive_load(target, context, attrs=None):
    target.x_plus_y = target.x + target.y
2. Above, the attrs attribute will be present for the refresh and refresh_flush events and indicate a list of attribute names that are being refreshed.
Runtime Introspection of Mapped classes, Instances and Mappers
A class that is mapped using registry will also feature a few attributes that are common to all mappings:
• The __mapper__ attribute will refer to the Mapper that is associated with the class:
mapper = User.__mapper__
This Mapper is also what’s returned when using the inspect() function against the mapped class:
from sqlalchemy import inspect

mapper = inspect(User)
The __table__ attribute will refer to the Table, or more generically to the FromClause object, to which the class is mapped:
table = User.__table__
This FromClause is also what’s returned when using the Mapper.local_table attribute of the Mapper:
table = inspect(User).local_table
For a single-table inheritance mapping, where the class is a subclass that does not have a table of its own, the Mapper.local_table attribute as well as the .__table__ attribute will be None. To retrieve the “selectable” that is actually selected from during a query for this class, this is available via the Mapper.selectable attribute:
table = inspect(User).selectable
Inspection of Mapper objects
As illustrated in the previous section, the Mapper object is available from any mapped class, regardless of method, using the Runtime Inspection API system. Using the inspect() function, one can acquire the Mapper from a mapped class:
 from sqlalchemy import inspect
 insp = inspect(User)
Detailed information is available including Mapper.columns:
 insp.columns
<sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>
This is a namespace that can be viewed in a list format or via individual names:
 list(insp.columns)
[Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
 insp.columns.name
Column('name', String(length=50), table=<user>)
Other namespaces include Mapper.all_orm_descriptors, which includes all mapped attributes as well as hybrids, association proxies:
 insp.all_orm_descriptors
<sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
 insp.all_orm_descriptors.keys()
['fullname', 'nickname', 'name', 'id']
As well as Mapper.column_attrs:
 list(insp.column_attrs)
[<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
 insp.column_attrs.name
<ColumnProperty at 0x10403fce8; name>
 insp.column_attrs.name.expression
Column('name', String(length=50), table=<user>)
See also
Mapper
Inspection of Mapped Instances
The inspect() function also provides information about instances of a mapped class. When applied to an instance of a mapped class, rather than the class itself, the object returned is known as InstanceState, which will provide links to not only the Mapper in use by the class, but also a detailed interface that provides information on the state of individual attributes within the instance including their current value and how this relates to what their database-loaded value is.
Given an instance of the User class loaded from the database:
 u1 = session.scalars(select(User)).first()
The inspect() function will return to us an InstanceState object:
 insp = inspect(u1)
 insp
<sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>
With this object we can see elements such as the Mapper:
 insp.mapper
<Mapper at 0x7f07e614ef50; User>
The Session to which the object is attached, if any:
 insp.session
<sqlalchemy.orm.session.Session object at 0x7f07e614f160>
Information about the current persistence state for the object:
 insp.persistent
True
 insp.pending
False
Attribute state information such as attributes that have not been loaded or lazy loaded (assume addresses refers to a relationship() on the mapped class to a related class):
 insp.unloaded
{'addresses'}
Information regarding the current in-Python status of attributes, such as attributes that have not been modified since the last flush:
 insp.unmodified
{'nickname', 'name', 'fullname', 'id'}
as well as specific history on modifications to attributes since the last flush:
 insp.attrs.nickname.value
'nickname'
 u1.nickname = "new nickname"
 insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
See also
InstanceState
InstanceState.attrs
AttributeState
[1] 
When running under Python 2, a Python 2 “old style” class is the only kind of class that isn’t compatible. When running code on Python 2, all classes must extend from the Python object class. Under Python 3 this is always the case.
[2] 
There is a legacy feature known as a “non primary mapper”, where additional Mapper objects may be associated with a class that’s already mapped, however they don’t apply instrumentation to the class. This feature is deprecated as of SQLAlchemy 1.3.


Mapping Classes with Declarative
The Declarative mapping style is the primary style of mapping that is used with SQLAlchemy. See the section Declarative Mapping for the top level introduction.
• Declarative Mapping Styles
o Using a Declarative Base Class
o Declarative Mapping using a Decorator (no declarative base)
• Table Configuration with Declarative
o Declarative Table with mapped_column()
• ORM Annotated Declarative - Automated Mapping with Type Annotations
• Dataclass features in mapped_column()
• Accessing Table and Metadata
• Declarative Table Configuration
• Explicit Schema Name with Declarative Table
• Setting Load and Persistence Options for Declarative Mapped Columns
• Naming Declarative Mapped Columns Explicitly
• Appending additional columns to an existing Declarative mapped class
o ORM Annotated Declarative - Complete Guide
• mapped_column() derives the datatype and nullability from the Mapped annotation
• Customizing the Type Map
• Union types inside the Type Map
• Support for Type Alias Types (defined by PEP 695) and NewType
• Mapping Multiple Type Configurations to Python Types
• Mapping Whole Column Declarations to Python Types
• Using Python Enum or pep-586 Literal types in the type map
o Declarative with Imperative Table (a.k.a. Hybrid Declarative)
• Alternate Attribute Names for Mapping Table Columns
• Applying Load, Persistence and Mapping Options for Imperative Table Columns
o Mapping Declaratively with Reflected Tables
• Using DeferredReflection
• Using Automap
• Automating Column Naming Schemes from Reflected Tables
• Mapping to an Explicit Set of Primary Key Columns
• Mapping a Subset of Table Columns
• Mapper Configuration with Declarative
o Defining Mapped Properties with Declarative
o Mapper Configuration Options with Declarative
• Constructing mapper arguments dynamically
o Other Declarative Mapping Directives
• __declare_last__()
• __declare_first__()
• metadata
• __abstract__
• __table_cls__
• Composing Mapped Hierarchies with Mixins
o Augmenting the Base
o Mixing in Columns
o Mixing in Relationships
o Mixing in _orm.column_property() and other _orm.MapperProperty classes
o Using Mixins and Base Classes with Mapped Inheritance Patterns
• Using _orm.declared_attr() with inheriting Table and Mapper arguments
• Using _orm.declared_attr() to generate table-specific inheriting columns
o Combining Table/Mapper Arguments from Multiple Mixins
o Creating Indexes and Constraints with Naming Conventions on Mixins


Declarative Mapping Styles
As introduced at Declarative Mapping, the Declarative Mapping is the typical way that mappings are constructed in modern SQLAlchemy. This section will provide an overview of forms that may be used for Declarative mapper configuration.
Using a Declarative Base Class
The most common approach is to generate a “Declarative Base” class by subclassing the DeclarativeBase superclass:
from sqlalchemy.orm import DeclarativeBase


# declarative base class
class Base(DeclarativeBase):
    pass
The Declarative Base class may also be created given an existing registry by assigning it as a class variable named registry:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import registry

reg = registry()


# declarative base class
class Base(DeclarativeBase):
    registry = reg
Changed in version 2.0: The DeclarativeBase superclass supersedes the use of the declarative_base() function and registry.generate_base() methods; the superclass approach integrates with PEP 484 tools without the use of plugins. See ORM Declarative Models for migration notes.
With the declarative base class, new mapped classes are declared as subclasses of the base:
from datetime import datetime
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")


class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
Above, the Base class serves as a base for new classes that are to be mapped, as above new mapped classes User and Address are constructed.
For each subclass constructed, the body of the class then follows the declarative mapping approach which defines both a Table as well as a Mapper object behind the scenes which comprise a full mapping.
See also
Table Configuration with Declarative - describes how to specify the components of the mapped Table to be generated, including notes and options on the use of the mapped_column() construct and how it interacts with the Mapped annotation type
Mapper Configuration with Declarative - describes all other aspects of ORM mapper configuration within Declarative including relationship() configuration, SQL expressions and Mapper parameters
Declarative Mapping using a Decorator (no declarative base)
As an alternative to using the “declarative base” class is to apply declarative mapping to a class explicitly, using either an imperative technique similar to that of a “classical” mapping, or more succinctly by using a decorator. The registry.mapped() function is a class decorator that can be applied to any Python class with no hierarchy in place. The Python class otherwise is configured in declarative style normally.
The example below sets up the identical mapping as seen in the previous section, using the registry.mapped() decorator rather than using the DeclarativeBase superclass:
from datetime import datetime
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()


@mapper_registry.mapped
class User:
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")


@mapper_registry.mapped
class Address:
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
When using the above style, the mapping of a particular class will only proceed if the decorator is applied to that class directly. For inheritance mappings (described in detail at Mapping Class Inheritance Hierarchies), the decorator should be applied to each subclass that is to be mapped:
from sqlalchemy.orm import registry

mapper_registry = registry()


@mapper_registry.mapped
class Person:
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "person",
    }


@mapper_registry.mapped
class Employee(Person):
    __tablename__ = "employee"

    person_id = mapped_column(ForeignKey("person.person_id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
    }
Both the declarative table and imperative table table configuration styles may be used with either the Declarative Base or decorator styles of Declarative mapping.
The decorator form of mapping is useful when combining a SQLAlchemy declarative mapping with other class instrumentation systems such as dataclasses and attrs, though note that SQLAlchemy 2.0 now features dataclasses integration with Declarative Base classes as well.


Table Configuration with Declarative
As introduced at Declarative Mapping, the Declarative style includes the ability to generate a mapped Table object at the same time, or to accommodate a Table or other FromClause object directly.
The following examples assume a declarative base class as:
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
All of the examples that follow illustrate a class inheriting from the above Base. The decorator style introduced at Declarative Mapping using a Decorator (no declarative base) is fully supported with all the following examples as well, as are legacy forms of Declarative Base including base classes generated by declarative_base().
Declarative Table with mapped_column()
When using Declarative, the body of the class to be mapped in most cases includes an attribute __tablename__ that indicates the string name of a Table that should be generated along with the mapping. The mapped_column() construct, which features additional ORM-specific configuration capabilities not present in the plain Column class, is then used within the class body to indicate columns in the table. The example below illustrates the most basic use of this construct within a Declarative mapping:
from sqlalchemy import Integer, String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    fullname = mapped_column(String)
    nickname = mapped_column(String(30))
Above, mapped_column() constructs are placed inline within the class definition as class level attributes. At the point at which the class is declared, the Declarative mapping process will generate a new Table object against the MetaData collection associated with the Declarative Base; each instance of mapped_column() will then be used to generate a Column object during this process, which will become part of the Table.columns collection of this Table object.
In the above example, Declarative will build a Table construct that is equivalent to the following:
# equivalent Table object produced
user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String()),
    Column("nickname", String(30)),
)
When the User class above is mapped, this Table object can be accessed directly via the __table__ attribute; this is described further at Accessing Table and Metadata.
mapped_column() supersedes the use of Column()
Users of 1.x SQLAlchemy will note the use of the mapped_column() construct, which is new as of the SQLAlchemy 2.0 series. This ORM-specific construct is intended first and foremost to be a drop-in replacement for the use of Column within Declarative mappings only, adding new ORM-specific convenience features such as the ability to establish mapped_column.deferred within the construct, and most importantly to indicate to typing tools such as Mypy and Pylance an accurate representation of how the attribute will behave at runtime at both the class level as well as the instance level. As will be seen in the following sections, it’s also at the forefront of a new annotation-driven configuration style introduced in SQLAlchemy 2.0.
Users of legacy code should be aware that the Column form will always work in Declarative in the same way it always has. The different forms of attribute mapping may also be mixed within a single mapping on an attribute by attribute basis, so migration to the new form can be at any pace. See the section ORM Declarative Models for a step by step guide to migrating a Declarative model to the new form.
The mapped_column() construct accepts all arguments that are accepted by the Column construct, as well as additional ORM-specific arguments. The mapped_column.__name positional parameter, indicating the name of the database column, is typically omitted, as the Declarative process will make use of the attribute name given to the construct and assign this as the name of the column (in the above example, this refers to the names id, name, fullname, nickname). Assigning an alternate mapped_column.__name is valid as well, where the resulting Column will use the given name in SQL and DDL statements, while the User mapped class will continue to allow access to the attribute using the attribute name given, independent of the name given to the column itself (more on this at Naming Declarative Mapped Columns Explicitly).
Tip
The mapped_column() construct is only valid within a Declarative class mapping. When constructing a Table object using Core as well as when using imperative table configuration, the Column construct is still required in order to indicate the presence of a database column.
See also
Mapping Table Columns - contains additional notes on affecting how Mapper interprets incoming Column objects.
ORM Annotated Declarative - Automated Mapping with Type Annotations
The mapped_column() construct in modern Python is normally augmented by the use of PEP 484 Python type annotations, where it is capable of deriving its column-configuration information from type annotations associated with the attribute as declared in the Declarative mapped class. These type annotations, if used, must be present within a special SQLAlchemy type called Mapped, which is a generic type that indicates a specific Python type within it.
Using this technique, the example in the previous section can be written more succinctly as below:
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[str | None]
    nickname: Mapped[str | None] = mapped_column(String(30))
The example above demonstrates that if a class attribute is type-hinted with Mapped but doesn’t have an explicit mapped_column() assigned to it, SQLAlchemy will automatically create one. Furthermore, details like the column’s datatype and whether it can be null (nullability) are inferred from the Mapped annotation. However, you can always explicitly provide these arguments to mapped_column() to override these automatically-derived settings.
For complete details on using the ORM Annotated Declarative system, see ORM Annotated Declarative - Complete Guide later in this chapter.
See also
ORM Annotated Declarative - Complete Guide - complete reference for ORM Annotated Declarative
Dataclass features in mapped_column()
The mapped_column() construct integrates with SQLAlchemy’s “native dataclasses” feature, discussed at Declarative Dataclass Mapping. See that section for current background on additional directives supported by mapped_column().
Accessing Table and Metadata
A declaratively mapped class will always include an attribute called __table__; when the above configuration using __tablename__ is complete, the declarative process makes the Table available via the __table__ attribute:
# access the Table
user_table = User.__table__
The above table is ultimately the same one that corresponds to the Mapper.local_table attribute, which we can see through the runtime inspection system:
from sqlalchemy import inspect

user_table = inspect(User).local_table
The MetaData collection associated with both the declarative registry as well as the base class is frequently necessary in order to run DDL operations such as CREATE, as well as in use with migration tools such as Alembic. This object is available via the .metadata attribute of registry as well as the declarative base class. Below, for a small script we may wish to emit a CREATE for all tables against a SQLite database:
engine = create_engine("sqlite://")

Base.metadata.create_all(engine)
Declarative Table Configuration
When using Declarative Table configuration with the __tablename__ declarative class attribute, additional arguments to be supplied to the Table constructor should be provided using the __table_args__ declarative class attribute.
This attribute accommodates both positional as well as keyword arguments that are normally sent to the Table constructor. The attribute can be specified in one of two forms. One is as a dictionary:
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"mysql_engine": "InnoDB"}
The other, a tuple, where each argument is positional (usually constraints):
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
    )
Keyword arguments can be specified with the above form by specifying the last argument as a dictionary:
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
        {"autoload": True},
    )
A class may also specify the __table_args__ declarative attribute, as well as the __tablename__ attribute, in a dynamic style using the declared_attr() method decorator. See Composing Mapped Hierarchies with Mixins for background.
Explicit Schema Name with Declarative Table
The schema name for a Table as documented at Specifying the Schema Name is applied to an individual Table using the Table.schema argument. When using Declarative tables, this option is passed like any other to the __table_args__ dictionary:
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"schema": "some_schema"}
The schema name can also be applied to all Table objects globally by using the MetaData.schema parameter documented at Specifying a Default Schema Name with MetaData. The MetaData object may be constructed separately and associated with a DeclarativeBase subclass by assigning to the metadata attribute directly:
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

metadata_obj = MetaData(schema="some_schema")


class Base(DeclarativeBase):
    metadata = metadata_obj


class MyClass(Base):
    # will use "some_schema" by default
    __tablename__ = "sometable"
See also
Specifying the Schema Name - in the Describing Databases with MetaData documentation.
Setting Load and Persistence Options for Declarative Mapped Columns
The mapped_column() construct accepts additional ORM-specific arguments that affect how the generated Column is mapped, affecting its load and persistence-time behavior. Options that are commonly used include:
• deferred column loading - The mapped_column.deferred boolean establishes the Column using deferred column loading by default. In the example below, the User.bio column will not be loaded by default, but only when accessed:
• class User(Base):
•     __tablename__ = "user"
• 
•     id: Mapped[int] = mapped_column(primary_key=True)
•     name: Mapped[str]
    bio: Mapped[str] = mapped_column(Text, deferred=True)
•  See also
Limiting which Columns Load with Column Deferral - full description of deferred column loading
•  active history - The mapped_column.active_history ensures that upon change of value for the attribute, the previous value will have been loaded and made part of the AttributeState.history collection when inspecting the history of the attribute. This may incur additional SQL statements:
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    important_identifier: Mapped[str] = mapped_column(active_history=True)
• 
See the docstring for mapped_column() for a list of supported parameters.
See also
Applying Load, Persistence and Mapping Options for Imperative Table Columns - describes using column_property() and deferred() for use with Imperative Table configuration
Naming Declarative Mapped Columns Explicitly
All of the examples thus far feature the mapped_column() construct linked to an ORM mapped attribute, where the Python attribute name given to the mapped_column() is also that of the column as we see in CREATE TABLE statements as well as queries. The name for a column as expressed in SQL may be indicated by passing the string positional argument mapped_column.__name as the first positional argument. In the example below, the User class is mapped with alternate names given to the columns themselves:
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column("user_id", primary_key=True)
    name: Mapped[str] = mapped_column("user_name")
Where above User.id resolves to a column named user_id and User.name resolves to a column named user_name. We may write a select() statement using our Python attribute names and will see the SQL names generated:
 from sqlalchemy import select
 print(select(User.id, User.name).where(User.name == "x"))
SELECT "user".user_id, "user".user_name
FROM "user"
WHERE "user".user_name = :user_name_1
See also
Alternate Attribute Names for Mapping Table Columns - applies to Imperative Table
Appending additional columns to an existing Declarative mapped class
A declarative table configuration allows the addition of new Column objects to an existing mapping after the Table metadata has already been generated.
For a declarative class that is declared using a declarative base class, the underlying metaclass DeclarativeMeta includes a __setattr__() method that will intercept additional mapped_column() or Core Column objects and add them to both the Table using Table.append_column() as well as to the existing Mapper using Mapper.add_property():
MyClass.some_new_column = mapped_column(String)
Using core Column:
MyClass.some_new_column = Column(String)
All arguments are supported including an alternate name, such as MyClass.some_new_column = mapped_column("some_name", String). However, the SQL type must be passed to the mapped_column() or Column object explicitly, as in the above examples where the String type is passed. There’s no capability for the Mapped annotation type to take part in the operation.
Additional Column objects may also be added to a mapping in the specific circumstance of using single table inheritance, where additional columns are present on mapped subclasses that have no Table of their own. This is illustrated in the section Single Table Inheritance.
See also
Adding Relationships to Mapped Classes After Declaration - similar examples for relationship()
Note
Assignment of mapped properties to an already mapped class will only function correctly if the “declarative base” class is used, meaning the user-defined subclass of DeclarativeBase or the dynamically generated class returned by declarative_base() or registry.generate_base(). This “base” class includes a Python metaclass which implements a special __setattr__() method that intercepts these operations.
Runtime assignment of class-mapped attributes to a mapped class will not work if the class is mapped using decorators like registry.mapped() or imperative functions like registry.map_imperatively().
ORM Annotated Declarative - Complete Guide
The mapped_column() construct is capable of deriving its column-configuration information from PEP 484 type annotations associated with the attribute as declared in the Declarative mapped class. These type annotations, if used, must be present within a special SQLAlchemy type called Mapped, which is a generic type that then indicates a specific Python type within it.
Using this technique, the User example from previous sections may be written as below:
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[str | None]
    nickname: Mapped[str | None] = mapped_column(String(30))
Above, when Declarative processes each class attribute, each mapped_column() will derive additional arguments from the corresponding Mapped type annotation on the left side, if present. Additionally, Declarative will generate an empty mapped_column() directive implicitly, whenever a Mapped type annotation is encountered that does not have a value assigned to the attribute (this form is inspired by the similar style used in Python dataclasses); this mapped_column() construct proceeds to derive its configuration from the Mapped annotation present.
mapped_column() derives the datatype and nullability from the Mapped annotation
The two qualities that mapped_column() derives from the Mapped annotation are:
• datatype - the Python type given inside Mapped, as contained within the typing.Optional construct if present, is associated with a TypeEngine subclass such as Integer, String, DateTime, or Uuid, to name a few common types.
The datatype is determined based on a dictionary of Python type to SQLAlchemy datatype. This dictionary is completely customizable, as detailed in the next section Customizing the Type Map. The default type map is implemented as in the code example below:
from typing import Any
from typing import Dict
from typing import Type

import datetime
import decimal
import uuid

from sqlalchemy import types

# default type mapping, deriving the type for mapped_column()
# from a Mapped[] annotation
type_map: Dict[Type[Any], TypeEngine[Any]] = {
    bool: types.Boolean(),
    bytes: types.LargeBinary(),
    datetime.date: types.Date(),
    datetime.datetime: types.DateTime(),
    datetime.time: types.Time(),
    datetime.timedelta: types.Interval(),
    decimal.Decimal: types.Numeric(),
    float: types.Float(),
    int: types.Integer(),
    str: types.String(),
    uuid.UUID: types.Uuid(),
}
•  If the mapped_column() construct indicates an explicit type as passed to the mapped_column.__type argument, then the given Python type is disregarded.
•  nullability - The mapped_column() construct will indicate its Column as NULL or NOT NULL first and foremost by the presence of the mapped_column.nullable parameter, passed either as True or False. Additionally , if the mapped_column.primary_key parameter is present and set to True, that will also imply that the column should be NOT NULL.
In the absence of both of these parameters, the presence of typing.Optional[] (or its equivalent) within the Mapped type annotation will be used to determine nullability, where typing.Optional[] means NULL, and the absence of typing.Optional[] means NOT NULL. If there is no Mapped[] annotation present at all, and there is no mapped_column.nullable or mapped_column.primary_key parameter, then SQLAlchemy’s usual default for Column of NULL is used.
In the example below, the id and data columns will be NOT NULL, and the additional_info column will be NULL:
from typing import Optional

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class SomeClass(Base):
    __tablename__ = "some_table"

    # primary_key=True, therefore will be NOT NULL
    id: Mapped[int] = mapped_column(primary_key=True)

    # not Optional[], therefore will be NOT NULL
    data: Mapped[str]

    # Optional[], therefore will be NULL
    additional_info: Mapped[Optional[str]]
It is also perfectly valid to have a mapped_column() whose nullability is different from what would be implied by the annotation. For example, an ORM mapped attribute may be annotated as allowing None within Python code that works with the object as it is first being created and populated, however the value will ultimately be written to a database column that is NOT NULL. The mapped_column.nullable parameter, when present, will always take precedence:
class SomeClass(Base):
    # 

    # will be String() NOT NULL, but can be None in Python
    data: Mapped[Optional[str]] = mapped_column(nullable=False)
Similarly, a non-None attribute that’s written to a database column that for whatever reason needs to be NULL at the schema level, mapped_column.nullable may be set to True:
class SomeClass(Base):
    # 

    # will be String() NULL, but type checker will not expect
    # the attribute to be None
    data: Mapped[str] = mapped_column(nullable=True)
• 
Customizing the Type Map
The mapping of Python types to SQLAlchemy TypeEngine types described in the previous section defaults to a hardcoded dictionary present in the sqlalchemy.sql.sqltypes module. However, the registry object that coordinates the Declarative mapping process will first consult a local, user defined dictionary of types which may be passed as the registry.type_annotation_map parameter when constructing the registry, which may be associated with the DeclarativeBase superclass when first used.
As an example, if we wish to make use of the BIGINT datatype for int, the TIMESTAMP datatype with timezone=True for datetime.datetime, and then only on Microsoft SQL Server we’d like to use NVARCHAR datatype when Python str is used, the registry and Declarative base could be configured as:
import datetime

from sqlalchemy import BIGINT, NVARCHAR, String, TIMESTAMP
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    type_annotation_map = {
        int: BIGINT,
        datetime.datetime: TIMESTAMP(timezone=True),
        str: String().with_variant(NVARCHAR, "mssql"),
    }


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.datetime]
    status: Mapped[str]
Below illustrates the CREATE TABLE statement generated for the above mapping, first on the Microsoft SQL Server backend, illustrating the NVARCHAR datatype:
 from sqlalchemy.schema import CreateTable
 from sqlalchemy.dialects import mssql, postgresql
 print(CreateTable(SomeClass.__table__).compile(dialect=mssql.dialect()))
CREATE TABLE some_table (
  id BIGINT NOT NULL IDENTITY,
  date TIMESTAMP NOT NULL,
  status NVARCHAR(max) NOT NULL,
  PRIMARY KEY (id)
)
Then on the PostgreSQL backend, illustrating TIMESTAMP WITH TIME ZONE:
 print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE TABLE some_table (
  id BIGSERIAL NOT NULL,
  date TIMESTAMP WITH TIME ZONE NOT NULL,
  status VARCHAR NOT NULL,
  PRIMARY KEY (id)
)
By making use of methods such as TypeEngine.with_variant(), we’re able to build up a type map that’s customized to what we need for different backends, while still being able to use succinct annotation-only mapped_column() configurations. There are two more levels of Python-type configurability available beyond this, described in the next two sections.
Union types inside the Type Map
Changed in version 2.0.37: The features described in this section have been repaired and enhanced to work consistently. Prior to this change, union types were supported in type_annotation_map, however the feature exhibited inconsistent behaviors between union syntaxes as well as in how None was handled. Please ensure SQLAlchemy is up to date before attempting to use the features described in this section.
SQLAlchemy supports mapping union types inside the type_annotation_map to allow mapping database types that can support multiple Python types, such as JSON or JSONB:
from typing import Union, Optional
from sqlalchemy import JSON
from sqlalchemy.dialects import postgresql
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.schema import CreateTable

# new style Union using a pipe operator
json_list = list[int] | list[str]

# old style Union using Union explicitly
json_scalar = Union[float, str, bool]


class Base(DeclarativeBase):
    type_annotation_map = {
        json_list: postgresql.JSONB,
        json_scalar: JSON,
    }


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    list_col: Mapped[list[str] | list[int]]

    # uses JSON
    scalar_col: Mapped[json_scalar]

    # uses JSON and is also nullable=True
    scalar_col_nullable: Mapped[json_scalar | None]

    # these forms all use JSON as well due to the json_scalar entry
    scalar_col_newstyle: Mapped[float | str | bool]
    scalar_col_oldstyle: Mapped[Union[float, str, bool]]
    scalar_col_mixedstyle: Mapped[Optional[float | str | bool]]
The above example maps the union of list[int] and list[str] to the Postgresql JSONB datatype, while naming a union of float, str, bool will match to the JSON datatype. An equivalent union, stated in the Mapped construct, will match into the corresponding entry in the type map.
The matching of a union type is based on the contents of the union regardless of how the individual types are named, and additionally excluding the use of the None type. That is, json_scalar will also match to str | bool | float | None. It will not match to a union that is a subset or superset of this union; that is, str | bool would not match, nor would str | bool | float | int. The individual contents of the union excluding None must be an exact match.
The None value is never significant as far as matching from type_annotation_map to Mapped, however is significant as an indicator for nullability of the Column. When None is present in the union either as it is placed in the Mapped construct. When present in Mapped, it indicates the Column would be nullable, in the absense of more specific indicators. This logic works in the same way as indicating an Optional type as described at mapped_column() derives the datatype and nullability from the Mapped annotation.
The CREATE TABLE statement for the above mapping will look as below:
 print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE TABLE some_table (
    id SERIAL NOT NULL,
    list_col JSONB NOT NULL,
    scalar_col JSON,
    scalar_col_not_null JSON NOT NULL,
    PRIMARY KEY (id)
)
While union types use a “loose” matching approach that matches on any equivalent set of subtypes, Python typing also features a way to create “type aliases” that are treated as distinct types that are non-equivalent to another type that includes the same composition. Integration of these types with type_annotation_map is described in the next section, Support for Type Alias Types (defined by PEP 695) and NewType.
Support for Type Alias Types (defined by PEP 695) and NewType
In contrast to the typing lookup described in Union types inside the Type Map, Python typing also includes two ways to create a composed type in a more formal way, using typing.NewType as well as the type keyword introduced in PEP 695. These types behave differently from ordinary type aliases (i.e. assigning a type to a variable name), and this difference is honored in how SQLAlchemy resolves these types from the type map.
Changed in version 2.0.37: The behaviors described in this section for typing.NewType as well as PEP 695 type have been formalized and corrected. Deprecation warnings are now emitted for “loose matching” patterns that have worked in some 2.0 releases, but are to be removed in SQLAlchemy 2.1. Please ensure SQLAlchemy is up to date before attempting to use the features described in this section.
The typing module allows the creation of “new types” using typing.NewType:
from typing import NewType

nstr30 = NewType("nstr30", str)
nstr50 = NewType("nstr50", str)
Additionally, in Python 3.12, a new feature defined by PEP 695 was introduced which provides the type keyword to accomplish a similar task; using type produces an object that is similar in many ways to typing.NewType which is internally referred to as typing.TypeAliasType:
type SmallInt = int
type BigInt = int
type JsonScalar = str | float | bool | None
For the purposes of how SQLAlchemy treats these type objects when used for SQL type lookup inside of Mapped, it’s important to note that Python does not consider two equivalent typing.TypeAliasType or typing.NewType objects to be equal:
# two typing.NewType objects are not equal even if they are both str
 nstr50 == nstr30
False

# two TypeAliasType objects are not equal even if they are both int
 SmallInt == BigInt
False

# an equivalent union is not equal to JsonScalar
 JsonScalar == str | float | bool | None
False
This is the opposite behavior from how ordinary unions are compared, and informs the correct behavior for SQLAlchemy’s type_annotation_map. When using typing.NewType or PEP 695 type objects, the type object is expected to be explicit within the type_annotation_map for it to be matched from a Mapped type, where the same object must be stated in order for a match to be made (excluding whether or not the type inside of Mapped also unions on None). This is distinct from the behavior described at Union types inside the Type Map, where a plain Union that is referenced directly will match to other Unions based on the composition, rather than the object identity, of a particular type in type_annotation_map.
In the example below, the composed types for nstr30, nstr50, SmallInt, BigInt, and JsonScalar have no overlap with each other and can be named distinctly within each Mapped construct, and are also all explicit in type_annotation_map. Any of these types may also be unioned with None or declared as Optional[] without affecting the lookup, only deriving column nullability:
from typing import NewType

from sqlalchemy import SmallInteger, BigInteger, JSON, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.schema import CreateTable

nstr30 = NewType("nstr30", str)
nstr50 = NewType("nstr50", str)
type SmallInt = int
type BigInt = int
type JsonScalar = str | float | bool | None


class TABase(DeclarativeBase):
    type_annotation_map = {
        nstr30: String(30),
        nstr50: String(50),
        SmallInt: SmallInteger,
        BigInteger: BigInteger,
        JsonScalar: JSON,
    }


class SomeClass(TABase):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    normal_str: Mapped[str]

    short_str: Mapped[nstr30]
    long_str_nullable: Mapped[nstr50 | None]

    small_int: Mapped[SmallInt]
    big_int: Mapped[BigInteger]
    scalar_col: Mapped[JsonScalar]
a CREATE TABLE for the above mapping will illustrate the different variants of integer and string we’ve configured, and looks like:
 print(CreateTable(SomeClass.__table__))
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    normal_str VARCHAR NOT NULL,
    short_str VARCHAR(30) NOT NULL,
    long_str_nullable VARCHAR(50),
    small_int SMALLINT NOT NULL,
    big_int BIGINT NOT NULL,
    scalar_col JSON,
    PRIMARY KEY (id)
)
Regarding nullability, the JsonScalar type includes None in its definition, which indicates a nullable column. Similarly the long_str_nullable column applies a union of None to nstr50, which matches to the nstr50 type in the type_annotation_map while also applying nullability to the mapped column. The other columns all remain NOT NULL as they are not indicated as optional.
Mapping Multiple Type Configurations to Python Types
As individual Python types may be associated with TypeEngine configurations of any variety by using the registry.type_annotation_map parameter, an additional capability is the ability to associate a single Python type with different variants of a SQL type based on additional type qualifiers. One typical example of this is mapping the Python str datatype to VARCHAR SQL types of different lengths. Another is mapping different varieties of decimal.Decimal to differently sized NUMERIC columns.
Python’s typing system provides a great way to add additional metadata to a Python type which is by using the PEP 593 Annotated generic type, which allows additional information to be bundled along with a Python type. The mapped_column() construct will correctly interpret an Annotated object by identity when resolving it in the registry.type_annotation_map, as in the example below where we declare two variants of String and Numeric:
from decimal import Decimal

from typing_extensions import Annotated

from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

str_30 = Annotated[str, 30]
str_50 = Annotated[str, 50]
num_12_4 = Annotated[Decimal, 12]
num_6_2 = Annotated[Decimal, 6]


class Base(DeclarativeBase):
    registry = registry(
        type_annotation_map={
            str_30: String(30),
            str_50: String(50),
            num_12_4: Numeric(12, 4),
            num_6_2: Numeric(6, 2),
        }
    )
The Python type passed to the Annotated container, in the above example the str and Decimal types, is important only for the benefit of typing tools; as far as the mapped_column() construct is concerned, it will only need perform a lookup of each type object in the registry.type_annotation_map dictionary without actually looking inside of the Annotated object, at least in this particular context. Similarly, the arguments passed to Annotated beyond the underlying Python type itself are also not important, it’s only that at least one argument must be present for the Annotated construct to be valid. We can then use these augmented types directly in our mapping where they will be matched to the more specific type constructions, as in the following example:
class SomeClass(Base):
    __tablename__ = "some_table"

    short_name: Mapped[str_30] = mapped_column(primary_key=True)
    long_name: Mapped[str_50]
    num_value: Mapped[num_12_4]
    short_num_value: Mapped[num_6_2]
a CREATE TABLE for the above mapping will illustrate the different variants of VARCHAR and NUMERIC we’ve configured, and looks like:
 from sqlalchemy.schema import CreateTable
 print(CreateTable(SomeClass.__table__))
CREATE TABLE some_table (
  short_name VARCHAR(30) NOT NULL,
  long_name VARCHAR(50) NOT NULL,
  num_value NUMERIC(12, 4) NOT NULL,
  short_num_value NUMERIC(6, 2) NOT NULL,
  PRIMARY KEY (short_name)
)
While variety in linking Annotated types to different SQL types grants us a wide degree of flexibility, the next section illustrates a second way in which Annotated may be used with Declarative that is even more open ended.
Mapping Whole Column Declarations to Python Types
The previous section illustrated using PEP 593 Annotated type instances as keys within the registry.type_annotation_map dictionary. In this form, the mapped_column() construct does not actually look inside the Annotated object itself, it’s instead used only as a dictionary key. However, Declarative also has the ability to extract an entire pre-established mapped_column() construct from an Annotated object directly. Using this form, we can define not only different varieties of SQL datatypes linked to Python types without using the registry.type_annotation_map dictionary, we can also set up any number of arguments such as nullability, column defaults, and constraints in a reusable fashion.
A set of ORM models will usually have some kind of primary key style that is common to all mapped classes. There also may be common column configurations such as timestamps with defaults and other fields of pre-established sizes and configurations. We can compose these configurations into mapped_column() instances that we then bundle directly into instances of Annotated, which are then re-used in any number of class declarations. Declarative will unpack an Annotated object when provided in this manner, skipping over any other directives that don’t apply to SQLAlchemy and searching only for SQLAlchemy ORM constructs.
The example below illustrates a variety of pre-configured field types used in this way, where we define intpk that represents an Integer primary key column, timestamp that represents a DateTime type which will use CURRENT_TIMESTAMP as a DDL level column default, and required_name which is a String of length 30 that’s NOT NULL:
import datetime

from typing_extensions import Annotated

from sqlalchemy import func
from sqlalchemy import String
from sqlalchemy.orm import mapped_column


intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]
required_name = Annotated[str, mapped_column(String(30), nullable=False)]
The above Annotated objects can then be used directly within Mapped, where the pre-configured mapped_column() constructs will be extracted and copied to a new instance that will be specific to each attribute:
class Base(DeclarativeBase):
    pass


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[timestamp]
CREATE TABLE for our above mapping looks like:
 from sqlalchemy.schema import CreateTable
 print(CreateTable(SomeClass.__table__))
CREATE TABLE some_table (
  id INTEGER NOT NULL,
  name VARCHAR(30) NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
  PRIMARY KEY (id)
)
When using Annotated types in this way, the configuration of the type may also be affected on a per-attribute basis. For the types in the above example that feature explicit use of mapped_column.nullable, we can apply the Optional[] generic modifier to any of our types so that the field is optional or not at the Python level, which will be independent of the NULL / NOT NULL setting that takes place in the database:
from typing_extensions import Annotated

import datetime
from typing import Optional

from sqlalchemy.orm import DeclarativeBase

timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False),
]


class Base(DeclarativeBase):
    pass


class SomeClass(Base):
    # 

    # pep-484 type will be Optional, but column will be
    # NOT NULL
    created_at: Mapped[Optional[timestamp]]
The mapped_column() construct is also reconciled with an explicitly passed mapped_column() construct, whose arguments will take precedence over those of the Annotated construct. Below we add a ForeignKey constraint to our integer primary key and also use an alternate server default for the created_at column:
import datetime

from typing_extensions import Annotated

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.schema import CreateTable

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]


class Base(DeclarativeBase):
    pass


class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[intpk]


class SomeClass(Base):
    __tablename__ = "some_table"

    # add ForeignKey to mapped_column(Integer, primary_key=True)
    id: Mapped[intpk] = mapped_column(ForeignKey("parent.id"))

    # change server default from CURRENT_TIMESTAMP to UTC_TIMESTAMP
    created_at: Mapped[timestamp] = mapped_column(server_default=func.UTC_TIMESTAMP())
The CREATE TABLE statement illustrates these per-attribute settings, adding a FOREIGN KEY constraint as well as substituting UTC_TIMESTAMP for CURRENT_TIMESTAMP:
 from sqlalchemy.schema import CreateTable
 print(CreateTable(SomeClass.__table__))
CREATE TABLE some_table (
  id INTEGER NOT NULL,
  created_at DATETIME DEFAULT UTC_TIMESTAMP() NOT NULL,
  PRIMARY KEY (id),
  FOREIGN KEY(id) REFERENCES parent (id)
)
Note
The feature of mapped_column() just described, where a fully constructed set of column arguments may be indicated using PEP 593 Annotated objects that contain a “template” mapped_column() object to be copied into the attribute, is currently not implemented for other ORM constructs such as relationship() and composite(). While this functionality is in theory possible, for the moment attempting to use Annotated to indicate further arguments for relationship() and similar will raise a NotImplementedError exception at runtime, but may be implemented in future releases.
Using Python Enum or pep-586 Literal types in the type map
Added in version 2.0.0b4: - Added Enum support
Added in version 2.0.1: - Added Literal support
User-defined Python types which derive from the Python built-in enum.Enum as well as the typing.Literal class are automatically linked to the SQLAlchemy Enum datatype when used in an ORM declarative mapping. The example below uses a custom enum.Enum within the Mapped[] constructor:
import enum

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
In the above example, the mapped attribute SomeClass.status will be linked to a Column with the datatype of Enum(Status). We can see this for example in the CREATE TABLE output for the PostgreSQL database:
CREATE TYPE status AS ENUM ('PENDING', 'RECEIVED', 'COMPLETED')

CREATE TABLE some_table (
  id SERIAL NOT NULL,
  status status NOT NULL,
  PRIMARY KEY (id)
)
In a similar way, typing.Literal may be used instead, using a typing.Literal that consists of all strings:
from typing import Literal

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


Status = Literal["pending", "received", "completed"]


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
The entries used in registry.type_annotation_map link the base enum.Enum Python type as well as the typing.Literal type to the SQLAlchemy Enum SQL type, using a special form which indicates to the Enum datatype that it should automatically configure itself against an arbitrary enumerated type. This configuration, which is implicit by default, would be indicated explicitly as:
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum),
        typing.Literal: sqlalchemy.Enum(enum.Enum),
    }
The resolution logic within Declarative is able to resolve subclasses of enum.Enum as well as instances of typing.Literal to match the enum.Enum or typing.Literal entry in the registry.type_annotation_map dictionary. The Enum SQL type then knows how to produce a configured version of itself with the appropriate settings, including default string length. If a typing.Literal that does not consist of only string values is passed, an informative error is raised.
typing.TypeAliasType can also be used to create enums, by assigning them to a typing.Literal of strings:
from typing import Literal

type Status = Literal["on", "off", "unknown"]
Since this is a typing.TypeAliasType, it represents a unique type object, so it must be placed in the type_annotation_map for it to be looked up successfully, keyed to the Enum type as follows:
import enum
import sqlalchemy


class Base(DeclarativeBase):
    type_annotation_map = {Status: sqlalchemy.Enum(enum.Enum)}
Since SQLAlchemy supports mapping different typing.TypeAliasType objects that are otherwise structurally equivalent individually, these must be present in type_annotation_map to avoid ambiguity.
Native Enums and Naming
The Enum.native_enum parameter refers to if the Enum datatype should create a so-called “native” enum, which on MySQL/MariaDB is the ENUM datatype and on PostgreSQL is a new TYPE object created by CREATE TYPE, or a “non-native” enum, which means that VARCHAR will be used to create the datatype. For backends other than MySQL/MariaDB or PostgreSQL, VARCHAR is used in all cases (third party dialects may have their own behaviors).
Because PostgreSQL’s CREATE TYPE requires that there’s an explicit name for the type to be created, special fallback logic exists when working with implicitly generated Enum without specifying an explicit Enum datatype within a mapping:
1. If the Enum is linked to an enum.Enum object, the Enum.native_enum parameter defaults to True and the name of the enum will be taken from the name of the enum.Enum datatype. The PostgreSQL backend will assume CREATE TYPE with this name.
2. If the Enum is linked to a typing.Literal object, the Enum.native_enum parameter defaults to False; no name is generated and VARCHAR is assumed.
To use typing.Literal with a PostgreSQL CREATE TYPE type, an explicit Enum must be used, either within the type map:
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]


class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
Or alternatively within mapped_column():
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]


class Base(DeclarativeBase):
    pass


class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
Altering the Configuration of the Default Enum
In order to modify the fixed configuration of the Enum datatype that’s generated implicitly, specify new entries in the registry.type_annotation_map, indicating additional arguments. For example, to use “non native enumerations” unconditionally, the Enum.native_enum parameter may be set to False for all types:
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
Changed in version 2.0.1: Implemented support for overriding parameters such as Enum.native_enum within the Enum datatype when establishing the registry.type_annotation_map. Previously, this functionality was not working.
To use a specific configuration for a specific enum.Enum subtype, such as setting the string length to 50 when using the example Status datatype:
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase


class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"


class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
By default Enum that are automatically generated are not associated with the MetaData instance used by the Base, so if the metadata defines a schema it will not be automatically associated with the enum. To automatically associate the enum with the schema in the metadata or table they belong to the Enum.inherit_schema can be set:
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
Linking Specific enum.Enum or typing.Literal to other datatypes
The above examples feature the use of an Enum that is automatically configuring itself to the arguments / attributes present on an enum.Enum or typing.Literal type object. For use cases where specific kinds of enum.Enum or typing.Literal should be linked to other types, these specific types may be placed in the type map also. In the example below, an entry for Literal[] that contains non-string types is linked to the JSON datatype:
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]


class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
In the above configuration, the my_literal datatype will resolve to a JSON instance. Other Literal variants will continue to resolve to Enum datatypes.
Declarative with Imperative Table (a.k.a. Hybrid Declarative)
Declarative mappings may also be provided with a pre-existing Table object, or otherwise a Table or other arbitrary FromClause construct (such as a Join or Subquery) that is constructed separately.
This is referred to as a “hybrid declarative” mapping, as the class is mapped using the declarative style for everything involving the mapper configuration, however the mapped Table object is produced separately and passed to the declarative process directly:
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


# construct a Table directly.  The Base.metadata collection is
# usually a good choice for MetaData but any MetaData
# collection may be used.

user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String),
    Column("fullname", String),
    Column("nickname", String),
)


# construct the User class using this table.
class User(Base):
    __table__ = user_table
Above, a Table object is constructed using the approach described at Describing Databases with MetaData. It can then be applied directly to a class that is declaratively mapped. The __tablename__ and __table_args__ declarative class attributes are not used in this form. The above configuration is often more readable as an inline definition:
class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("fullname", String),
        Column("nickname", String),
    )
A natural effect of the above style is that the __table__ attribute is itself defined within the class definition block. As such it may be immediately referenced within subsequent attributes, such as the example below which illustrates referring to the type column in a polymorphic mapper configuration:
class Person(Base):
    __table__ = Table(
        "person",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("type", String(50)),
    )

    __mapper_args__ = {
        "polymorphic_on": __table__.c.type,
        "polymorphic_identity": "person",
    }
The “imperative table” form is also used when a non-Table construct, such as a Join or Subquery object, is to be mapped. An example below:
from sqlalchemy import func, select

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)


class Customer(Base):
    __table__ = customer_select
For background on mapping to non-Table constructs see the sections Mapping a Class against Multiple Tables and Mapping a Class against Arbitrary Subqueries.
The “imperative table” form is of particular use when the class itself is using an alternative form of attribute declaration, such as Python dataclasses. See the section Applying ORM Mappings to an existing dataclass (legacy dataclass use) for detail.
See also
Describing Databases with MetaData
Applying ORM Mappings to an existing dataclass (legacy dataclass use)
Alternate Attribute Names for Mapping Table Columns
The section Naming Declarative Mapped Columns Explicitly illustrated how to use mapped_column() to provide a specific name for the generated Column object separate from the attribute name under which it is mapped.
When using Imperative Table configuration, we already have Column objects present. To map these to alternate names we may assign the Column to the desired attributes directly:
user_table = Table(
    "user",
    Base.metadata,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String),
)


class User(Base):
    __table__ = user_table

    id = user_table.c.user_id
    name = user_table.c.user_name
The User mapping above will refer to the "user_id" and "user_name" columns via the User.id and User.name attributes, in the same way as demonstrated at Naming Declarative Mapped Columns Explicitly.
One caveat to the above mapping is that the direct inline link to Column will not be typed correctly when using PEP 484 typing tools. A strategy to resolve this is to apply the Column objects within the column_property() function; while the Mapper already generates this property object for its internal use automatically, by naming it in the class declaration, typing tools will be able to match the attribute to the Mapped annotation:
from sqlalchemy.orm import column_property
from sqlalchemy.orm import Mapped


class User(Base):
    __table__ = user_table

    id: Mapped[int] = column_property(user_table.c.user_id)
    name: Mapped[str] = column_property(user_table.c.user_name)
See also
Naming Declarative Mapped Columns Explicitly - applies to Declarative Table
Applying Load, Persistence and Mapping Options for Imperative Table Columns
The section Setting Load and Persistence Options for Declarative Mapped Columns reviewed how to set load and persistence options when using the mapped_column() construct with Declarative Table configuration. When using Imperative Table configuration, we already have existing Column objects that are mapped. In order to map these Column objects along with additional parameters that are specific to the ORM mapping, we may use the column_property() and deferred() constructs in order to associate additional parameters with the column. Options include:
• deferred column loading - The deferred() function is shorthand for invoking column_property() with the column_property.deferred parameter set to True; this construct establishes the Column using deferred column loading by default. In the example below, the User.bio column will not be loaded by default, but only when accessed:
• from sqlalchemy.orm import deferred
• 
• user_table = Table(
•     "user",
•     Base.metadata,
•     Column("id", Integer, primary_key=True),
•     Column("name", String),
•     Column("bio", Text),
• )
• 
• 
• class User(Base):
•     __table__ = user_table
• 
    bio = deferred(user_table.c.bio)
• 
See also
Limiting which Columns Load with Column Deferral - full description of deferred column loading
• active history - The column_property.active_history ensures that upon change of value for the attribute, the previous value will have been loaded and made part of the AttributeState.history collection when inspecting the history of the attribute. This may incur additional SQL statements:
• from sqlalchemy.orm import column_property
• 
• user_table = Table(
•     "user",
•     Base.metadata,
•     Column("id", Integer, primary_key=True),
•     Column("important_identifier", String),
• )
• 
• 
• class User(Base):
•     __table__ = user_table
• 
•     important_identifier = column_property(
•         user_table.c.important_identifier, active_history=True
    )
• 
See also
The column_property() construct is also important for cases where classes are mapped to alternative FROM clauses such as joins and selects. More background on these cases is at:
• Mapping a Class against Multiple Tables
• SQL Expressions as Mapped Attributes
For Declarative Table configuration with mapped_column(), most options are available directly; see the section Setting Load and Persistence Options for Declarative Mapped Columns for examples.
Mapping Declaratively with Reflected Tables
There are several patterns available which provide for producing mapped classes against a series of Table objects that were introspected from the database, using the reflection process described at Reflecting Database Objects.
A simple way to map a class to a table reflected from the database is to use a declarative hybrid mapping, passing the Table.autoload_with parameter to the constructor for Table:
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")


class Base(DeclarativeBase):
    pass


class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        autoload_with=engine,
    )
A variant on the above pattern that scales for many tables is to use the MetaData.reflect() method to reflect a full set of Table objects at once, then refer to them from the MetaData:
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")


class Base(DeclarativeBase):
    pass


Base.metadata.reflect(engine)


class MyClass(Base):
    __table__ = Base.metadata.tables["mytable"]
One caveat to the approach of using __table__ is that the mapped classes cannot be declared until the tables have been reflected, which requires the database connectivity source to be present while the application classes are being declared; it’s typical that classes are declared as the modules of an application are being imported, but database connectivity isn’t available until the application starts running code so that it can consume configuration information and create an engine. There are currently two approaches to working around this, described in the next two sections.
Using DeferredReflection
To accommodate the use case of declaring mapped classes where reflection of table metadata can occur afterwards, a simple extension called the DeferredReflection mixin is available, which alters the declarative mapping process to be delayed until a special class-level DeferredReflection.prepare() method is called, which will perform the reflection process against a target database, and will integrate the results with the declarative table mapping process, that is, classes which use the __tablename__ attribute:
from sqlalchemy.ext.declarative import DeferredReflection
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class Reflected(DeferredReflection):
    __abstract__ = True


class Foo(Reflected, Base):
    __tablename__ = "foo"
    bars = relationship("Bar")


class Bar(Reflected, Base):
    __tablename__ = "bar"

    foo_id = mapped_column(Integer, ForeignKey("foo.id"))
Above, we create a mixin class Reflected that will serve as a base for classes in our declarative hierarchy that should become mapped when the Reflected.prepare method is called. The above mapping is not complete until we do so, given an Engine:
engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")
Reflected.prepare(engine)
The purpose of the Reflected class is to define the scope at which classes should be reflectively mapped. The plugin will search among the subclass tree of the target against which .prepare() is called and reflect all tables which are named by declared classes; tables in the target database that are not part of mappings and are not related to the target tables via foreign key constraint will not be reflected.
Using Automap
A more automated solution to mapping against an existing database where table reflection is to be used is to use the Automap extension. This extension will generate entire mapped classes from a database schema, including relationships between classes based on observed foreign key constraints. While it includes hooks for customization, such as hooks that allow custom class naming and relationship naming schemes, automap is oriented towards an expedient zero-configuration style of working. If an application wishes to have a fully explicit model that makes use of table reflection, the DeferredReflection class may be preferable for its less automated approach.
See also
Automap
Automating Column Naming Schemes from Reflected Tables
When using any of the previous reflection techniques, we have the option to change the naming scheme by which columns are mapped. The Column object includes a parameter Column.key which is a string name that determines under what name this Column will be present in the Table.c collection, independently of the SQL name of the column. This key is also used by Mapper as the attribute name under which the Column will be mapped, if not supplied through other means such as that illustrated at Alternate Attribute Names for Mapping Table Columns.
When working with table reflection, we can intercept the parameters that will be used for Column as they are received using the DDLEvents.column_reflect() event and apply whatever changes we need, including the .key attribute but also things like datatypes.
The event hook is most easily associated with the MetaData object that’s in use as illustrated below:
from sqlalchemy import event
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()
With the above event, the reflection of Column objects will be intercepted with our event that adds a new “.key” element, such as in a mapping as below:
class MyClass(Base):
    __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)
The approach also works with both the DeferredReflection base class as well as with the Automap extension. For automap specifically, see the section Intercepting Column Definitions for background.
See also
Mapping Declaratively with Reflected Tables
DDLEvents.column_reflect()
Intercepting Column Definitions - in the Automap documentation
Mapping to an Explicit Set of Primary Key Columns
The Mapper construct in order to successfully map a table always requires that at least one column be identified as the “primary key” for that selectable. This is so that when an ORM object is loaded or persisted, it can be placed in the identity map with an appropriate identity key.
In those cases where a reflected table to be mapped does not include a primary key constraint, as well as in the general case for mapping against arbitrary selectables where primary key columns might not be present, the Mapper.primary_key parameter is provided so that any set of columns may be configured as the “primary key” for the table, as far as ORM mapping is concerned.
Given the following example of an Imperative Table mapping against an existing Table object where the table does not have any declared primary key (as may occur in reflection scenarios), we may map such a table as in the following example:
from sqlalchemy import Column
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase


metadata = MetaData()
group_users = Table(
    "group_users",
    metadata,
    Column("user_id", String(40), nullable=False),
    Column("group_id", String(40), nullable=False),
    UniqueConstraint("user_id", "group_id"),
)


class Base(DeclarativeBase):
    pass


class GroupUsers(Base):
    __table__ = group_users
    __mapper_args__ = {"primary_key": [group_users.c.user_id, group_users.c.group_id]}
Above, the group_users table is an association table of some kind with string columns user_id and group_id, but no primary key is set up; instead, there is only a UniqueConstraint establishing that the two columns represent a unique key. The Mapper does not automatically inspect unique constraints for primary keys; instead, we make use of the Mapper.primary_key parameter, passing a collection of [group_users.c.user_id, group_users.c.group_id], indicating that these two columns should be used in order to construct the identity key for instances of the GroupUsers class.
Mapping a Subset of Table Columns
Sometimes table reflection may provide a Table with many columns that are not important for our needs and may be safely ignored. For such a table that has lots of columns that don’t need to be referenced in the application, the Mapper.include_properties or Mapper.exclude_properties parameters can indicate a subset of columns to be mapped, where other columns from the target Table will not be considered by the ORM in any way. Example:
class User(Base):
    __table__ = user_table
    __mapper_args__ = {"include_properties": ["user_id", "user_name"]}
In the above example, the User class will map to the user_table table, only including the user_id and user_name columns - the rest are not referenced.
Similarly:
class Address(Base):
    __table__ = address_table
    __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}
will map the Address class to the address_table table, including all columns present except street, city, state, and zip.
As indicated in the two examples, columns may be referenced either by string name or by referring to the Column object directly. Referring to the object directly may be useful for explicitness as well as to resolve ambiguities when mapping to multi-table constructs that might have repeated names:
class User(Base):
    __table__ = user_table
    __mapper_args__ = {
        "include_properties": [user_table.c.user_id, user_table.c.user_name]
    }
When columns are not included in a mapping, these columns will not be referenced in any SELECT statements emitted when executing select() or legacy Query objects, nor will there be any mapped attribute on the mapped class which represents the column; assigning an attribute of that name will have no effect beyond that of a normal Python attribute assignment.
However, it is important to note that schema level column defaults WILL still be in effect for those Column objects that include them, even though they may be excluded from the ORM mapping.
“Schema level column defaults” refers to the defaults described at Column INSERT/UPDATE Defaults including those configured by the Column.default, Column.onupdate, Column.server_default and Column.server_onupdate parameters. These constructs continue to have normal effects because in the case of Column.default and Column.onupdate, the Column object is still present on the underlying Table, thus allowing the default functions to take place when the ORM emits an INSERT or UPDATE, and in the case of Column.server_default and Column.server_onupdate, the relational database itself emits these defaults as a server side behavior.


Mapper Configuration with Declarative
The section Mapped Class Essential Components discusses the general configurational elements of a Mapper construct, which is the structure that defines how a particular user defined class is mapped to a database table or other SQL construct. The following sections describe specific details about how the declarative system goes about constructing the Mapper.
Defining Mapped Properties with Declarative
The examples given at Table Configuration with Declarative illustrate mappings against table-bound columns, using the mapped_column() construct. There are several other varieties of ORM mapped constructs that may be configured besides table-bound columns, the most common being the relationship() construct. Other kinds of properties include SQL expressions that are defined using the column_property() construct and multiple-column mappings using the composite() construct.
While an imperative mapping makes use of the properties dictionary to establish all the mapped class attributes, in the declarative mapping, these properties are all specified inline with the class definition, which in the case of a declarative table mapping are inline with the Column objects that will be used to generate a Table object.
Working with the example mapping of User and Address, we may illustrate a declarative table mapping that includes not just mapped_column() objects but also relationships and SQL expressions:
from typing import List
from typing import Optional

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy import Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    firstname: Mapped[str] = mapped_column(String(50))
    lastname: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[str] = column_property(firstname + " " + lastname)

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")


class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]
    address_statistics: Mapped[Optional[str]] = mapped_column(Text, deferred=True)

    user: Mapped["User"] = relationship(back_populates="addresses")
The above declarative table mapping features two tables, each with a relationship() referring to the other, as well as a simple SQL expression mapped by column_property(), and an additional mapped_column() that indicates loading should be on a “deferred” basis as defined by the mapped_column.deferred keyword. More documentation on these particular concepts may be found at Basic Relationship Patterns, Using column_property, and Limiting which Columns Load with Column Deferral.
Properties may be specified with a declarative mapping as above using “hybrid table” style as well; the Column objects that are directly part of a table move into the Table definition but everything else, including composed SQL expressions, would still be inline with the class definition. Constructs that need to refer to a Column directly would reference it in terms of the Table object. To illustrate the above mapping using hybrid table style:
# mapping attributes using declarative with imperative table
# i.e. __table__

from sqlalchemy import Column, ForeignKey, Integer, String, Table, Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import deferred
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("firstname", String(50)),
        Column("lastname", String(50)),
    )

    fullname = column_property(__table__.c.firstname + " " + __table__.c.lastname)

    addresses = relationship("Address", back_populates="user")


class Address(Base):
    __table__ = Table(
        "address",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", ForeignKey("user.id")),
        Column("email_address", String),
        Column("address_statistics", Text),
    )

    address_statistics = deferred(__table__.c.address_statistics)

    user = relationship("User", back_populates="addresses")
Things to note above:
• The address Table contains a column called address_statistics, however we re-map this column under the same attribute name to be under the control of a deferred() construct.
• With both declararative table and hybrid table mappings, when we define a ForeignKey construct, we always name the target table using the table name, and not the mapped class name.
• When we define relationship() constructs, as these constructs create a linkage between two mapped classes where one necessarily is defined before the other, we can refer to the remote class using its string name. This functionality also extends into the area of other arguments specified on the relationship() such as the “primary join” and “order by” arguments. See the section Late-Evaluation of Relationship Arguments for details on this.
Mapper Configuration Options with Declarative
With all mapping forms, the mapping of the class is configured through parameters that become part of the Mapper object. The function which ultimately receives these arguments is the Mapper function, and are delivered to it from one of the front-facing mapping functions defined on the registry object.
For the declarative form of mapping, mapper arguments are specified using the __mapper_args__ declarative class variable, which is a dictionary that is passed as keyword arguments to the Mapper function. Some examples:
Map Specific Primary Key Columns
The example below illustrates Declarative-level settings for the Mapper.primary_key parameter, which establishes particular columns as part of what the ORM should consider to be a primary key for the class, independently of schema-level primary key constraints:
class GroupUsers(Base):
    __tablename__ = "group_users"

    user_id = mapped_column(String(40))
    group_id = mapped_column(String(40))

    __mapper_args__ = {"primary_key": [user_id, group_id]}
See also
Mapping to an Explicit Set of Primary Key Columns - further background on ORM mapping of explicit columns as primary key columns
Version ID Column
The example below illustrates Declarative-level settings for the Mapper.version_id_col and Mapper.version_id_generator parameters, which configure an ORM-maintained version counter that is updated and checked within the unit of work flush process:
from datetime import datetime


class Widget(Base):
    __tablename__ = "widgets"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime, nullable=False)

    __mapper_args__ = {
        "version_id_col": timestamp,
        "version_id_generator": lambda v: datetime.now(),
    }
See also
Configuring a Version Counter - background on the ORM version counter feature
Single Table Inheritance
The example below illustrates Declarative-level settings for the Mapper.polymorphic_on and Mapper.polymorphic_identity parameters, which are used when configuring a single-table inheritance mapping:
class Person(Base):
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = dict(
        polymorphic_on=type,
        polymorphic_identity="person",
    )


class Employee(Person):
    __mapper_args__ = dict(
        polymorphic_identity="employee",
    )
See also
Single Table Inheritance - background on the ORM single table inheritance mapping feature.
Constructing mapper arguments dynamically
The __mapper_args__ dictionary may be generated from a class-bound descriptor method rather than from a fixed dictionary by making use of the declared_attr() construct. This is useful to create arguments for mappers that are programmatically derived from the table configuration or other aspects of the mapped class. A dynamic __mapper_args__ attribute will typically be useful when using a Declarative Mixin or abstract base class.
For example, to omit from the mapping any columns that have a special Column.info value, a mixin can use a __mapper_args__ method that scans for these columns from the cls.__table__ attribute and passes them to the Mapper.exclude_properties collection:
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr


class ExcludeColsWFlag:
    @declared_attr
    def __mapper_args__(cls):
        return {
            "exclude_properties": [
                column.key
                for column in cls.__table__.c
                if column.info.get("exclude", False)
            ]
        }


class Base(DeclarativeBase):
    pass


class SomeClass(ExcludeColsWFlag, Base):
    __tablename__ = "some_table"

    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String)
    not_needed = mapped_column(String, info={"exclude": True})
Above, the ExcludeColsWFlag mixin provides a per-class __mapper_args__ hook that will scan for Column objects that include the key/value 'exclude': True passed to the Column.info parameter, and then add their string “key” name to the Mapper.exclude_properties collection which will prevent the resulting Mapper from considering these columns for any SQL operations.
See also
Composing Mapped Hierarchies with Mixins
Other Declarative Mapping Directives
__declare_last__()
The __declare_last__() hook allows definition of a class level function that is automatically called by the MapperEvents.after_configured() event, which occurs after mappings are assumed to be completed and the ‘configure’ step has finished:
class MyClass(Base):
    @classmethod
    def __declare_last__(cls):
        """ """
        # do something with mappings
__declare_first__()
Like __declare_last__(), but is called at the beginning of mapper configuration via the MapperEvents.before_configured() event:
class MyClass(Base):
    @classmethod
    def __declare_first__(cls):
        """ """
        # do something before mappings are configured
metadata
The MetaData collection normally used to assign a new Table is the registry.metadata attribute associated with the registry object in use. When using a declarative base class such as that produced by the DeclarativeBase superclass, as well as legacy functions such as declarative_base() and registry.generate_base(), this MetaData is also normally present as an attribute named .metadata that’s directly on the base class, and thus also on the mapped class via inheritance. Declarative uses this attribute, when present, in order to determine the target MetaData collection, or if not present, uses the MetaData associated directly with the registry.
This attribute may also be assigned towards in order to affect the MetaData collection to be used on a per-mapped-hierarchy basis for a single base and/or registry. This takes effect whether a declarative base class is used or if the registry.mapped() decorator is used directly, thus allowing patterns such as the metadata-per-abstract base example in the next section, __abstract__. A similar pattern can be illustrated using registry.mapped() as follows:
reg = registry()


class BaseOne:
    metadata = MetaData()


class BaseTwo:
    metadata = MetaData()


@reg.mapped
class ClassOne:
    __tablename__ = "t1"  # will use reg.metadata

    id = mapped_column(Integer, primary_key=True)


@reg.mapped
class ClassTwo(BaseOne):
    __tablename__ = "t1"  # will use BaseOne.metadata

    id = mapped_column(Integer, primary_key=True)


@reg.mapped
class ClassThree(BaseTwo):
    __tablename__ = "t1"  # will use BaseTwo.metadata

    id = mapped_column(Integer, primary_key=True)
See also
__abstract__
__abstract__
__abstract__ causes declarative to skip the production of a table or mapper for the class entirely. A class can be added within a hierarchy in the same way as mixin (see Mixin and Custom Base Classes), allowing subclasses to extend just from the special class:
class SomeAbstractBase(Base):
    __abstract__ = True

    def some_helpful_method(self):
        """ """

    @declared_attr
    def __mapper_args__(cls):
        return {"helpful mapper arguments": True}


class MyMappedClass(SomeAbstractBase):
    pass
One possible use of __abstract__ is to use a distinct MetaData for different bases:
class Base(DeclarativeBase):
    pass


class DefaultBase(Base):
    __abstract__ = True
    metadata = MetaData()


class OtherBase(Base):
    __abstract__ = True
    metadata = MetaData()
Above, classes which inherit from DefaultBase will use one MetaData as the registry of tables, and those which inherit from OtherBase will use a different one. The tables themselves can then be created perhaps within distinct databases:
DefaultBase.metadata.create_all(some_engine)
OtherBase.metadata.create_all(some_other_engine)
See also
Building Deeper Hierarchies with polymorphic_abstract - an alternative form of “abstract” mapped class that is appropriate for inheritance hierarchies.
__table_cls__
Allows the callable / class used to generate a Table to be customized. This is a very open-ended hook that can allow special customizations to a Table that one generates here:
class MyMixin:
    @classmethod
    def __table_cls__(cls, name, metadata_obj, *arg, **kw):
        return Table(f"my_{name}", metadata_obj, *arg, **kw)
The above mixin would cause all Table objects generated to include the prefix "my_", followed by the name normally specified using the __tablename__ attribute.
__table_cls__ also supports the case of returning None, which causes the class to be considered as single-table inheritance vs. its subclass. This may be useful in some customization schemes to determine that single-table inheritance should take place based on the arguments for the table itself, such as, define as single-inheritance if there is no primary key present:
class AutoTable:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__

    @classmethod
    def __table_cls__(cls, *arg, **kw):
        for obj in arg[1:]:
            if (isinstance(obj, Column) and obj.primary_key) or isinstance(
                obj, PrimaryKeyConstraint
            ):
                return Table(*arg, **kw)

        return None


class Person(AutoTable, Base):
    id = mapped_column(Integer, primary_key=True)


class Employee(Person):
    employee_name = mapped_column(String)
The above Employee class would be mapped as single-table inheritance against Person; the employee_name column would be added as a member of the Person table.


Composing Mapped Hierarchies with Mixins
A common need when mapping classes using the Declarative style is to share common functionality, such as particular columns, table or mapper options, naming schemes, or other mapped properties, across many classes. When using declarative mappings, this idiom is supported via the use of mixin classes, as well as via augmenting the declarative base class itself.
Tip
In addition to mixin classes, common column options may also be shared among many classes using PEP 593 Annotated types; see Mapping Multiple Type Configurations to Python Types and Mapping Whole Column Declarations to Python Types for background on these SQLAlchemy 2.0 features.
An example of some commonly mixed-in idioms is below:
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class CommonMixin:
    """define a series of common elements that may be applied to mapped
    classes using this class as a mixin class."""

    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id: Mapped[int] = mapped_column(primary_key=True)


class HasLogRecord:
    """mark classes that have a many-to-one relationship to the
    ``LogRecord`` class."""

    log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self) -> Mapped["LogRecord"]:
        return relationship("LogRecord")


class LogRecord(CommonMixin, Base):
    log_info: Mapped[str]


class MyModel(CommonMixin, HasLogRecord, Base):
    name: Mapped[str]
The above example illustrates a class MyModel which includes two mixins CommonMixin and HasLogRecord in its bases, as well as a supplementary class LogRecord which also includes CommonMixin, demonstrating a variety of constructs that are supported on mixins and base classes, including:
• columns declared using mapped_column(), Mapped or Column are copied from mixins or base classes onto the target class to be mapped; above this is illustrated via the column attributes CommonMixin.id and HasLogRecord.log_record_id.
• Declarative directives such as __table_args__ and __mapper_args__ can be assigned to a mixin or base class, where they will take effect automatically for any classes which inherit from the mixin or base. The above example illustrates this using the __table_args__ and __mapper_args__ attributes.
• All Declarative directives, including all of __tablename__, __table__, __table_args__ and __mapper_args__, may be implemented using user-defined class methods, which are decorated with the declared_attr decorator (specifically the declared_attr.directive sub-member, more on that in a moment). Above, this is illustrated using a def __tablename__(cls) classmethod that generates a Table name dynamically; when applied to the MyModel class, the table name will be generated as "mymodel", and when applied to the LogRecord class, the table name will be generated as "logrecord".
• Other ORM properties such as relationship() can be generated on the target class to be mapped using user-defined class methods also decorated with the declared_attr decorator. Above, this is illustrated by generating a many-to-one relationship() to a mapped object called LogRecord.
The features above may all be demonstrated using a select() example:
 from sqlalchemy import select
 print(select(MyModel).join(MyModel.log_record))
SELECT mymodel.name, mymodel.id, mymodel.log_record_id
FROM mymodel JOIN logrecord ON logrecord.id = mymodel.log_record_id
Tip
The examples of declared_attr will attempt to illustrate the correct PEP 484 annotations for each method example. The use of annotations with declared_attr functions are completely optional, and are not consumed by Declarative; however, these annotations are required in order to pass Mypy --strict type checking.
Additionally, the declared_attr.directive sub-member illustrated above is optional as well, and is only significant for PEP 484 typing tools, as it adjusts for the expected return type when creating methods to override Declarative directives such as __tablename__, __mapper_args__ and __table_args__.
Added in version 2.0: As part of PEP 484 typing support for the SQLAlchemy ORM, added the declared_attr.directive to declared_attr to distinguish between Mapped attributes and Declarative configurational attributes
There’s no fixed convention for the order of mixins and base classes. Normal Python method resolution rules apply, and the above example would work just as well with:
class MyModel(Base, HasLogRecord, CommonMixin):
    name: Mapped[str] = mapped_column()
This works because Base here doesn’t define any of the variables that CommonMixin or HasLogRecord defines, i.e. __tablename__, __table_args__, id, etc. If the Base did define an attribute of the same name, the class placed first in the inherits list would determine which attribute is used on the newly defined class.
Tip
While the above example is using Annotated Declarative Table form based on the Mapped annotation class, mixin classes also work perfectly well with non-annotated and legacy Declarative forms, such as when using Column directly instead of mapped_column().
Changed in version 2.0: For users coming from the 1.4 series of SQLAlchemy who may have been using the mypy plugin, the declarative_mixin() class decorator is no longer needed to mark declarative mixins, assuming the mypy plugin is no longer in use.
Augmenting the Base
In addition to using a pure mixin, most of the techniques in this section can also be applied to the base class directly, for patterns that should apply to all classes derived from a particular base. The example below illustrates some of the previous section’s example in terms of the Base class:
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    """define a series of common elements that may be applied to mapped
    classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id: Mapped[int] = mapped_column(primary_key=True)


class HasLogRecord:
    """mark classes that have a many-to-one relationship to the
    ``LogRecord`` class."""

    log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self) -> Mapped["LogRecord"]:
        return relationship("LogRecord")


class LogRecord(Base):
    log_info: Mapped[str]


class MyModel(HasLogRecord, Base):
    name: Mapped[str]
Where above, MyModel as well as LogRecord, in deriving from Base, will both have their table name derived from their class name, a primary key column named id, as well as the above table and mapper arguments defined by Base.__table_args__ and Base.__mapper_args__.
When using legacy declarative_base() or registry.generate_base(), the declarative_base.cls parameter may be used as follows to generate an equivalent effect, as illustrated in the non-annotated example below:
# legacy declarative_base() use

from sqlalchemy import Integer, String
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base:
    """define a series of common elements that may be applied to mapped
    classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls):
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id = mapped_column(Integer, primary_key=True)


Base = declarative_base(cls=Base)


class HasLogRecord:
    """mark classes that have a many-to-one relationship to the
    ``LogRecord`` class."""

    log_record_id = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self):
        return relationship("LogRecord")


class LogRecord(Base):
    log_info = mapped_column(String)


class MyModel(HasLogRecord, Base):
    name = mapped_column(String)
Mixing in Columns
Columns can be indicated in mixins assuming the Declarative table style of configuration is in use (as opposed to imperative table configuration), so that columns declared on the mixin can then be copied to be part of the Table that the Declarative process generates. All three of the mapped_column(), Mapped, and Column constructs may be declared inline in a declarative mixin:
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime]


class MyModel(TimestampMixin, Base):
    __tablename__ = "test"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
Where above, all declarative classes that include TimestampMixin in their class bases will automatically include a column created_at that applies a timestamp to all row insertions, as well as an updated_at column, which does not include a default for the purposes of the example (if it did, we would use the Column.onupdate parameter which is accepted by mapped_column()). These column constructs are always copied from the originating mixin or base class, so that the same mixin/base class may be applied to any number of target classes which will each have their own column constructs.
All Declarative column forms are supported by mixins, including:
• Annotated attributes - with or without mapped_column() present:
• class TimestampMixin:
•     created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime]
•  •  mapped_column - with or without Mapped present:
class TimestampMixin:
    created_at = mapped_column(default=func.now())
    updated_at: Mapped[datetime] = mapped_column()
•  •  Column - legacy Declarative form:
class TimestampMixin:
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime)
• 
In each of the above forms, Declarative handles the column-based attributes on the mixin class by creating a copy of the construct, which is then applied to the target class.
Changed in version 2.0: The declarative API can now accommodate Column objects as well as mapped_column() constructs of any form when using mixins without the need to use declared_attr(). Previous limitations which prevented columns with ForeignKey elements from being used directly in mixins have been removed.
Mixing in Relationships
Relationships created by relationship() are provided with declarative mixin classes exclusively using the declared_attr approach, eliminating any ambiguity which could arise when copying a relationship and its possibly column-bound contents. Below is an example which combines a foreign key column and a relationship so that two classes Foo and Bar can both be configured to reference a common target class via many-to-one:
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        return relationship("Target")


class Foo(RefTargetMixin, Base):
    __tablename__ = "foo"
    id: Mapped[int] = mapped_column(primary_key=True)


class Bar(RefTargetMixin, Base):
    __tablename__ = "bar"
    id: Mapped[int] = mapped_column(primary_key=True)


class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)
With the above mapping, each of Foo and Bar contain a relationship to Target accessed along the .target attribute:
 from sqlalchemy import select
 print(select(Foo).join(Foo.target))
SELECT foo.id, foo.target_id
FROM foo JOIN target ON target.id = foo.target_id
 print(select(Bar).join(Bar.target))
SELECT bar.id, bar.target_id
FROM bar JOIN target ON target.id = bar.target_id
Special arguments such as relationship.primaryjoin may also be used within mixed-in classmethods, which often need to refer to the class that’s being mapped. For schemes that need to refer to locally mapped columns, in ordinary cases these columns are made available by Declarative as attributes on the mapped class which is passed as the cls argument to the decorated classmethod. Using this feature, we could for example rewrite the RefTargetMixin.target method using an explicit primaryjoin which refers to pending mapped columns on both Target and cls:
class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)


class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        # illustrates explicit 'primaryjoin' argument
        return relationship("Target", primaryjoin=Target.id == cls.target_id)
Mixing in column_property() and other MapperProperty classes
Like relationship(), other MapperProperty subclasses such as column_property() also need to have class-local copies generated when used by mixins, so are also declared within functions that are decorated by declared_attr. Within the function, other ordinary mapped columns that were declared with mapped_column(), Mapped, or Column will be made available from the cls argument so that they may be used to compose new attributes, as in the example below which adds two columns together:
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)


class Something(SomethingMixin, Base):
    __tablename__ = "something"

    id: Mapped[int] = mapped_column(primary_key=True)
Above, we may make use of Something.x_plus_y in a statement where it produces the full expression:
 from sqlalchemy import select
 print(select(Something.x_plus_y))
SELECT something.x + something.y AS anon_1
FROM something
Tip
The declared_attr decorator causes the decorated callable to behave exactly as a classmethod. However, typing tools like Pylance may not be able to recognize this, which can sometimes cause it to complain about access to the cls variable inside the body of the function. To resolve this issue when it occurs, the @classmethod decorator may be combined directly with declared_attr as:
class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    @classmethod
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)
Added in version 2.0: - declared_attr can accommodate a function decorated with @classmethod to help with PEP 484 integration where needed.
Using Mixins and Base Classes with Mapped Inheritance Patterns
When dealing with mapper inheritance patterns as documented at Mapping Class Inheritance Hierarchies, some additional capabilities are present when using declared_attr either with mixin classes, or when augmenting both mapped and un-mapped superclasses in a class hierarchy.
When defining functions decorated by declared_attr on mixins or base classes to be interpreted by subclasses in a mapped inheritance hierarchy, there is an important distinction made between functions that generate the special names used by Declarative such as __tablename__, __mapper_args__ vs. those that may generate ordinary mapped attributes such as mapped_column() and relationship(). Functions that define Declarative directives are invoked for each subclass in a hierarchy, whereas functions that generate mapped attributes are invoked only for the first mapped superclass in a hierarchy.
The rationale for this difference in behavior is based on the fact that mapped properties are already inheritable by classes, such as a particular column on a superclass’ mapped table should not be duplicated to that of a subclass as well, whereas elements that are specific to a particular class or its mapped table are not inheritable, such as the name of the table that is locally mapped.
The difference in behavior between these two use cases is demonstrated in the following two sections.
Using declared_attr() with inheriting Table and Mapper arguments
A common recipe with mixins is to create a def __tablename__(cls) function that generates a name for the mapped Table dynamically.
This recipe can be used to generate table names for an inheriting mapper hierarchy as in the example below which creates a mixin that gives every class a simple table name based on class name. The recipe is illustrated below where a table name is generated for the Person mapped class and the Engineer subclass of Person, but not for the Manager subclass of Person:
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class Tablename:
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
        return cls.__name__.lower()


class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}


class Engineer(Person):
    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}


class Manager(Person):
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
        """override __tablename__ so that Manager is single-inheritance to Person"""

        return None

    __mapper_args__ = {"polymorphic_identity": "manager"}
In the above example, both the Person base class as well as the Engineer class, being subclasses of the Tablename mixin class which generates new table names, will have a generated __tablename__ attribute, which to Declarative indicates that each class should have its own Table generated to which it will be mapped. For the Engineer subclass, the style of inheritance applied is joined table inheritance, as it will be mapped to a table engineer that joins to the base person table. Any other subclasses that inherit from Person will also have this style of inheritance applied by default (and within this particular example, would need to each specify a primary key column; more on that in the next section).
By contrast, the Manager subclass of Person overrides the __tablename__ classmethod to return None. This indicates to Declarative that this class should not have a Table generated, and will instead make use exclusively of the base Table to which Person is mapped. For the Manager subclass, the style of inheritance applied is single table inheritance.
The example above illustrates that Declarative directives like __tablename__ are necessarily applied to each subclass individually, as each mapped class needs to state which Table it will be mapped towards, or if it will map itself to the inheriting superclass’ Table.
If we instead wanted to reverse the default table scheme illustrated above, so that single table inheritance were the default and joined table inheritance could be defined only when a __tablename__ directive were supplied to override it, we can make use of Declarative helpers within the top-most __tablename__() method, in this case a helper called has_inherited_table(). This function will return True if a superclass is already mapped to a Table. We may use this helper within the base-most __tablename__() classmethod so that we may conditionally return None for the table name, if a table is already present, thus indicating single-table inheritance for inheriting subclasses by default:
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import has_inherited_table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class Tablename:
    @declared_attr.directive
    def __tablename__(cls):
        if has_inherited_table(cls):
            return None
        return cls.__name__.lower()


class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}


class Engineer(Person):
    @declared_attr.directive
    def __tablename__(cls):
        """override __tablename__ so that Engineer is joined-inheritance to Person"""

        return cls.__name__.lower()

    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}


class Manager(Person):
    __mapper_args__ = {"polymorphic_identity": "manager"}
Using declared_attr() to generate table-specific inheriting columns
In contrast to how __tablename__ and other special names are handled when used with declared_attr, when we mix in columns and properties (e.g. relationships, column properties, etc.), the function is invoked for the base class only in the hierarchy, unless the declared_attr directive is used in combination with the declared_attr.cascading sub-directive. Below, only the Person class will receive a column called id; the mapping will fail on Engineer, which is not given a primary key:
class HasId:
    id: Mapped[int] = mapped_column(primary_key=True)


class Person(HasId, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}


# this mapping will fail, as there's no primary key
class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
It is usually the case in joined-table inheritance that we want distinctly named columns on each subclass. However in this case, we may want to have an id column on every table, and have them refer to each other via foreign key. We can achieve this as a mixin by using the declared_attr.cascading modifier, which indicates that the function should be invoked for each class in the hierarchy, in almost (see warning below) the same way as it does for __tablename__:
class HasIdMixin:
    @declared_attr.cascading
    def id(cls) -> Mapped[int]:
        if has_inherited_table(cls):
            return mapped_column(ForeignKey("person.id"), primary_key=True)
        else:
            return mapped_column(Integer, primary_key=True)


class Person(HasIdMixin, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}


class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
Warning
The declared_attr.cascading feature currently does not allow for a subclass to override the attribute with a different function or value. This is a current limitation in the mechanics of how @declared_attr is resolved, and a warning is emitted if this condition is detected. This limitation only applies to ORM mapped columns, relationships, and other MapperProperty styles of attribute. It does not apply to Declarative directives such as __tablename__, __mapper_args__, etc., which resolve in a different way internally than that of declared_attr.cascading.
Combining Table/Mapper Arguments from Multiple Mixins
In the case of __table_args__ or __mapper_args__ specified with declarative mixins, you may want to combine some parameters from several mixins with those you wish to define on the class itself. The declared_attr decorator can be used here to create user-defined collation routines that pull from multiple collections:
from sqlalchemy.orm import declared_attr


class MySQLSettings:
    __table_args__ = {"mysql_engine": "InnoDB"}


class MyOtherMixin:
    __table_args__ = {"info": "foo"}


class MyModel(MySQLSettings, MyOtherMixin, Base):
    __tablename__ = "my_model"

    @declared_attr.directive
    def __table_args__(cls):
        args = dict()
        args.update(MySQLSettings.__table_args__)
        args.update(MyOtherMixin.__table_args__)
        return args

    id = mapped_column(Integer, primary_key=True)
Creating Indexes and Constraints with Naming Conventions on Mixins
Using named constraints such as Index, UniqueConstraint, CheckConstraint, where each object is to be unique to a specific table descending from a mixin, requires that an individual instance of each object is created per actual mapped class.
As a simple example, to define a named, potentially multicolumn Index that applies to all tables derived from a mixin, use the “inline” form of Index and establish it as part of __table_args__, using declared_attr to establish __table_args__() as a class method that will be invoked for each subclass:
class MyMixin:
    a = mapped_column(Integer)
    b = mapped_column(Integer)

    @declared_attr.directive
    def __table_args__(cls):
        return (Index(f"test_idx_{cls.__tablename__}", "a", "b"),)


class MyModelA(MyMixin, Base):
    __tablename__ = "table_a"
    id = mapped_column(Integer, primary_key=True)


class MyModelB(MyMixin, Base):
    __tablename__ = "table_b"
    id = mapped_column(Integer, primary_key=True)
The above example would generate two tables "table_a" and "table_b", with indexes "test_idx_table_a" and "test_idx_table_b"
Typically, in modern SQLAlchemy we would use a naming convention, as documented at Configuring Constraint Naming Conventions. While naming conventions take place automatically using the MetaData.naming_convention as new Constraint objects are created, as this convention is applied at object construction time based on the parent Table for a particular Constraint, a distinct Constraint object needs to be created for each inheriting subclass with its own Table, again using declared_attr with __table_args__(), below illustrated using an abstract mapped base:
from uuid import UUID

from sqlalchemy import CheckConstraint
from sqlalchemy import create_engine
from sqlalchemy import MetaData
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

constraint_naming_conventions = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=constraint_naming_conventions)


class MyAbstractBase(Base):
    __abstract__ = True

    @declared_attr.directive
    def __table_args__(cls):
        return (
            UniqueConstraint("uuid"),
            CheckConstraint("x > 0 OR y < 100", name="xy_chk"),
        )

    id: Mapped[int] = mapped_column(primary_key=True)
    uuid: Mapped[UUID]
    x: Mapped[int]
    y: Mapped[int]


class ModelAlpha(MyAbstractBase):
    __tablename__ = "alpha"


class ModelBeta(MyAbstractBase):
    __tablename__ = "beta"
The above mapping will generate DDL that includes table-specific names for all constraints, including primary key, CHECK constraint, unique constraint:
CREATE TABLE alpha (
    id INTEGER NOT NULL,
    uuid CHAR(32) NOT NULL,
    x INTEGER NOT NULL,
    y INTEGER NOT NULL,
    CONSTRAINT pk_alpha PRIMARY KEY (id),
    CONSTRAINT uq_alpha_uuid UNIQUE (uuid),
    CONSTRAINT ck_alpha_xy_chk CHECK (x > 0 OR y < 100)
)


CREATE TABLE beta (
    id INTEGER NOT NULL,
    uuid CHAR(32) NOT NULL,
    x INTEGER NOT NULL,
    y INTEGER NOT NULL,
    CONSTRAINT pk_beta PRIMARY KEY (id),
    CONSTRAINT uq_beta_uuid UNIQUE (uuid),
    CONSTRAINT ck_beta_xy_chk CHECK (x > 0 OR y < 100)
)


Integration with dataclasses and attrs
SQLAlchemy as of version 2.0 features “native dataclass” integration where an Annotated Declarative Table mapping may be turned into a Python dataclass by adding a single mixin or decorator to mapped classes.
Added in version 2.0: Integrated dataclass creation with ORM Declarative classes
There are also patterns available that allow existing dataclasses to be mapped, as well as to map classes instrumented by the attrs third party integration library.
Declarative Dataclass Mapping
SQLAlchemy Annotated Declarative Table mappings may be augmented with an additional mixin class or decorator directive, which will add an additional step to the Declarative process after the mapping is complete that will convert the mapped class in-place into a Python dataclass, before completing the mapping process which applies ORM-specific instrumentation to the class. The most prominent behavioral addition this provides is generation of an __init__() method with fine-grained control over positional and keyword arguments with or without defaults, as well as generation of methods like __repr__() and __eq__().
From a PEP 484 typing perspective, the class is recognized as having Dataclass-specific behaviors, most notably by taking advantage of PEP 681 “Dataclass Transforms”, which allows typing tools to consider the class as though it were explicitly decorated using the @dataclasses.dataclass decorator.
Note
Support for PEP 681 in typing tools as of April 4, 2023 is limited and is currently known to be supported by Pyright as well as Mypy as of version 1.2. Note that Mypy 1.1.1 introduced PEP 681 support but did not correctly accommodate Python descriptors which will lead to errors when using SQLAlchemy’s ORM mapping scheme.
See also
https://peps.python.org/pep-0681/#the-dataclass-transform-decorator - background on how libraries like SQLAlchemy enable PEP 681 support
Dataclass conversion may be added to any Declarative class either by adding the MappedAsDataclass mixin to a DeclarativeBase class hierarchy, or for decorator mapping by using the registry.mapped_as_dataclass() class decorator.
The MappedAsDataclass mixin may be applied either to the Declarative Base class or any superclass, as in the example below:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass


class Base(MappedAsDataclass, DeclarativeBase):
    """subclasses will be converted to dataclasses"""


class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
Or may be applied directly to classes that extend from the Declarative base:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass


class Base(DeclarativeBase):
    pass


class User(MappedAsDataclass, Base):
    """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
When using the decorator form, only the registry.mapped_as_dataclass() decorator is supported:
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry


reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
Class level feature configuration
Support for dataclasses features is partial. Currently supported are the init, repr, eq, order and unsafe_hash features, match_args and kw_only are supported on Python 3.10+. Currently not supported are the frozen and slots features.
When using the mixin class form with MappedAsDataclass, class configuration arguments are passed as class-level parameters:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass


class Base(DeclarativeBase):
    pass


class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
    """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
When using the decorator form with registry.mapped_as_dataclass(), class configuration arguments are passed to the decorator directly:
from sqlalchemy.orm import registry
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


reg = registry()


@reg.mapped_as_dataclass(unsafe_hash=True)
class User:
    """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
For background on dataclass class options, see the dataclasses documentation at @dataclasses.dataclass.
Attribute Configuration
SQLAlchemy native dataclasses differ from normal dataclasses in that attributes to be mapped are described using the Mapped generic annotation container in all cases. Mappings follow the same forms as those documented at Declarative Table with mapped_column(), and all features of mapped_column() and Mapped are supported.
Additionally, ORM attribute configuration constructs including mapped_column(), relationship() and composite() support per-attribute field options, including init, default, default_factory and repr. The names of these arguments is fixed as specified in PEP 681. Functionality is equivalent to dataclasses:
• init, as in mapped_column.init, relationship.init, if False indicates the field should not be part of the __init__() method
• default, as in mapped_column.default, relationship.default indicates a default value for the field as given as a keyword argument in the __init__() method.
• default_factory, as in mapped_column.default_factory, relationship.default_factory, indicates a callable function that will be invoked to generate a new default value for a parameter if not passed explicitly to the __init__() method.
• repr True by default, indicates the field should be part of the generated __repr__() method
Another key difference from dataclasses is that default values for attributes must be configured using the default parameter of the ORM construct, such as mapped_column(default=None). A syntax that resembles dataclass syntax which accepts simple Python values as defaults without using @dataclases.field() is not supported.
As an example using mapped_column(), the mapping below will produce an __init__() method that accepts only the fields name and fullname, where name is required and may be passed positionally, and fullname is optional. The id field, which we expect to be database-generated, is not part of the constructor at all:
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(default=None)


# 'fullname' is optional keyword argument
u1 = User("name")
Column Defaults
In order to accommodate the name overlap of the default argument with the existing Column.default parameter of the Column construct, the mapped_column() construct disambiguates the two names by adding a new parameter mapped_column.insert_default, which will be populated directly into the Column.default parameter of Column, independently of what may be set on mapped_column.default, which is always used for the dataclasses configuration. For example, to configure a datetime column with a Column.default set to the func.utc_timestamp() SQL function, but where the parameter is optional in the constructor:
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.utc_timestamp(), default=None
    )
With the above mapping, an INSERT for a new User object where no parameter for created_at were passed proceeds as:
 with Session(e) as session:
    session.add(User())
    session.commit()
BEGIN (implicit)
INSERT INTO user_account (created_at) VALUES (utc_timestamp())
[generated in 0.00010s] ()
COMMIT
Integration with Annotated
The approach introduced at Mapping Whole Column Declarations to Python Types illustrates how to use PEP 593 Annotated objects to package whole mapped_column() constructs for re-use. While Annotated objects can be combined with the use of dataclasses, dataclass-specific keyword arguments unfortunately cannot be used within the Annotated construct. This includes PEP 681-specific arguments init, default, repr, and default_factory, which must be present in a mapped_column() or similar construct inline with the class attribute.
Changed in version 2.0.14/2.0.22: the Annotated construct when used with an ORM construct like mapped_column() cannot accommodate dataclass field parameters such as init and repr - this use goes against the design of Python dataclasses and is not supported by PEP 681, and therefore is also rejected by the SQLAlchemy ORM at runtime. A deprecation warning is now emitted and the attribute will be ignored.
As an example, the init=False parameter below will be ignored and additionally emit a deprecation warning:
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

# typing tools as well as SQLAlchemy will ignore init=False here
intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"
    id: Mapped[intpk]


# typing error as well as runtime error: Argument missing for parameter "id"
u1 = User()
Instead, mapped_column() must be present on the right side as well with an explicit setting for mapped_column.init; the other arguments can remain within the Annotated construct:
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

intpk = Annotated[int, mapped_column(primary_key=True)]

reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    # init=False and other pep-681 arguments must be inline
    id: Mapped[intpk] = mapped_column(init=False)


u1 = User()
Using mixins and abstract superclasses
Any mixins or base classes that are used in a MappedAsDataclass mapped class which include Mapped attributes must themselves be part of a MappedAsDataclass hierarchy, such as in the example below using a mixin:
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)


class Base(DeclarativeBase, MappedAsDataclass):
    pass


class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
Python type checkers which support PEP 681 will otherwise not consider attributes from non-dataclass mixins to be part of the dataclass.
Deprecated since version 2.0.8: Using mixins and abstract bases within MappedAsDataclass or registry.mapped_as_dataclass() hierarchies which are not themselves dataclasses is deprecated, as these fields are not supported by PEP 681 as belonging to the dataclass. A warning is emitted for this case which will later be an error.
See also
When transforming <cls> to a dataclass, attribute(s) originate from superclass <cls> which is not a dataclass. - background on rationale
Relationship Configuration
The Mapped annotation in combination with relationship() is used in the same way as described at Basic Relationship Patterns. When specifying a collection-based relationship() as an optional keyword argument, the relationship.default_factory parameter must be passed and it must refer to the collection class that’s to be used. Many-to-one and scalar object references may make use of relationship.default if the default value is to be None:
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

reg = registry()


@reg.mapped_as_dataclass
class Parent:
    __tablename__ = "parent"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        default_factory=list, back_populates="parent"
    )


@reg.mapped_as_dataclass
class Child:
    __tablename__ = "child"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
    parent: Mapped["Parent"] = relationship(default=None)
The above mapping will generate an empty list for Parent.children when a new Parent() object is constructed without passing children, and similarly a None value for Child.parent when a new Child() object is constructed without passing parent.
While the relationship.default_factory can be automatically derived from the given collection class of the relationship() itself, this would break compatibility with dataclasses, as the presence of relationship.default_factory or relationship.default is what determines if the parameter is to be required or optional when rendered into the __init__() method.
Using Non-Mapped Dataclass Fields
When using Declarative dataclasses, non-mapped fields may be used on the class as well, which will be part of the dataclass construction process but will not be mapped. Any field that does not use Mapped will be ignored by the mapping process. In the example below, the fields ctrl_one and ctrl_two will be part of the instance-level state of the object, but will not be persisted by the ORM:
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()


@reg.mapped_as_dataclass
class Data:
    __tablename__ = "data"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    status: Mapped[str]

    ctrl_one: Optional[str] = None
    ctrl_two: Optional[str] = None
Instance of Data above can be created as:
d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")
A more real world example might be to make use of the Dataclasses InitVar feature in conjunction with the __post_init__() feature to receive init-only fields that can be used to compose persisted data. In the example below, the User class is declared using id, name and password_hash as mapped features, but makes use of init-only password and repeat_password fields to represent the user creation process (note: to run this example, replace the function your_crypt_function_here() with a third party crypt function, such as bcrypt or argon2-cffi):
from dataclasses import InitVar
from typing import Optional

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()


@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]

    password: InitVar[str]
    repeat_password: InitVar[str]

    password_hash: Mapped[str] = mapped_column(init=False, nullable=False)

    def __post_init__(self, password: str, repeat_password: str):
        if password != repeat_password:
            raise ValueError("passwords do not match")

        self.password_hash = your_crypt_function_here(password)
The above object is created with parameters password and repeat_password, which are consumed up front so that the password_hash variable may be generated:
 u1 = User(name="some_user", password="xyz", repeat_password="xyz")
 u1.password_hash
'$6$9ppc(example crypted string.)'
Changed in version 2.0.0rc1: When using registry.mapped_as_dataclass() or MappedAsDataclass, fields that do not include the Mapped annotation may be included, which will be treated as part of the resulting dataclass but not be mapped, without the need to also indicate the __allow_unmapped__ class attribute. Previous 2.0 beta releases would require this attribute to be explicitly present, even though the purpose of this attribute was only to allow legacy ORM typed mappings to continue to function.
Integrating with Alternate Dataclass Providers such as Pydantic
Warning
The dataclass layer of Pydantic is not fully compatible with SQLAlchemy’s class instrumentation without additional internal changes, and many features such as related collections may not work correctly.
For Pydantic compatibility, please consider the SQLModel ORM which is built with Pydantic on top of SQLAlchemy ORM, which includes special implementation details which explicitly resolve these incompatibilities.
SQLAlchemy’s MappedAsDataclass class and registry.mapped_as_dataclass() method call directly into the Python standard library dataclasses.dataclass class decorator, after the declarative mapping process has been applied to the class. This function call may be swapped out for alternateive dataclasses providers, such as that of Pydantic, using the dataclass_callable parameter accepted by MappedAsDataclass as a class keyword argument as well as by registry.mapped_as_dataclass():
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import registry


class Base(
    MappedAsDataclass,
    DeclarativeBase,
    dataclass_callable=pydantic.dataclasses.dataclass,
):
    pass


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
The above User class will be applied as a dataclass, using Pydantic’s pydantic.dataclasses.dataclasses callable. The process is available both for mapped classes as well as mixins that extend from MappedAsDataclass or which have registry.mapped_as_dataclass() applied directly.
Added in version 2.0.4: Added the dataclass_callable class and method parameters for MappedAsDataclass and registry.mapped_as_dataclass(), and adjusted some of the dataclass internals to accommodate more strict dataclass functions such as that of Pydantic.
Applying ORM Mappings to an existing dataclass (legacy dataclass use)
Legacy Feature
The approaches described here are superseded by the Declarative Dataclass Mapping feature new in the 2.0 series of SQLAlchemy. This newer version of the feature builds upon the dataclass support first added in version 1.4, which is described in this section.
To map an existing dataclass, SQLAlchemy’s “inline” declarative directives cannot be used directly; ORM directives are assigned using one of three techniques:
• Using “Declarative with Imperative Table”, the table / column to be mapped is defined using a Table object assigned to the __table__ attribute of the class; relationships are defined within __mapper_args__ dictionary. The class is mapped using the registry.mapped() decorator. An example is below at Mapping pre-existing dataclasses using Declarative With Imperative Table.
• Using full “Declarative”, the Declarative-interpreted directives such as Column, relationship() are added to the .metadata dictionary of the dataclasses.field() construct, where they are consumed by the declarative process. The class is again mapped using the registry.mapped() decorator. See the example below at Mapping pre-existing dataclasses using Declarative-style fields.
• An “Imperative” mapping can be applied to an existing dataclass using the registry.map_imperatively() method to produce the mapping in exactly the same way as described at Imperative Mapping. This is illustrated below at Mapping pre-existing dataclasses using Imperative Mapping.
The general process by which SQLAlchemy applies mappings to a dataclass is the same as that of an ordinary class, but also includes that SQLAlchemy will detect class-level attributes that were part of the dataclasses declaration process and replace them at runtime with the usual SQLAlchemy ORM mapped attributes. The __init__ method that would have been generated by dataclasses is left intact, as is the same for all the other methods that dataclasses generates such as __eq__(), __repr__(), etc.
Mapping pre-existing dataclasses using Declarative With Imperative Table
An example of a mapping using @dataclass using Declarative with Imperative Table (a.k.a. Hybrid Declarative) is below. A complete Table object is constructed explicitly and assigned to the __table__ attribute. Instance fields are defined using normal dataclass syntaxes. Additional MapperProperty definitions such as relationship(), are placed in the __mapper_args__ class-level dictionary underneath the properties key, corresponding to the Mapper.properties parameter:
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List, Optional

from sqlalchemy import Column, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()


@mapper_registry.mapped
@dataclass
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )
    id: int = field(init=False)
    name: Optional[str] = None
    fullname: Optional[str] = None
    nickname: Optional[str] = None
    addresses: List[Address] = field(default_factory=list)

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }


@mapper_registry.mapped
@dataclass
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: Optional[str] = None
In the above example, the User.id, Address.id, and Address.user_id attributes are defined as field(init=False). This means that parameters for these won’t be added to __init__() methods, but Session will still be able to set them after getting their values during flush from autoincrement or other default value generator. To allow them to be specified in the constructor explicitly, they would instead be given a default value of None.
For a relationship() to be declared separately, it needs to be specified directly within the Mapper.properties dictionary which itself is specified within the __mapper_args__ dictionary, so that it is passed to the constructor for Mapper. An alternative to this approach is in the next example.
Warning
Declaring a dataclass field() setting a default together with init=False will not work as would be expected with a totally plain dataclass, since the SQLAlchemy class instrumentation will replace the default value set on the class by the dataclass creation process. Use default_factory instead. This adaptation is done automatically when making use of Declarative Dataclass Mapping.
Mapping pre-existing dataclasses using Declarative-style fields
Legacy Feature
This approach to Declarative mapping with dataclasses should be considered as legacy. It will remain supported however is unlikely to offer any advantages against the new approach detailed at Declarative Dataclass Mapping.
Note that mapped_column() is not supported with this use; the Column construct should continue to be used to declare table metadata within the metadata field of dataclasses.field().
The fully declarative approach requires that Column objects are declared as class attributes, which when using dataclasses would conflict with the dataclass-level attributes. An approach to combine these together is to make use of the metadata attribute on the dataclass.field object, where SQLAlchemy-specific mapping information may be supplied. Declarative supports extraction of these parameters when the class specifies the attribute __sa_dataclass_metadata_key__. This also provides a more succinct method of indicating the relationship() association:
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List

from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()


@mapper_registry.mapped
@dataclass
class User:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    name: str = field(default=None, metadata={"sa": Column(String(50))})
    fullname: str = field(default=None, metadata={"sa": Column(String(50))})
    nickname: str = field(default=None, metadata={"sa": Column(String(12))})
    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": relationship("Address")}
    )


@mapper_registry.mapped
@dataclass
class Address:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(init=False, metadata={"sa": Column(ForeignKey("user.id"))})
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})
Using Declarative Mixins with pre-existing dataclasses
In the section Composing Mapped Hierarchies with Mixins, Declarative Mixin classes are introduced. One requirement of declarative mixins is that certain constructs that can’t be easily duplicated must be given as callables, using the declared_attr decorator, such as in the example at Mixing in Relationships:
class RefTargetMixin:
    @declared_attr
    def target_id(cls) -> Mapped[int]:
        return mapped_column("target_id", ForeignKey("target.id"))

    @declared_attr
    def target(cls):
        return relationship("Target")
This form is supported within the Dataclasses field() object by using a lambda to indicate the SQLAlchemy construct inside the field(). Using declared_attr() to surround the lambda is optional. If we wanted to produce our User class above where the ORM fields came from a mixin that is itself a dataclass, the form would be:
@dataclass
class UserMixin:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": lambda: relationship("Address")}
    )


@dataclass
class AddressMixin:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(
        init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
    )
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})


@mapper_registry.mapped
class User(UserMixin):
    pass


@mapper_registry.mapped
class Address(AddressMixin):
    pass
Added in version 1.4.2: Added support for “declared attr” style mixin attributes, namely relationship() constructs as well as Column objects with foreign key declarations, to be used within “Dataclasses with Declarative Table” style mappings.
Mapping pre-existing dataclasses using Imperative Mapping
As described previously, a class which is set up as a dataclass using the @dataclass decorator can then be further decorated using the registry.mapped() decorator in order to apply declarative-style mapping to the class. As an alternative to using the registry.mapped() decorator, we may also pass the class through the registry.map_imperatively() method instead, so that we may pass all Table and Mapper configuration imperatively to the function rather than having them defined on the class itself as class variables:
from __future__ import annotations

from dataclasses import dataclass
from dataclasses import field
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()


@dataclass
class User:
    id: int = field(init=False)
    name: str = None
    fullname: str = None
    nickname: str = None
    addresses: List[Address] = field(default_factory=list)


@dataclass
class Address:
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: str = None


metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
The same warning mentioned in Mapping pre-existing dataclasses using Declarative With Imperative Table applies when using this mapping style.
Applying ORM mappings to an existing attrs class
Warning
The attrs library is not part of SQLAlchemy’s continuous integration testing, and compatibility with this library may change without notice due to incompatibilities introduced by either side.
The attrs library is a popular third party library that provides similar features as dataclasses, with many additional features provided not found in ordinary dataclasses.
A class augmented with attrs uses the @define decorator. This decorator initiates a process to scan the class for attributes that define the class’ behavior, which are then used to generate methods, documentation, and annotations.
The SQLAlchemy ORM supports mapping an attrs class using Imperative mapping. The general form of this style is equivalent to the Mapping pre-existing dataclasses using Imperative Mapping mapping form used with dataclasses, where the class construction uses attrs alone, with ORM mappings applied after the fact without any class attribute scanning.
The @define decorator of attrs by default replaces the annotated class with a new __slots__ based class, which is not supported. When using the old style annotation @attr.s or using define(slots=False), the class does not get replaced. Furthermore attrs removes its own class-bound attributes after the decorator runs, so that SQLAlchemy’s mapping process takes over these attributes without any issue. Both decorators, @attr.s and @define(slots=False) work with SQLAlchemy.
Changed in version 2.0: SQLAlchemy integration with attrs works only with imperative mapping style, that is, not using Declarative. The introduction of ORM Annotated Declarative style is not cross-compatible with attrs.
The attrs class is built first. The SQLAlchemy ORM mapping can be applied after the fact using registry.map_imperatively():
from __future__ import annotations

from typing import List

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()


@define(slots=False)
class User:
    id: int
    name: str
    fullname: str
    nickname: str
    addresses: List[Address]


@define(slots=False)
class Address:
    id: int
    user_id: int
    email_address: Optional[str]


metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)


SQL Expressions as Mapped Attributes
Attributes on a mapped class can be linked to SQL expressions, which can be used in queries.
Using a Hybrid
The easiest and most flexible way to link relatively simple SQL expressions to a class is to use a so-called “hybrid attribute”, described in the section Hybrid Attributes. The hybrid provides for an expression that works at both the Python level as well as at the SQL expression level. For example, below we map a class User, containing attributes firstname and lastname, and include a hybrid that will provide for us the fullname, which is the string concatenation of the two:
from sqlalchemy.ext.hybrid import hybrid_property


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        return self.firstname + " " + self.lastname
Above, the fullname attribute is interpreted at both the instance and class level, so that it is available from an instance:
some_user = session.scalars(select(User).limit(1)).first()
print(some_user.fullname)
as well as usable within queries:
some_user = session.scalars(
    select(User).where(User.fullname == "John Smith").limit(1)
).first()
The string concatenation example is a simple one, where the Python expression can be dual purposed at the instance and class level. Often, the SQL expression must be distinguished from the Python expression, which can be achieved using hybrid_property.expression(). Below we illustrate the case where a conditional needs to be present inside the hybrid, using the if statement in Python and the case() construct for SQL expressions:
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.sql import case


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        if self.firstname is not None:
            return self.firstname + " " + self.lastname
        else:
            return self.lastname

    @fullname.expression
    def fullname(cls):
        return case(
            (cls.firstname != None, cls.firstname + " " + cls.lastname),
            else_=cls.lastname,
        )
Using column_property
The column_property() function can be used to map a SQL expression in a manner similar to a regularly mapped Column. With this technique, the attribute is loaded along with all other column-mapped attributes at load time. This is in some cases an advantage over the usage of hybrids, as the value can be loaded up front at the same time as the parent row of the object, particularly if the expression is one which links to other tables (typically as a correlated subquery) to access data that wouldn’t normally be available on an already loaded object.
Disadvantages to using column_property() for SQL expressions include that the expression must be compatible with the SELECT statement emitted for the class as a whole, and there are also some configurational quirks which can occur when using column_property() from declarative mixins.
Our “fullname” example can be expressed using column_property() as follows:
from sqlalchemy.orm import column_property


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))
    fullname = column_property(firstname + " " + lastname)
Correlated subqueries may be used as well. Below we use the select() construct to create a ScalarSelect, representing a column-oriented SELECT statement, that links together the count of Address objects available for a particular User:
from sqlalchemy.orm import column_property
from sqlalchemy import select, func
from sqlalchemy import Column, Integer, String, ForeignKey

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    address_count = column_property(
        select(func.count(Address.id))
        .where(Address.user_id == id)
        .correlate_except(Address)
        .scalar_subquery()
    )
In the above example, we define a ScalarSelect() construct like the following:
stmt = (
    select(func.count(Address.id))
    .where(Address.user_id == id)
    .correlate_except(Address)
    .scalar_subquery()
)
Above, we first use select() to create a Select construct, which we then convert into a scalar subquery using the Select.scalar_subquery() method, indicating our intent to use this Select statement in a column expression context.
Within the Select itself, we select the count of Address.id rows where the Address.user_id column is equated to id, which in the context of the User class is the Column named id (note that id is also the name of a Python built in function, which is not what we want to use here - if we were outside of the User class definition, we’d use User.id).
The Select.correlate_except() method indicates that each element in the FROM clause of this select() may be omitted from the FROM list (that is, correlated to the enclosing SELECT statement against User) except for the one corresponding to Address. This isn’t strictly necessary, but prevents Address from being inadvertently omitted from the FROM list in the case of a long string of joins between User and Address tables where SELECT statements against Address are nested.
For a column_property() that refers to columns linked from a many-to-many relationship, use and_() to join the fields of the association table to both tables in a relationship:
from sqlalchemy import and_


class Author(Base):
    # 

    book_count = column_property(
        select(func.count(books.c.id))
        .where(
            and_(
                book_authors.c.author_id == authors.c.id,
                book_authors.c.book_id == books.c.id,
            )
        )
        .scalar_subquery()
    )
Adding column_property() to an existing Declarative mapped class
If import issues prevent the column_property() from being defined inline with the class, it can be assigned to the class after both are configured. When using mappings that make use of a Declarative base class (i.e. produced by the DeclarativeBase superclass or legacy functions such as declarative_base()), this attribute assignment has the effect of calling Mapper.add_property() to add an additional property after the fact:
# only works if a declarative base class is in use
User.address_count = column_property(
    select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
)
When using mapping styles that don’t use Declarative base classes such as the registry.mapped() decorator, the Mapper.add_property() method may be invoked explicitly on the underlying Mapper object, which can be obtained using inspect():
from sqlalchemy.orm import registry

reg = registry()


@reg.mapped
class User:
    __tablename__ = "user"

    # additional mapping directives


# later 

# works for any kind of mapping
from sqlalchemy import inspect

inspect(User).add_property(
    column_property(
        select(func.count(Address.id))
        .where(Address.user_id == User.id)
        .scalar_subquery()
    )
)
See also
Appending additional columns to an existing Declarative mapped class
Composing from Column Properties at Mapping Time
It is possible to create mappings that combine multiple ColumnProperty objects together. The ColumnProperty will be interpreted as a SQL expression when used in a Core expression context, provided that it is targeted by an existing expression object; this works by the Core detecting that the object has a __clause_element__() method which returns a SQL expression. However, if the ColumnProperty is used as a lead object in an expression where there is no other Core SQL expression object to target it, the ColumnProperty.expression attribute will return the underlying SQL expression so that it can be used to build SQL expressions consistently. Below, the File class contains an attribute File.path that concatenates a string token to the File.filename attribute, which is itself a ColumnProperty:
class File(Base):
    __tablename__ = "file"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(64))
    extension = mapped_column(String(8))
    filename = column_property(name + "." + extension)
    path = column_property("C:/" + filename.expression)
When the File class is used in expressions normally, the attributes assigned to filename and path are usable directly. The use of the ColumnProperty.expression attribute is only necessary when using the ColumnProperty directly within the mapping definition:
stmt = select(File.path).where(File.filename == "foo.txt")
Using Column Deferral with column_property()
The column deferral feature introduced in the ORM Querying Guide at Limiting which Columns Load with Column Deferral may be applied at mapping time to a SQL expression mapped by column_property() by using the deferred() function in place of column_property():
from sqlalchemy.orm import deferred


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
See also
Using deferred() for imperative mappers, mapped SQL expressions
Using a plain descriptor
In cases where a SQL query more elaborate than what column_property() or hybrid_property can provide must be emitted, a regular Python function accessed as an attribute can be used, assuming the expression only needs to be available on an already-loaded instance. The function is decorated with Python’s own @property decorator to mark it as a read-only attribute. Within the function, object_session() is used to locate the Session corresponding to the current object, which is then used to emit a query:
from sqlalchemy.orm import object_session
from sqlalchemy import select, func


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @property
    def address_count(self):
        return object_session(self).scalar(
            select(func.count(Address.id)).where(Address.user_id == self.id)
        )
The plain descriptor approach is useful as a last resort, but is less performant in the usual case than both the hybrid and column property approaches, in that it needs to emit a SQL query upon each access.
Query-time SQL expressions as mapped attributes
In addition to being able to configure fixed SQL expressions on mapped classes, the SQLAlchemy ORM also includes a feature wherein objects may be loaded with the results of arbitrary SQL expressions which are set up at query time as part of their state. This behavior is available by configuring an ORM mapped attribute using query_expression() and then using the with_expression() loader option at query time. See the section Loading Arbitrary SQL Expressions onto Objects for an example mapping and usage.


Changing Attribute Behavior
This section will discuss features and techniques used to modify the behavior of ORM mapped attributes, including those mapped with mapped_column(), relationship(), and others.
Simple Validators
A quick way to add a “validation” routine to an attribute is to use the validates() decorator. An attribute validator can raise an exception, halting the process of mutating the attribute’s value, or can change the given value into something different. Validators, like all attribute extensions, are only called by normal userland code; they are not issued when the ORM is populating the object:
from sqlalchemy.orm import validates


class EmailAddress(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)

    @validates("email")
    def validate_email(self, key, address):
        if "@" not in address:
            raise ValueError("failed simple email validation")
        return address
Validators also receive collection append events, when items are added to a collection:
from sqlalchemy.orm import validates


class User(Base):
    # 

    addresses = relationship("Address")

    @validates("addresses")
    def validate_address(self, key, address):
        if "@" not in address.email:
            raise ValueError("failed simplified email validation")
        return address
The validation function by default does not get emitted for collection remove events, as the typical expectation is that a value being discarded doesn’t require validation. However, validates() supports reception of these events by specifying include_removes=True to the decorator. When this flag is set, the validation function must receive an additional boolean argument which if True indicates that the operation is a removal:
from sqlalchemy.orm import validates


class User(Base):
    # 

    addresses = relationship("Address")

    @validates("addresses", include_removes=True)
    def validate_address(self, key, address, is_remove):
        if is_remove:
            raise ValueError("not allowed to remove items from the collection")
        else:
            if "@" not in address.email:
                raise ValueError("failed simplified email validation")
            return address
The case where mutually dependent validators are linked via a backref can also be tailored, using the include_backrefs=False option; this option, when set to False, prevents a validation function from emitting if the event occurs as a result of a backref:
from sqlalchemy.orm import validates


class User(Base):
    # 

    addresses = relationship("Address", backref="user")

    @validates("addresses", include_backrefs=False)
    def validate_address(self, key, address):
        if "@" not in address:
            raise ValueError("failed simplified email validation")
        return address
Above, if we were to assign to Address.user as in some_address.user = some_user, the validate_address() function would not be emitted, even though an append occurs to some_user.addresses - the event is caused by a backref.
Note that the validates() decorator is a convenience function built on top of attribute events. An application that requires more control over configuration of attribute change behavior can make use of this system, described at AttributeEvents.
Object NameDescriptionvalidates(*names, [include_removes, include_backrefs])Decorate a method as a ‘validator’ for one or more named properties.function sqlalchemy.orm.validates(*names: str, include_removes: bool = False, include_backrefs: bool = True) → Callable[[_Fn], _Fn]
Decorate a method as a ‘validator’ for one or more named properties.
Designates a method as a validator, a method which receives the name of the attribute as well as a value to be assigned, or in the case of a collection, the value to be added to the collection. The function can then raise validation exceptions to halt the process from continuing (where Python’s built-in ValueError and AssertionError exceptions are reasonable choices), or can modify or replace the value before proceeding. The function should otherwise return the given value.
Note that a validator for a collection cannot issue a load of that collection within the validation routine - this usage raises an assertion to avoid recursion overflows. This is a reentrant condition which is not supported.
Parameters:
• *names – list of attribute names to be validated.
• include_removes – if True, “remove” events will be sent as well - the validation function must accept an additional argument “is_remove” which will be a boolean.
• include_backrefs – 
defaults to True; if False, the validation function will not emit if the originator is an attribute event related via a backref. This can be used for bi-directional validates() usage where only one validator should emit per attribute operation.
Changed in version 2.0.16: This paramter inadvertently defaulted to False for releases 2.0.0 through 2.0.15. Its correct default of True is restored in 2.0.16.
See also
Simple Validators - usage examples for validates()
Using Custom Datatypes at the Core Level
A non-ORM means of affecting the value of a column in a way that suits converting data between how it is represented in Python, vs. how it is represented in the database, can be achieved by using a custom datatype that is applied to the mapped Table metadata. This is more common in the case of some style of encoding / decoding that occurs both as data goes to the database and as it is returned; read more about this in the Core documentation at Augmenting Existing Types.
Using Descriptors and Hybrids
A more comprehensive way to produce modified behavior for an attribute is to use descriptors. These are commonly used in Python using the property() function. The standard SQLAlchemy technique for descriptors is to create a plain descriptor, and to have it read/write from a mapped attribute with a different name. Below we illustrate this using Python 2.6-style properties:
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    # name the attribute with an underscore,
    # different from the column name
    _email = mapped_column("email", String)

    # then create an ".email" attribute
    # to get/set "._email"
    @property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
The approach above will work, but there’s more we can add. While our EmailAddress object will shuttle the value through the email descriptor and into the _email mapped attribute, the class level EmailAddress.email attribute does not have the usual expression semantics usable with Select. To provide these, we instead use the hybrid extension as follows:
from sqlalchemy.ext.hybrid import hybrid_property


class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
The .email attribute, in addition to providing getter/setter behavior when we have an instance of EmailAddress, also provides a SQL expression when used at the class level, that is, from the EmailAddress class directly:
from sqlalchemy.orm import Session
from sqlalchemy import select

session = Session()

address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address@example.com")
).one()
SELECT address.email AS address_email, address.id AS address_id
FROM address
WHERE address.email = ?
('address@example.com',)
address.email = "otheraddress@example.com"
session.commit()
UPDATE address SET email=? WHERE address.id = ?
('otheraddress@example.com', 1)
COMMIT
The hybrid_property also allows us to change the behavior of the attribute, including defining separate behaviors when the attribute is accessed at the instance level versus at the class/expression level, using the hybrid_property.expression() modifier. Such as, if we wanted to add a host name automatically, we might define two sets of string manipulation logic:
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
        """Return the value of _email up until the last twelve
        characters."""

        return self._email[:-12]

    @email.setter
    def email(self, email):
        """Set the value of _email, tacking on the twelve character
        value @example.com."""

        self._email = email + "@example.com"

    @email.expression
    def email(cls):
        """Produce a SQL expression that represents the value
        of the _email column, minus the last twelve characters."""

        return func.substr(cls._email, 1, func.length(cls._email) - 12)
Above, accessing the email property of an instance of EmailAddress will return the value of the _email attribute, removing or adding the hostname @example.com from the value. When we query against the email attribute, a SQL function is rendered which produces the same effect:
address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address")
).one()
SELECT address.email AS address_email, address.id AS address_id
FROM address
WHERE substr(address.email, ?, length(address.email) - ?) = ?
(1, 12, 'address')
Read more about Hybrids at Hybrid Attributes.
Synonyms
Synonyms are a mapper-level construct that allow any attribute on a class to “mirror” another attribute that is mapped.
In the most basic sense, the synonym is an easy way to make a certain attribute available by an additional name:
from sqlalchemy.orm import synonym


class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    job_status = mapped_column(String(50))

    status = synonym("job_status")
The above class MyClass has two attributes, .job_status and .status that will behave as one attribute, both at the expression level:
 print(MyClass.job_status == "some_status")
my_table.job_status = :job_status_1
 print(MyClass.status == "some_status")
my_table.job_status = :job_status_1
and at the instance level:
 m1 = MyClass(status="x")
 m1.status, m1.job_status
('x', 'x')

 m1.job_status = "y"
 m1.status, m1.job_status
('y', 'y')
The synonym() can be used for any kind of mapped attribute that subclasses MapperProperty, including mapped columns and relationships, as well as synonyms themselves.
Beyond a simple mirror, synonym() can also be made to reference a user-defined descriptor. We can supply our status synonym with a @property:
class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @property
    def job_status(self):
        return "Status: " + self.status

    job_status = synonym("status", descriptor=job_status)
When using Declarative, the above pattern can be expressed more succinctly using the synonym_for() decorator:
from sqlalchemy.ext.declarative import synonym_for


class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @synonym_for("status")
    @property
    def job_status(self):
        return "Status: " + self.status
While the synonym() is useful for simple mirroring, the use case of augmenting attribute behavior with descriptors is better handled in modern usage using the hybrid attribute feature, which is more oriented towards Python descriptors. Technically, a synonym() can do everything that a hybrid_property can do, as it also supports injection of custom SQL capabilities, but the hybrid is more straightforward to use in more complex situations.
Object NameDescriptionsynonym(name, *, [map_column, descriptor, comparator_factory, init, repr, default, default_factory, compare, kw_only, hash, info, doc])Denote an attribute name as a synonym to a mapped property, in that the attribute will mirror the value and expression behavior of another attribute.function sqlalchemy.orm.synonym(name: str, *, map_column: bool | None = None, descriptor: Any | None = None, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: _NoArg | _T = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None) → Synonym[Any]
Denote an attribute name as a synonym to a mapped property, in that the attribute will mirror the value and expression behavior of another attribute.
e.g.:
class MyClass(Base):
    __tablename__ = "my_table"

    id = Column(Integer, primary_key=True)
    job_status = Column(String(50))

    status = synonym("job_status")
Parameters:
• name – the name of the existing mapped property. This can refer to the string name ORM-mapped attribute configured on the class, including column-bound attributes and relationships.
• descriptor – a Python descriptor that will be used as a getter (and potentially a setter) when this attribute is accessed at the instance level.
• map_column – 
For classical mappings and mappings against an existing Table object only. if True, the synonym() construct will locate the Column object upon the mapped table that would normally be associated with the attribute name of this synonym, and produce a new ColumnProperty that instead maps this Column to the alternate name given as the “name” argument of the synonym; in this way, the usual step of redefining the mapping of the Column to be under a different name is unnecessary. This is usually intended to be used when a Column is to be replaced with an attribute that also uses a descriptor, that is, in conjunction with the synonym.descriptor parameter:
my_table = Table(
    "my_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("job_status", String(50)),
)


class MyClass:
    @property
    def _job_status_descriptor(self):
        return "Status: %s" % self._job_status


mapper(
    MyClass,
    my_table,
    properties={
        "job_status": synonym(
            "_job_status",
            map_column=True,
            descriptor=MyClass._job_status_descriptor,
        )
    },
)
Above, the attribute named _job_status is automatically mapped to the job_status column:
 j1 = MyClass()
 j1._job_status = "employed"
 j1.job_status
Status: employed
• When using Declarative, in order to provide a descriptor in conjunction with a synonym, use the sqlalchemy.ext.declarative.synonym_for() helper. However, note that the hybrid properties feature should usually be preferred, particularly when redefining attribute behavior.
• info – Optional data dictionary which will be populated into the InspectionAttr.info attribute of this object.
• comparator_factory – 
A subclass of PropComparator that will provide custom comparison behavior at the SQL expression level.
Note
For the use case of providing an attribute which redefines both Python-level and SQL-expression level behavior of an attribute, please refer to the Hybrid attribute introduced at Using Descriptors and Hybrids for a more effective technique.
See also
Synonyms - Overview of synonyms
synonym_for() - a helper oriented towards Declarative
Using Descriptors and Hybrids - The Hybrid Attribute extension provides an updated approach to augmenting attribute behavior more flexibly than can be achieved with synonyms.
Operator Customization
The “operators” used by the SQLAlchemy ORM and Core expression language are fully customizable. For example, the comparison expression User.name == 'ed' makes usage of an operator built into Python itself called operator.eq - the actual SQL construct which SQLAlchemy associates with such an operator can be modified. New operations can be associated with column expressions as well. The operators which take place for column expressions are most directly redefined at the type level - see the section Redefining and Creating New Operators for a description.
ORM level functions like column_property(), relationship(), and composite() also provide for operator redefinition at the ORM level, by passing a PropComparator subclass to the comparator_factory argument of each function. Customization of operators at this level is a rare use case. See the documentation at PropComparator for an overview.
Composite Column Types
Sets of columns can be associated with a single user-defined datatype, which in modern use is normally a Python dataclass. The ORM provides a single attribute which represents the group of columns using the class you provide.
A simple example represents pairs of Integer columns as a Point object, with attributes .x and .y. Using a dataclass, these attributes are defined with the corresponding int Python type:
import dataclasses


@dataclasses.dataclass
class Point:
    x: int
    y: int
Non-dataclass forms are also accepted, but require additional methods to be implemented. For an example using a non-dataclass class, see the section Using Legacy Non-Dataclasses.
Added in version 2.0: The composite() construct fully supports Python dataclasses including the ability to derive mapped column datatypes from the composite class.
We will create a mapping to a table vertices, which represents two points as x1/y1 and x2/y2. The Point class is associated with the mapped columns using the composite() construct.
The example below illustrates the most modern form of composite() as used with a fully Annotated Declarative Table configuration. mapped_column() constructs representing each column are passed directly to composite(), indicating zero or more aspects of the columns to be generated, in this case the names; the composite() construct derives the column types (in this case int, corresponding to Integer) from the dataclass directly:
from sqlalchemy.orm import DeclarativeBase, Mapped
from sqlalchemy.orm import composite, mapped_column


class Base(DeclarativeBase):
    pass


class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)

    start: Mapped[Point] = composite(mapped_column("x1"), mapped_column("y1"))
    end: Mapped[Point] = composite(mapped_column("x2"), mapped_column("y2"))

    def __repr__(self):
        return f"Vertex(start={self.start}, end={self.end})"
Tip
In the example above the columns that represent the composites (x1, y1, etc.) are also accessible on the class but are not correctly understood by type checkers. If accessing the single columns is important they can be explicitly declared, as shown in Map columns directly, pass attribute names to composite.
The above mapping would correspond to a CREATE TABLE statement as:
 from sqlalchemy.schema import CreateTable
 print(CreateTable(Vertex.__table__))
CREATE TABLE vertices (
  id INTEGER NOT NULL,
  x1 INTEGER NOT NULL,
  y1 INTEGER NOT NULL,
  x2 INTEGER NOT NULL,
  y2 INTEGER NOT NULL,
  PRIMARY KEY (id)
)
Working with Mapped Composite Column Types
With a mapping as illustrated in the top section, we can work with the Vertex class, where the .start and .end attributes will transparently refer to the columns referenced by the Point class, as well as with instances of the Vertex class, where the .start and .end attributes will refer to instances of the Point class. The x1, y1, x2, and y2 columns are handled transparently:
• Persisting Point objects
We can create a Vertex object, assign Point objects as members, and they will be persisted as expected:
 v = Vertex(start=Point(3, 4), end=Point(5, 6))
 session.add(v)
 session.commit()
BEGIN (implicit)
INSERT INTO vertices (x1, y1, x2, y2) VALUES (?, ?, ?, ?)
[generated in ] (3, 4, 5, 6)
COMMIT
Selecting Point objects as columns
composite() will allow the Vertex.start and Vertex.end attributes to behave like a single SQL expression to as much an extent as possible when using the ORM Session (including the legacy Query object) to select Point objects:
 stmt = select(Vertex.start, Vertex.end)
 session.execute(stmt).all()
SELECT vertices.x1, vertices.y1, vertices.x2, vertices.y2
FROM vertices
[] ()
[(Point(x=3, y=4), Point(x=5, y=6))]
Comparing Point objects in SQL expressions
The Vertex.start and Vertex.end attributes may be used in WHERE criteria and similar, using ad-hoc Point objects for comparisons:
 stmt = select(Vertex).where(Vertex.start == Point(3, 4)).where(Vertex.end < Point(7, 8))
 session.scalars(stmt).all()
SELECT vertices.id, vertices.x1, vertices.y1, vertices.x2, vertices.y2
FROM vertices
WHERE vertices.x1 = ? AND vertices.y1 = ? AND vertices.x2 < ? AND vertices.y2 < ?
[] (3, 4, 7, 8)
[Vertex(Point(x=3, y=4), Point(x=5, y=6))]
•  Added in version 2.0: composite() constructs now support “ordering” comparisons such as <, >=, and similar, in addition to the already-present support for ==, !=.
Tip
The “ordering” comparison above using the “less than” operator (<) as well as the “equality” comparison using ==, when used to generate SQL expressions, are implemented by the Comparator class, and don’t make use of the comparison methods on the composite class itself, e.g. the __lt__() or __eq__() methods. From this it follows that the Point dataclass above also need not implement the dataclasses order=True parameter for the above SQL operations to work. The section Redefining Comparison Operations for Composites contains background on how to customize the comparison operations.
•  Updating Point objects on Vertex Instances
By default, the Point object must be replaced by a new object for changes to be detected:
 v1 = session.scalars(select(Vertex)).one()
SELECT vertices.id, vertices.x1, vertices.y1, vertices.x2, vertices.y2
FROM vertices
[] ()
 v1.end = Point(x=10, y=14)
 session.commit()
UPDATE vertices SET x2=?, y2=? WHERE vertices.id = ?
[] (10, 14, 1)
COMMIT
• In order to allow in place changes on the composite object, the Mutation Tracking extension must be used. See the section Establishing Mutability on Composites for examples.
Other mapping forms for composites
The composite() construct may be passed the relevant columns using a mapped_column() construct, a Column, or the string name of an existing mapped column. The following examples illustrate an equivalent mapping as that of the main section above.
Map columns directly, then pass to composite
Here we pass the existing mapped_column() instances to the composite() construct, as in the non-annotated example below where we also pass the Point class as the first argument to composite():
from sqlalchemy import Integer
from sqlalchemy.orm import mapped_column, composite


class Vertex(Base):
    __tablename__ = "vertices"

    id = mapped_column(Integer, primary_key=True)
    x1 = mapped_column(Integer)
    y1 = mapped_column(Integer)
    x2 = mapped_column(Integer)
    y2 = mapped_column(Integer)

    start = composite(Point, x1, y1)
    end = composite(Point, x2, y2)
Map columns directly, pass attribute names to composite
We can write the same example above using more annotated forms where we have the option to pass attribute names to composite() instead of full column constructs:
from sqlalchemy.orm import mapped_column, composite, Mapped


class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    start: Mapped[Point] = composite("x1", "y1")
    end: Mapped[Point] = composite("x2", "y2")
Imperative mapping and imperative table
When using imperative table or fully imperative mappings, we have access to Column objects directly. These may be passed to composite() as well, as in the imperative example below:
mapper_registry.map_imperatively(
    Vertex,
    vertices_table,
    properties={
        "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
        "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
    },
)
Using Legacy Non-Dataclasses
If not using a dataclass, the requirements for the custom datatype class are that it have a constructor which accepts positional arguments corresponding to its column format, and also provides a method __composite_values__() which returns the state of the object as a list or tuple, in order of its column-based attributes. It also should supply adequate __eq__() and __ne__() methods which test the equality of two instances.
To illustrate the equivalent Point class from the main section not using a dataclass:
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __repr__(self):
        return f"Point(x={self.x!r}, y={self.y!r})"

    def __eq__(self, other):
        return isinstance(other, Point) and other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
Usage with composite() then proceeds where the columns to be associated with the Point class must also be declared with explicit types, using one of the forms at Other mapping forms for composites.
Tracking In-Place Mutations on Composites
In-place changes to an existing composite value are not tracked automatically. Instead, the composite class needs to provide events to its parent object explicitly. This task is largely automated via the usage of the MutableComposite mixin, which uses events to associate each user-defined composite object with all parent associations. Please see the example in Establishing Mutability on Composites.
Redefining Comparison Operations for Composites
The “equals” comparison operation by default produces an AND of all corresponding columns equated to one another. This can be changed using the comparator_factory argument to composite(), where we specify a custom Comparator class to define existing or new operations. Below we illustrate the “greater than” operator, implementing the same expression that the base “greater than” does:
import dataclasses

from sqlalchemy.orm import composite
from sqlalchemy.orm import CompositeProperty
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.sql import and_


@dataclasses.dataclass
class Point:
    x: int
    y: int


class PointComparator(CompositeProperty.Comparator):
    def __gt__(self, other):
        """redefine the 'greater than' operation"""

        return and_(
            *[
                a > b
                for a, b in zip(
                    self.__clause_element__().clauses,
                    dataclasses.astuple(other),
                )
            ]
        )


class Base(DeclarativeBase):
    pass


class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)

    start: Mapped[Point] = composite(
        mapped_column("x1"), mapped_column("y1"), comparator_factory=PointComparator
    )
    end: Mapped[Point] = composite(
        mapped_column("x2"), mapped_column("y2"), comparator_factory=PointComparator
    )
Since Point is a dataclass, we may make use of dataclasses.astuple() to get a tuple form of Point instances.
The custom comparator then returns the appropriate SQL expression:
 print(Vertex.start > Point(5, 6))
vertices.x1 > :x1_1 AND vertices.y1 > :y1_1
Nesting Composites
Composite objects can be defined to work in simple nested schemes, by redefining behaviors within the composite class to work as desired, then mapping the composite class to the full length of individual columns normally. This requires that additional methods to move between the “nested” and “flat” forms are defined.
Below we reorganize the Vertex class to itself be a composite object which refers to Point objects. Vertex and Point can be dataclasses, however we will add a custom construction method to Vertex that can be used to create new Vertex objects given four column values, which will will arbitrarily name _generate() and define as a classmethod so that we can make new Vertex objects by passing values to the Vertex._generate() method.
We will also implement the __composite_values__() method, which is a fixed name recognized by the composite() construct (introduced previously at Using Legacy Non-Dataclasses) that indicates a standard way of receiving the object as a flat tuple of column values, which in this case will supersede the usual dataclass-oriented methodology.
With our custom _generate() constructor and __composite_values__() serializer method, we can now move between a flat tuple of columns and Vertex objects that contain Point instances. The Vertex._generate method is passed as the first argument to the composite() construct as the source of new Vertex instances, and the __composite_values__() method will be used implicitly by composite().
For the purposes of the example, the Vertex composite is then mapped to a class called HasVertex, which is where the Table containing the four source columns ultimately resides:
from __future__ import annotations

import dataclasses
from typing import Any
from typing import Tuple

from sqlalchemy.orm import composite
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


@dataclasses.dataclass
class Point:
    x: int
    y: int


@dataclasses.dataclass
class Vertex:
    start: Point
    end: Point

    @classmethod
    def _generate(cls, x1: int, y1: int, x2: int, y2: int) -> Vertex:
        """generate a Vertex from a row"""
        return Vertex(Point(x1, y1), Point(x2, y2))

    def __composite_values__(self) -> Tuple[Any, ]:
        """generate a row from a Vertex"""
        return dataclasses.astuple(self.start) + dataclasses.astuple(self.end)


class Base(DeclarativeBase):
    pass


class HasVertex(Base):
    __tablename__ = "has_vertex"
    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    vertex: Mapped[Vertex] = composite(Vertex._generate, "x1", "y1", "x2", "y2")
The above mapping can then be used in terms of HasVertex, Vertex, and Point:
hv = HasVertex(vertex=Vertex(Point(1, 2), Point(3, 4)))

session.add(hv)
session.commit()

stmt = select(HasVertex).where(HasVertex.vertex == Vertex(Point(1, 2), Point(3, 4)))

hv = session.scalars(stmt).first()
print(hv.vertex.start)
print(hv.vertex.end)
Composite API
Object NameDescriptioncomposite([_class_or_attr], *attrs, [group, deferred, raiseload, comparator_factory, active_history, init, repr, default, default_factory, compare, kw_only, hash, info, doc], **__kw)Return a composite column-based property for use with a Mapper.function sqlalchemy.orm.composite(_class_or_attr: None | Type[_CC] | Callable[, _CC] | _CompositeAttrType[Any] = None, *attrs: _CompositeAttrType[Any], group: str | None = None, deferred: bool = False, raiseload: bool = False, comparator_factory: Type[Composite.Comparator[_T]] | None = None, active_history: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None, **__kw: Any) → Composite[Any]
Return a composite column-based property for use with a Mapper.
See the mapping documentation section Composite Column Types for a full usage example.
The MapperProperty returned by composite() is the Composite.
Parameters:
• class_ – The “composite type” class, or any classmethod or callable which will produce a new instance of the composite object given the column values in order.
• *attrs – 
List of elements to be mapped, which may include:
o Column objects
o mapped_column() constructs
o string names of other attributes on the mapped class, which may be any other SQL or object-mapped attribute. This can for example allow a composite that refers to a many-to-one relationship
• active_history=False – When True, indicates that the “previous” value for a scalar attribute should be loaded when replaced, if not already loaded. See the same flag on column_property().
• group – A group name for this property when marked as deferred.
• deferred – When True, the column property is “deferred”, meaning that it does not load immediately, and is instead loaded when the attribute is first accessed on an instance. See also deferred().
• comparator_factory – a class which extends Comparator which provides custom SQL clause generation for comparison operations.
• doc – optional string that will be applied as the doc on the class-bound descriptor.
• info – Optional data dictionary which will be populated into the MapperProperty.info attribute of this object.
• init – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __init__() method as generated by the dataclass process.
• repr – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __repr__() method as generated by the dataclass process.
• default_factory – Specific to Declarative Dataclass Mapping, specifies a default-value generation function that will take place as part of the __init__() method as generated by the dataclass process.
• compare – 
Specific to Declarative Dataclass Mapping, indicates if this field should be included in comparison operations when generating the __eq__() and __ne__() methods for the mapped class.
Added in version 2.0.0b4.
• kw_only – Specific to Declarative Dataclass Mapping, indicates if this field should be marked as keyword-only when generating the __init__().
• hash – 
Specific to Declarative Dataclass Mapping, controls if this field is included when generating the __hash__() method for the mapped class.
Added in version 2.0.36.


Mapping Class Inheritance Hierarchies
SQLAlchemy supports three forms of inheritance:
• single table inheritance – several types of classes are represented by a single table;
• concrete table inheritance – each type of class is represented by independent tables;
• joined table inheritance – the class hierarchy is broken up among dependent tables. Each class represented by its own table that only includes those attributes local to that class.
The most common forms of inheritance are single and joined table, while concrete inheritance presents more configurational challenges.
When mappers are configured in an inheritance relationship, SQLAlchemy has the ability to load elements polymorphically, meaning that a single query can return objects of multiple types.
See also
Writing SELECT statements for Inheritance Mappings - in the ORM Querying Guide
Inheritance Mapping Recipes - complete examples of joined, single and concrete inheritance
Joined Table Inheritance
In joined table inheritance, each class along a hierarchy of classes is represented by a distinct table. Querying for a particular subclass in the hierarchy will render as a SQL JOIN along all tables in its inheritance path. If the queried class is the base class, the base table is queried instead, with options to include other tables at the same time or to allow attributes specific to sub-tables to load later.
In all cases, the ultimate class to instantiate for a given row is determined by a discriminator column or SQL expression, defined on the base class, which will yield a scalar value that is associated with a particular subclass.
The base class in a joined inheritance hierarchy is configured with additional arguments that will indicate to the polymorphic discriminator column, and optionally a polymorphic identifier for the base class itself:
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

    def __repr__(self):
        return f"{self.__class__.__name__}({self.name!r})"
In the above example, the discriminator is the type column, whichever is configured using the Mapper.polymorphic_on parameter. This parameter accepts a column-oriented expression, specified either as a string name of the mapped attribute to use or as a column expression object such as Column or mapped_column() construct.
The discriminator column will store a value which indicates the type of object represented within the row. The column may be of any datatype, though string and integer are the most common. The actual data value to be applied to this column for a particular row in the database is specified using the Mapper.polymorphic_identity parameter, described below.
While a polymorphic discriminator expression is not strictly necessary, it is required if polymorphic loading is desired. Establishing a column on the base table is the easiest way to achieve this, however very sophisticated inheritance mappings may make use of SQL expressions, such as a CASE expression, as the polymorphic discriminator.
Note
Currently, only one discriminator column or SQL expression may be configured for the entire inheritance hierarchy, typically on the base- most class in the hierarchy. “Cascading” polymorphic discriminator expressions are not yet supported.
We next define Engineer and Manager subclasses of Employee. Each contains columns that represent the attributes unique to the subclass they represent. Each table also must contain a primary key column (or columns), as well as a foreign key reference to the parent table:
class Engineer(Employee):
    __tablename__ = "engineer"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    engineer_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }


class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
In the above example, each mapping specifies the Mapper.polymorphic_identity parameter within its mapper arguments. This value populates the column designated by the Mapper.polymorphic_on parameter established on the base mapper. The Mapper.polymorphic_identity parameter should be unique to each mapped class across the whole hierarchy, and there should only be one “identity” per mapped class; as noted above, “cascading” identities where some subclasses introduce a second identity are not supported.
The ORM uses the value set up by Mapper.polymorphic_identity in order to determine which class a row belongs towards when loading rows polymorphically. In the example above, every row which represents an Employee will have the value 'employee' in its type column; similarly, every Engineer will get the value 'engineer', and each Manager will get the value 'manager'. Regardless of whether the inheritance mapping uses distinct joined tables for subclasses as in joined table inheritance, or all one table as in single table inheritance, this value is expected to be persisted and available to the ORM when querying. The Mapper.polymorphic_identity parameter also applies to concrete table inheritance, but is not actually persisted; see the later section at Concrete Table Inheritance for details.
In a polymorphic setup, it is most common that the foreign key constraint is established on the same column or columns as the primary key itself, however this is not required; a column distinct from the primary key may also be made to refer to the parent via foreign key. The way that a JOIN is constructed from the base table to subclasses is also directly customizable, however this is rarely necessary.
Joined inheritance primary keys
One natural effect of the joined table inheritance configuration is that the identity of any mapped object can be determined entirely from rows in the base table alone. This has obvious advantages, so SQLAlchemy always considers the primary key columns of a joined inheritance class to be those of the base table only. In other words, the id columns of both the engineer and manager tables are not used to locate Engineer or Manager objects - only the value in employee.id is considered. engineer.id and manager.id are still of course critical to the proper operation of the pattern overall as they are used to locate the joined row, once the parent row has been determined within a statement.
With the joined inheritance mapping complete, querying against Employee will return a combination of Employee, Engineer and Manager objects. Newly saved Engineer, Manager, and Employee objects will automatically populate the employee.type column with the correct “discriminator” value in this case "engineer", "manager", or "employee", as appropriate.
Relationships with Joined Inheritance
Relationships are fully supported with joined table inheritance. The relationship involving a joined-inheritance class should target the class in the hierarchy that also corresponds to the foreign key constraint; below, as the employee table has a foreign key constraint back to the company table, the relationships are set up between Company and Employee:
from __future__ import annotations

from sqlalchemy.orm import relationship


class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }


class Manager(Employee): 


class Engineer(Employee): 
If the foreign key constraint is on a table corresponding to a subclass, the relationship should target that subclass instead. In the example below, there is a foreign key constraint from manager to company, so the relationships are established between the Manager and Company classes:
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }


class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }


class Engineer(Employee): 
Above, the Manager class will have a Manager.company attribute; Company will have a Company.managers attribute that always loads against a join of the employee and manager tables together.
Loading Joined Inheritance Mappings
See the section Writing SELECT statements for Inheritance Mappings for background on inheritance loading techniques, including configuration of tables to be queried both at mapper configuration time as well as query time.
Single Table Inheritance
Single table inheritance represents all attributes of all subclasses within a single table. A particular subclass that has attributes unique to that class will persist them within columns in the table that are otherwise NULL if the row refers to a different kind of object.
Querying for a particular subclass in the hierarchy will render as a SELECT against the base table, which will include a WHERE clause that limits rows to those with a particular value or values present in the discriminator column or expression.
Single table inheritance has the advantage of simplicity compared to joined table inheritance; queries are much more efficient as only one table needs to be involved in order to load objects of every represented class.
Single-table inheritance configuration looks much like joined-table inheritance, except only the base class specifies __tablename__. A discriminator column is also required on the base table so that classes can be differentiated from each other.
Even though subclasses share the base table for all of their attributes, when using Declarative, mapped_column objects may still be specified on subclasses, indicating that the column is to be mapped only to that subclass; the mapped_column will be applied to the same base Table object:
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }


class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }


class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
Note that the mappers for the derived classes Manager and Engineer omit the __tablename__, indicating they do not have a mapped table of their own. Additionally, a mapped_column() directive with nullable=True is included; as the Python types declared for these classes do not include Optional[], the column would normally be mapped as NOT NULL, which would not be appropriate as this column only expects to be populated for those rows that correspond to that particular subclass.
Resolving Column Conflicts with use_existing_column
Note in the previous section that the manager_name and engineer_info columns are “moved up” to be applied to Employee.__table__, as a result of their declaration on a subclass that has no table of its own. A tricky case comes up when two subclasses want to specify the same column, as below:
from datetime import datetime


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }


class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)


class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)
Above, the start_date column declared on both Engineer and Manager will result in an error:
sqlalchemy.exc.ArgumentError: Column 'start_date' on class Manager conflicts
with existing column 'employee.start_date'.  If using Declarative,
consider using the use_existing_column parameter of mapped_column() to
resolve conflicts.
The above scenario presents an ambiguity to the Declarative mapping system that may be resolved by using the mapped_column.use_existing_column parameter on mapped_column(), which instructs mapped_column() to look on the inheriting superclass present and use the column that’s already mapped, if already present, else to map a new column:
from sqlalchemy import DateTime


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }


class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )


class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )
Above, when Manager is mapped, the start_date column is already present on the Employee class, having been provided by the Engineer mapping already. The mapped_column.use_existing_column parameter indicates to mapped_column() that it should look for the requested Column on the mapped Table for Employee first, and if present, maintain that existing mapping. If not present, mapped_column() will map the column normally, adding it as one of the columns in the Table referenced by the Employee superclass.
Added in version 2.0.0b4: - Added mapped_column.use_existing_column, which provides a 2.0-compatible means of mapping a column on an inheriting subclass conditionally. The previous approach which combines declared_attr with a lookup on the parent .__table__ continues to function as well, but lacks PEP 484 typing support.
A similar concept can be used with mixin classes (see Composing Mapped Hierarchies with Mixins) to define a particular series of columns and/or other mapped attributes from a reusable mixin class:
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "employee",
    }


class HasStartDate:
    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )


class Engineer(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }


class Manager(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
Relationships with Single Table Inheritance
Relationships are fully supported with single table inheritance. Configuration is done in the same manner as that of joined inheritance; a foreign key attribute should be on the same class that’s the “foreign” side of the relationship:
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }


class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }


class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
Also, like the case of joined inheritance, we can create relationships that involve a specific subclass. When queried, the SELECT statement will include a WHERE clause that limits the class selection to that subclass or subclasses:
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }


class Manager(Employee):
    manager_name: Mapped[str] = mapped_column(nullable=True)

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }


class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
Above, the Manager class will have a Manager.company attribute; Company will have a Company.managers attribute that always loads against the employee with an additional WHERE clause that limits rows to those with type = 'manager'.
Building Deeper Hierarchies with polymorphic_abstract
Added in version 2.0.
When building any kind of inheritance hierarchy, a mapped class may include the Mapper.polymorphic_abstract parameter set to True, which indicates that the class should be mapped normally, however would not expect to be instantiated directly and would not include a Mapper.polymorphic_identity. Subclasses may then be declared as subclasses of this mapped class, which themselves can include a Mapper.polymorphic_identity and therefore be used normally. This allows a series of subclasses to be referenced at once by a common base class which is considered to be “abstract” within the hierarchy, both in queries as well as in relationship() declarations. This use differs from the use of the __abstract__ attribute with Declarative, which leaves the target class entirely unmapped and thus not usable as a mapped class by itself. Mapper.polymorphic_abstract may be applied to any class or classes at any level in the hierarchy, including on multiple levels at once.
As an example, suppose Manager and Principal were both to be classified against a superclass Executive, and Engineer and Sysadmin were classified against a superclass Technologist. Neither Executive or Technologist is ever instantiated, therefore have no Mapper.polymorphic_identity. These classes can be configured using Mapper.polymorphic_abstract as follows:
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }


class Executive(Employee):
    """An executive of the company"""

    executive_background: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}


class Technologist(Employee):
    """An employee who works with technology"""

    competencies: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}


class Manager(Executive):
    """a manager"""

    __mapper_args__ = {"polymorphic_identity": "manager"}


class Principal(Executive):
    """a principal of the company"""

    __mapper_args__ = {"polymorphic_identity": "principal"}


class Engineer(Technologist):
    """an engineer"""

    __mapper_args__ = {"polymorphic_identity": "engineer"}


class SysAdmin(Technologist):
    """a systems administrator"""

    __mapper_args__ = {"polymorphic_identity": "sysadmin"}
In the above example, the new classes Technologist and Executive are ordinary mapped classes, and also indicate new columns to be added to the superclass called executive_background and competencies. However, they both lack a setting for Mapper.polymorphic_identity; this is because it’s not expected that Technologist or Executive would ever be instantiated directly; we’d always have one of Manager, Principal, Engineer or SysAdmin. We can however query for Principal and Technologist roles, as well as have them be targets of relationship(). The example below demonstrates a SELECT statement for Technologist objects:
session.scalars(select(Technologist)).all()
SELECT employee.id, employee.name, employee.type, employee.competencies
FROM employee
WHERE employee.type IN (?, ?)
[] ('engineer', 'sysadmin')
The Technologist and Executive abstract mapped classes may also be made the targets of relationship() mappings, like any other mapped class. We can extend the above example to include Company, with separate collections Company.technologists and Company.principals:
class Company(Base):
    __tablename__ = "company"
    id = Column(Integer, primary_key=True)

    executives: Mapped[List[Executive]] = relationship()
    technologists: Mapped[List[Technologist]] = relationship()


class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)

    # foreign key to "company.id" is added
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))

    # rest of mapping is the same
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
    }


# Executive, Technologist, Manager, Principal, Engineer, SysAdmin
# classes from previous example would follow here unchanged
Using the above mapping we can use joins and relationship loading techniques across Company.technologists and Company.executives individually:
session.scalars(
    select(Company)
    .join(Company.technologists)
    .where(Technologist.competency.ilike("%java%"))
    .options(selectinload(Company.executives))
).all()
SELECT company.id
FROM company JOIN employee ON company.id = employee.company_id AND employee.type IN (?, ?)
WHERE lower(employee.competencies) LIKE lower(?)
[] ('engineer', 'sysadmin', '%java%')

SELECT employee.company_id AS employee_company_id, employee.id AS employee_id,
employee.name AS employee_name, employee.type AS employee_type,
employee.executive_background AS employee_executive_background
FROM employee
WHERE employee.company_id IN (?) AND employee.type IN (?, ?)
[] (1, 'manager', 'principal')
See also
__abstract__ - Declarative parameter which allows a Declarative class to be completely un-mapped within a hierarchy, while still extending from a mapped superclass.
Loading Single Inheritance Mappings
The loading techniques for single-table inheritance are mostly identical to those used for joined-table inheritance, and a high degree of abstraction is provided between these two mapping types such that it is easy to switch between them as well as to intermix them in a single hierarchy (just omit __tablename__ from whichever subclasses are to be single-inheriting). See the sections Writing SELECT statements for Inheritance Mappings and SELECT Statements for Single Inheritance Mappings for documentation on inheritance loading techniques, including configuration of classes to be queried both at mapper configuration time as well as query time.
Concrete Table Inheritance
Concrete inheritance maps each subclass to its own distinct table, each of which contains all columns necessary to produce an instance of that class. A concrete inheritance configuration by default queries non-polymorphically; a query for a particular class will only query that class’ table and only return instances of that class. Polymorphic loading of concrete classes is enabled by configuring within the mapper a special SELECT that typically is produced as a UNION of all the tables.
Warning
Concrete table inheritance is much more complicated than joined or single table inheritance, and is much more limited in functionality especially pertaining to using it with relationships, eager loading, and polymorphic loading. When used polymorphically it produces very large queries with UNIONS that won’t perform as well as simple joins. It is strongly advised that if flexibility in relationship loading and polymorphic loading is required, that joined or single table inheritance be used if at all possible. If polymorphic loading isn’t required, then plain non-inheriting mappings can be used if each class refers to its own table completely.
Whereas joined and single table inheritance are fluent in “polymorphic” loading, it is a more awkward affair in concrete inheritance. For this reason, concrete inheritance is more appropriate when polymorphic loading is not required. Establishing relationships that involve concrete inheritance classes is also more awkward.
To establish a class as using concrete inheritance, add the Mapper.concrete parameter within the __mapper_args__. This indicates to Declarative as well as the mapping that the superclass table should not be considered as part of the mapping:
class Employee(Base):
    __tablename__ = "employee"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))


class Manager(Employee):
    __tablename__ = "manager"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }


class Engineer(Employee):
    __tablename__ = "engineer"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }
Two critical points should be noted:
• We must define all columns explicitly on each subclass, even those of the same name. A column such as Employee.name here is not copied out to the tables mapped by Manager or Engineer for us.
• while the Engineer and Manager classes are mapped in an inheritance relationship with Employee, they still do not include polymorphic loading. Meaning, if we query for Employee objects, the manager and engineer tables are not queried at all.
Concrete Polymorphic Loading Configuration
Polymorphic loading with concrete inheritance requires that a specialized SELECT is configured against each base class that should have polymorphic loading. This SELECT needs to be capable of accessing all the mapped tables individually, and is typically a UNION statement that is constructed using a SQLAlchemy helper polymorphic_union().
As discussed in Writing SELECT statements for Inheritance Mappings, mapper inheritance configurations of any type can be configured to load from a special selectable by default using the Mapper.with_polymorphic argument. Current public API requires that this argument is set on a Mapper when it is first constructed.
However, in the case of Declarative, both the mapper and the Table that is mapped are created at once, the moment the mapped class is defined. This means that the Mapper.with_polymorphic argument cannot be provided yet, since the Table objects that correspond to the subclasses haven’t yet been defined.
There are a few strategies available to resolve this cycle, however Declarative provides helper classes ConcreteBase and AbstractConcreteBase which handle this issue behind the scenes.
Using ConcreteBase, we can set up our concrete mapping in almost the same way as we do other forms of inheritance mappings:
from sqlalchemy.ext.declarative import ConcreteBase
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
Above, Declarative sets up the polymorphic selectable for the Employee class at mapper “initialization” time; this is the late-configuration step for mappers that resolves other dependent mappers. The ConcreteBase helper uses the polymorphic_union() function to create a UNION of all concrete-mapped tables after all the other classes are set up, and then configures this statement with the already existing base-class mapper.
Upon select, the polymorphic union produces a query like this:
session.scalars(select(Employee)).all()
SELECT
    pjoin.id,
    pjoin.name,
    pjoin.type,
    pjoin.manager_data,
    pjoin.engineer_info
FROM (
    SELECT
        employee.id AS id,
        employee.name AS name,
        CAST(NULL AS VARCHAR(40)) AS manager_data,
        CAST(NULL AS VARCHAR(40)) AS engineer_info,
        'employee' AS type
    FROM employee
    UNION ALL
    SELECT
        manager.id AS id,
        manager.name AS name,
        manager.manager_data AS manager_data,
        CAST(NULL AS VARCHAR(40)) AS engineer_info,
        'manager' AS type
    FROM manager
    UNION ALL
    SELECT
        engineer.id AS id,
        engineer.name AS name,
        CAST(NULL AS VARCHAR(40)) AS manager_data,
        engineer.engineer_info AS engineer_info,
        'engineer' AS type
    FROM engineer
) AS pjoin
The above UNION query needs to manufacture “NULL” columns for each subtable in order to accommodate for those columns that aren’t members of that particular subclass.
See also
ConcreteBase
Abstract Concrete Classes
The concrete mappings illustrated thus far show both the subclasses as well as the base class mapped to individual tables. In the concrete inheritance use case, it is common that the base class is not represented within the database, only the subclasses. In other words, the base class is “abstract”.
Normally, when one would like to map two different subclasses to individual tables, and leave the base class unmapped, this can be achieved very easily. When using Declarative, just declare the base class with the __abstract__ indicator:
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class Employee(Base):
    __abstract__ = True


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
Above, we are not actually making use of SQLAlchemy’s inheritance mapping facilities; we can load and persist instances of Manager and Engineer normally. The situation changes however when we need to query polymorphically, that is, we’d like to emit select(Employee) and get back a collection of Manager and Engineer instances. This brings us back into the domain of concrete inheritance, and we must build a special mapper against Employee in order to achieve this.
To modify our concrete inheritance example to illustrate an “abstract” base that is capable of polymorphic loading, we will have only an engineer and a manager table and no employee table, however the Employee mapper will be mapped directly to the “polymorphic union”, rather than specifying it locally to the Mapper.with_polymorphic parameter.
To help with this, Declarative offers a variant of the ConcreteBase class called AbstractConcreteBase which achieves this automatically:
from sqlalchemy.ext.declarative import AbstractConcreteBase
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class Employee(AbstractConcreteBase, Base):
    strict_attrs = True

    name = mapped_column(String(50))


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }


Base.registry.configure()
Above, the registry.configure() method is invoked, which will trigger the Employee class to be actually mapped; before the configuration step, the class has no mapping as the sub-tables which it will query from have not yet been defined. This process is more complex than that of ConcreteBase, in that the entire mapping of the base class must be delayed until all the subclasses have been declared. With a mapping like the above, only instances of Manager and Engineer may be persisted; querying against the Employee class will always produce Manager and Engineer objects.
Using the above mapping, queries can be produced in terms of the Employee class and any attributes that are locally declared upon it, such as the Employee.name:
 stmt = select(Employee).where(Employee.name == "n1")
 print(stmt)
SELECT pjoin.id, pjoin.name, pjoin.type, pjoin.manager_data, pjoin.engineer_info
FROM (
  SELECT engineer.id AS id, engineer.name AS name, engineer.engineer_info AS engineer_info,
  CAST(NULL AS VARCHAR(40)) AS manager_data, 'engineer' AS type
  FROM engineer
  UNION ALL
  SELECT manager.id AS id, manager.name AS name, CAST(NULL AS VARCHAR(40)) AS engineer_info,
  manager.manager_data AS manager_data, 'manager' AS type
  FROM manager
) AS pjoin
WHERE pjoin.name = :name_1
The AbstractConcreteBase.strict_attrs parameter indicates that the Employee class should directly map only those attributes which are local to the Employee class, in this case the Employee.name attribute. Other attributes such as Manager.manager_data and Engineer.engineer_info are present only on their corresponding subclass. When AbstractConcreteBase.strict_attrs is not set, then all subclass attributes such as Manager.manager_data and Engineer.engineer_info get mapped onto the base Employee class. This is a legacy mode of use which may be more convenient for querying but has the effect that all subclasses share the full set of attributes for the whole hierarchy; in the above example, not using AbstractConcreteBase.strict_attrs would have the effect of generating non-useful Engineer.manager_name and Manager.engineer_info attributes.
Added in version 2.0: Added AbstractConcreteBase.strict_attrs parameter to AbstractConcreteBase which produces a cleaner mapping; the default is False to allow legacy mappings to continue working as they did in 1.x versions.
See also
AbstractConcreteBase
Classical and Semi-Classical Concrete Polymorphic Configuration
The Declarative configurations illustrated with ConcreteBase and AbstractConcreteBase are equivalent to two other forms of configuration that make use of polymorphic_union() explicitly. These configurational forms make use of the Table object explicitly so that the “polymorphic union” can be created first, then applied to the mappings. These are illustrated here to clarify the role of the polymorphic_union() function in terms of mapping.
A semi-classical mapping for example makes use of Declarative, but establishes the Table objects separately:
metadata_obj = Base.metadata

employees_table = Table(
    "employee",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

managers_table = Table(
    "manager",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("manager_data", String(50)),
)

engineers_table = Table(
    "engineer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("engineer_info", String(50)),
)
Next, the UNION is produced using polymorphic_union():
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "employee": employees_table,
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)
With the above Table objects, the mappings can be produced using “semi-classical” style, where we use Declarative in conjunction with the __table__ argument; our polymorphic union above is passed via __mapper_args__ to the Mapper.with_polymorphic parameter:
class Employee(Base):
    __table__ = employee_table
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": ("*", pjoin),
        "polymorphic_identity": "employee",
    }


class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }


class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
Alternatively, the same Table objects can be used in fully “classical” style, without using Declarative at all. A constructor similar to that supplied by Declarative is illustrated:
class Employee:
    def __init__(self, **kw):
        for k in kw:
            setattr(self, k, kw[k])


class Manager(Employee):
    pass


class Engineer(Employee):
    pass


employee_mapper = mapper_registry.map_imperatively(
    Employee,
    pjoin,
    with_polymorphic=("*", pjoin),
    polymorphic_on=pjoin.c.type,
)
manager_mapper = mapper_registry.map_imperatively(
    Manager,
    managers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="manager",
)
engineer_mapper = mapper_registry.map_imperatively(
    Engineer,
    engineers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="engineer",
)
The “abstract” example can also be mapped using “semi-classical” or “classical” style. The difference is that instead of applying the “polymorphic union” to the Mapper.with_polymorphic parameter, we apply it directly as the mapped selectable on our basemost mapper. The semi-classical mapping is illustrated below:
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)


class Employee(Base):
    __table__ = pjoin
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": "*",
        "polymorphic_identity": "employee",
    }


class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }


class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
Above, we use polymorphic_union() in the same manner as before, except that we omit the employee table.
See also
Imperative Mapping - background information on imperative, or “classical” mappings
Relationships with Concrete Inheritance
In a concrete inheritance scenario, mapping relationships is challenging since the distinct classes do not share a table. If the relationships only involve specific classes, such as a relationship between Company in our previous examples and Manager, special steps aren’t needed as these are just two related tables.
However, if Company is to have a one-to-many relationship to Employee, indicating that the collection may include both Engineer and Manager objects, that implies that Employee must have polymorphic loading capabilities and also that each table to be related must have a foreign key back to the company table. An example of such a configuration is as follows:
from sqlalchemy.ext.declarative import ConcreteBase


class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee")


class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
The next complexity with concrete inheritance and relationships involves when we’d like one or all of Employee, Manager and Engineer to themselves refer back to Company. For this case, SQLAlchemy has special behavior in that a relationship() placed on Employee which links to Company does not work against the Manager and Engineer classes, when exercised at the instance level. Instead, a distinct relationship() must be applied to each class. In order to achieve bi-directional behavior in terms of three separate relationships which serve as the opposite of Company.employees, the relationship.back_populates parameter is used between each of the relationships:
from sqlalchemy.ext.declarative import ConcreteBase


class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee", back_populates="company")


class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
The above limitation is related to the current implementation, including that concrete inheriting classes do not share any of the attributes of the superclass and therefore need distinct relationships to be set up.
Loading Concrete Inheritance Mappings
The options for loading with concrete inheritance are limited; generally, if polymorphic loading is configured on the mapper using one of the declarative concrete mixins, it can’t be modified at query time in current SQLAlchemy versions. Normally, the with_polymorphic() function would be able to override the style of loading used by concrete, however due to current limitations this is not yet supported.


Non-Traditional Mappings
Mapping a Class against Multiple Tables
Mappers can be constructed against arbitrary relational units (called selectables) in addition to plain tables. For example, the join() function creates a selectable unit comprised of multiple tables, complete with its own composite primary key, which can be mapped in the same way as a Table:
from sqlalchemy import Table, Column, Integer, String, MetaData, join, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import column_property

metadata_obj = MetaData()

# define two Table objects
user_table = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String),
)

address_table = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String),
)

# define a join between them.  This
# takes place across the user.id and address.user_id
# columns.
user_address_join = join(user_table, address_table)


class Base(DeclarativeBase):
    metadata = metadata_obj


# map to it
class AddressUser(Base):
    __table__ = user_address_join

    id = column_property(user_table.c.id, address_table.c.user_id)
    address_id = address_table.c.id
In the example above, the join expresses columns for both the user and the address table. The user.id and address.user_id columns are equated by foreign key, so in the mapping they are defined as one attribute, AddressUser.id, using column_property() to indicate a specialized column mapping. Based on this part of the configuration, the mapping will copy new primary key values from user.id into the address.user_id column when a flush occurs.
Additionally, the address.id column is mapped explicitly to an attribute named address_id. This is to disambiguate the mapping of the address.id column from the same-named AddressUser.id attribute, which here has been assigned to refer to the user table combined with the address.user_id foreign key.
The natural primary key of the above mapping is the composite of (user.id, address.id), as these are the primary key columns of the user and address table combined together. The identity of an AddressUser object will be in terms of these two values, and is represented from an AddressUser object as (AddressUser.id, AddressUser.address_id).
When referring to the AddressUser.id column, most SQL expressions will make use of only the first column in the list of columns mapped, as the two columns are synonymous. However, for the special use case such as a GROUP BY expression where both columns must be referenced at the same time while making use of the proper context, that is, accommodating for aliases and similar, the accessor Comparator.expressions may be used:
stmt = select(AddressUser).group_by(*AddressUser.id.expressions)
Added in version 1.3.17: Added the Comparator.expressions accessor.
Note
A mapping against multiple tables as illustrated above supports persistence, that is, INSERT, UPDATE and DELETE of rows within the targeted tables. However, it does not support an operation that would UPDATE one table and perform INSERT or DELETE on others at the same time for one record. That is, if a record PtoQ is mapped to tables “p” and “q”, where it has a row based on a LEFT OUTER JOIN of “p” and “q”, if an UPDATE proceeds that is to alter data in the “q” table in an existing record, the row in “q” must exist; it won’t emit an INSERT if the primary key identity is already present. If the row does not exist, for most DBAPI drivers which support reporting the number of rows affected by an UPDATE, the ORM will fail to detect an updated row and raise an error; otherwise, the data would be silently ignored.
A recipe to allow for an on-the-fly “insert” of the related row might make use of the .MapperEvents.before_update event and look like:
from sqlalchemy import event


@event.listens_for(PtoQ, "before_update")
def receive_before_update(mapper, connection, target):
    if target.some_required_attr_on_q is None:
        connection.execute(q_table.insert(), {"id": target.id})
where above, a row is INSERTed into the q_table table by creating an INSERT construct with Table.insert(), then executing it using the given Connection which is the same one being used to emit other SQL for the flush process. The user-supplied logic would have to detect that the LEFT OUTER JOIN from “p” to “q” does not have an entry for the “q” side.
Mapping a Class against Arbitrary Subqueries
Similar to mapping against a join, a plain select() object can be used with a mapper as well. The example fragment below illustrates mapping a class called Customer to a select() which includes a join to a subquery:
from sqlalchemy import select, func

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)


class Customer(Base):
    __table__ = customer_select
Above, the full row represented by customer_select will be all the columns of the customers table, in addition to those columns exposed by the subq subquery, which are order_count, highest_order, and customer_id. Mapping the Customer class to this selectable then creates a class which will contain those attributes.
When the ORM persists new instances of Customer, only the customers table will actually receive an INSERT. This is because the primary key of the orders table is not represented in the mapping; the ORM will only emit an INSERT into a table for which it has mapped the primary key.
Note
The practice of mapping to arbitrary SELECT statements, especially complex ones as above, is almost never needed; it necessarily tends to produce complex queries which are often less efficient than that which would be produced by direct query construction. The practice is to some degree based on the very early history of SQLAlchemy where the Mapper construct was meant to represent the primary querying interface; in modern usage, the Query object can be used to construct virtually any SELECT statement, including complex composites, and should be favored over the “map-to-selectable” approach.
Multiple Mappers for One Class
In modern SQLAlchemy, a particular class is mapped by only one so-called primary mapper at a time. This mapper is involved in three main areas of functionality: querying, persistence, and instrumentation of the mapped class. The rationale of the primary mapper relates to the fact that the Mapper modifies the class itself, not only persisting it towards a particular Table, but also instrumenting attributes upon the class which are structured specifically according to the table metadata. It’s not possible for more than one mapper to be associated with a class in equal measure, since only one mapper can actually instrument the class.
The concept of a “non-primary” mapper had existed for many versions of SQLAlchemy however as of version 1.3 this feature is deprecated. The one case where such a non-primary mapper is useful is when constructing a relationship to a class against an alternative selectable. This use case is now suited using the aliased construct and is described at Relationship to Aliased Class.
As far as the use case of a class that can actually be fully persisted to different tables under different scenarios, very early versions of SQLAlchemy offered a feature for this adapted from Hibernate, known as the “entity name” feature. However, this use case became infeasible within SQLAlchemy once the mapped class itself became the source of SQL expression construction; that is, the class’ attributes themselves link directly to mapped table columns. The feature was removed and replaced with a simple recipe-oriented approach to accomplishing this task without any ambiguity of instrumentation - to create new subclasses, each mapped individually. This pattern is now available as a recipe at Entity Name.


Configuring a Version Counter
The Mapper supports management of a version id column, which is a single table column that increments or otherwise updates its value each time an UPDATE to the mapped table occurs. This value is checked each time the ORM emits an UPDATE or DELETE against the row to ensure that the value held in memory matches the database value.
Warning
Because the versioning feature relies upon comparison of the in memory record of an object, the feature only applies to the Session.flush() process, where the ORM flushes individual in-memory rows to the database. It does not take effect when performing a multirow UPDATE or DELETE using Query.update() or Query.delete() methods, as these methods only emit an UPDATE or DELETE statement but otherwise do not have direct access to the contents of those rows being affected.
The purpose of this feature is to detect when two concurrent transactions are modifying the same row at roughly the same time, or alternatively to provide a guard against the usage of a “stale” row in a system that might be re-using data from a previous transaction without refreshing (e.g. if one sets expire_on_commit=False with a Session, it is possible to re-use the data from a previous transaction).
Concurrent transaction updates
When detecting concurrent updates within transactions, it is typically the case that the database’s transaction isolation level is below the level of repeatable read; otherwise, the transaction will not be exposed to a new row value created by a concurrent update which conflicts with the locally updated value. In this case, the SQLAlchemy versioning feature will typically not be useful for in-transaction conflict detection, though it still can be used for cross-transaction staleness detection.
The database that enforces repeatable reads will typically either have locked the target row against a concurrent update, or is employing some form of multi version concurrency control such that it will emit an error when the transaction is committed. SQLAlchemy’s version_id_col is an alternative which allows version tracking to occur for specific tables within a transaction that otherwise might not have this isolation level set.
See also
Repeatable Read Isolation Level - PostgreSQL’s implementation of repeatable read, including a description of the error condition.
Simple Version Counting
The most straightforward way to track versions is to add an integer column to the mapped table, then establish it as the version_id_col within the mapper options:
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_id = mapped_column(Integer, nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_id}
Note
It is strongly recommended that the version_id column be made NOT NULL. The versioning feature does not support a NULL value in the versioning column.
Above, the User mapping tracks integer versions using the column version_id. When an object of type User is first flushed, the version_id column will be given a value of “1”. Then, an UPDATE of the table later on will always be emitted in a manner similar to the following:
UPDATE user SET version_id=:version_id, name=:name
WHERE user.id = :user_id AND user.version_id = :user_version_id
-- {"name": "new name", "version_id": 2, "user_id": 1, "user_version_id": 1}
The above UPDATE statement is updating the row that not only matches user.id = 1, it also is requiring that user.version_id = 1, where “1” is the last version identifier we’ve been known to use on this object. If a transaction elsewhere has modified the row independently, this version id will no longer match, and the UPDATE statement will report that no rows matched; this is the condition that SQLAlchemy tests, that exactly one row matched our UPDATE (or DELETE) statement. If zero rows match, that indicates our version of the data is stale, and a StaleDataError is raised.
Custom Version Counters / Types
Other kinds of values or counters can be used for versioning. Common types include dates and GUIDs. When using an alternate type or counter scheme, SQLAlchemy provides a hook for this scheme using the version_id_generator argument, which accepts a version generation callable. This callable is passed the value of the current known version, and is expected to return the subsequent version.
For example, if we wanted to track the versioning of our User class using a randomly generated GUID, we could do this (note that some backends support a native GUID type, but we illustrate here using a simple string):
import uuid


class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {
        "version_id_col": version_uuid,
        "version_id_generator": lambda version: uuid.uuid4().hex,
    }
The persistence engine will call upon uuid.uuid4() each time a User object is subject to an INSERT or an UPDATE. In this case, our version generation function can disregard the incoming value of version, as the uuid4() function generates identifiers without any prerequisite value. If we were using a sequential versioning scheme such as numeric or a special character system, we could make use of the given version in order to help determine the subsequent value.
See also
Backend-agnostic GUID Type
Server Side Version Counters
The version_id_generator can also be configured to rely upon a value that is generated by the database. In this case, the database would need some means of generating new identifiers when a row is subject to an INSERT as well as with an UPDATE. For the UPDATE case, typically an update trigger is needed, unless the database in question supports some other native version identifier. The PostgreSQL database in particular supports a system column called xmin which provides UPDATE versioning. We can make use of the PostgreSQL xmin column to version our User class as follows:
from sqlalchemy import FetchedValue


class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    xmin = mapped_column("xmin", String, system=True, server_default=FetchedValue())

    __mapper_args__ = {"version_id_col": xmin, "version_id_generator": False}
With the above mapping, the ORM will rely upon the xmin column for automatically providing the new value of the version id counter.
creating tables that refer to system columns
In the above scenario, as xmin is a system column provided by PostgreSQL, we use the system=True argument to mark it as a system-provided column, omitted from the CREATE TABLE statement. The datatype of this column is an internal PostgreSQL type called xid which acts mostly like a string, so we use the String datatype.
The ORM typically does not actively fetch the values of database-generated values when it emits an INSERT or UPDATE, instead leaving these columns as “expired” and to be fetched when they are next accessed, unless the eager_defaults Mapper flag is set. However, when a server side version column is used, the ORM needs to actively fetch the newly generated value. This is so that the version counter is set up before any concurrent transaction may update it again. This fetching is also best done simultaneously within the INSERT or UPDATE statement using RETURNING, otherwise if emitting a SELECT statement afterwards, there is still a potential race condition where the version counter may change before it can be fetched.
When the target database supports RETURNING, an INSERT statement for our User class will look like this:
INSERT INTO "user" (name) VALUES (%(name)s) RETURNING "user".id, "user".xmin
-- {'name': 'ed'}
Where above, the ORM can acquire any newly generated primary key values along with server-generated version identifiers in one statement. When the backend does not support RETURNING, an additional SELECT must be emitted for every INSERT and UPDATE, which is much less efficient, and also introduces the possibility of missed version counters:
INSERT INTO "user" (name) VALUES (%(name)s)
-- {'name': 'ed'}

SELECT "user".version_id AS user_version_id FROM "user" where
"user".id = :param_1
-- {"param_1": 1}
It is strongly recommended that server side version counters only be used when absolutely necessary and only on backends that support RETURNING, currently PostgreSQL, Oracle Database, MariaDB 10.5, SQLite 3.35, and SQL Server.
Programmatic or Conditional Version Counters
When version_id_generator is set to False, we can also programmatically (and conditionally) set the version identifier on our object in the same way we assign any other mapped attribute. Such as if we used our UUID example, but set version_id_generator to False, we can set the version identifier at our choosing:
import uuid


class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_uuid, "version_id_generator": False}


u1 = User(name="u1", version_uuid=uuid.uuid4().hex)

session.add(u1)

session.commit()

u1.name = "u2"
u1.version_uuid = uuid.uuid4().hex

session.commit()
We can update our User object without incrementing the version counter as well; the value of the counter will remain unchanged, and the UPDATE statement will still check against the previous value. This may be useful for schemes where only certain classes of UPDATE are sensitive to concurrency issues:
# will leave version_uuid unchanged
u1.name = "u3"
session.commit()


Class Mapping API
Object NameDescriptionadd_mapped_attribute(target, key, attr)Add a new mapped attribute to an ORM mapped class.as_declarative(**kw)Class decorator which will adapt a given class into a declarative_base().class_mapper(class_[, configure])Given a class, return the primary Mapper associated with the key.clear_mappers()Remove all mappers from all classes.column_property(column, *additional_columns, [group, deferred, raiseload, comparator_factory, init, repr, default, default_factory, compare, kw_only, hash, active_history, expire_on_flush, info, doc])Provide a column-level property for use with a mapping.configure_mappers()Initialize the inter-mapper relationships of all mappers that have been constructed thus far across all registry collections.declarative_base(*, [metadata, mapper, cls, name, class_registry, type_annotation_map, constructor, metaclass])Construct a base class for declarative class definitions.declarative_mixin(cls)Mark a class as providing the feature of “declarative mixin”.DeclarativeBaseBase class used for declarative class definitions.DeclarativeBaseNoMetaSame as DeclarativeBase, but does not use a metaclass to intercept new attributes.declared_attrMark a class-level method as representing the definition of a mapped property or Declarative directive.has_inherited_table(cls)Given a class, return True if any of the classes it inherits from has a mapped table, otherwise return False.identity_key([class_, ident], *, [instance, row, identity_token])Generate “identity key” tuples, as are used as keys in the Session.identity_map dictionary.mapped_column([__name_pos, __type_pos], *args, [init, repr, default, default_factory, compare, kw_only, hash, nullable, primary_key, deferred, deferred_group, deferred_raiseload, use_existing_column, name, type_, autoincrement, doc, key, index, unique, info, onupdate, insert_default, server_default, server_onupdate, active_history, quote, system, comment, sort_order], **kw)declare a new ORM-mapped Column construct for use within Declarative Table configuration.MappedAsDataclassMixin class to indicate when mapping this class, also convert it to be a dataclass.MappedClassProtocolA protocol representing a SQLAlchemy mapped class.MapperDefines an association between a Python class and a database table or other relational structure, so that ORM operations against the class may proceed.object_mapper(instance)Given an object, return the primary Mapper associated with the object instance.orm_insert_sentinel([name, type_], *, [default, omit_from_statements])Provides a surrogate mapped_column() that generates a so-called sentinel column, allowing efficient bulk inserts with deterministic RETURNING sorting for tables that don’t otherwise have qualifying primary key configurations.polymorphic_union(table_map, typecolname[, aliasname, cast_nulls])Create a UNION statement used by a polymorphic mapper.reconstructor(fn)Decorate a method as the ‘reconstructor’ hook.registryGeneralized registry for mapping classes.synonym_for(name[, map_column])Decorator that produces an synonym() attribute in conjunction with a Python descriptor.class sqlalchemy.orm.registry
Generalized registry for mapping classes.
The registry serves as the basis for maintaining a collection of mappings, and provides configurational hooks used to map classes.
The three general kinds of mappings supported are Declarative Base, Declarative Decorator, and Imperative Mapping. All of these mapping styles may be used interchangeably:
• registry.generate_base() returns a new declarative base class, and is the underlying implementation of the declarative_base() function.
• registry.mapped() provides a class decorator that will apply declarative mapping to a class without the use of a declarative base class.
• registry.map_imperatively() will produce a Mapper for a class without scanning the class for declarative class attributes. This method suits the use case historically provided by the sqlalchemy.orm.mapper() classical mapping function, which is removed as of SQLAlchemy 2.0.
Added in version 1.4.
Members
__init__(), as_declarative_base(), configure(), dispose(), generate_base(), map_declaratively(), map_imperatively(), mapped(), mapped_as_dataclass(), mappers, update_type_annotation_map()
See also
ORM Mapped Class Overview - overview of class mapping styles.
method sqlalchemy.orm.registry.__init__(*, metadata: Optional[MetaData] = None, class_registry: Optional[clsregistry._ClsRegistryType] = None, type_annotation_map: Optional[_TypeAnnotationMapType] = None, constructor: Callable[, None] = <function _declarative_constructor>)
Construct a new registry
Parameters:
• metadata – An optional MetaData instance. All Table objects generated using declarative table mapping will make use of this MetaData collection. If this argument is left at its default of None, a blank MetaData collection is created.
• constructor – Specify the implementation for the __init__ function on a mapped class that has no __init__ of its own. Defaults to an implementation that assigns **kwargs for declared fields and relationships to an instance. If None is supplied, no __init__ will be provided and construction will fall back to cls.__init__ by way of the normal Python semantics.
• class_registry – optional dictionary that will serve as the registry of class names-> mapped classes when string names are used to identify classes inside of relationship() and others. Allows two or more declarative base classes to share the same registry of class names for simplified inter-base relationships.
• type_annotation_map – 
optional dictionary of Python types to SQLAlchemy TypeEngine classes or instances. The provided dict will update the default type mapping. This is used exclusively by the MappedColumn construct to produce column types based on annotations within the Mapped type.
Added in version 2.0.
See also
Customizing the Type Map
method sqlalchemy.orm.registry.as_declarative_base(**kw: Any) → Callable[[Type[_T]], Type[_T]]
Class decorator which will invoke registry.generate_base() for a given base class.
E.g.:
from sqlalchemy.orm import registry

mapper_registry = registry()


@mapper_registry.as_declarative_base()
class Base:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    id = Column(Integer, primary_key=True)


class MyMappedClass(Base): 
All keyword arguments passed to registry.as_declarative_base() are passed along to registry.generate_base().
method sqlalchemy.orm.registry.configure(cascade: bool = False) → None
Configure all as-yet unconfigured mappers in this registry.
The configure step is used to reconcile and initialize the relationship() linkages between mapped classes, as well as to invoke configuration events such as the MapperEvents.before_configured() and MapperEvents.after_configured(), which may be used by ORM extensions or user-defined extension hooks.
If one or more mappers in this registry contain relationship() constructs that refer to mapped classes in other registries, this registry is said to be dependent on those registries. In order to configure those dependent registries automatically, the configure.cascade flag should be set to True. Otherwise, if they are not configured, an exception will be raised. The rationale behind this behavior is to allow an application to programmatically invoke configuration of registries while controlling whether or not the process implicitly reaches other registries.
As an alternative to invoking registry.configure(), the ORM function configure_mappers() function may be used to ensure configuration is complete for all registry objects in memory. This is generally simpler to use and also predates the usage of registry objects overall. However, this function will impact all mappings throughout the running Python process and may be more memory/time consuming for an application that has many registries in use for different purposes that may not be needed immediately.
See also
configure_mappers()
Added in version 1.4.0b2.
method sqlalchemy.orm.registry.dispose(cascade: bool = False) → None
Dispose of all mappers in this registry.
After invocation, all the classes that were mapped within this registry will no longer have class instrumentation associated with them. This method is the per-registry analogue to the application-wide clear_mappers() function.
If this registry contains mappers that are dependencies of other registries, typically via relationship() links, then those registries must be disposed as well. When such registries exist in relation to this one, their registry.dispose() method will also be called, if the dispose.cascade flag is set to True; otherwise, an error is raised if those registries were not already disposed.
Added in version 1.4.0b2.
See also
clear_mappers()
method sqlalchemy.orm.registry.generate_base(mapper: ~typing.Callable[[], ~sqlalchemy.orm.mapper.Mapper[~typing.Any]] | None = None, cls: ~typing.Type[~typing.Any] = <class 'object'>, name: str = 'Base', metaclass: ~typing.Type[~typing.Any] = <class 'sqlalchemy.orm.decl_api.DeclarativeMeta'>) → Any
Generate a declarative base class.
Classes that inherit from the returned class object will be automatically mapped using declarative mapping.
E.g.:
from sqlalchemy.orm import registry

mapper_registry = registry()

Base = mapper_registry.generate_base()


class MyClass(Base):
    __tablename__ = "my_table"
    id = Column(Integer, primary_key=True)
The above dynamically generated class is equivalent to the non-dynamic example below:
from sqlalchemy.orm import registry
from sqlalchemy.orm.decl_api import DeclarativeMeta

mapper_registry = registry()


class Base(metaclass=DeclarativeMeta):
    __abstract__ = True
    registry = mapper_registry
    metadata = mapper_registry.metadata

    __init__ = mapper_registry.constructor
Changed in version 2.0: Note that the registry.generate_base() method is superseded by the new DeclarativeBase class, which generates a new “base” class using subclassing, rather than return value of a function. This allows an approach that is compatible with PEP 484 typing tools.
The registry.generate_base() method provides the implementation for the declarative_base() function, which creates the registry and base class all at once.
See the section Declarative Mapping for background and examples.
Parameters:
• mapper – An optional callable, defaults to Mapper. This function is used to generate new Mapper objects.
• cls – Defaults to object. A type to use as the base for the generated declarative base class. May be a class or tuple of classes.
• name – Defaults to Base. The display name for the generated class. Customizing this is not required, but can improve clarity in tracebacks and debugging.
• metaclass – Defaults to DeclarativeMeta. A metaclass or __metaclass__ compatible callable to use as the meta type of the generated declarative base class.
See also
Declarative Mapping
declarative_base()
method sqlalchemy.orm.registry.map_declaratively(cls: Type[_O]) → Mapper[_O]
Map a class declaratively.
In this form of mapping, the class is scanned for mapping information, including for columns to be associated with a table, and/or an actual table object.
Returns the Mapper object.
E.g.:
from sqlalchemy.orm import registry

mapper_registry = registry()


class Foo:
    __tablename__ = "some_table"

    id = Column(Integer, primary_key=True)
    name = Column(String)


mapper = mapper_registry.map_declaratively(Foo)
This function is more conveniently invoked indirectly via either the registry.mapped() class decorator or by subclassing a declarative metaclass generated from registry.generate_base().
See the section Declarative Mapping for complete details and examples.
Parameters:
cls – class to be mapped.
Returns:
a Mapper object.
See also
Declarative Mapping
registry.mapped() - more common decorator interface to this function.
registry.map_imperatively()
method sqlalchemy.orm.registry.map_imperatively(class_: Type[_O], local_table: FromClause | None = None, **kw: Any) → Mapper[_O]
Map a class imperatively.
In this form of mapping, the class is not scanned for any mapping information. Instead, all mapping constructs are passed as arguments.
This method is intended to be fully equivalent to the now-removed SQLAlchemy mapper() function, except that it’s in terms of a particular registry.
E.g.:
from sqlalchemy.orm import registry

mapper_registry = registry()

my_table = Table(
    "my_table",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
)


class MyClass:
    pass


mapper_registry.map_imperatively(MyClass, my_table)
See the section Imperative Mapping for complete background and usage examples.
Parameters:
• class_ – The class to be mapped. Corresponds to the Mapper.class_ parameter.
• local_table – the Table or other FromClause object that is the subject of the mapping. Corresponds to the Mapper.local_table parameter.
• **kw – all other keyword arguments are passed to the Mapper constructor directly.
See also
Imperative Mapping
Declarative Mapping
method sqlalchemy.orm.registry.mapped(cls: Type[_O]) → Type[_O]
Class decorator that will apply the Declarative mapping process to a given class.
E.g.:
from sqlalchemy.orm import registry

mapper_registry = registry()


@mapper_registry.mapped
class Foo:
    __tablename__ = "some_table"

    id = Column(Integer, primary_key=True)
    name = Column(String)
See the section Declarative Mapping for complete details and examples.
Parameters:
cls – class to be mapped.
Returns:
the class that was passed.
See also
Declarative Mapping
registry.generate_base() - generates a base class that will apply Declarative mapping to subclasses automatically using a Python metaclass.
See also
registry.mapped_as_dataclass()
method sqlalchemy.orm.registry.mapped_as_dataclass(_registry__cls: Type[_O] | None = None, *, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, eq: _NoArg | bool = _NoArg.NO_ARG, order: _NoArg | bool = _NoArg.NO_ARG, unsafe_hash: _NoArg | bool = _NoArg.NO_ARG, match_args: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, dataclass_callable: _NoArg | Callable[, Type[Any]] = _NoArg.NO_ARG) → Type[_O] | Callable[[Type[_O]], Type[_O]]
Class decorator that will apply the Declarative mapping process to a given class, and additionally convert the class to be a Python dataclass.
See also
Declarative Dataclass Mapping - complete background on SQLAlchemy native dataclass mapping
Added in version 2.0.
attribute sqlalchemy.orm.registry.mappers
read only collection of all Mapper objects.
method sqlalchemy.orm.registry.update_type_annotation_map(type_annotation_map: _TypeAnnotationMapType) → None
update the registry.type_annotation_map with new values.
function sqlalchemy.orm.add_mapped_attribute(target: Type[_O], key: str, attr: MapperProperty[Any]) → None
Add a new mapped attribute to an ORM mapped class.
E.g.:
add_mapped_attribute(User, "addresses", relationship(Address))
This may be used for ORM mappings that aren’t using a declarative metaclass that intercepts attribute set operations.
Added in version 2.0.
function sqlalchemy.orm.column_property(column: _ORMColumnExprArgument[_T], *additional_columns: _ORMColumnExprArgument[Any], group: str | None = None, deferred: bool = False, raiseload: bool = False, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, active_history: bool = False, expire_on_flush: bool = True, info: _InfoType | None = None, doc: str | None = None) → MappedSQLExpression[_T]
Provide a column-level property for use with a mapping.
With Declarative mappings, column_property() is used to map read-only SQL expressions to a mapped class.
When using Imperative mappings, column_property() also takes on the role of mapping table columns with additional features. When using fully Declarative mappings, the mapped_column() construct should be used for this purpose.
With Declarative Dataclass mappings, column_property() is considered to be read only, and will not be included in the Dataclass __init__() constructor.
The column_property() function returns an instance of ColumnProperty.
See also
Using column_property - general use of column_property() to map SQL expressions
Applying Load, Persistence and Mapping Options for Imperative Table Columns - usage of column_property() with Imperative Table mappings to apply additional options to a plain Column object
Parameters:
• *cols – list of Column objects to be mapped.
• active_history=False – Used only for Imperative Table mappings, or legacy-style Declarative mappings (i.e. which have not been upgraded to mapped_column()), for column-based attributes that are expected to be writeable; use mapped_column() with mapped_column.active_history for Declarative mappings. See that parameter for functional details.
• comparator_factory – a class which extends Comparator which provides custom SQL clause generation for comparison operations.
• group – a group name for this property when marked as deferred.
• deferred – when True, the column property is “deferred”, meaning that it does not load immediately, and is instead loaded when the attribute is first accessed on an instance. See also deferred().
• doc – optional string that will be applied as the doc on the class-bound descriptor.
• expire_on_flush=True – Disable expiry on flush. A column_property() which refers to a SQL expression (and not a single table-bound column) is considered to be a “read only” property; populating it has no effect on the state of data, and it can only return database state. For this reason a column_property()’s value is expired whenever the parent object is involved in a flush, that is, has any kind of “dirty” state within a flush. Setting this parameter to False will have the effect of leaving any existing value present after the flush proceeds. Note that the Session with default expiration settings still expires all attributes after a Session.commit() call, however.
• info – Optional data dictionary which will be populated into the MapperProperty.info attribute of this object.
• raiseload – 
if True, indicates the column should raise an error when undeferred, rather than loading the value. This can be altered at query time by using the deferred() option with raiseload=False.
Added in version 1.4.
See also
Using raiseload to prevent deferred column loads
• init – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __init__() method as generated by the dataclass process.
• repr – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __repr__() method as generated by the dataclass process.
• default_factory – 
Specific to Declarative Dataclass Mapping, specifies a default-value generation function that will take place as part of the __init__() method as generated by the dataclass process.
See also
What are default, default_factory and insert_default and what should I use?
mapped_column.default
mapped_column.insert_default
• compare – 
Specific to Declarative Dataclass Mapping, indicates if this field should be included in comparison operations when generating the __eq__() and __ne__() methods for the mapped class.
Added in version 2.0.0b4.
• kw_only – Specific to Declarative Dataclass Mapping, indicates if this field should be marked as keyword-only when generating the __init__().
• hash – 
Specific to Declarative Dataclass Mapping, controls if this field is included when generating the __hash__() method for the mapped class.
Added in version 2.0.36.
function sqlalchemy.orm.declarative_base(*, metadata: Optional[MetaData] = None, mapper: Optional[Callable[, Mapper[Any]]] = None, cls: Type[Any] = <class 'object'>, name: str = 'Base', class_registry: Optional[clsregistry._ClsRegistryType] = None, type_annotation_map: Optional[_TypeAnnotationMapType] = None, constructor: Callable[, None] = <function _declarative_constructor>, metaclass: Type[Any] = <class 'sqlalchemy.orm.decl_api.DeclarativeMeta'>) → Any
Construct a base class for declarative class definitions.
The new base class will be given a metaclass that produces appropriate Table objects and makes the appropriate Mapper calls based on the information provided declaratively in the class and any subclasses of the class.
Changed in version 2.0: Note that the declarative_base() function is superseded by the new DeclarativeBase class, which generates a new “base” class using subclassing, rather than return value of a function. This allows an approach that is compatible with PEP 484 typing tools.
The declarative_base() function is a shorthand version of using the registry.generate_base() method. That is, the following:
from sqlalchemy.orm import declarative_base

Base = declarative_base()
Is equivalent to:
from sqlalchemy.orm import registry

mapper_registry = registry()
Base = mapper_registry.generate_base()
See the docstring for registry and registry.generate_base() for more details.
Changed in version 1.4: The declarative_base() function is now a specialization of the more generic registry class. The function also moves to the sqlalchemy.orm package from the declarative.ext package.
Parameters:
• metadata – An optional MetaData instance. All Table objects implicitly declared by subclasses of the base will share this MetaData. A MetaData instance will be created if none is provided. The MetaData instance will be available via the metadata attribute of the generated declarative base class.
• mapper – An optional callable, defaults to Mapper. Will be used to map subclasses to their Tables.
• cls – Defaults to object. A type to use as the base for the generated declarative base class. May be a class or tuple of classes.
• name – Defaults to Base. The display name for the generated class. Customizing this is not required, but can improve clarity in tracebacks and debugging.
• constructor – Specify the implementation for the __init__ function on a mapped class that has no __init__ of its own. Defaults to an implementation that assigns **kwargs for declared fields and relationships to an instance. If None is supplied, no __init__ will be provided and construction will fall back to cls.__init__ by way of the normal Python semantics.
• class_registry – optional dictionary that will serve as the registry of class names-> mapped classes when string names are used to identify classes inside of relationship() and others. Allows two or more declarative base classes to share the same registry of class names for simplified inter-base relationships.
• type_annotation_map – 
optional dictionary of Python types to SQLAlchemy TypeEngine classes or instances. This is used exclusively by the MappedColumn construct to produce column types based on annotations within the Mapped type.
Added in version 2.0.
See also
Customizing the Type Map
• metaclass – Defaults to DeclarativeMeta. A metaclass or __metaclass__ compatible callable to use as the meta type of the generated declarative base class.
See also
registry
function sqlalchemy.orm.declarative_mixin(cls: Type[_T]) → Type[_T]
Mark a class as providing the feature of “declarative mixin”.
E.g.:
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import declarative_mixin


@declarative_mixin
class MyMixin:

    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"always_refresh": True}

    id = Column(Integer, primary_key=True)


class MyModel(MyMixin, Base):
    name = Column(String(1000))
The declarative_mixin() decorator currently does not modify the given class in any way; it’s current purpose is strictly to assist the Mypy plugin in being able to identify SQLAlchemy declarative mixin classes when no other context is present.
Added in version 1.4.6.
Legacy Feature
This api is considered legacy and will be deprecated in the next SQLAlchemy version.
See also
Composing Mapped Hierarchies with Mixins
Using @declared_attr and Declarative Mixins - in the Mypy plugin documentation
function sqlalchemy.orm.as_declarative(**kw: Any) → Callable[[Type[_T]], Type[_T]]
Class decorator which will adapt a given class into a declarative_base().
This function makes use of the registry.as_declarative_base() method, by first creating a registry automatically and then invoking the decorator.
E.g.:
from sqlalchemy.orm import as_declarative


@as_declarative()
class Base:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    id = Column(Integer, primary_key=True)


class MyMappedClass(Base): 
See also
registry.as_declarative_base()
function sqlalchemy.orm.mapped_column(__name_pos: str | _TypeEngineArgument[Any] | SchemaEventTarget | None = None, __type_pos: _TypeEngineArgument[Any] | SchemaEventTarget | None = None, *args: SchemaEventTarget, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, nullable: bool | Literal[SchemaConst.NULL_UNSPECIFIED] | None = SchemaConst.NULL_UNSPECIFIED, primary_key: bool | None = False, deferred: _NoArg | bool = _NoArg.NO_ARG, deferred_group: str | None = None, deferred_raiseload: bool | None = None, use_existing_column: bool = False, name: str | None = None, type_: _TypeEngineArgument[Any] | None = None, autoincrement: _AutoIncrementType = 'auto', doc: str | None = None, key: str | None = None, index: bool | None = None, unique: bool | None = None, info: _InfoType | None = None, onupdate: Any | None = None, insert_default: Any | None = _NoArg.NO_ARG, server_default: _ServerDefaultArgument | None = None, server_onupdate: _ServerOnUpdateArgument | None = None, active_history: bool = False, quote: bool | None = None, system: bool = False, comment: str | None = None, sort_order: _NoArg | int = _NoArg.NO_ARG, **kw: Any) → MappedColumn[Any]
declare a new ORM-mapped Column construct for use within Declarative Table configuration.
The mapped_column() function provides an ORM-aware and Python-typing-compatible construct which is used with declarative mappings to indicate an attribute that’s mapped to a Core Column object. It provides the equivalent feature as mapping an attribute to a Column object directly when using Declarative, specifically when using Declarative Table configuration.
Added in version 2.0.
mapped_column() is normally used with explicit typing along with the Mapped annotation type, where it can derive the SQL type and nullability for the column based on what’s present within the Mapped annotation. It also may be used without annotations as a drop-in replacement for how Column is used in Declarative mappings in SQLAlchemy 1.x style.
For usage examples of mapped_column(), see the documentation at Declarative Table with mapped_column().
See also
Declarative Table with mapped_column() - complete documentation
ORM Declarative Models - migration notes for Declarative mappings using 1.x style mappings
Parameters:
• __name – String name to give to the Column. This is an optional, positional only argument that if present must be the first positional argument passed. If omitted, the attribute name to which the mapped_column() is mapped will be used as the SQL column name.
• __type – TypeEngine type or instance which will indicate the datatype to be associated with the Column. This is an optional, positional-only argument that if present must immediately follow the __name parameter if present also, or otherwise be the first positional parameter. If omitted, the ultimate type for the column may be derived either from the annotated type, or if a ForeignKey is present, from the datatype of the referenced column.
• *args – Additional positional arguments include constructs such as ForeignKey, CheckConstraint, and Identity, which are passed through to the constructed Column.
• nullable – Optional bool, whether the column should be “NULL” or “NOT NULL”. If omitted, the nullability is derived from the type annotation based on whether or not typing.Optional (or its equivalent) is present. nullable defaults to True otherwise for non-primary key columns, and False for primary key columns.
• primary_key – optional bool, indicates the Column would be part of the table’s primary key or not.
• deferred – 
Optional bool - this keyword argument is consumed by the ORM declarative process, and is not part of the Column itself; instead, it indicates that this column should be “deferred” for loading as though mapped by deferred().
See also
Configuring Column Deferral on Mappings
• deferred_group – 
Implies mapped_column.deferred to True, and set the deferred.group parameter.
See also
Loading deferred columns in groups
• deferred_raiseload – 
Implies mapped_column.deferred to True, and set the deferred.raiseload parameter.
See also
Using raiseload to prevent deferred column loads
• use_existing_column – 
if True, will attempt to locate the given column name on an inherited superclass (typically single inheriting superclass), and if present, will not produce a new column, mapping to the superclass column as though it were omitted from this class. This is used for mixins that add new columns to an inherited superclass.
See also
Resolving Column Conflicts with use_existing_column
Added in version 2.0.0b4.
• default – 
Passed directly to the Column.default parameter if the mapped_column.insert_default parameter is not present. Additionally, when used with Declarative Dataclass Mapping, indicates a default Python value that should be applied to the keyword constructor within the generated __init__() method.
Note that in the case of dataclass generation when mapped_column.insert_default is not present, this means the mapped_column.default value is used in two places, both the __init__() method as well as the Column.default parameter. While this behavior may change in a future release, for the moment this tends to “work out”; a default of None will mean that the Column gets no default generator, whereas a default that refers to a non-None Python or SQL expression value will be assigned up front on the object when __init__() is called, which is the same value that the Core Insert construct would use in any case, leading to the same end result.
Note
When using Core level column defaults that are callables to be interpreted by the underlying Column in conjunction with ORM-mapped dataclasses, especially those that are context-aware default functions, the mapped_column.insert_default parameter must be used instead. This is necessary to disambiguate the callable from being interpreted as a dataclass level default.
See also
What are default, default_factory and insert_default and what should I use?
mapped_column.insert_default
mapped_column.default_factory
• insert_default – 
Passed directly to the Column.default parameter; will supersede the value of mapped_column.default when present, however mapped_column.default will always apply to the constructor default for a dataclasses mapping.
See also
What are default, default_factory and insert_default and what should I use?
mapped_column.default
mapped_column.default_factory
• sort_order – 
An integer that indicates how this mapped column should be sorted compared to the others when the ORM is creating a Table. Among mapped columns that have the same value the default ordering is used, placing first the mapped columns defined in the main class, then the ones in the super classes. Defaults to 0. The sort is ascending.
Added in version 2.0.4.
• active_history=False – 
When True, indicates that the “previous” value for a scalar attribute should be loaded when replaced, if not already loaded. Normally, history tracking logic for simple non-primary-key scalar values only needs to be aware of the “new” value in order to perform a flush. This flag is available for applications that make use of get_history() or Session.is_modified() which also need to know the “previous” value of the attribute.
Added in version 2.0.10.
• init – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __init__() method as generated by the dataclass process.
• repr – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __repr__() method as generated by the dataclass process.
• default_factory – 
Specific to Declarative Dataclass Mapping, specifies a default-value generation function that will take place as part of the __init__() method as generated by the dataclass process.
See also
What are default, default_factory and insert_default and what should I use?
mapped_column.default
mapped_column.insert_default
• compare – 
Specific to Declarative Dataclass Mapping, indicates if this field should be included in comparison operations when generating the __eq__() and __ne__() methods for the mapped class.
Added in version 2.0.0b4.
• kw_only – Specific to Declarative Dataclass Mapping, indicates if this field should be marked as keyword-only when generating the __init__().
• hash – 
Specific to Declarative Dataclass Mapping, controls if this field is included when generating the __hash__() method for the mapped class.
Added in version 2.0.36.
• **kw – All remaining keyword arguments are passed through to the constructor for the Column.
class sqlalchemy.orm.declared_attr
Mark a class-level method as representing the definition of a mapped property or Declarative directive.
declared_attr is typically applied as a decorator to a class level method, turning the attribute into a scalar-like property that can be invoked from the uninstantiated class. The Declarative mapping process looks for these declared_attr callables as it scans classes, and assumes any attribute marked with declared_attr will be a callable that will produce an object specific to the Declarative mapping or table configuration.
declared_attr is usually applicable to mixins, to define relationships that are to be applied to different implementors of the class. It may also be used to define dynamically generated column expressions and other Declarative attributes.
Example:
class ProvidesUserMixin:
    "A mixin that adds a 'user' relationship to classes."

    user_id: Mapped[int] = mapped_column(ForeignKey("user_table.id"))

    @declared_attr
    def user(cls) -> Mapped["User"]:
        return relationship("User")
When used with Declarative directives such as __tablename__, the declared_attr.directive() modifier may be used which indicates to PEP 484 typing tools that the given method is not dealing with Mapped attributes:
class CreateTableName:
    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()
declared_attr can also be applied directly to mapped classes, to allow for attributes that dynamically configure themselves on subclasses when using mapped inheritance schemes. Below illustrates declared_attr to create a dynamic scheme for generating the Mapper.polymorphic_identity parameter for subclasses:
class Employee(Base):
    __tablename__ = "employee"

    id: Mapped[int] = mapped_column(primary_key=True)
    type: Mapped[str] = mapped_column(String(50))

    @declared_attr.directive
    def __mapper_args__(cls) -> Dict[str, Any]:
        if cls.__name__ == "Employee":
            return {
                "polymorphic_on": cls.type,
                "polymorphic_identity": "Employee",
            }
        else:
            return {"polymorphic_identity": cls.__name__}


class Engineer(Employee):
    pass
declared_attr supports decorating functions that are explicitly decorated with @classmethod. This is never necessary from a runtime perspective, however may be needed in order to support PEP 484 typing tools that don’t otherwise recognize the decorated function as having class-level behaviors for the cls parameter:
class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    @classmethod
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)
Added in version 2.0: - declared_attr can accommodate a function decorated with @classmethod to help with PEP 484 integration where needed.
See also
Composing Mapped Hierarchies with Mixins - Declarative Mixin documentation with background on use patterns for declared_attr.
Members
cascading, directive
Class signature
class sqlalchemy.orm.declared_attr (sqlalchemy.orm.base._MappedAttribute, sqlalchemy.orm.decl_api._declared_attr_common)
attribute sqlalchemy.orm.declared_attr.cascading
Mark a declared_attr as cascading.
This is a special-use modifier which indicates that a column or MapperProperty-based declared attribute should be configured distinctly per mapped subclass, within a mapped-inheritance scenario.
Warning
The declared_attr.cascading modifier has several limitations:
• The flag only applies to the use of declared_attr on declarative mixin classes and __abstract__ classes; it currently has no effect when used on a mapped class directly.
• The flag only applies to normally-named attributes, e.g. not any special underscore attributes such as __tablename__. On these attributes it has no effect.
• The flag currently does not allow further overrides down the class hierarchy; if a subclass tries to override the attribute, a warning is emitted and the overridden attribute is skipped. This is a limitation that it is hoped will be resolved at some point.
Below, both MyClass as well as MySubClass will have a distinct id Column object established:
class HasIdMixin:
    @declared_attr.cascading
    def id(cls) -> Mapped[int]:
        if has_inherited_table(cls):
            return mapped_column(ForeignKey("myclass.id"), primary_key=True)
        else:
            return mapped_column(Integer, primary_key=True)


class MyClass(HasIdMixin, Base):
    __tablename__ = "myclass"
    # 


class MySubClass(MyClass):
    """ """

    # 
The behavior of the above configuration is that MySubClass will refer to both its own id column as well as that of MyClass underneath the attribute named some_id.
See also
Declarative Inheritance
Using _orm.declared_attr() to generate table-specific inheriting columns
attribute sqlalchemy.orm.declared_attr.directive
Mark a declared_attr as decorating a Declarative directive such as __tablename__ or __mapper_args__.
The purpose of declared_attr.directive is strictly to support PEP 484 typing tools, by allowing the decorated function to have a return type that is not using the Mapped generic class, as would normally be the case when declared_attr is used for columns and mapped properties. At runtime, the declared_attr.directive returns the declared_attr class unmodified.
E.g.:
class CreateTableName:
    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()
Added in version 2.0.
See also
Composing Mapped Hierarchies with Mixins
declared_attr
class sqlalchemy.orm.DeclarativeBase
Base class used for declarative class definitions.
The DeclarativeBase allows for the creation of new declarative bases in such a way that is compatible with type checkers:
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
The above Base class is now usable as the base for new declarative mappings. The superclass makes use of the __init_subclass__() method to set up new classes and metaclasses aren’t used.
When first used, the DeclarativeBase class instantiates a new registry to be used with the base, assuming one was not provided explicitly. The DeclarativeBase class supports class-level attributes which act as parameters for the construction of this registry; such as to indicate a specific MetaData collection as well as a specific value for registry.type_annotation_map:
from typing_extensions import Annotated

from sqlalchemy import BigInteger
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase

bigint = Annotated[int, "bigint"]
my_metadata = MetaData()


class Base(DeclarativeBase):
    metadata = my_metadata
    type_annotation_map = {
        str: String().with_variant(String(255), "mysql", "mariadb"),
        bigint: BigInteger(),
    }
Class-level attributes which may be specified include:
Parameters:
• metadata – optional MetaData collection. If a registry is constructed automatically, this MetaData collection will be used to construct it. Otherwise, the local MetaData collection will supercede that used by an existing registry passed using the DeclarativeBase.registry parameter.
• type_annotation_map – optional type annotation map that will be passed to the registry as registry.type_annotation_map.
• registry – supply a pre-existing registry directly.
Added in version 2.0: Added DeclarativeBase, so that declarative base classes may be constructed in such a way that is also recognized by PEP 484 type checkers. As a result, DeclarativeBase and other subclassing-oriented APIs should be seen as superseding previous “class returned by a function” APIs, namely declarative_base() and registry.generate_base(), where the base class returned cannot be recognized by type checkers without using plugins.
__init__ behavior
In a plain Python class, the base-most __init__() method in the class hierarchy is object.__init__(), which accepts no arguments. However, when the DeclarativeBase subclass is first declared, the class is given an __init__() method that links to the registry.constructor constructor function, if no __init__() method is already present; this is the usual declarative constructor that will assign keyword arguments as attributes on the instance, assuming those attributes are established at the class level (i.e. are mapped, or are linked to a descriptor). This constructor is never accessed by a mapped class without being called explicitly via super(), as mapped classes are themselves given an __init__() method directly which calls registry.constructor, so in the default case works independently of what the base-most __init__() method does.
Changed in version 2.0.1: DeclarativeBase has a default constructor that links to registry.constructor by default, so that calls to super().__init__() can access this constructor. Previously, due to an implementation mistake, this default constructor was missing, and calling super().__init__() would invoke object.__init__().
The DeclarativeBase subclass may also declare an explicit __init__() method which will replace the use of the registry.constructor function at this level:
class Base(DeclarativeBase):
    def __init__(self, id=None):
        self.id = id
Mapped classes still will not invoke this constructor implicitly; it remains only accessible by calling super().__init__():
class MyClass(Base):
    def __init__(self, id=None, name=None):
        self.name = name
        super().__init__(id=id)
Note that this is a different behavior from what functions like the legacy declarative_base() would do; the base created by those functions would always install registry.constructor for __init__().
Members
__mapper__, __mapper_args__, __table__, __table_args__, __tablename__, metadata, registry
Class signature
class sqlalchemy.orm.DeclarativeBase (sqlalchemy.inspection.Inspectable)
attribute sqlalchemy.orm.DeclarativeBase.__mapper__: ClassVar[Mapper[Any]]
The Mapper object to which a particular class is mapped.
May also be acquired using inspect(), e.g. inspect(klass).
attribute sqlalchemy.orm.DeclarativeBase.__mapper_args__: Any
Dictionary of arguments which will be passed to the Mapper constructor.
See also
Mapper Configuration Options with Declarative
attribute sqlalchemy.orm.DeclarativeBase.__table__: ClassVar[FromClause]
The FromClause to which a particular subclass is mapped.
This is usually an instance of Table but may also refer to other kinds of FromClause such as Subquery, depending on how the class is mapped.
See also
Accessing Table and Metadata
attribute sqlalchemy.orm.DeclarativeBase.__table_args__: Any
A dictionary or tuple of arguments that will be passed to the Table constructor. See Declarative Table Configuration for background on the specific structure of this collection.
See also
Declarative Table Configuration
attribute sqlalchemy.orm.DeclarativeBase.__tablename__: Any
String name to assign to the generated Table object, if not specified directly via DeclarativeBase.__table__.
See also
Declarative Table with mapped_column()
attribute sqlalchemy.orm.DeclarativeBase.metadata: ClassVar[MetaData]
Refers to the MetaData collection that will be used for new Table objects.
See also
Accessing Table and Metadata
attribute sqlalchemy.orm.DeclarativeBase.registry: ClassVar[registry]
Refers to the registry in use where new Mapper objects will be associated.
class sqlalchemy.orm.DeclarativeBaseNoMeta
Same as DeclarativeBase, but does not use a metaclass to intercept new attributes.
The DeclarativeBaseNoMeta base may be used when use of custom metaclasses is desirable.
Added in version 2.0.
Members
__mapper__, __mapper_args__, __table__, __table_args__, __tablename__, metadata, registry
Class signature
class sqlalchemy.orm.DeclarativeBaseNoMeta (sqlalchemy.inspection.Inspectable)
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.__mapper__: ClassVar[Mapper[Any]]
The Mapper object to which a particular class is mapped.
May also be acquired using inspect(), e.g. inspect(klass).
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.__mapper_args__: Any
Dictionary of arguments which will be passed to the Mapper constructor.
See also
Mapper Configuration Options with Declarative
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.__table__: FromClause | None
The FromClause to which a particular subclass is mapped.
This is usually an instance of Table but may also refer to other kinds of FromClause such as Subquery, depending on how the class is mapped.
See also
Accessing Table and Metadata
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.__table_args__: Any
A dictionary or tuple of arguments that will be passed to the Table constructor. See Declarative Table Configuration for background on the specific structure of this collection.
See also
Declarative Table Configuration
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.__tablename__: Any
String name to assign to the generated Table object, if not specified directly via DeclarativeBase.__table__.
See also
Declarative Table with mapped_column()
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.metadata: ClassVar[MetaData]
Refers to the MetaData collection that will be used for new Table objects.
See also
Accessing Table and Metadata
attribute sqlalchemy.orm.DeclarativeBaseNoMeta.registry: ClassVar[registry]
Refers to the registry in use where new Mapper objects will be associated.
function sqlalchemy.orm.has_inherited_table(cls: Type[_O]) → bool
Given a class, return True if any of the classes it inherits from has a mapped table, otherwise return False.
This is used in declarative mixins to build attributes that behave differently for the base class vs. a subclass in an inheritance hierarchy.
See also
Using Mixins and Base Classes with Mapped Inheritance Patterns
function sqlalchemy.orm.synonym_for(name: str, map_column: bool = False) → Callable[[Callable[[], Any]], Synonym[Any]]
Decorator that produces an synonym() attribute in conjunction with a Python descriptor.
The function being decorated is passed to synonym() as the synonym.descriptor parameter:
class MyClass(Base):
    __tablename__ = "my_table"

    id = Column(Integer, primary_key=True)
    _job_status = Column("job_status", String(50))

    @synonym_for("job_status")
    @property
    def job_status(self):
        return "Status: %s" % self._job_status
The hybrid properties feature of SQLAlchemy is typically preferred instead of synonyms, which is a more legacy feature.
See also
Synonyms - Overview of synonyms
synonym() - the mapper-level function
Using Descriptors and Hybrids - The Hybrid Attribute extension provides an updated approach to augmenting attribute behavior more flexibly than can be achieved with synonyms.
function sqlalchemy.orm.object_mapper(instance: _T) → Mapper[_T]
Given an object, return the primary Mapper associated with the object instance.
Raises sqlalchemy.orm.exc.UnmappedInstanceError if no mapping is configured.
This function is available via the inspection system as:
inspect(instance).mapper
Using the inspection system will raise sqlalchemy.exc.NoInspectionAvailable if the instance is not part of a mapping.
function sqlalchemy.orm.class_mapper(class_: Type[_O], configure: bool = True) → Mapper[_O]
Given a class, return the primary Mapper associated with the key.
Raises UnmappedClassError if no mapping is configured on the given class, or ArgumentError if a non-class object is passed.
Equivalent functionality is available via the inspect() function as:
inspect(some_mapped_class)
Using the inspection system will raise sqlalchemy.exc.NoInspectionAvailable if the class is not mapped.
function sqlalchemy.orm.configure_mappers() → None
Initialize the inter-mapper relationships of all mappers that have been constructed thus far across all registry collections.
The configure step is used to reconcile and initialize the relationship() linkages between mapped classes, as well as to invoke configuration events such as the MapperEvents.before_configured() and MapperEvents.after_configured(), which may be used by ORM extensions or user-defined extension hooks.
Mapper configuration is normally invoked automatically, the first time mappings from a particular registry are used, as well as whenever mappings are used and additional not-yet-configured mappers have been constructed. The automatic configuration process however is local only to the registry involving the target mapper and any related registry objects which it may depend on; this is equivalent to invoking the registry.configure() method on a particular registry.
By contrast, the configure_mappers() function will invoke the configuration process on all registry objects that exist in memory, and may be useful for scenarios where many individual registry objects that are nonetheless interrelated are in use.
Changed in version 1.4: As of SQLAlchemy 1.4.0b2, this function works on a per-registry basis, locating all registry objects present and invoking the registry.configure() method on each. The registry.configure() method may be preferred to limit the configuration of mappers to those local to a particular registry and/or declarative base class.
Points at which automatic configuration is invoked include when a mapped class is instantiated into an instance, as well as when ORM queries are emitted using Session.query() or Session.execute() with an ORM-enabled statement.
The mapper configure process, whether invoked by configure_mappers() or from registry.configure(), provides several event hooks that can be used to augment the mapper configuration step. These hooks include:
• MapperEvents.before_configured() - called once before configure_mappers() or registry.configure() does any work; this can be used to establish additional options, properties, or related mappings before the operation proceeds.
• MapperEvents.mapper_configured() - called as each individual Mapper is configured within the process; will include all mapper state except for backrefs set up by other mappers that are still to be configured.
• MapperEvents.after_configured() - called once after configure_mappers() or registry.configure() is complete; at this stage, all Mapper objects that fall within the scope of the configuration operation will be fully configured. Note that the calling application may still have other mappings that haven’t been produced yet, such as if they are in modules as yet unimported, and may also have mappings that are still to be configured, if they are in other registry collections not part of the current scope of configuration.
function sqlalchemy.orm.clear_mappers() → None
Remove all mappers from all classes.
Changed in version 1.4: This function now locates all registry objects and calls upon the registry.dispose() method of each.
This function removes all instrumentation from classes and disposes of their associated mappers. Once called, the classes are unmapped and can be later re-mapped with new mappers.
clear_mappers() is not for normal use, as there is literally no valid usage for it outside of very specific testing scenarios. Normally, mappers are permanent structural components of user-defined classes, and are never discarded independently of their class. If a mapped class itself is garbage collected, its mapper is automatically disposed of as well. As such, clear_mappers() is only for usage in test suites that re-use the same classes with different mappings, which is itself an extremely rare use case - the only such use case is in fact SQLAlchemy’s own test suite, and possibly the test suites of other ORM extension libraries which intend to test various combinations of mapper construction upon a fixed set of classes.
function sqlalchemy.orm.util.identity_key(class_: Type[_T] | None = None, ident: Any | Tuple[Any, ] = None, *, instance: _T | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[_T]
Generate “identity key” tuples, as are used as keys in the Session.identity_map dictionary.
This function has several call styles:
• identity_key(class, ident, identity_token=token)
This form receives a mapped class and a primary key scalar or tuple as an argument.
E.g.:
 identity_key(MyClass, (1, 2))
(<class '__main__.MyClass'>, (1, 2), None)
•  param class:
mapped class (must be a positional argument)
param ident:
primary key, may be a scalar or tuple argument.
param identity_token:
optional identity token
Added in version 1.2: added identity_token
•  identity_key(instance=instance)
This form will produce the identity key for a given instance. The instance need not be persistent, only that its primary key attributes are populated (else the key will contain None for those missing values).
E.g.:
 instance = MyClass(1, 2)
 identity_key(instance=instance)
(<class '__main__.MyClass'>, (1, 2), None)
•  In this form, the given instance is ultimately run though Mapper.identity_key_from_instance(), which will have the effect of performing a database check for the corresponding row if the object is expired.
param instance:
object instance (must be given as a keyword arg)
•  identity_key(class, row=row, identity_token=token)
This form is similar to the class/tuple form, except is passed a database result row as a Row or RowMapping object.
E.g.:
 row = engine.execute(text("select * from table where a=1 and b=2")).first()
 identity_key(MyClass, row=row)
(<class '__main__.MyClass'>, (1, 2), None)
• param class:
mapped class (must be a positional argument)
param row:
Row row returned by a CursorResult (must be given as a keyword arg)
param identity_token:
optional identity token
Added in version 1.2: added identity_token
function sqlalchemy.orm.polymorphic_union(table_map, typecolname, aliasname='p_union', cast_nulls=True)
Create a UNION statement used by a polymorphic mapper.
See Concrete Table Inheritance for an example of how this is used.
Parameters:
• table_map – mapping of polymorphic identities to Table objects.
• typecolname – string name of a “discriminator” column, which will be derived from the query, producing the polymorphic identity for each row. If None, no polymorphic discriminator is generated.
• aliasname – name of the alias() construct generated.
• cast_nulls – if True, non-existent columns, which are represented as labeled NULLs, will be passed into CAST. This is a legacy behavior that is problematic on some backends such as Oracle - in which case it can be set to False.
function sqlalchemy.orm.orm_insert_sentinel(name: str | None = None, type_: _TypeEngineArgument[Any] | None = None, *, default: Any | None = None, omit_from_statements: bool = True) → MappedColumn[Any]
Provides a surrogate mapped_column() that generates a so-called sentinel column, allowing efficient bulk inserts with deterministic RETURNING sorting for tables that don’t otherwise have qualifying primary key configurations.
Use of orm_insert_sentinel() is analogous to the use of the insert_sentinel() construct within a Core Table construct.
Guidelines for adding this construct to a Declarative mapped class are the same as that of the insert_sentinel() construct; the database table itself also needs to have a column with this name present.
For background on how this object is used, see the section Configuring Sentinel Columns as part of the section “Insert Many Values” Behavior for INSERT statements.
See also
insert_sentinel()
“Insert Many Values” Behavior for INSERT statements
Configuring Sentinel Columns
Added in version 2.0.10.
function sqlalchemy.orm.reconstructor(fn: _Fn) → _Fn
Decorate a method as the ‘reconstructor’ hook.
Designates a single method as the “reconstructor”, an __init__-like method that will be called by the ORM after the instance has been loaded from the database or otherwise reconstituted.
Tip
The reconstructor() decorator makes use of the InstanceEvents.load() event hook, which can be used directly.
The reconstructor will be invoked with no arguments. Scalar (non-collection) database-mapped attributes of the instance will be available for use within the function. Eagerly-loaded collections are generally not yet available and will usually only contain the first element. ORM state changes made to objects at this stage will not be recorded for the next flush() operation, so the activity within a reconstructor should be conservative.
See also
InstanceEvents.load()
class sqlalchemy.orm.Mapper
Defines an association between a Python class and a database table or other relational structure, so that ORM operations against the class may proceed.
The Mapper object is instantiated using mapping methods present on the registry object. For information about instantiating new Mapper objects, see ORM Mapped Class Overview.
Members
__init__(), add_properties(), add_property(), all_orm_descriptors, attrs, base_mapper, c, cascade_iterator(), class_, class_manager, column_attrs, columns, common_parent(), composites, concrete, configured, entity, get_property(), get_property_by_column(), identity_key_from_instance(), identity_key_from_primary_key(), identity_key_from_row(), inherits, is_mapper, is_sibling(), isa(), iterate_properties, local_table, mapped_table, mapper, non_primary, persist_selectable, polymorphic_identity, polymorphic_iterator(), polymorphic_map, polymorphic_on, primary_key, primary_key_from_instance(), primary_mapper(), relationships, selectable, self_and_descendants, single, synonyms, tables, validators, with_polymorphic_mappers
Class signature
class sqlalchemy.orm.Mapper (sqlalchemy.orm.ORMFromClauseRole, sqlalchemy.orm.ORMEntityColumnsClauseRole, sqlalchemy.sql.cache_key.MemoizedHasCacheKey, sqlalchemy.orm.base.InspectionAttr, sqlalchemy.log.Identified, sqlalchemy.inspection.Inspectable, sqlalchemy.event.registry.EventTarget, typing.Generic)
method sqlalchemy.orm.Mapper.__init__(class_: Type[_O], local_table: FromClause | None = None, properties: Mapping[str, MapperProperty[Any]] | None = None, primary_key: Iterable[_ORMColumnExprArgument[Any]] | None = None, non_primary: bool = False, inherits: Mapper[Any] | Type[Any] | None = None, inherit_condition: _ColumnExpressionArgument[bool] | None = None, inherit_foreign_keys: Sequence[_ORMColumnExprArgument[Any]] | None = None, always_refresh: bool = False, version_id_col: _ORMColumnExprArgument[Any] | None = None, version_id_generator: Literal[False] | Callable[[Any], Any] | None = None, polymorphic_on: _ORMColumnExprArgument[Any] | str | MapperProperty[Any] | None = None, _polymorphic_map: Dict[Any, Mapper[Any]] | None = None, polymorphic_identity: Any | None = None, concrete: bool = False, with_polymorphic: _WithPolymorphicArg | None = None, polymorphic_abstract: bool = False, polymorphic_load: Literal['selectin', 'inline'] | None = None, allow_partial_pks: bool = True, batch: bool = True, column_prefix: str | None = None, include_properties: Sequence[str] | None = None, exclude_properties: Sequence[str] | None = None, passive_updates: bool = True, passive_deletes: bool = False, confirm_deleted_rows: bool = True, eager_defaults: Literal[True, False, 'auto'] = 'auto', legacy_is_orphan: bool = False, _compiled_cache_size: int = 100)
Direct constructor for a new Mapper object.
The Mapper constructor is not called directly, and is normally invoked through the use of the registry object through either the Declarative or Imperative mapping styles.
Changed in version 2.0: The public facing mapper() function is removed; for a classical mapping configuration, use the registry.map_imperatively() method.
Parameters documented below may be passed to either the registry.map_imperatively() method, or may be passed in the __mapper_args__ declarative class attribute described at Mapper Configuration Options with Declarative.
Parameters:
• class_ – The class to be mapped. When using Declarative, this argument is automatically passed as the declared class itself.
• local_table – The Table or other FromClause (i.e. selectable) to which the class is mapped. May be None if this mapper inherits from another mapper using single-table inheritance. When using Declarative, this argument is automatically passed by the extension, based on what is configured via the DeclarativeBase.__table__ attribute or via the Table produced as a result of the DeclarativeBase.__tablename__ attribute being present.
• polymorphic_abstract – 
Indicates this class will be mapped in a polymorphic hierarchy, but not directly instantiated. The class is mapped normally, except that it has no requirement for a Mapper.polymorphic_identity within an inheritance hierarchy. The class however must be part of a polymorphic inheritance scheme which uses Mapper.polymorphic_on at the base.
Added in version 2.0.
See also
Building Deeper Hierarchies with polymorphic_abstract
• always_refresh – If True, all query operations for this mapped class will overwrite all data within object instances that already exist within the session, erasing any in-memory changes with whatever information was loaded from the database. Usage of this flag is highly discouraged; as an alternative, see the method Query.populate_existing().
• allow_partial_pks – 
Defaults to True. Indicates that a composite primary key with some NULL values should be considered as possibly existing within the database. This affects whether a mapper will assign an incoming row to an existing identity, as well as if Session.merge() will check the database first for a particular primary key value. A “partial primary key” can occur if one has mapped to an OUTER JOIN, for example.
The Mapper.allow_partial_pks parameter also indicates to the ORM relationship lazy loader, when loading a many-to-one related object, if a composite primary key that has partial NULL values should result in an attempt to load from the database, or if a load attempt is not necessary.
Added in version 2.0.36: Mapper.allow_partial_pks is consulted by the relationship lazy loader strategy, such that when set to False, a SELECT for a composite primary key that has partial NULL values will not be emitted.
• batch – Defaults to True, indicating that save operations of multiple entities can be batched together for efficiency. Setting to False indicates that an instance will be fully saved before saving the next instance. This is used in the extremely rare case that a MapperEvents listener requires being called in between individual row persistence operations.
• column_prefix – 
A string which will be prepended to the mapped attribute name when Column objects are automatically assigned as attributes to the mapped class. Does not affect Column objects that are mapped explicitly in the Mapper.properties dictionary.
This parameter is typically useful with imperative mappings that keep the Table object separate. Below, assuming the user_table Table object has columns named user_id, user_name, and password:
class User(Base):
    __table__ = user_table
    __mapper_args__ = {"column_prefix": "_"}
•  The above mapping will assign the user_id, user_name, and password columns to attributes named _user_id, _user_name, and _password on the mapped User class.
The Mapper.column_prefix parameter is uncommon in modern use. For dealing with reflected tables, a more flexible approach to automating a naming scheme is to intercept the Column objects as they are reflected; see the section Automating Column Naming Schemes from Reflected Tables for notes on this usage pattern.
•  concrete – 
If True, indicates this mapper should use concrete table inheritance with its parent mapper.
See the section Concrete Table Inheritance for an example.
•  confirm_deleted_rows – defaults to True; when a DELETE occurs of one more rows based on specific primary keys, a warning is emitted when the number of rows matched does not equal the number of rows expected. This parameter may be set to False to handle the case where database ON DELETE CASCADE rules may be deleting some of those rows automatically. The warning may be changed to an exception in a future release.
•  eager_defaults – 
if True, the ORM will immediately fetch the value of server-generated default values after an INSERT or UPDATE, rather than leaving them as expired to be fetched on next access. This can be used for event schemes where the server-generated values are needed immediately before the flush completes.
The fetch of values occurs either by using RETURNING inline with the INSERT or UPDATE statement, or by adding an additional SELECT statement subsequent to the INSERT or UPDATE, if the backend does not support RETURNING.
The use of RETURNING is extremely performant in particular for INSERT statements where SQLAlchemy can take advantage of insertmanyvalues, whereas the use of an additional SELECT is relatively poor performing, adding additional SQL round trips which would be unnecessary if these new attributes are not to be accessed in any case.
For this reason, Mapper.eager_defaults defaults to the string value "auto", which indicates that server defaults for INSERT should be fetched using RETURNING if the backing database supports it and if the dialect in use supports “insertmanyreturning” for an INSERT statement. If the backing database does not support RETURNING or “insertmanyreturning” is not available, server defaults will not be fetched.
Changed in version 2.0.0rc1: added the “auto” option for Mapper.eager_defaults
See also
Fetching Server-Generated Defaults
Changed in version 2.0.0: RETURNING now works with multiple rows INSERTed at once using the insertmanyvalues feature, which among other things allows the Mapper.eager_defaults feature to be very performant on supporting backends.
•  exclude_properties – 
A list or set of string column names to be excluded from mapping.
See also
Mapping a Subset of Table Columns
•  include_properties – 
An inclusive list or set of string column names to map.
See also
Mapping a Subset of Table Columns
•  inherits – 
A mapped class or the corresponding Mapper of one indicating a superclass to which this Mapper should inherit from. The mapped class here must be a subclass of the other mapper’s class. When using Declarative, this argument is passed automatically as a result of the natural class hierarchy of the declared classes.
See also
Mapping Class Inheritance Hierarchies
•  inherit_condition – For joined table inheritance, a SQL expression which will define how the two tables are joined; defaults to a natural join between the two tables.
•  inherit_foreign_keys – When inherit_condition is used and the columns present are missing a ForeignKey configuration, this parameter can be used to specify which columns are “foreign”. In most cases can be left as None.
•  legacy_is_orphan – 
Boolean, defaults to False. When True, specifies that “legacy” orphan consideration is to be applied to objects mapped by this mapper, which means that a pending (that is, not persistent) object is auto-expunged from an owning Session only when it is de-associated from all parents that specify a delete-orphan cascade towards this mapper. The new default behavior is that the object is auto-expunged when it is de-associated with any of its parents that specify delete-orphan cascade. This behavior is more consistent with that of a persistent object, and allows behavior to be consistent in more scenarios independently of whether or not an orphan object has been flushed yet or not.
See the change note and example at The consideration of a “pending” object as an “orphan” has been made more aggressive for more detail on this change.
•  non_primary – 
Specify that this Mapper is in addition to the “primary” mapper, that is, the one used for persistence. The Mapper created here may be used for ad-hoc mapping of the class to an alternate selectable, for loading only.
See also
Relationship to Aliased Class - the new pattern that removes the need for the Mapper.non_primary flag.
•  passive_deletes – 
Indicates DELETE behavior of foreign key columns when a joined-table inheritance entity is being deleted. Defaults to False for a base mapper; for an inheriting mapper, defaults to False unless the value is set to True on the superclass mapper.
When True, it is assumed that ON DELETE CASCADE is configured on the foreign key relationships that link this mapper’s table to its superclass table, so that when the unit of work attempts to delete the entity, it need only emit a DELETE statement for the superclass table, and not this table.
When False, a DELETE statement is emitted for this mapper’s table individually. If the primary key attributes local to this table are unloaded, then a SELECT must be emitted in order to validate these attributes; note that the primary key columns of a joined-table subclass are not part of the “primary key” of the object as a whole.
Note that a value of True is always forced onto the subclass mappers; that is, it’s not possible for a superclass to specify passive_deletes without this taking effect for all subclass mappers.
See also
Using foreign key ON DELETE cascade with ORM relationships - description of similar feature as used with relationship()
mapper.passive_updates - supporting ON UPDATE CASCADE for joined-table inheritance mappers
•  passive_updates – 
Indicates UPDATE behavior of foreign key columns when a primary key column changes on a joined-table inheritance mapping. Defaults to True.
When True, it is assumed that ON UPDATE CASCADE is configured on the foreign key in the database, and that the database will handle propagation of an UPDATE from a source column to dependent columns on joined-table rows.
When False, it is assumed that the database does not enforce referential integrity and will not be issuing its own CASCADE operation for an update. The unit of work process will emit an UPDATE statement for the dependent columns during a primary key change.
See also
Mutable Primary Keys / Update Cascades - description of a similar feature as used with relationship()
mapper.passive_deletes - supporting ON DELETE CASCADE for joined-table inheritance mappers
•  polymorphic_load – 
Specifies “polymorphic loading” behavior for a subclass in an inheritance hierarchy (joined and single table inheritance only). Valid values are:
• “‘inline’” - specifies this class should be part of the “with_polymorphic” mappers, e.g. its columns will be included in a SELECT query against the base.
• “‘selectin’” - specifies that when instances of this class are loaded, an additional SELECT will be emitted to retrieve the columns specific to this subclass. The SELECT uses IN to fetch multiple subclasses at once.
Added in version 1.2.
See also
Configuring with_polymorphic() on mappers
Using selectin_polymorphic()
•  polymorphic_on – 
Specifies the column, attribute, or SQL expression used to determine the target class for an incoming row, when inheriting classes are present.
May be specified as a string attribute name, or as a SQL expression such as a Column or in a Declarative mapping a mapped_column() object. It is typically expected that the SQL expression corresponds to a column in the base-most mapped Table:
class Employee(Base):
    __tablename__ = "employee"

    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str] = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_on": discriminator,
        "polymorphic_identity": "employee",
    }
It may also be specified as a SQL expression, as in this example where we use the case() construct to provide a conditional approach:
class Employee(Base):
    __tablename__ = "employee"

    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str] = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_on": case(
            (discriminator == "EN", "engineer"),
            (discriminator == "MA", "manager"),
            else_="employee",
        ),
        "polymorphic_identity": "employee",
    }
It may also refer to any attribute using its string name, which is of particular use when using annotated column configurations:
class Employee(Base):
    __tablename__ = "employee"

    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "discriminator",
        "polymorphic_identity": "employee",
    }
When setting polymorphic_on to reference an attribute or expression that’s not present in the locally mapped Table, yet the value of the discriminator should be persisted to the database, the value of the discriminator is not automatically set on new instances; this must be handled by the user, either through manual means or via event listeners. A typical approach to establishing such a listener looks like:
from sqlalchemy import event
from sqlalchemy.orm import object_mapper


@event.listens_for(Employee, "init", propagate=True)
def set_identity(instance, *arg, **kw):
    mapper = object_mapper(instance)
    instance.discriminator = mapper.polymorphic_identity
•  Where above, we assign the value of polymorphic_identity for the mapped class to the discriminator attribute, thus persisting the value to the discriminator column in the database.
Warning
Currently, only one discriminator column may be set, typically on the base-most class in the hierarchy. “Cascading” polymorphic columns are not yet supported.
See also
Mapping Class Inheritance Hierarchies
•  polymorphic_identity – 
Specifies the value which identifies this particular class as returned by the column expression referred to by the Mapper.polymorphic_on setting. As rows are received, the value corresponding to the Mapper.polymorphic_on column expression is compared to this value, indicating which subclass should be used for the newly reconstructed object.
See also
Mapping Class Inheritance Hierarchies
•  properties – 
A dictionary mapping the string names of object attributes to MapperProperty instances, which define the persistence behavior of that attribute. Note that Column objects present in the mapped Table are automatically placed into ColumnProperty instances upon mapping, unless overridden. When using Declarative, this argument is passed automatically, based on all those MapperProperty instances declared in the declared class body.
See also
The properties dictionary - in the ORM Mapped Class Overview
•  primary_key – 
A list of Column objects, or alternatively string names of attribute names which refer to Column, which define the primary key to be used against this mapper’s selectable unit. This is normally simply the primary key of the local_table, but can be overridden here.
Changed in version 2.0.2: Mapper.primary_key arguments may be indicated as string attribute names as well.
See also
Mapping to an Explicit Set of Primary Key Columns - background and example use
•  version_id_col – 
A Column that will be used to keep a running version id of rows in the table. This is used to detect concurrent updates or the presence of stale data in a flush. The methodology is to detect if an UPDATE statement does not match the last known version id, a StaleDataError exception is thrown. By default, the column must be of Integer type, unless version_id_generator specifies an alternative version generator.
See also
Configuring a Version Counter - discussion of version counting and rationale.
•  version_id_generator – 
Define how new version ids should be generated. Defaults to None, which indicates that a simple integer counting scheme be employed. To provide a custom versioning scheme, provide a callable function of the form:
def generate_version(version):
    return next_version
• Alternatively, server-side versioning functions such as triggers, or programmatic versioning schemes outside of the version id generator may be used, by specifying the value False. Please see Server Side Version Counters for a discussion of important points when using this option.
See also
Custom Version Counters / Types
Server Side Version Counters
• with_polymorphic – 
A tuple in the form (<classes>, <selectable>) indicating the default style of “polymorphic” loading, that is, which tables are queried at once. <classes> is any single or list of mappers and/or classes indicating the inherited classes that should be loaded at once. The special value '*' may be used to indicate all descending classes should be loaded immediately. The second tuple argument <selectable> indicates a selectable that will be used to query for multiple classes.
The Mapper.polymorphic_load parameter may be preferable over the use of Mapper.with_polymorphic in modern mappings to indicate a per-subclass technique of indicating polymorphic loading styles.
See also
Configuring with_polymorphic() on mappers
method sqlalchemy.orm.Mapper.add_properties(dict_of_properties)
Add the given dictionary of properties to this mapper, using add_property.
method sqlalchemy.orm.Mapper.add_property(key: str, prop: Column[Any] | MapperProperty[Any]) → None
Add an individual MapperProperty to this mapper.
If the mapper has not been configured yet, just adds the property to the initial properties dictionary sent to the constructor. If this Mapper has already been configured, then the given MapperProperty is configured immediately.
attribute sqlalchemy.orm.Mapper.all_orm_descriptors
A namespace of all InspectionAttr attributes associated with the mapped class.
These attributes are in all cases Python descriptors associated with the mapped class or its superclasses.
This namespace includes attributes that are mapped to the class as well as attributes declared by extension modules. It includes any Python descriptor type that inherits from InspectionAttr. This includes QueryableAttribute, as well as extension types such as hybrid_property, hybrid_method and AssociationProxy.
To distinguish between mapped attributes and extension attributes, the attribute InspectionAttr.extension_type will refer to a constant that distinguishes between different extension types.
The sorting of the attributes is based on the following rules:
1. Iterate through the class and its superclasses in order from subclass to superclass (i.e. iterate through cls.__mro__)
2. For each class, yield the attributes in the order in which they appear in __dict__, with the exception of those in step 3 below. In Python 3.6 and above this ordering will be the same as that of the class’ construction, with the exception of attributes that were added after the fact by the application or the mapper.
3. If a certain attribute key is also in the superclass __dict__, then it’s included in the iteration for that class, and not the class in which it first appeared.
The above process produces an ordering that is deterministic in terms of the order in which attributes were assigned to the class.
Changed in version 1.3.19: ensured deterministic ordering for Mapper.all_orm_descriptors().
When dealing with a QueryableAttribute, the QueryableAttribute.property attribute refers to the MapperProperty property, which is what you get when referring to the collection of mapped properties via Mapper.attrs.
Warning
The Mapper.all_orm_descriptors accessor namespace is an instance of OrderedProperties. This is a dictionary-like object which includes a small number of named methods such as OrderedProperties.items() and OrderedProperties.values(). When accessing attributes dynamically, favor using the dict-access scheme, e.g. mapper.all_orm_descriptors[somename] over getattr(mapper.all_orm_descriptors, somename) to avoid name collisions.
See also
Mapper.attrs
attribute sqlalchemy.orm.Mapper.attrs
A namespace of all MapperProperty objects associated this mapper.
This is an object that provides each property based on its key name. For instance, the mapper for a User class which has User.name attribute would provide mapper.attrs.name, which would be the ColumnProperty representing the name column. The namespace object can also be iterated, which would yield each MapperProperty.
Mapper has several pre-filtered views of this attribute which limit the types of properties returned, including synonyms, column_attrs, relationships, and composites.
Warning
The Mapper.attrs accessor namespace is an instance of OrderedProperties. This is a dictionary-like object which includes a small number of named methods such as OrderedProperties.items() and OrderedProperties.values(). When accessing attributes dynamically, favor using the dict-access scheme, e.g. mapper.attrs[somename] over getattr(mapper.attrs, somename) to avoid name collisions.
See also
Mapper.all_orm_descriptors
attribute sqlalchemy.orm.Mapper.base_mapper: Mapper[Any]
The base-most Mapper in an inheritance chain.
In a non-inheriting scenario, this attribute will always be this Mapper. In an inheritance scenario, it references the Mapper which is parent to all other Mapper objects in the inheritance chain.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.c: ReadOnlyColumnCollection[str, Column[Any]]
A synonym for Mapper.columns.
method sqlalchemy.orm.Mapper.cascade_iterator(type_: str, state: InstanceState[_O], halt_on: Callable[[InstanceState[Any]], bool] | None = None) → Iterator[Tuple[object, Mapper[Any], InstanceState[Any], _InstanceDict]]
Iterate each element and its mapper in an object graph, for all relationships that meet the given cascade rule.
Parameters:
• type_ – 
The name of the cascade rule (i.e. "save-update", "delete", etc.).
Note
the "all" cascade is not accepted here. For a generic object traversal function, see How do I walk all objects that are related to a given object?.
• state – The lead InstanceState. child items will be processed per the relationships defined for this object’s mapper.
Returns:
the method yields individual object instances.
See also
Cascades
How do I walk all objects that are related to a given object? - illustrates a generic function to traverse all objects without relying on cascades.
attribute sqlalchemy.orm.Mapper.class_: Type[_O]
The class to which this Mapper is mapped.
attribute sqlalchemy.orm.Mapper.class_manager: ClassManager[_O]
The ClassManager which maintains event listeners and class-bound descriptors for this Mapper.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.column_attrs
Return a namespace of all ColumnProperty properties maintained by this Mapper.
See also
Mapper.attrs - namespace of all MapperProperty objects.
attribute sqlalchemy.orm.Mapper.columns: ReadOnlyColumnCollection[str, Column[Any]]
A collection of Column or other scalar expression objects maintained by this Mapper.
The collection behaves the same as that of the c attribute on any Table object, except that only those columns included in this mapping are present, and are keyed based on the attribute name defined in the mapping, not necessarily the key attribute of the Column itself. Additionally, scalar expressions mapped by column_property() are also present here.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
method sqlalchemy.orm.Mapper.common_parent(other: Mapper[Any]) → bool
Return true if the given mapper shares a common inherited parent as this mapper.
attribute sqlalchemy.orm.Mapper.composites
Return a namespace of all Composite properties maintained by this Mapper.
See also
Mapper.attrs - namespace of all MapperProperty objects.
attribute sqlalchemy.orm.Mapper.concrete: bool
Represent True if this Mapper is a concrete inheritance mapper.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.configured: bool = False
Represent True if this Mapper has been configured.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
See also
configure_mappers().
attribute sqlalchemy.orm.Mapper.entity
Part of the inspection API.
Returns self.class_.
method sqlalchemy.orm.Mapper.get_property(key: str, _configure_mappers: bool = False) → MapperProperty[Any]
return a MapperProperty associated with the given key.
method sqlalchemy.orm.Mapper.get_property_by_column(column: ColumnElement[_T]) → MapperProperty[_T]
Given a Column object, return the MapperProperty which maps this column.
method sqlalchemy.orm.Mapper.identity_key_from_instance(instance: _O) → _IdentityKeyType[_O]
Return the identity key for the given instance, based on its primary key attributes.
If the instance’s state is expired, calling this method will result in a database check to see if the object has been deleted. If the row no longer exists, ObjectDeletedError is raised.
This value is typically also found on the instance state under the attribute name key.
method sqlalchemy.orm.Mapper.identity_key_from_primary_key(primary_key: Tuple[Any, ], identity_token: Any | None = None) → _IdentityKeyType[_O]
Return an identity-map key for use in storing/retrieving an item from an identity map.
Parameters:
primary_key – A list of values indicating the identifier.
method sqlalchemy.orm.Mapper.identity_key_from_row(row: Row[Any] | RowMapping, identity_token: Any | None = None, adapter: ORMAdapter | None = None) → _IdentityKeyType[_O]
Return an identity-map key for use in storing/retrieving an item from the identity map.
Parameters:
row – 
A Row or RowMapping produced from a result set that selected from the ORM mapped primary key columns.
Changed in version 2.0: Row or RowMapping are accepted for the “row” argument
attribute sqlalchemy.orm.Mapper.inherits: Mapper[Any] | None
References the Mapper which this Mapper inherits from, if any.
attribute sqlalchemy.orm.Mapper.is_mapper = True
Part of the inspection API.
method sqlalchemy.orm.Mapper.is_sibling(other: Mapper[Any]) → bool
return true if the other mapper is an inheriting sibling to this one. common parent but different branch
method sqlalchemy.orm.Mapper.isa(other: Mapper[Any]) → bool
Return True if the this mapper inherits from the given mapper.
attribute sqlalchemy.orm.Mapper.iterate_properties
return an iterator of all MapperProperty objects.
attribute sqlalchemy.orm.Mapper.local_table: FromClause
The immediate FromClause to which this Mapper refers.
Typically is an instance of Table, may be any FromClause.
The “local” table is the selectable that the Mapper is directly responsible for managing from an attribute access and flush perspective. For non-inheriting mappers, Mapper.local_table will be the same as Mapper.persist_selectable. For inheriting mappers, Mapper.local_table refers to the specific portion of Mapper.persist_selectable that includes the columns to which this Mapper is loading/persisting, such as a particular Table within a join.
See also
Mapper.persist_selectable.
Mapper.selectable.
attribute sqlalchemy.orm.Mapper.mapped_table
Deprecated since version 1.3: Use .persist_selectable
attribute sqlalchemy.orm.Mapper.mapper
Part of the inspection API.
Returns self.
attribute sqlalchemy.orm.Mapper.non_primary: bool
Represent True if this Mapper is a “non-primary” mapper, e.g. a mapper that is used only to select rows but not for persistence management.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.persist_selectable: FromClause
The FromClause to which this Mapper is mapped.
Typically is an instance of Table, may be any FromClause.
The Mapper.persist_selectable is similar to Mapper.local_table, but represents the FromClause that represents the inheriting class hierarchy overall in an inheritance scenario.
:attr.`.Mapper.persist_selectable` is also separate from the Mapper.selectable attribute, the latter of which may be an alternate subquery used for selecting columns. :attr.`.Mapper.persist_selectable` is oriented towards columns that will be written on a persist operation.
See also
Mapper.selectable.
Mapper.local_table.
attribute sqlalchemy.orm.Mapper.polymorphic_identity: Any | None
Represent an identifier which is matched against the Mapper.polymorphic_on column during result row loading.
Used only with inheritance, this object can be of any type which is comparable to the type of column represented by Mapper.polymorphic_on.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
method sqlalchemy.orm.Mapper.polymorphic_iterator() → Iterator[Mapper[Any]]
Iterate through the collection including this mapper and all descendant mappers.
This includes not just the immediately inheriting mappers but all their inheriting mappers as well.
To iterate through an entire hierarchy, use mapper.base_mapper.polymorphic_iterator().
attribute sqlalchemy.orm.Mapper.polymorphic_map: Dict[Any, Mapper[Any]]
A mapping of “polymorphic identity” identifiers mapped to Mapper instances, within an inheritance scenario.
The identifiers can be of any type which is comparable to the type of column represented by Mapper.polymorphic_on.
An inheritance chain of mappers will all reference the same polymorphic map object. The object is used to correlate incoming result rows to target mappers.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.polymorphic_on: KeyedColumnElement[Any] | None
The Column or SQL expression specified as the polymorphic_on argument for this Mapper, within an inheritance scenario.
This attribute is normally a Column instance but may also be an expression, such as one derived from cast().
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.primary_key: Tuple[ColumnElement[Any], ]
An iterable containing the collection of Column objects which comprise the ‘primary key’ of the mapped table, from the perspective of this Mapper.
This list is against the selectable in Mapper.persist_selectable. In the case of inheriting mappers, some columns may be managed by a superclass mapper. For example, in the case of a Join, the primary key is determined by all of the primary key columns across all tables referenced by the Join.
The list is also not necessarily the same as the primary key column collection associated with the underlying tables; the Mapper features a primary_key argument that can override what the Mapper considers as primary key columns.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
method sqlalchemy.orm.Mapper.primary_key_from_instance(instance: _O) → Tuple[Any, ]
Return the list of primary key values for the given instance.
If the instance’s state is expired, calling this method will result in a database check to see if the object has been deleted. If the row no longer exists, ObjectDeletedError is raised.
method sqlalchemy.orm.Mapper.primary_mapper() → Mapper[Any]
Return the primary mapper corresponding to this mapper’s class key (class).
attribute sqlalchemy.orm.Mapper.relationships
A namespace of all Relationship properties maintained by this Mapper.
Warning
the Mapper.relationships accessor namespace is an instance of OrderedProperties. This is a dictionary-like object which includes a small number of named methods such as OrderedProperties.items() and OrderedProperties.values(). When accessing attributes dynamically, favor using the dict-access scheme, e.g. mapper.relationships[somename] over getattr(mapper.relationships, somename) to avoid name collisions.
See also
Mapper.attrs - namespace of all MapperProperty objects.
attribute sqlalchemy.orm.Mapper.selectable
The FromClause construct this Mapper selects from by default.
Normally, this is equivalent to persist_selectable, unless the with_polymorphic feature is in use, in which case the full “polymorphic” selectable is returned.
attribute sqlalchemy.orm.Mapper.self_and_descendants
The collection including this mapper and all descendant mappers.
This includes not just the immediately inheriting mappers but all their inheriting mappers as well.
attribute sqlalchemy.orm.Mapper.single: bool
Represent True if this Mapper is a single table inheritance mapper.
Mapper.local_table will be None if this flag is set.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.synonyms
Return a namespace of all Synonym properties maintained by this Mapper.
See also
Mapper.attrs - namespace of all MapperProperty objects.
attribute sqlalchemy.orm.Mapper.tables: Sequence[TableClause]
A sequence containing the collection of Table or TableClause objects which this Mapper is aware of.
If the mapper is mapped to a Join, or an Alias representing a Select, the individual Table objects that comprise the full construct will be represented here.
This is a read only attribute determined during mapper construction. Behavior is undefined if directly modified.
attribute sqlalchemy.orm.Mapper.validators: util.immutabledict[str, Tuple[str, Dict[str, Any]]]
An immutable dictionary of attributes which have been decorated using the validates() decorator.
The dictionary contains string attribute names as keys mapped to the actual validation method.
attribute sqlalchemy.orm.Mapper.with_polymorphic_mappers
The list of Mapper objects included in the default “polymorphic” query.
class sqlalchemy.orm.MappedAsDataclass
Mixin class to indicate when mapping this class, also convert it to be a dataclass.
See also
Declarative Dataclass Mapping - complete background on SQLAlchemy native dataclass mapping
Added in version 2.0.
class sqlalchemy.orm.MappedClassProtocol
A protocol representing a SQLAlchemy mapped class.
The protocol is generic on the type of class, use MappedClassProtocol[Any] to allow any mapped class.
Class signature
class sqlalchemy.orm.MappedClassProtocol (typing.Protocol)
Mapping SQL Expressions¶
This page has been merged into the ORM Mapped Class Configuration index.
Mapping Table Columns¶
This section has been integrated into the Table Configuration with Declarative section.


Relationship Configuration
This section describes the relationship() function and in depth discussion of its usage. For an introduction to relationships, start with Working with ORM Related Objects in the SQLAlchemy Unified Tutorial.
• Basic Relationship Patterns
o Declarative vs. Imperative Forms
o One To Many
• Using Sets, Lists, or other Collection Types for One To Many
• Configuring Delete Behavior for One to Many
o Many To One
• Nullable Many-to-One
o One To One
• Setting uselist=False for non-annotated configurations
o Many To Many
• Setting Bi-Directional Many-to-many
• Using a late-evaluated form for the “secondary” argument
• Using Sets, Lists, or other Collection Types for Many To Many
• Deleting Rows from the Many to Many Table
o Association Object
• Combining Association Object with Many-to-Many Access Patterns
o Late-Evaluation of Relationship Arguments
• Adding Relationships to Mapped Classes After Declaration
• Using a late-evaluated form for the “secondary” argument of many-to-many
• Adjacency List Relationships
o Composite Adjacency Lists
o Self-Referential Query Strategies
o Configuring Self-Referential Eager Loading
• Configuring how Relationship Joins
o Handling Multiple Join Paths
o Specifying Alternate Join Conditions
o Creating Custom Foreign Conditions
o Using custom operators in join conditions
o Custom operators based on SQL functions
o Overlapping Foreign Keys
o Non-relational Comparisons / Materialized Path
o Self-Referential Many-to-Many Relationship
o Composite “Secondary” Joins
o Relationship to Aliased Class
• Integrating AliasedClass Mappings with Typing and Avoiding Early Mapper Configuration
• Using the AliasedClass target in Queries
o Row-Limited Relationships with Window Functions
o Building Query-Enabled Properties
o Notes on using the viewonly relationship parameter
• In-Python mutations including backrefs are not appropriate with viewonly=True
• viewonly=True collections / attributes do not get re-queried until expired
• Working with Large Collections
o Write Only Relationships
• Creating and Persisting New Write Only Collections
• Adding New Items to an Existing Collection
• Querying Items
• Removing Items
• Bulk INSERT of New Items
• Bulk UPDATE and DELETE of Items
• Write Only Collections - API Documentation
o Dynamic Relationship Loaders
• Dynamic Relationship Loaders - API
o Setting RaiseLoad
o Using Passive Deletes
• Collection Customization and API Details
o Customizing Collection Access
• Dictionary Collections
o Custom Collection Implementations
• Annotating Custom Collections via Decorators
• Custom Dictionary-Based Collections
• Instrumentation and Custom Types
o Collection API
• attribute_keyed_dict()
• column_keyed_dict()
• keyfunc_mapping()
• attribute_mapped_collection
• column_mapped_collection
• mapped_collection
• KeyFuncDict
• MappedCollection
o Collection Internals
• bulk_replace()
• collection
• collection_adapter
• CollectionAdapter
• InstrumentedDict
• InstrumentedList
• InstrumentedSet
• prepare_instrumentation()
• Special Relationship Persistence Patterns
o Rows that point to themselves / Mutually Dependent Rows
o Mutable Primary Keys / Update Cascades
• Simulating limited ON UPDATE CASCADE without foreign key support
• Using the legacy ‘backref’ relationship parameter
o Backref Default Arguments
o Specifying Backref Arguments
• Relationships API
o relationship()
o backref()
o dynamic_loader()
o foreign()
o remote()


Basic Relationship Patterns
A quick walkthrough of the basic relational patterns, which in this section are illustrated using Declarative style mappings based on the use of the Mapped annotation type.
The setup for each of the following sections is as follows:
from __future__ import annotations
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass
Declarative vs. Imperative Forms
As SQLAlchemy has evolved, different ORM configurational styles have emerged. For examples in this section and others that use annotated Declarative mappings with Mapped, the corresponding non-annotated form should use the desired class, or string class name, as the first argument passed to relationship(). The example below illustrates the form used in this document, which is a fully Declarative example using PEP 484 annotations, where the relationship() construct is also deriving the target class and collection type from the Mapped annotation, which is the most modern form of SQLAlchemy Declarative mapping:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
In contrast, using a Declarative mapping without annotations is the more “classic” form of mapping, where relationship() requires all parameters passed to it directly, as in the example below:
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")


class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="children")
Finally, using Imperative Mapping, which is SQLAlchemy’s original mapping form before Declarative was made (which nonetheless remains preferred by a vocal minority of users), the above configuration looks like:
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
Additionally, the default collection style for non-annotated mappings is list. To use a set or other collection without annotations, indicate it using the relationship.collection_class parameter:
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", collection_class=set, )
Detail on collection configuration for relationship() is at Customizing Collection Access.
Additional differences between annotated and non-annotated / imperative styles will be noted as needed.
One To Many
A one to many relationship places a foreign key on the child table referencing the parent. relationship() is then specified on the parent, as referencing a collection of items represented by the child:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship()


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
To establish a bidirectional relationship in one-to-many, where the “reverse” side is a many to one, specify an additional relationship() and connect the two using the relationship.back_populates parameter, using the attribute name of each relationship() as the value for relationship.back_populates on the other:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
Child will get a parent attribute with many-to-one semantics.
Using Sets, Lists, or other Collection Types for One To Many
Using annotated Declarative mappings, the type of collection used for the relationship() is derived from the collection type passed to the Mapped container type. The example from the previous section may be written to use a set rather than a list for the Parent.children collection using Mapped[Set["Child"]]:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(back_populates="parent")
When using non-annotated forms including imperative mappings, the Python class to use as a collection may be passed using the relationship.collection_class parameter.
See also
Customizing Collection Access - contains further detail on collection configuration including some techniques to map relationship() to dictionaries.
Configuring Delete Behavior for One to Many
It is often the case that all Child objects should be deleted when their owning Parent is deleted. To configure this behavior, the delete cascade option described at delete is used. An additional option is that a Child object can itself be deleted when it is deassociated from its parent. This behavior is described at delete-orphan.
See also
delete
Using foreign key ON DELETE cascade with ORM relationships
delete-orphan
Many To One
Many to one places a foreign key in the parent table referencing the child. relationship() is declared on the parent, where a new scalar-holding attribute will be created:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship()


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
The above example shows a many-to-one relationship that assumes non-nullable behavior; the next section, Nullable Many-to-One, illustrates a nullable version.
Bidirectional behavior is achieved by adding a second relationship() and applying the relationship.back_populates parameter in both directions, using the attribute name of each relationship() as the value for relationship.back_populates on the other:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship(back_populates="parents")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
Nullable Many-to-One
In the preceding example, the Parent.child relationship is not typed as allowing None; this follows from the Parent.child_id column itself not being nullable, as it is typed with Mapped[int]. If we wanted Parent.child to be a nullable many-to-one, we can set both Parent.child_id and Parent.child to be Optional[] (or its equivalent), in which case the configuration would look like:
from typing import Optional


class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[Optional[int]] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Optional["Child"]] = relationship(back_populates="parents")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
Above, the column for Parent.child_id will be created in DDL to allow NULL values. When using mapped_column() with explicit typing declarations, the specification of child_id: Mapped[Optional[int]] is equivalent to setting Column.nullable to True on the Column, whereas child_id: Mapped[int] is equivalent to setting it to False. See mapped_column() derives the datatype and nullability from the Mapped annotation for background on this behavior.
Tip
If using Python 3.10 or greater, PEP 604 syntax is more convenient to indicate optional types using | None, which when combined with PEP 563 postponed annotation evaluation so that string-quoted types aren’t required, would look like:
from __future__ import annotations


class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int | None] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Child | None] = relationship(back_populates="parents")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(back_populates="child")
One To One
One To One is essentially a One To Many relationship from a foreign key perspective, but indicates that there will only be one row at any time that refers to a particular parent row.
When using annotated mappings with Mapped, the “one-to-one” convention is achieved by applying a non-collection type to the Mapped annotation on both sides of the relationship, which will imply to the ORM that a collection should not be used on either side, as in the example below:
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child: Mapped["Child"] = relationship(back_populates="parent")


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")
Above, when we load a Parent object, the Parent.child attribute will refer to a single Child object rather than a collection. If we replace the value of Parent.child with a new Child object, the ORM’s unit of work process will replace the previous Child row with the new one, setting the previous child.parent_id column to NULL by default unless there are specific cascade behaviors set up.
Tip
As mentioned previously, the ORM considers the “one-to-one” pattern as a convention, where it makes the assumption that when it loads the Parent.child attribute on a Parent object, it will get only one row back. If more than one row is returned, the ORM will emit a warning.
However, the Child.parent side of the above relationship remains as a “many-to-one” relationship. By itself, it will not detect assignment of more than one Child, unless the relationship.single_parent parameter is set, which may be useful:
class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child", single_parent=True)
Outside of setting this parameter, the “one-to-many” side (which here is one-to-one by convention) will also not reliably detect if more than one Child is associated with a single Parent, such as in the case where the multiple Child objects are pending and not database-persistent.
Whether or not relationship.single_parent is used, it is recommended that the database schema include a unique constraint to indicate that the Child.parent_id column should be unique, to ensure at the database level that only one Child row may refer to a particular Parent row at a time (see Declarative Table Configuration for background on the __table_args__ tuple syntax):
from sqlalchemy import UniqueConstraint


class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")

    __table_args__ = (UniqueConstraint("parent_id"),)
Added in version 2.0: The relationship() construct can derive the effective value of the relationship.uselist parameter from a given Mapped annotation.
Setting uselist=False for non-annotated configurations
When using relationship() without the benefit of Mapped annotations, the one-to-one pattern can be enabled using the relationship.uselist parameter set to False on what would normally be the “many” side, illustrated in a non-annotated Declarative configuration below:
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    child = relationship("Child", uselist=False, back_populates="parent")


class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="child")
Many To Many
Many to Many adds an association table between two classes. The association table is nearly always given as a Core Table object or other Core selectable such as a Join object, and is indicated by the relationship.secondary argument to relationship(). Usually, the Table uses the MetaData object associated with the declarative base class, so that the ForeignKey directives can locate the remote tables with which to link:
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


# note for a Core table, we use the sqlalchemy.Column construct,
# not sqlalchemy.orm.mapped_column
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id")),
    Column("right_id", ForeignKey("right_table.id")),
)


class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(secondary=association_table)


class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
Tip
The “association table” above has foreign key constraints established that refer to the two entity tables on either side of the relationship. The data type of each of association.left_id and association.right_id is normally inferred from that of the referenced table and may be omitted. It is also recommended, though not in any way required by SQLAlchemy, that the columns which refer to the two entity tables are established within either a unique constraint or more commonly as the primary key constraint; this ensures that duplicate rows won’t be persisted within the table regardless of issues on the application side:
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)
Setting Bi-Directional Many-to-many
For a bidirectional relationship, both sides of the relationship contain a collection. Specify using relationship.back_populates, and for each relationship() specify the common association table:
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)


class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(
        secondary=association_table, back_populates="parents"
    )


class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(
        secondary=association_table, back_populates="children"
    )
Using a late-evaluated form for the “secondary” argument
The relationship.secondary parameter of relationship() also accepts two different “late evaluated” forms, including string table name as well as lambda callable. See the section Using a late-evaluated form for the “secondary” argument of many-to-many for background and examples.
Using Sets, Lists, or other Collection Types for Many To Many
Configuration of collections for a Many to Many relationship is identical to that of One To Many, as described at Using Sets, Lists, or other Collection Types for One To Many. For an annotated mapping using Mapped, the collection can be indicated by the type of collection used within the Mapped generic class, such as set:
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(secondary=association_table)
When using non-annotated forms including imperative mappings, as is the case with one-to-many, the Python class to use as a collection may be passed using the relationship.collection_class parameter.
See also
Customizing Collection Access - contains further detail on collection configuration including some techniques to map relationship() to dictionaries.
Deleting Rows from the Many to Many Table
A behavior which is unique to the relationship.secondary argument to relationship() is that the Table which is specified here is automatically subject to INSERT and DELETE statements, as objects are added or removed from the collection. There is no need to delete from this table manually. The act of removing a record from the collection will have the effect of the row being deleted on flush:
# row will be deleted from the "secondary" table
# automatically
myparent.children.remove(somechild)
A question which often arises is how the row in the “secondary” table can be deleted when the child object is handed directly to Session.delete():
session.delete(somechild)
There are several possibilities here:
• If there is a relationship() from Parent to Child, but there is not a reverse-relationship that links a particular Child to each Parent, SQLAlchemy will not have any awareness that when deleting this particular Child object, it needs to maintain the “secondary” table that links it to the Parent. No delete of the “secondary” table will occur.
• If there is a relationship that links a particular Child to each Parent, suppose it’s called Child.parents, SQLAlchemy by default will load in the Child.parents collection to locate all Parent objects, and remove each row from the “secondary” table which establishes this link. Note that this relationship does not need to be bidirectional; SQLAlchemy is strictly looking at every relationship() associated with the Child object being deleted.
• A higher performing option here is to use ON DELETE CASCADE directives with the foreign keys used by the database. Assuming the database supports this feature, the database itself can be made to automatically delete rows in the “secondary” table as referencing rows in “child” are deleted. SQLAlchemy can be instructed to forego actively loading in the Child.parents collection in this case using the relationship.passive_deletes directive on relationship(); see Using foreign key ON DELETE cascade with ORM relationships for more details on this.
Note again, these behaviors are only relevant to the relationship.secondary option used with relationship(). If dealing with association tables that are mapped explicitly and are not present in the relationship.secondary option of a relevant relationship(), cascade rules can be used instead to automatically delete entities in reaction to a related entity being deleted - see Cascades for information on this feature.
See also
Using delete cascade with many-to-many relationships
Using foreign key ON DELETE with many-to-many relationships
Association Object
The association object pattern is a variant on many-to-many: it’s used when an association table contains additional columns beyond those which are foreign keys to the parent and child (or left and right) tables, columns which are most ideally mapped to their own ORM mapped class. This mapped class is mapped against the Table that would otherwise be noted as relationship.secondary when using the many-to-many pattern.
In the association object pattern, the relationship.secondary parameter is not used; instead, a class is mapped directly to the association table. Two individual relationship() constructs then link first the parent side to the mapped association class via one to many, and then the mapped association class to the child side via many-to-one, to form a uni-directional association object relationship from parent, to association, to child. For a bi-directional relationship, four relationship() constructs are used to link the mapped association class to both parent and child in both directions.
The example below illustrates a new class Association which maps to the Table named association; this table now includes an additional column called extra_data, which is a string value that is stored along with each association between Parent and Child. By mapping the table to an explicit class, rudimental access from Parent to Child makes explicit use of Association:
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship()


class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship()


class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
To illustrate the bi-directional version, we add two more relationship() constructs, linked to the existing ones using relationship.back_populates:
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship(back_populates="parents")
    parent: Mapped["Parent"] = relationship(back_populates="children")


class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship(back_populates="parent")


class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Association"]] = relationship(back_populates="child")
Working with the association pattern in its direct form requires that child objects are associated with an association instance before being appended to the parent; similarly, access from parent to child goes through the association object:
# create parent, append a child via association
p = Parent()
a = Association(extra_data="some data")
a.child = Child()
p.children.append(a)

# iterate through child objects via association, including association
# attributes
for assoc in p.children:
    print(assoc.extra_data)
    print(assoc.child)
To enhance the association object pattern such that direct access to the Association object is optional, SQLAlchemy provides the Association Proxy extension. This extension allows the configuration of attributes which will access two “hops” with a single access, one “hop” to the associated object, and a second to a target attribute.
See also
Association Proxy - allows direct “many to many” style access between parent and child for a three-class association object mapping.
Warning
Avoid mixing the association object pattern with the many-to-many pattern directly, as this produces conditions where data may be read and written in an inconsistent fashion without special steps; the association proxy is typically used to provide more succinct access. For more detailed background on the caveats introduced by this combination, see the next section Combining Association Object with Many-to-Many Access Patterns.
Combining Association Object with Many-to-Many Access Patterns
As mentioned in the previous section, the association object pattern does not automatically integrate with usage of the many-to-many pattern against the same tables/columns at the same time. From this it follows that read operations may return conflicting data and write operations may also attempt to flush conflicting changes, causing either integrity errors or unexpected inserts or deletes.
To illustrate, the example below configures a bidirectional many-to-many relationship between Parent and Child via Parent.children and Child.parents. At the same time, an association object relationship is also configured, between Parent.child_associations -> Association.child and Child.parent_associations -> Association.parent:
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Association(Base):
    __tablename__ = "association_table"

    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]

    # association between Assocation -> Child
    child: Mapped["Child"] = relationship(back_populates="parent_associations")

    # association between Assocation -> Parent
    parent: Mapped["Parent"] = relationship(back_populates="child_associations")


class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents"
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )


class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children"
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
When using this ORM model to make changes, changes made to Parent.children will not be coordinated with changes made to Parent.child_associations or Child.parent_associations in Python; while all of these relationships will continue to function normally by themselves, changes on one will not show up in another until the Session is expired, which normally occurs automatically after Session.commit().
Additionally, if conflicting changes are made, such as adding a new Association object while also appending the same related Child to Parent.children, this will raise integrity errors when the unit of work flush process proceeds, as in the example below:
p1 = Parent()
c1 = Child()
p1.children.append(c1)

# redundant, will cause a duplicate INSERT on Association
p1.child_associations.append(Association(child=c1))
Appending Child to Parent.children directly also implies the creation of rows in the association table without indicating any value for the association.extra_data column, which will receive NULL for its value.
It’s fine to use a mapping like the above if you know what you’re doing; there may be good reason to use many-to-many relationships in the case where use of the “association object” pattern is infrequent, which is that it’s easier to load relationships along a single many-to-many relationship, which can also optimize slightly better how the “secondary” table is used in SQL statements, compared to how two separate relationships to an explicit association class is used. It’s at least a good idea to apply the relationship.viewonly parameter to the “secondary” relationship to avoid the issue of conflicting changes occurring, as well as preventing NULL being written to the additional association columns, as below:
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents", viewonly=True
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )


class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children", viewonly=True
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
The above mapping will not write any changes to Parent.children or Child.parents to the database, preventing conflicting writes. However, reads of Parent.children or Child.parents will not necessarily match the data that’s read from Parent.child_associations or Child.parent_associations, if changes are being made to these collections within the same transaction or Session as where the viewonly collections are being read. If use of the association object relationships is infrequent and is carefully organized against code that accesses the many-to-many collections to avoid stale reads (in extreme cases, making direct use of Session.expire() to cause collections to be refreshed within the current transaction), the pattern may be feasible.
A popular alternative to the above pattern is one where the direct many-to-many Parent.children and Child.parents relationships are replaced with an extension that will transparently proxy through the Association class, while keeping everything consistent from the ORM’s point of view. This extension is known as the Association Proxy.
See also
Association Proxy - allows direct “many to many” style access between parent and child for a three-class association object mapping.
Late-Evaluation of Relationship Arguments
Most of the examples in the preceding sections illustrate mappings where the various relationship() constructs refer to their target classes using a string name, rather than the class itself, such as when using Mapped, a forward reference is generated that exists at runtime only as a string:
class Parent(Base):
    # 

    children: Mapped[List["Child"]] = relationship(back_populates="parent")


class Child(Base):
    # 

    parent: Mapped["Parent"] = relationship(back_populates="children")
Similarly, when using non-annotated forms such as non-annotated Declarative or Imperative mappings, a string name is also supported directly by the relationship() construct:
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
These string names are resolved into classes in the mapper resolution stage, which is an internal process that occurs typically after all mappings have been defined and is normally triggered by the first usage of the mappings themselves. The registry object is the container where these names are stored and resolved to the mapped classes to which they refer.
In addition to the main class argument for relationship(), other arguments which depend upon the columns present on an as-yet undefined class may also be specified either as Python functions, or more commonly as strings. For most of these arguments except that of the main argument, string inputs are evaluated as Python expressions using Python’s built-in eval() function, as they are intended to receive complete SQL expressions.
Warning
As the Python eval() function is used to interpret the late-evaluated string arguments passed to relationship() mapper configuration construct, these arguments should not be repurposed such that they would receive untrusted user input; eval() is not secure against untrusted user input.
The full namespace available within this evaluation includes all classes mapped for this declarative base, as well as the contents of the sqlalchemy package, including expression functions like desc() and sqlalchemy.sql.functions.func:
class Parent(Base):
    # 

    children: Mapped[List["Child"]] = relationship(
        order_by="desc(Child.email_address)",
        primaryjoin="Parent.id == Child.parent_id",
    )
For the case where more than one module contains a class of the same name, string class names can also be specified as module-qualified paths within any of these string expressions:
class Parent(Base):
    # 

    children: Mapped[List["myapp.mymodel.Child"]] = relationship(
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
In an example like the above, the string passed to Mapped can be disambiguated from a specific class argument by passing the class location string directly to the first positional parameter (relationship.argument) as well. Below illustrates a typing-only import for Child, combined with a runtime specifier for the target class that will search for the correct name within the registry:
import typing

if typing.TYPE_CHECKING:
    from myapp.mymodel import Child


class Parent(Base):
    # 

    children: Mapped[List["Child"]] = relationship(
        "myapp.mymodel.Child",
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
The qualified path can be any partial path that removes ambiguity between the names. For example, to disambiguate between myapp.model1.Child and myapp.model2.Child, we can specify model1.Child or model2.Child:
class Parent(Base):
    # 

    children: Mapped[List["Child"]] = relationship(
        "model1.Child",
        order_by="desc(mymodel1.Child.email_address)",
        primaryjoin="Parent.id == model1.Child.parent_id",
    )
The relationship() construct also accepts Python functions or lambdas as input for these arguments. A Python functional approach might look like the following:
import typing

from sqlalchemy import desc

if typing.TYPE_CHECKING:
    from myapplication import Child


def _resolve_child_model():
    from myapplication import Child

    return Child


class Parent(Base):
    # 

    children: Mapped[List["Child"]] = relationship(
        _resolve_child_model,
        order_by=lambda: desc(_resolve_child_model().email_address),
        primaryjoin=lambda: Parent.id == _resolve_child_model().parent_id,
    )
The full list of parameters which accept Python functions/lambdas or strings that will be passed to eval() are:
• relationship.order_by
• relationship.primaryjoin
• relationship.secondaryjoin
• relationship.secondary
• relationship.remote_side
• relationship.foreign_keys
• relationship._user_defined_foreign_keys
Warning
As stated previously, the above parameters to relationship() are evaluated as Python code expressions using eval(). DO NOT PASS UNTRUSTED INPUT TO THESE ARGUMENTS.
Adding Relationships to Mapped Classes After Declaration
It should also be noted that in a similar way as described at Appending additional columns to an existing Declarative mapped class, any MapperProperty construct can be added to a declarative base mapping at any time (noting that annotated forms are not supported in this context). If we wanted to implement this relationship() after the Address class were available, we could also apply it afterwards:
# first, module A, where Child has not been created yet,
# we create a Parent class which knows nothing about Child


class Parent(Base): 


# later, in Module B, which is imported after module A:


class Child(Base): 


from module_a import Parent

# assign the User.addresses relationship as a class variable.  The
# declarative base class will intercept this and map the relationship.
Parent.children = relationship(Child, primaryjoin=Child.parent_id == Parent.id)
As is the case for ORM mapped columns, there’s no capability for the Mapped annotation type to take part in this operation; therefore, the related class must be specified directly within the relationship() construct, either as the class itself, the string name of the class, or a callable function that returns a reference to the target class.
Note
As is the case for ORM mapped columns, assignment of mapped properties to an already mapped class will only function correctly if the “declarative base” class is used, meaning the user-defined subclass of DeclarativeBase or the dynamically generated class returned by declarative_base() or registry.generate_base(). This “base” class includes a Python metaclass which implements a special __setattr__() method that intercepts these operations.
Runtime assignment of class-mapped attributes to a mapped class will not work if the class is mapped using decorators like registry.mapped() or imperative functions like registry.map_imperatively().
Using a late-evaluated form for the “secondary” argument of many-to-many
Many-to-many relationships make use of the relationship.secondary parameter, which ordinarily indicates a reference to a typically non-mapped Table object or other Core selectable object. Late evaluation using a lambda callable is typical.
For the example given at Many To Many, if we assumed that the association_table Table object would be defined at a point later on in the module than the mapped class itself, we may write the relationship() using a lambda as:
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        "Child", secondary=lambda: association_table
    )
As a shortcut for table names that are also valid Python identifiers, the relationship.secondary parameter may also be passed as a string, where resolution works by evaluation of the string as a Python expression, with simple identifier names linked to same-named Table objects that are present in the same MetaData collection referenced by the current registry.
In the example below, the expression "association_table" is evaluated as a variable named “association_table” that is resolved against the table names within the MetaData collection:
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(secondary="association_table")
Note
When passed as a string, the name passed to relationship.secondary must be a valid Python identifier starting with a letter and containing only alphanumeric characters or underscores. Other characters such as dashes etc. will be interpreted as Python operators which will not resolve to the name given. Please consider using lambda expressions rather than strings for improved clarity.
Warning
When passed as a string, relationship.secondary argument is interpreted using Python’s eval() function, even though it’s typically the name of a table. DO NOT PASS UNTRUSTED INPUT TO THIS STRING.


Adjacency List Relationships
The adjacency list pattern is a common relational pattern whereby a table contains a foreign key reference to itself, in other words is a self referential relationship. This is the most common way to represent hierarchical data in flat tables. Other methods include nested sets, sometimes called “modified preorder”, as well as materialized path. Despite the appeal that modified preorder has when evaluated for its fluency within SQL queries, the adjacency list model is probably the most appropriate pattern for the large majority of hierarchical storage needs, for reasons of concurrency, reduced complexity, and that modified preorder has little advantage over an application which can fully load subtrees into the application space.
See also
This section details the single-table version of a self-referential relationship. For a self-referential relationship that uses a second table as an association table, see the section Self-Referential Many-to-Many Relationship.
In this example, we’ll work with a single mapped class called Node, representing a tree structure:
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node")
With this structure, a graph such as the following:
root --+---> child1
       +---> child2 --+--> subchild1
       |              +--> subchild2
       +---> child3
Would be represented with data such as:
id       parent_id     data
---      -------       ----
1        NULL          root
2        1             child1
3        1             child2
4        3             subchild1
5        3             subchild2
6        1             child3
The relationship() configuration here works in the same way as a “normal” one-to-many relationship, with the exception that the “direction”, i.e. whether the relationship is one-to-many or many-to-one, is assumed by default to be one-to-many. To establish the relationship as many-to-one, an extra directive is added known as relationship.remote_side, which is a Column or collection of Column objects that indicate those which should be considered to be “remote”:
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    parent = relationship("Node", remote_side=[id])
Where above, the id column is applied as the relationship.remote_side of the parent relationship(), thus establishing parent_id as the “local” side, and the relationship then behaves as a many-to-one.
As always, both directions can be combined into a bidirectional relationship using two relationship() constructs linked by relationship.back_populates:
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node", back_populates="parent")
    parent = relationship("Node", back_populates="children", remote_side=[id])
See also
Adjacency List - working example, updated for SQLAlchemy 2.0
Composite Adjacency Lists
A sub-category of the adjacency list relationship is the rare case where a particular column is present on both the “local” and “remote” side of the join condition. An example is the Folder class below; using a composite primary key, the account_id column refers to itself, to indicate sub folders which are within the same account as that of the parent; while folder_id refers to a specific folder within that account:
class Folder(Base):
    __tablename__ = "folder"
    __table_args__ = (
        ForeignKeyConstraint(
            ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
        ),
    )

    account_id = mapped_column(Integer, primary_key=True)
    folder_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer)
    name = mapped_column(String)

    parent_folder = relationship(
        "Folder", back_populates="child_folders", remote_side=[account_id, folder_id]
    )

    child_folders = relationship("Folder", back_populates="parent_folder")
Above, we pass account_id into the relationship.remote_side list. relationship() recognizes that the account_id column here is on both sides, and aligns the “remote” column along with the folder_id column, which it recognizes as uniquely present on the “remote” side.
Self-Referential Query Strategies
Querying of self-referential structures works like any other query:
# get all nodes named 'child2'
session.scalars(select(Node).where(Node.data == "child2"))
However extra care is needed when attempting to join along the foreign key from one level of the tree to the next. In SQL, a join from a table to itself requires that at least one side of the expression be “aliased” so that it can be unambiguously referred to.
Recall from Selecting ORM Aliases in the ORM tutorial that the aliased() construct is normally used to provide an “alias” of an ORM entity. Joining from Node to itself using this technique looks like:
from sqlalchemy.orm import aliased

nodealias = aliased(Node)
session.scalars(
    select(Node)
    .where(Node.data == "subchild1")
    .join(Node.parent.of_type(nodealias))
    .where(nodealias.data == "child2")
).all()
SELECT node.id AS node_id,
        node.parent_id AS node_parent_id,
        node.data AS node_data
FROM node JOIN node AS node_1
    ON node.parent_id = node_1.id
WHERE node.data = ?
    AND node_1.data = ?
['subchild1', 'child2']
Configuring Self-Referential Eager Loading
Eager loading of relationships occurs using joins or outerjoins from parent to child table during a normal query operation, such that the parent and its immediate child collection or reference can be populated from a single SQL statement, or a second statement for all immediate child collections. SQLAlchemy’s joined and subquery eager loading use aliased tables in all cases when joining to related items, so are compatible with self-referential joining. However, to use eager loading with a self-referential relationship, SQLAlchemy needs to be told how many levels deep it should join and/or query; otherwise the eager load will not take place at all. This depth setting is configured via relationships.join_depth:
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node", lazy="joined", join_depth=2)


session.scalars(select(Node)).all()
SELECT node_1.id AS node_1_id,
        node_1.parent_id AS node_1_parent_id,
        node_1.data AS node_1_data,
        node_2.id AS node_2_id,
        node_2.parent_id AS node_2_parent_id,
        node_2.data AS node_2_data,
        node.id AS node_id,
        node.parent_id AS node_parent_id,
        node.data AS node_data
FROM node
    LEFT OUTER JOIN node AS node_2
        ON node.id = node_2.parent_id
    LEFT OUTER JOIN node AS node_1
        ON node_2.id = node_1.parent_id
[]


Configuring how Relationship Joins
relationship() will normally create a join between two tables by examining the foreign key relationship between the two tables to determine which columns should be compared. There are a variety of situations where this behavior needs to be customized.
Handling Multiple Join Paths
One of the most common situations to deal with is when there are more than one foreign key path between two tables.
Consider a Customer class that contains two foreign keys to an Address class:
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address")
    shipping_address = relationship("Address")


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
The above mapping, when we attempt to use it, will produce the error:
sqlalchemy.exc.AmbiguousForeignKeysError: Could not determine join
condition between parent/child tables on relationship
Customer.billing_address - there are multiple foreign key
paths linking the tables.  Specify the 'foreign_keys' argument,
providing a list of those columns which should be
counted as containing a foreign key reference to the parent table.
The above message is pretty long. There are many potential messages that relationship() can return, which have been carefully tailored to detect a variety of common configurational issues; most will suggest the additional configuration that’s needed to resolve the ambiguity or other missing information.
In this case, the message wants us to qualify each relationship() by instructing for each one which foreign key column should be considered, and the appropriate form is as follows:
class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address", foreign_keys=[billing_address_id])
    shipping_address = relationship("Address", foreign_keys=[shipping_address_id])
Above, we specify the foreign_keys argument, which is a Column or list of Column objects which indicate those columns to be considered “foreign”, or in other words, the columns that contain a value referring to a parent table. Loading the Customer.billing_address relationship from a Customer object will use the value present in billing_address_id in order to identify the row in Address to be loaded; similarly, shipping_address_id is used for the shipping_address relationship. The linkage of the two columns also plays a role during persistence; the newly generated primary key of a just-inserted Address object will be copied into the appropriate foreign key column of an associated Customer object during a flush.
When specifying foreign_keys with Declarative, we can also use string names to specify, however it is important that if using a list, the list is part of the string:
billing_address = relationship("Address", foreign_keys="[Customer.billing_address_id]")
In this specific example, the list is not necessary in any case as there’s only one Column we need:
billing_address = relationship("Address", foreign_keys="Customer.billing_address_id")
Warning
When passed as a Python-evaluable string, the relationship.foreign_keys argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
Specifying Alternate Join Conditions
The default behavior of relationship() when constructing a join is that it equates the value of primary key columns on one side to that of foreign-key-referring columns on the other. We can change this criterion to be anything we’d like using the relationship.primaryjoin argument, as well as the relationship.secondaryjoin argument in the case when a “secondary” table is used.
In the example below, using the User class as well as an Address class which stores a street address, we create a relationship boston_addresses which will only load those Address objects which specify a city of “Boston”:
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)
    boston_addresses = relationship(
        "Address",
        primaryjoin="and_(User.id==Address.user_id, Address.city=='Boston')",
    )


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
Within this string SQL expression, we made use of the and_() conjunction construct to establish two distinct predicates for the join condition - joining both the User.id and Address.user_id columns to each other, as well as limiting rows in Address to just city='Boston'. When using Declarative, rudimentary SQL functions like and_() are automatically available in the evaluated namespace of a string relationship() argument.
Warning
When passed as a Python-evaluable string, the relationship.primaryjoin argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
The custom criteria we use in a relationship.primaryjoin is generally only significant when SQLAlchemy is rendering SQL in order to load or represent this relationship. That is, it’s used in the SQL statement that’s emitted in order to perform a per-attribute lazy load, or when a join is constructed at query time, such as via Select.join(), or via the eager “joined” or “subquery” styles of loading. When in-memory objects are being manipulated, we can place any Address object we’d like into the boston_addresses collection, regardless of what the value of the .city attribute is. The objects will remain present in the collection until the attribute is expired and re-loaded from the database where the criterion is applied. When a flush occurs, the objects inside of boston_addresses will be flushed unconditionally, assigning value of the primary key user.id column onto the foreign-key-holding address.user_id column for each row. The city criteria has no effect here, as the flush process only cares about synchronizing primary key values into referencing foreign key values.
Creating Custom Foreign Conditions
Another element of the primary join condition is how those columns considered “foreign” are determined. Usually, some subset of Column objects will specify ForeignKey, or otherwise be part of a ForeignKeyConstraint that’s relevant to the join condition. relationship() looks to this foreign key status as it decides how it should load and persist data for this relationship. However, the relationship.primaryjoin argument can be used to create a join condition that doesn’t involve any “schema” level foreign keys. We can combine relationship.primaryjoin along with relationship.foreign_keys and relationship.remote_side explicitly in order to establish such a join.
Below, a class HostEntry joins to itself, equating the string content column to the ip_address column, which is a PostgreSQL type called INET. We need to use cast() in order to cast one side of the join to the type of the other:
from sqlalchemy import cast, String, Column, Integer
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import INET

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign_keys, remote_side
    parent_host = relationship(
        "HostEntry",
        primaryjoin=ip_address == cast(content, INET),
        foreign_keys=content,
        remote_side=ip_address,
    )
The above relationship will produce a join like:
SELECT host_entry.id, host_entry.ip_address, host_entry.content
FROM host_entry JOIN host_entry AS host_entry_1
ON host_entry_1.ip_address = CAST(host_entry.content AS INET)
An alternative syntax to the above is to use the foreign() and remote() annotations, inline within the relationship.primaryjoin expression. This syntax represents the annotations that relationship() normally applies by itself to the join condition given the relationship.foreign_keys and relationship.remote_side arguments. These functions may be more succinct when an explicit join condition is present, and additionally serve to mark exactly the column that is “foreign” or “remote” independent of whether that column is stated multiple times or within complex SQL expressions:
from sqlalchemy.orm import foreign, remote


class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign() and remote() annotations
    # in lieu of separate arguments
    parent_host = relationship(
        "HostEntry",
        primaryjoin=remote(ip_address) == cast(foreign(content), INET),
    )
Using custom operators in join conditions
Another use case for relationships is the use of custom operators, such as PostgreSQL’s “is contained within” << operator when joining with types such as INET and CIDR. For custom boolean operators we use the Operators.bool_op() function:
inet_column.bool_op("<<")(cidr_column)
A comparison like the above may be used directly with relationship.primaryjoin when constructing a relationship():
class IPA(Base):
    __tablename__ = "ip_address"

    id = mapped_column(Integer, primary_key=True)
    v4address = mapped_column(INET)

    network = relationship(
        "Network",
        primaryjoin="IPA.v4address.bool_op('<<')(foreign(Network.v4representation))",
        viewonly=True,
    )


class Network(Base):
    __tablename__ = "network"

    id = mapped_column(Integer, primary_key=True)
    v4representation = mapped_column(CIDR)
Above, a query such as:
select(IPA).join(IPA.network)
Will render as:
SELECT ip_address.id AS ip_address_id, ip_address.v4address AS ip_address_v4address
FROM ip_address JOIN network ON ip_address.v4address << network.v4representation
Custom operators based on SQL functions
A variant to the use case for Operators.op.is_comparison is when we aren’t using an operator, but a SQL function. The typical example of this use case is the PostgreSQL PostGIS functions however any SQL function on any database that resolves to a binary condition may apply. To suit this use case, the FunctionElement.as_comparison() method can modify any SQL function, such as those invoked from the func namespace, to indicate to the ORM that the function produces a comparison of two expressions. The below example illustrates this with the Geoalchemy2 library:
from geoalchemy2 import Geometry
from sqlalchemy import Column, Integer, func
from sqlalchemy.orm import relationship, foreign


class Polygon(Base):
    __tablename__ = "polygon"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POLYGON", srid=4326))
    points = relationship(
        "Point",
        primaryjoin="func.ST_Contains(foreign(Polygon.geom), Point.geom).as_comparison(1, 2)",
        viewonly=True,
    )


class Point(Base):
    __tablename__ = "point"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POINT", srid=4326))
Above, the FunctionElement.as_comparison() indicates that the func.ST_Contains() SQL function is comparing the Polygon.geom and Point.geom expressions. The foreign() annotation additionally notes which column takes on the “foreign key” role in this particular relationship.
Added in version 1.3: Added FunctionElement.as_comparison().
Overlapping Foreign Keys
A rare scenario can arise when composite foreign keys are used, such that a single column may be the subject of more than one column referred to via foreign key constraint.
Consider an (admittedly complex) mapping such as the Magazine object, referred to both by the Writer object and the Article object using a composite primary key scheme that includes magazine_id for both; then to make Article refer to Writer as well, Article.magazine_id is involved in two separate relationships; Article.magazine and Article.writer:
class Magazine(Base):
    __tablename__ = "magazine"

    id = mapped_column(Integer, primary_key=True)


class Article(Base):
    __tablename__ = "article"

    article_id = mapped_column(Integer)
    magazine_id = mapped_column(ForeignKey("magazine.id"))
    writer_id = mapped_column(Integer)

    magazine = relationship("Magazine")
    writer = relationship("Writer")

    __table_args__ = (
        PrimaryKeyConstraint("article_id", "magazine_id"),
        ForeignKeyConstraint(
            ["writer_id", "magazine_id"], ["writer.id", "writer.magazine_id"]
        ),
    )


class Writer(Base):
    __tablename__ = "writer"

    id = mapped_column(Integer, primary_key=True)
    magazine_id = mapped_column(ForeignKey("magazine.id"), primary_key=True)
    magazine = relationship("Magazine")
When the above mapping is configured, we will see this warning emitted:
SAWarning: relationship 'Article.writer' will copy column
writer.magazine_id to column article.magazine_id,
which conflicts with relationship(s): 'Article.magazine'
(copies magazine.id to article.magazine_id). Consider applying
viewonly=True to read-only relationships, or provide a primaryjoin
condition marking writable columns with the foreign() annotation.
What this refers to originates from the fact that Article.magazine_id is the subject of two different foreign key constraints; it refers to Magazine.id directly as a source column, but also refers to Writer.magazine_id as a source column in the context of the composite key to Writer.
When objects are added to an ORM Session using Session.add(), the ORM flush process takes on the task of reconciling object refereneces that correspond to relationship() configurations and delivering this state to the databse using INSERT/UPDATE/DELETE statements. In this specific example, if we associate an Article with a particular Magazine, but then associate the Article with a Writer that’s associated with a different Magazine, this flush process will overwrite Article.magazine_id non-deterministically, silently changing which magazine to which we refer; it may also attempt to place NULL into this column if we de-associate a Writer from an Article. The warning lets us know that this scenario may occur during ORM flush sequences.
To solve this, we need to break out the behavior of Article to include all three of the following features:
1. Article first and foremost writes to Article.magazine_id based on data persisted in the Article.magazine relationship only, that is a value copied from Magazine.id.
2. Article can write to Article.writer_id on behalf of data persisted in the Article.writer relationship, but only the Writer.id column; the Writer.magazine_id column should not be written into Article.magazine_id as it ultimately is sourced from Magazine.id.
3. Article takes Article.magazine_id into account when loading Article.writer, even though it doesn’t write to it on behalf of this relationship.
To get just #1 and #2, we could specify only Article.writer_id as the “foreign keys” for Article.writer:
class Article(Base):
    # 

    writer = relationship("Writer", foreign_keys="Article.writer_id")
However, this has the effect of Article.writer not taking Article.magazine_id into account when querying against Writer:
SELECT article.article_id AS article_article_id,
    article.magazine_id AS article_magazine_id,
    article.writer_id AS article_writer_id
FROM article
JOIN writer ON writer.id = article.writer_id
Therefore, to get at all of #1, #2, and #3, we express the join condition as well as which columns to be written by combining relationship.primaryjoin fully, along with either the relationship.foreign_keys argument, or more succinctly by annotating with foreign():
class Article(Base):
    # 

    writer = relationship(
        "Writer",
        primaryjoin="and_(Writer.id == foreign(Article.writer_id), "
        "Writer.magazine_id == Article.magazine_id)",
    )
Non-relational Comparisons / Materialized Path
Warning
this section details an experimental feature.
Using custom expressions means we can produce unorthodox join conditions that don’t obey the usual primary/foreign key model. One such example is the materialized path pattern, where we compare strings for overlapping path tokens in order to produce a tree structure.
Through careful use of foreign() and remote(), we can build a relationship that effectively produces a rudimentary materialized path system. Essentially, when foreign() and remote() are on the same side of the comparison expression, the relationship is considered to be “one to many”; when they are on different sides, the relationship is considered to be “many to one”. For the comparison we’ll use here, we’ll be dealing with collections so we keep things configured as “one to many”:
class Element(Base):
    __tablename__ = "element"

    path = mapped_column(String, primary_key=True)

    descendants = relationship(
        "Element",
        primaryjoin=remote(foreign(path)).like(path.concat("/%")),
        viewonly=True,
        order_by=path,
    )
Above, if given an Element object with a path attribute of "/foo/bar2", we seek for a load of Element.descendants to look like:
SELECT element.path AS element_path
FROM element
WHERE element.path LIKE ('/foo/bar2' || '/%') ORDER BY element.path
Self-Referential Many-to-Many Relationship
See also
This section documents a two-table variant of the “adjacency list” pattern, which is documented at Adjacency List Relationships. Be sure to review the self-referential querying patterns in subsections Self-Referential Query Strategies and Configuring Self-Referential Eager Loading which apply equally well to the mapping pattern discussed here.
Many to many relationships can be customized by one or both of relationship.primaryjoin and relationship.secondaryjoin - the latter is significant for a relationship that specifies a many-to-many reference using the relationship.secondary argument. A common situation which involves the usage of relationship.primaryjoin and relationship.secondaryjoin is when establishing a many-to-many relationship from a class to itself, as shown below:
from typing import List

from sqlalchemy import Integer, ForeignKey, Column, Table
from sqlalchemy.orm import DeclarativeBase, Mapped
from sqlalchemy.orm import mapped_column, relationship


class Base(DeclarativeBase):
    pass


node_to_node = Table(
    "node_to_node",
    Base.metadata,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)


class Node(Base):
    __tablename__ = "node"
    id: Mapped[int] = mapped_column(primary_key=True)
    label: Mapped[str]
    right_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.left_node_id,
        secondaryjoin=id == node_to_node.c.right_node_id,
        back_populates="left_nodes",
    )
    left_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.right_node_id,
        secondaryjoin=id == node_to_node.c.left_node_id,
        back_populates="right_nodes",
    )
Where above, SQLAlchemy can’t know automatically which columns should connect to which for the right_nodes and left_nodes relationships. The relationship.primaryjoin and relationship.secondaryjoin arguments establish how we’d like to join to the association table. In the Declarative form above, as we are declaring these conditions within the Python block that corresponds to the Node class, the id variable is available directly as the Column object we wish to join with.
Alternatively, we can define the relationship.primaryjoin and relationship.secondaryjoin arguments using strings, which is suitable in the case that our configuration does not have either the Node.id column object available yet or the node_to_node table perhaps isn’t yet available. When referring to a plain Table object in a declarative string, we use the string name of the table as it is present in the MetaData:
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    label = mapped_column(String)
    right_nodes = relationship(
        "Node",
        secondary="node_to_node",
        primaryjoin="Node.id==node_to_node.c.left_node_id",
        secondaryjoin="Node.id==node_to_node.c.right_node_id",
        backref="left_nodes",
    )
Warning
When passed as a Python-evaluable string, the relationship.primaryjoin and relationship.secondaryjoin arguments are interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THESE STRINGS. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
A classical mapping situation here is similar, where node_to_node can be joined to node.c.id:
from sqlalchemy import Integer, ForeignKey, String, Column, Table, MetaData
from sqlalchemy.orm import relationship, registry

metadata_obj = MetaData()
mapper_registry = registry()

node_to_node = Table(
    "node_to_node",
    metadata_obj,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)

node = Table(
    "node",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("label", String),
)


class Node:
    pass


mapper_registry.map_imperatively(
    Node,
    node,
    properties={
        "right_nodes": relationship(
            Node,
            secondary=node_to_node,
            primaryjoin=node.c.id == node_to_node.c.left_node_id,
            secondaryjoin=node.c.id == node_to_node.c.right_node_id,
            backref="left_nodes",
        )
    },
)
Note that in both examples, the relationship.backref keyword specifies a left_nodes backref - when relationship() creates the second relationship in the reverse direction, it’s smart enough to reverse the relationship.primaryjoin and relationship.secondaryjoin arguments.
See also
• Adjacency List Relationships - single table version
• Self-Referential Query Strategies - tips on querying with self-referential mappings
• Configuring Self-Referential Eager Loading - tips on eager loading with self- referential mapping
Composite “Secondary” Joins
Note
This section features far edge cases that are somewhat supported by SQLAlchemy, however it is recommended to solve problems like these in simpler ways whenever possible, by using reasonable relational layouts and / or in-Python attributes.
Sometimes, when one seeks to build a relationship() between two tables there is a need for more than just two or three tables to be involved in order to join them. This is an area of relationship() where one seeks to push the boundaries of what’s possible, and often the ultimate solution to many of these exotic use cases needs to be hammered out on the SQLAlchemy mailing list.
In more recent versions of SQLAlchemy, the relationship.secondary parameter can be used in some of these cases in order to provide a composite target consisting of multiple tables. Below is an example of such a join condition (requires version 0.9.2 at least to function as is):
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

    d = relationship(
        "D",
        secondary="join(B, D, B.d_id == D.id).join(C, C.d_id == D.id)",
        primaryjoin="and_(A.b_id == B.id, A.id == C.a_id)",
        secondaryjoin="D.id == B.d_id",
        uselist=False,
        viewonly=True,
    )


class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)
    d_id = mapped_column(ForeignKey("d.id"))


class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))
    d_id = mapped_column(ForeignKey("d.id"))


class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
In the above example, we provide all three of relationship.secondary, relationship.primaryjoin, and relationship.secondaryjoin, in the declarative style referring to the named tables a, b, c, d directly. A query from A to D looks like:
sess.scalars(select(A).join(A.d)).all()

SELECT a.id AS a_id, a.b_id AS a_b_id
FROM a JOIN (
    b AS b_1 JOIN d AS d_1 ON b_1.d_id = d_1.id
        JOIN c AS c_1 ON c_1.d_id = d_1.id)
    ON a.b_id = b_1.id AND a.id = c_1.a_id JOIN d ON d.id = b_1.d_id
In the above example, we take advantage of being able to stuff multiple tables into a “secondary” container, so that we can join across many tables while still keeping things “simple” for relationship(), in that there’s just “one” table on both the “left” and the “right” side; the complexity is kept within the middle.
Warning
A relationship like the above is typically marked as viewonly=True, using relationship.viewonly, and should be considered as read-only. While there are sometimes ways to make relationships like the above writable, this is generally complicated and error prone.
See also
Notes on using the viewonly relationship parameter
Relationship to Aliased Class
In the previous section, we illustrated a technique where we used relationship.secondary in order to place additional tables within a join condition. There is one complex join case where even this technique is not sufficient; when we seek to join from A to B, making use of any number of C, D, etc. in between, however there are also join conditions between A and B directly. In this case, the join from A to B may be difficult to express with just a complex relationship.primaryjoin condition, as the intermediary tables may need special handling, and it is also not expressible with a relationship.secondary object, since the A->secondary->B pattern does not support any references between A and B directly. When this extremely advanced case arises, we can resort to creating a second mapping as a target for the relationship. This is where we use AliasedClass in order to make a mapping to a class that includes all the additional tables we need for this join. In order to produce this mapper as an “alternative” mapping for our class, we use the aliased() function to produce the new construct, then use relationship() against the object as though it were a plain mapped class.
Below illustrates a relationship() with a simple join from A to B, however the primaryjoin condition is augmented with two additional entities C and D, which also must have rows that line up with the rows in both A and B simultaneously:
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))


class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)


class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))

    some_c_value = mapped_column(String)


class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
    c_id = mapped_column(ForeignKey("c.id"))
    b_id = mapped_column(ForeignKey("b.id"))

    some_d_value = mapped_column(String)


# 1. set up the join() as a variable, so we can refer
# to it in the mapping multiple times.
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

# 2. Create an AliasedClass to B
B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
With the above mapping, a simple join looks like:
sess.scalars(select(A).join(A.b)).all()

SELECT a.id AS a_id, a.b_id AS a_b_id
FROM a JOIN (b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) ON a.b_id = b.id
Integrating AliasedClass Mappings with Typing and Avoiding Early Mapper Configuration
The creation of the aliased() construct against a mapped class forces the configure_mappers() step to proceed, which will resolve all current classes and their relationships. This may be problematic if unrelated mapped classes needed by the current mappings have not yet been declared, or if the configuration of the relationship itself needs access to as-yet undeclared classes. Additionally, SQLAlchemy’s Declarative pattern works with Python typing most effectively when relationships are declared up front.
To organize the construction of the relationship to work with these issues, a configure level event hook like MapperEvents.before_mapper_configured() may be used, which will invoke the configuration code only when all mappings are ready for configuration:
from sqlalchemy import event


class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))


@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # do the above configuration in a configuration hook

    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, j, flat=True)
    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
Above, the function _configure_ab_relationship() will be invoked only when a fully configured version of A is requested, at which point the classes B, D and C would be available.
For an approach that integrates with inline typing, a similar technique can be used to effectively generate a “singleton” creation pattern for the aliased class where it is late-initialized as a global variable, which can then be used in the relationship inline:
from typing import Any

B_viacd: Any = None
b_viacd_join: Any = None


class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey("b.id"))

    # 1. the relationship can be declared using lambdas, allowing it to resolve
    #    to targets that are late-configured
    b: Mapped[B] = relationship(
        lambda: B_viacd, primaryjoin=lambda: A.b_id == b_viacd_join.c.b_id
    )


# 2. configure the targets of the relationship using a before_mapper_configured
#    hook.
@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # 3. set up the join() and AliasedClass as globals from within
    #    the configuration hook.

    global B_viacd, b_viacd_join

    b_viacd_join = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, b_viacd_join, flat=True)
Using the AliasedClass target in Queries
In the previous example, the A.b relationship refers to the B_viacd entity as the target, and not the B class directly. To add additional criteria involving the A.b relationship, it’s typically necessary to reference the B_viacd directly rather than using B, especially in a case where the target entity of A.b is to be transformed into an alias or a subquery. Below illustrates the same relationship using a subquery, rather than a join:
subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

B_viacd_subquery = aliased(B, subq)

A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)
A query using the above A.b relationship will render a subquery:
sess.scalars(select(A).join(A.b)).all()

SELECT a.id AS a_id, a.b_id AS a_b_id
FROM a JOIN (SELECT b.id AS id, b.some_b_column AS some_b_column
FROM b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) AS anon_1 ON a.b_id = anon_1.id
If we want to add additional criteria based on the A.b join, we must do so in terms of B_viacd_subquery rather than B directly:
sess.scalars(
    select(A)
    .join(A.b)
    .where(B_viacd_subquery.some_b_column == "some b")
    .order_by(B_viacd_subquery.id)
).all()

SELECT a.id AS a_id, a.b_id AS a_b_id
FROM a JOIN (SELECT b.id AS id, b.some_b_column AS some_b_column
FROM b JOIN d ON d.b_id = b.id JOIN c ON c.id = d.c_id) AS anon_1 ON a.b_id = anon_1.id
WHERE anon_1.some_b_column = ? ORDER BY anon_1.id
Row-Limited Relationships with Window Functions
Another interesting use case for relationships to AliasedClass objects are situations where the relationship needs to join to a specialized SELECT of any form. One scenario is when the use of a window function is desired, such as to limit how many rows should be returned for a relationship. The example below illustrates a non-primary mapper relationship that will load the first ten items for each collection:
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)


class B(Base):
    __tablename__ = "b"
    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))


partition = select(
    B, func.row_number().over(order_by=B.id, partition_by=B.a_id).label("index")
).alias()

partitioned_b = aliased(B, partition)

A.partitioned_bs = relationship(
    partitioned_b, primaryjoin=and_(partitioned_b.a_id == A.id, partition.c.index < 10)
)
We can use the above partitioned_bs relationship with most of the loader strategies, such as selectinload():
for a1 in session.scalars(select(A).options(selectinload(A.partitioned_bs))):
    print(a1.partitioned_bs)  # <-- will be no more than ten objects
Where above, the “selectinload” query looks like:
SELECT
    a_1.id AS a_1_id, anon_1.id AS anon_1_id, anon_1.a_id AS anon_1_a_id,
    anon_1.data AS anon_1_data, anon_1.index AS anon_1_index
FROM a AS a_1
JOIN (
    SELECT b.id AS id, b.a_id AS a_id, b.data AS data,
    row_number() OVER (PARTITION BY b.a_id ORDER BY b.id) AS index
    FROM b) AS anon_1
ON anon_1.a_id = a_1.id AND anon_1.index < %(index_1)s
WHERE a_1.id IN ( primary key collection )
ORDER BY a_1.id
Above, for each matching primary key in “a”, we will get the first ten “bs” as ordered by “b.id”. By partitioning on “a_id” we ensure that each “row number” is local to the parent “a_id”.
Such a mapping would ordinarily also include a “plain” relationship from “A” to “B”, for persistence operations as well as when the full set of “B” objects per “A” is desired.
Building Query-Enabled Properties
Very ambitious custom join conditions may fail to be directly persistable, and in some cases may not even load correctly. To remove the persistence part of the equation, use the flag relationship.viewonly on the relationship(), which establishes it as a read-only attribute (data written to the collection will be ignored on flush()). However, in extreme cases, consider using a regular Python property in conjunction with Query as follows:
class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)

    @property
    def addresses(self):
        return object_session(self).query(Address).with_parent(self).filter().all()
In other cases, the descriptor can be built to make use of existing in-Python data. See the section on Using Descriptors and Hybrids for more general discussion of special Python attributes.
See also
Using Descriptors and Hybrids
Notes on using the viewonly relationship parameter
The relationship.viewonly parameter when applied to a relationship() construct indicates that this relationship() will not take part in any ORM unit of work operations, and additionally that the attribute does not expect to participate within in-Python mutations of its represented collection. This means that while the viewonly relationship may refer to a mutable Python collection like a list or set, making changes to that list or set as present on a mapped instance will have no effect on the ORM flush process.
To explore this scenario consider this mapping:
from __future__ import annotations

import datetime

from sqlalchemy import and_
from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship()

    current_week_tasks: Mapped[list[Task]] = relationship(
        primaryjoin=lambda: and_(
            User.id == Task.user_account_id,
            # this expression works on PostgreSQL but may not be supported
            # by other database engines
            Task.task_date >= func.now() - datetime.timedelta(days=7),
        ),
        viewonly=True,
    )


class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="current_week_tasks")
The following sections will note different aspects of this configuration.
In-Python mutations including backrefs are not appropriate with viewonly=True
The above mapping targets the User.current_week_tasks viewonly relationship as the backref target of the Task.user attribute. This is not currently flagged by SQLAlchemy’s ORM configuration process, however is a configuration error. Changing the .user attribute on a Task will not affect the .current_week_tasks attribute:
 u1 = User()
 t1 = Task(task_date=datetime.datetime.now())
 t1.user = u1
 u1.current_week_tasks
[]
There is another parameter called relationship.sync_backrefs which can be turned on here to allow .current_week_tasks to be mutated in this case, however this is not considered to be a best practice with a viewonly relationship, which instead should not be relied upon for in-Python mutations.
In this mapping, backrefs can be configured between User.all_tasks and Task.user, as these are both not viewonly and will synchronize normally.
Beyond the issue of backref mutations being disabled for viewonly relationships, plain changes to the User.all_tasks collection in Python are also not reflected in the User.current_week_tasks collection until changes have been flushed to the database.
Overall, for a use case where a custom collection should respond immediately to in-Python mutations, the viewonly relationship is generally not appropriate. A better approach is to use the Hybrid Attributes feature of SQLAlchemy, or for instance-only cases to use a Python @property, where a user-defined collection that is generated in terms of the current Python instance can be implemented. To change our example to work this way, we repair the relationship.back_populates parameter on Task.user to reference User.all_tasks, and then illustrate a simple @property that will deliver results in terms of the immediate User.all_tasks collection:
class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship(back_populates="user")

    @property
    def current_week_tasks(self) -> list[Task]:
        past_seven_days = datetime.datetime.now() - datetime.timedelta(days=7)
        return [t for t in self.all_tasks if t.task_date >= past_seven_days]


class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="all_tasks")
Using an in-Python collection calculated on the fly each time, we are guaranteed to have the correct answer at all times, without the need to use a database at all:
 u1 = User()
 t1 = Task(task_date=datetime.datetime.now())
 t1.user = u1
 u1.current_week_tasks
[<__main__.Task object at 0x7f3d699523c0>]
viewonly=True collections / attributes do not get re-queried until expired
Continuing with the original viewonly attribute, if we do in fact make changes to the User.all_tasks collection on a persistent object, the viewonly collection can only show the net result of this change after two things occur. The first is that the change to User.all_tasks is flushed, so that the new data is available in the database, at least within the scope of the local transaction. The second is that the User.current_week_tasks attribute is expired and reloaded via a new SQL query to the database.
To support this requirement, the simplest flow to use is one where the viewonly relationship is consumed only in operations that are primarily read only to start with. Such as below, if we retrieve a User fresh from the database, the collection will be current:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b906b0>]
When we make modifications to u1.all_tasks, if we want to see these changes reflected in the u1.current_week_tasks viewonly relationship, these changes need to be flushed and the u1.current_week_tasks attribute needs to be expired, so that it will lazy load on next access. The simplest approach to this is to use Session.commit(), keeping the Session.expire_on_commit parameter set at its default of True:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
    sess.commit()
    print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b90ec0>, <__main__.Task object at 0x7f8711b90a10>]
Above, the call to Session.commit() flushed the changes to u1.all_tasks to the database, then expired all objects, so that when we accessed u1.current_week_tasks, a :term:` lazy load` occurred which fetched the contents for this attribute freshly from the database.
To intercept operations without actually committing the transaction, the attribute needs to be explicitly expired first. A simplistic way to do this is to just call it directly. In the example below, Session.flush() sends pending changes to the database, then Session.expire() is used to expire the u1.current_week_tasks collection so that it re-fetches on next access:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
    sess.flush()
    sess.expire(u1, ["current_week_tasks"])
    print(u1.current_week_tasks)
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
We can in fact skip the call to Session.flush(), assuming a Session that keeps Session.autoflush at its default value of True, as the expired current_week_tasks attribute will trigger autoflush when accessed after expiration:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
    sess.expire(u1, ["current_week_tasks"])
    print(u1.current_week_tasks)  # triggers autoflush before querying
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
Continuing with the above approach to something more elaborate, we can apply the expiration programmatically when the related User.all_tasks collection changes, using event hooks. This an advanced technique, where simpler architectures like @property or sticking to read-only use cases should be examined first. In our simple example, this would be configured as:
from sqlalchemy import event, inspect


@event.listens_for(User.all_tasks, "append")
@event.listens_for(User.all_tasks, "remove")
@event.listens_for(User.all_tasks, "bulk_replace")
def _expire_User_current_week_tasks(target, value, initiator):
    inspect(target).session.expire(target, ["current_week_tasks"])
With the above hooks, mutation operations are intercepted and result in the User.current_week_tasks collection to be expired automatically:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
    print(u1.current_week_tasks)
[<__main__.Task object at 0x7f66d093ccb0>, <__main__.Task object at 0x7f66d093cce0>]
The AttributeEvents event hooks used above are also triggered by backref mutations, so with the above hooks a change to Task.user is also intercepted:
 with Session(e) as sess:
    u1 = sess.scalar(select(User).where(User.id == 1))
    t1 = Task(task_date=datetime.datetime.now())
    t1.user = u1
    sess.add(t1)
    print(u1.current_week_tasks)
[<__main__.Task object at 0x7f3b0c070d10>, <__main__.Task object at 0x7f3b0c057d10>]


Working with Large Collections
The default behavior of relationship() is to fully load the contents of collections into memory, based on a configured loader strategy that controls when and how these contents are loaded from the database. Related collections may be loaded into memory not just when they are accessed, or eagerly loaded, but in most cases will require population when the collection itself is mutated, as well as in cases where the owning object is to be deleted by the unit of work system.
When a related collection is potentially very large, it may not be feasible for such a collection to be populated into memory under any circumstances, as the operation may be overly consuming of time, network and memory resources.
This section includes API features intended to allow relationship() to be used with large collections while maintaining adequate performance.
Write Only Relationships
The write only loader strategy is the primary means of configuring a relationship() that will remain writeable, but will not load its contents into memory. A write-only ORM configuration in modern type-annotated Declarative form is illustrated below:
 from decimal import Decimal
 from datetime import datetime

 from sqlalchemy import ForeignKey
 from sqlalchemy import func
 from sqlalchemy.orm import DeclarativeBase
 from sqlalchemy.orm import Mapped
 from sqlalchemy.orm import mapped_column
 from sqlalchemy.orm import relationship
 from sqlalchemy.orm import Session
 from sqlalchemy.orm import WriteOnlyMapped

 class Base(DeclarativeBase):
    pass

 class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]

    account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

    def __repr__(self):
        return f"Account(identifier={self.identifier!r})"

 class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())

    def __repr__(self):
        return (
            f"AccountTransaction(amount={self.amount:.2f}, "
            f"timestamp={self.timestamp.isoformat()!r})"
        )

    __mapper_args__ = {"eager_defaults": True}
Above, the account_transactions relationship is configured not using the ordinary Mapped annotation, but instead using the WriteOnlyMapped type annotation, which at runtime will assign the loader strategy of lazy="write_only" to the target relationship(). The WriteOnlyMapped annotation is an alternative form of the Mapped annotation which indicate the use of the WriteOnlyCollection collection type on instances of the object.
The above relationship() configuration also includes several elements that are specific to what action to take when Account objects are deleted, as well as when AccountTransaction objects are removed from the account_transactions collection. These elements are:
• passive_deletes=True - allows the unit of work to forego having to load the collection when Account is deleted; see Using foreign key ON DELETE cascade with ORM relationships.
• ondelete="cascade" configured on the ForeignKey constraint. This is also detailed at Using foreign key ON DELETE cascade with ORM relationships.
• cascade="all, delete-orphan" - instructs the unit of work to delete AccountTransaction objects when they are removed from the collection. See delete-orphan in the Cascades document.
Added in version 2.0: Added “Write only” relationship loaders.
Creating and Persisting New Write Only Collections
The write-only collection allows for direct assignment of the collection as a whole only for transient or pending objects. With our above mapping, this indicates we can create a new Account object with a sequence of AccountTransaction objects to be added to a Session. Any Python iterable may be used as the source of objects to start, where below we use a Python list:
 new_account = Account(
    identifier="account_01",
    account_transactions=[
        AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
        AccountTransaction(description="transfer", amount=Decimal("1000.00")),
        AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
    ],
)

 with Session(engine) as session:
    session.add(new_account)
    session.commit()
BEGIN (implicit)
INSERT INTO account (identifier) VALUES (?)
[] ('account_01',)
INSERT INTO account_transaction (account_id, description, amount, timestamp)
VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
[(insertmanyvalues) 1/3 (ordered; batch not supported)] (1, 'initial deposit', 500.0)
INSERT INTO account_transaction (account_id, description, amount, timestamp)
VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
[insertmanyvalues 2/3 (ordered; batch not supported)] (1, 'transfer', 1000.0)
INSERT INTO account_transaction (account_id, description, amount, timestamp)
VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
[insertmanyvalues 3/3 (ordered; batch not supported)] (1, 'withdrawal', -29.5)
COMMIT
Once an object is database-persisted (i.e. in the persistent or detached state), the collection has the ability to be extended with new items as well as the ability for individual items to be removed. However, the collection may no longer be re-assigned with a full replacement collection, as such an operation requires that the previous collection is fully loaded into memory in order to reconcile the old entries with the new ones:
 new_account.account_transactions = [
    AccountTransaction(description="some transaction", amount=Decimal("10.00"))
]
Traceback (most recent call last):

sqlalchemy.exc.InvalidRequestError: Collection "Account.account_transactions" does not
support implicit iteration; collection replacement operations can't be used
Adding New Items to an Existing Collection
For write-only collections of persistent objects, modifications to the collection using unit of work processes may proceed only by using the WriteOnlyCollection.add(), WriteOnlyCollection.add_all() and WriteOnlyCollection.remove() methods:
 from sqlalchemy import select
 session = Session(engine, expire_on_commit=False)
 existing_account = session.scalar(select(Account).filter_by(identifier="account_01"))
BEGIN (implicit)
SELECT account.id, account.identifier
FROM account
WHERE account.identifier = ?
[] ('account_01',)
 existing_account.account_transactions.add_all(
    [
        AccountTransaction(description="paycheck", amount=Decimal("2000.00")),
        AccountTransaction(description="rent", amount=Decimal("-800.00")),
    ]
)
 session.commit()
INSERT INTO account_transaction (account_id, description, amount, timestamp)
VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
[(insertmanyvalues) 1/2 (ordered; batch not supported)] (1, 'paycheck', 2000.0)
INSERT INTO account_transaction (account_id, description, amount, timestamp)
VALUES (?, ?, ?, CURRENT_TIMESTAMP) RETURNING id, timestamp
[insertmanyvalues 2/2 (ordered; batch not supported)] (1, 'rent', -800.0)
COMMIT
The items added above are held in a pending queue within the Session until the next flush, at which point they are INSERTed into the database, assuming the added objects were previously transient.
Querying Items
The WriteOnlyCollection does not at any point store a reference to the current contents of the collection, nor does it have any behavior where it would directly emit a SELECT to the database in order to load them; the overriding assumption is that the collection may contain many thousands or millions of rows, and should never be fully loaded into memory as a side effect of any other operation.
Instead, the WriteOnlyCollection includes SQL-generating helpers such as WriteOnlyCollection.select(), which will generate a Select construct pre-configured with the correct WHERE / FROM criteria for the current parent row, which can then be further modified in order to SELECT any range of rows desired, as well as invoked using features like server side cursors for processes that wish to iterate through the full collection in a memory-efficient manner.
The statement generated is illustrated below. Note it also includes ORDER BY criteria, indicated in the example mapping by the relationship.order_by parameter of relationship(); this criteria would be omitted if the parameter were not configured:
 print(existing_account.account_transactions.select())
SELECT account_transaction.id, account_transaction.account_id, account_transaction.description,
account_transaction.amount, account_transaction.timestamp
FROM account_transaction
WHERE :param_1 = account_transaction.account_id ORDER BY account_transaction.timestamp
We may use this Select construct along with the Session in order to query for AccountTransaction objects, most easily using the Session.scalars() method that will return a Result that yields ORM objects directly. It’s typical, though not required, that the Select would be modified further to limit the records returned; in the example below, additional WHERE criteria to load only “debit” account transactions is added, along with “LIMIT 10” to retrieve only the first ten rows:
 account_transactions = session.scalars(
    existing_account.account_transactions.select()
    .where(AccountTransaction.amount < 0)
    .limit(10)
).all()
BEGIN (implicit)
SELECT account_transaction.id, account_transaction.account_id, account_transaction.description,
account_transaction.amount, account_transaction.timestamp
FROM account_transaction
WHERE ? = account_transaction.account_id AND account_transaction.amount < ?
ORDER BY account_transaction.timestamp  LIMIT ? OFFSET ?
[] (1, 0, 10, 0)
 print(account_transactions)
[AccountTransaction(amount=-29.50, timestamp=''), AccountTransaction(amount=-800.00, timestamp='')]
Removing Items
Individual items that are loaded in the persistent state against the current Session may be marked for removal from the collection using the WriteOnlyCollection.remove() method. The flush process will implicitly consider the object to be already part of the collection when the operation proceeds. The example below illustrates removal of an individual AccountTransaction item, which per cascade settings results in a DELETE of that row:
 existing_transaction = account_transactions[0]
 existing_account.account_transactions.remove(existing_transaction)
 session.commit()
DELETE FROM account_transaction WHERE account_transaction.id = ?
[] (3,)
COMMIT
As with any ORM-mapped collection, object removal may proceed either to de-associate the object from the collection while leaving the object present in the database, or may issue a DELETE for its row, based on the delete-orphan configuration of the relationship().
Collection removal without deletion involves setting foreign key columns to NULL for a one-to-many relationship, or deleting the corresponding association row for a many-to-many relationship.
Bulk INSERT of New Items
The WriteOnlyCollection can generate DML constructs such as Insert objects, which may be used in an ORM context to produce bulk insert behavior. See the section ORM Bulk INSERT Statements for an overview of ORM bulk inserts.
One to Many Collections
For a regular one to many collection only, the WriteOnlyCollection.insert() method will produce an Insert construct which is pre-established with VALUES criteria corresponding to the parent object. As this VALUES criteria is entirely against the related table, the statement can be used to INSERT new rows that will at the same time become new records in the related collection:
 session.execute(
    existing_account.account_transactions.insert(),
    [
        {"description": "transaction 1", "amount": Decimal("47.50")},
        {"description": "transaction 2", "amount": Decimal("-501.25")},
        {"description": "transaction 3", "amount": Decimal("1800.00")},
        {"description": "transaction 4", "amount": Decimal("-300.00")},
    ],
)
BEGIN (implicit)
INSERT INTO account_transaction (account_id, description, amount, timestamp) VALUES (?, ?, ?, CURRENT_TIMESTAMP)
[] [(1, 'transaction 1', 47.5), (1, 'transaction 2', -501.25), (1, 'transaction 3', 1800.0), (1, 'transaction 4', -300.0)]
<>
 session.commit()
COMMIT
See also
ORM Bulk INSERT Statements - in the ORM Querying Guide
One To Many - at Basic Relationship Patterns
Many to Many Collections
For a many to many collection, the relationship between two classes involves a third table that is configured using the relationship.secondary parameter of relationship. To bulk insert rows into a collection of this type using WriteOnlyCollection, the new records may be bulk-inserted separately first, retrieved using RETURNING, and those records then passed to the WriteOnlyCollection.add_all() method where the unit of work process will proceed to persist them as part of the collection.
Supposing a class BankAudit referred to many AccountTransaction records using a many-to-many table:
 from sqlalchemy import Table, Column
 audit_to_transaction = Table(
    "audit_transaction",
    Base.metadata,
    Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
    Column(
        "transaction_id",
        ForeignKey("account_transaction.id", ondelete="CASCADE"),
        primary_key=True,
    ),
)
 class BankAudit(Base):
    __tablename__ = "audit"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
        secondary=audit_to_transaction, passive_deletes=True
    )
To illustrate the two operations, we add more AccountTransaction objects using bulk insert, which we retrieve using RETURNING by adding returning(AccountTransaction) to the bulk INSERT statement (note that we could just as easily use existing AccountTransaction objects as well):
 new_transactions = session.scalars(
    existing_account.account_transactions.insert().returning(AccountTransaction),
    [
        {"description": "odd trans 1", "amount": Decimal("50000.00")},
        {"description": "odd trans 2", "amount": Decimal("25000.00")},
        {"description": "odd trans 3", "amount": Decimal("45.00")},
    ],
).all()
BEGIN (implicit)
INSERT INTO account_transaction (account_id, description, amount, timestamp) VALUES
(?, ?, ?, CURRENT_TIMESTAMP), (?, ?, ?, CURRENT_TIMESTAMP), (?, ?, ?, CURRENT_TIMESTAMP)
RETURNING id, account_id, description, amount, timestamp
[] (1, 'odd trans 1', 50000.0, 1, 'odd trans 2', 25000.0, 1, 'odd trans 3', 45.0)
With a list of AccountTransaction objects ready, the WriteOnlyCollection.add_all() method is used to associate many rows at once with a new BankAudit object:
 bank_audit = BankAudit()
 session.add(bank_audit)
 bank_audit.account_transactions.add_all(new_transactions)
 session.commit()
INSERT INTO audit DEFAULT VALUES
[] ()
INSERT INTO audit_transaction (audit_id, transaction_id) VALUES (?, ?)
[] [(1, 10), (1, 11), (1, 12)]
COMMIT
See also
ORM Bulk INSERT Statements - in the ORM Querying Guide
Many To Many - at Basic Relationship Patterns
Bulk UPDATE and DELETE of Items
In a similar way in which WriteOnlyCollection can generate Select constructs with WHERE criteria pre-established, it can also generate Update and Delete constructs with that same WHERE criteria, to allow criteria-oriented UPDATE and DELETE statements against the elements in a large collection.
One To Many Collections
As is the case with INSERT, this feature is most straightforward with one to many collections.
In the example below, the WriteOnlyCollection.update() method is used to generate an UPDATE statement is emitted against the elements in the collection, locating rows where the “amount” is equal to -800 and adding the amount of 200 to them:
 session.execute(
    existing_account.account_transactions.update()
    .values(amount=AccountTransaction.amount + 200)
    .where(AccountTransaction.amount == -800),
)
BEGIN (implicit)
UPDATE account_transaction SET amount=(account_transaction.amount + ?)
WHERE ? = account_transaction.account_id AND account_transaction.amount = ?
[] (200, 1, -800)
<>
In a similar way, WriteOnlyCollection.delete() will produce a DELETE statement that is invoked in the same way:
 session.execute(
    existing_account.account_transactions.delete().where(
        AccountTransaction.amount.between(0, 30)
    ),
)
DELETE FROM account_transaction WHERE ? = account_transaction.account_id
AND account_transaction.amount BETWEEN ? AND ? RETURNING id
[] (1, 0, 30)
<>
Many to Many Collections
Tip
The techniques here involve multi-table UPDATE expressions, which are slightly more advanced.
For bulk UPDATE and DELETE of many to many collections, in order for an UPDATE or DELETE statement to relate to the primary key of the parent object, the association table must be explicitly part of the UPDATE/DELETE statement, which requires either that the backend includes supports for non-standard SQL syntaxes, or extra explicit steps when constructing the UPDATE or DELETE statement.
For backends that support multi-table versions of UPDATE, the WriteOnlyCollection.update() method should work without extra steps for a many-to-many collection, as in the example below where an UPDATE is emitted against AccountTransaction objects in terms of the many-to-many BankAudit.account_transactions collection:
 session.execute(
    bank_audit.account_transactions.update().values(
        description=AccountTransaction.description + " (audited)"
    )
)
UPDATE account_transaction SET description=(account_transaction.description || ?)
FROM audit_transaction WHERE ? = audit_transaction.audit_id
AND account_transaction.id = audit_transaction.transaction_id RETURNING id
[] (' (audited)', 1)
<>
The above statement automatically makes use of “UPDATE..FROM” syntax, supported by SQLite and others, to name the additional audit_transaction table in the WHERE clause.
To UPDATE or DELETE a many-to-many collection where multi-table syntax is not available, the many-to-many criteria may be moved into SELECT that for example may be combined with IN to match rows. The WriteOnlyCollection still helps us here, as we use the WriteOnlyCollection.select() method to generate this SELECT for us, making use of the Select.with_only_columns() method to produce a scalar subquery:
 from sqlalchemy import update
 subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
 session.execute(
    update(AccountTransaction)
    .values(description=AccountTransaction.description + " (audited)")
    .where(AccountTransaction.id.in_(subq))
)
UPDATE account_transaction SET description=(account_transaction.description || ?)
WHERE account_transaction.id IN (SELECT account_transaction.id
FROM audit_transaction
WHERE ? = audit_transaction.audit_id AND account_transaction.id = audit_transaction.transaction_id)
RETURNING id
[] (' (audited)', 1)
<>
Write Only Collections - API Documentation
Object NameDescriptionWriteOnlyCollectionWrite-only collection which can synchronize changes into the attribute event system.WriteOnlyMappedRepresent the ORM mapped attribute type for a “write only” relationship.class sqlalchemy.orm.WriteOnlyCollection
Write-only collection which can synchronize changes into the attribute event system.
The WriteOnlyCollection is used in a mapping by using the "write_only" lazy loading strategy with relationship(). For background on this configuration, see Write Only Relationships.
Added in version 2.0.
See also
Write Only Relationships
Members
add(), add_all(), delete(), insert(), remove(), select(), update()
Class signature
class sqlalchemy.orm.WriteOnlyCollection (sqlalchemy.orm.writeonly.AbstractCollectionWriter)
method sqlalchemy.orm.WriteOnlyCollection.add(item: _T) → None
Add an item to this WriteOnlyCollection.
The given item will be persisted to the database in terms of the parent instance’s collection on the next flush.
method sqlalchemy.orm.WriteOnlyCollection.add_all(iterator: Iterable[_T]) → None
Add an iterable of items to this WriteOnlyCollection.
The given items will be persisted to the database in terms of the parent instance’s collection on the next flush.
method sqlalchemy.orm.WriteOnlyCollection.delete() → Delete
Produce a Delete which will refer to rows in terms of this instance-local WriteOnlyCollection.
method sqlalchemy.orm.WriteOnlyCollection.insert() → Insert
For one-to-many collections, produce a Insert which will insert new rows in terms of this this instance-local WriteOnlyCollection.
This construct is only supported for a Relationship that does not include the relationship.secondary parameter. For relationships that refer to a many-to-many table, use ordinary bulk insert techniques to produce new objects, then use AbstractCollectionWriter.add_all() to associate them with the collection.
method sqlalchemy.orm.WriteOnlyCollection.remove(item: _T) → None
Remove an item from this WriteOnlyCollection.
The given item will be removed from the parent instance’s collection on the next flush.
method sqlalchemy.orm.WriteOnlyCollection.select() → Select[Tuple[_T]]
Produce a Select construct that represents the rows within this instance-local WriteOnlyCollection.
method sqlalchemy.orm.WriteOnlyCollection.update() → Update
Produce a Update which will refer to rows in terms of this instance-local WriteOnlyCollection.
class sqlalchemy.orm.WriteOnlyMapped
Represent the ORM mapped attribute type for a “write only” relationship.
The WriteOnlyMapped type annotation may be used in an Annotated Declarative Table mapping to indicate that the lazy="write_only" loader strategy should be used for a particular relationship().
E.g.:
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    addresses: WriteOnlyMapped[Address] = relationship(
        cascade="all,delete-orphan"
    )
See the section Write Only Relationships for background.
Added in version 2.0.
See also
Write Only Relationships - complete background
DynamicMapped - includes legacy Query support
Class signature
class sqlalchemy.orm.WriteOnlyMapped (sqlalchemy.orm.base._MappedAnnotationBase)
Dynamic Relationship Loaders
Legacy Feature
The “dynamic” lazy loader strategy is the legacy form of what is now the “write_only” strategy described in the section Write Only Relationships.
The “dynamic” strategy produces a legacy Query object from the related collection. However, a major drawback of “dynamic” relationships is that there are several cases where the collection will fully iterate, some of which are non-obvious, which can only be prevented with careful programming and testing on a case-by-case basis. Therefore, for truly large collection management, the WriteOnlyCollection should be preferred.
The dynamic loader is also not compatible with the Asynchronous I/O (asyncio) extension. It can be used with some limitations, as indicated in Asyncio dynamic guidelines, but again the WriteOnlyCollection, which is fully compatible with asyncio, should be preferred.
The dynamic relationship strategy allows configuration of a relationship() which when accessed on an instance will return a legacy Query object in place of the collection. The Query can then be modified further so that the database collection may be iterated based on filtering criteria. The returned Query object is an instance of AppenderQuery, which combines the loading and iteration behavior of Query along with rudimentary collection mutation methods such as AppenderQuery.append() and AppenderQuery.remove().
The “dynamic” loader strategy may be configured with type-annotated Declarative form using the DynamicMapped annotation class:
from sqlalchemy.orm import DynamicMapped


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    posts: DynamicMapped[Post] = relationship()
Above, the User.posts collection on an individual User object will return the AppenderQuery object, which is a subclass of Query that also supports basic collection mutation operations:
jack = session.get(User, id)

# filter Jack's blog posts
posts = jack.posts.filter(Post.headline == "this is a post")

# apply array slices
posts = jack.posts[5:20]
The dynamic relationship supports limited write operations, via the AppenderQuery.append() and AppenderQuery.remove() methods:
oldpost = jack.posts.filter(Post.headline == "old post").one()
jack.posts.remove(oldpost)

jack.posts.append(Post("new post"))
Since the read side of the dynamic relationship always queries the database, changes to the underlying collection will not be visible until the data has been flushed. However, as long as “autoflush” is enabled on the Session in use, this will occur automatically each time the collection is about to emit a query.
Dynamic Relationship Loaders - API
Object NameDescriptionAppenderQueryA dynamic query that supports basic collection storage operations.DynamicMappedRepresent the ORM mapped attribute type for a “dynamic” relationship.class sqlalchemy.orm.AppenderQuery
A dynamic query that supports basic collection storage operations.
Methods on AppenderQuery include all methods of Query, plus additional methods used for collection persistence.
Members
add(), add_all(), append(), count(), extend(), get_children(), prefix_with(), remove(), suffix_with(), with_hint(), with_statement_hint()
Class signature
class sqlalchemy.orm.AppenderQuery (sqlalchemy.orm.dynamic.AppenderMixin, sqlalchemy.orm.Query)
method sqlalchemy.orm.AppenderQuery.add(item: _T) → None
inherited from the AppenderMixin.add() method of AppenderMixin
Add an item to this AppenderQuery.
The given item will be persisted to the database in terms of the parent instance’s collection on the next flush.
This method is provided to assist in delivering forwards-compatibility with the WriteOnlyCollection collection class.
Added in version 2.0.
method sqlalchemy.orm.AppenderQuery.add_all(iterator: Iterable[_T]) → None
inherited from the AppenderMixin.add_all() method of AppenderMixin
Add an iterable of items to this AppenderQuery.
The given items will be persisted to the database in terms of the parent instance’s collection on the next flush.
This method is provided to assist in delivering forwards-compatibility with the WriteOnlyCollection collection class.
Added in version 2.0.
method sqlalchemy.orm.AppenderQuery.append(item: _T) → None
inherited from the AppenderMixin.append() method of AppenderMixin
Append an item to this AppenderQuery.
The given item will be persisted to the database in terms of the parent instance’s collection on the next flush.
method sqlalchemy.orm.AppenderQuery.count() → int
inherited from the AppenderMixin.count() method of AppenderMixin
Return a count of rows this the SQL formed by this Query would return.
This generates the SQL for this Query as follows:
SELECT count(1) AS count_1 FROM (
    SELECT <rest of query follows>
) AS anon_1
The above SQL returns a single row, which is the aggregate value of the count function; the Query.count() method then returns that single integer value.
Warning
It is important to note that the value returned by count() is not the same as the number of ORM objects that this Query would return from a method such as the .all() method. The Query object, when asked to return full entities, will deduplicate entries based on primary key, meaning if the same primary key value would appear in the results more than once, only one object of that primary key would be present. This does not apply to a query that is against individual columns.
See also
My Query does not return the same number of objects as query.count() tells me - why?
For fine grained control over specific columns to count, to skip the usage of a subquery or otherwise control of the FROM clause, or to use other aggregate functions, use expression.func expressions in conjunction with Session.query(), i.e.:
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
See also
2.0 Migration - ORM Usage
method sqlalchemy.orm.AppenderQuery.extend(iterator: Iterable[_T]) → None
inherited from the AppenderMixin.extend() method of AppenderMixin
Add an iterable of items to this AppenderQuery.
The given items will be persisted to the database in terms of the parent instance’s collection on the next flush.
method sqlalchemy.orm.AppenderQuery.get_children(*, omit_attrs: Tuple[str, ] = (), **kw: Any) → Iterable[HasTraverseInternals]
inherited from the HasTraverseInternals.get_children() method of HasTraverseInternals
Return immediate child HasTraverseInternals elements of this HasTraverseInternals.
This is used for visit traversal.
**kw may contain flags that change the collection that is returned, for example to return a subset of items in order to cut down on larger traversals, or to return child items from a different context (such as schema-level collections instead of clause-level).
method sqlalchemy.orm.AppenderQuery.prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
inherited from the HasPrefixes.prefix_with() method of HasPrefixes
Add one or more expressions following the statement keyword, i.e. SELECT, INSERT, UPDATE, or DELETE. Generative.
This is used to support backend-specific prefix keywords such as those provided by MySQL.
E.g.:
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with("/*+ BKA(t1) */", dialect="mysql")
Multiple prefixes can be specified by multiple calls to HasPrefixes.prefix_with().
Parameters:
• *prefixes – textual or ClauseElement construct which will be rendered following the INSERT, UPDATE, or DELETE keyword.
• dialect – optional string dialect name which will limit rendering of this prefix to only that dialect.
method sqlalchemy.orm.AppenderQuery.remove(item: _T) → None
inherited from the AppenderMixin.remove() method of AppenderMixin
Remove an item from this AppenderQuery.
The given item will be removed from the parent instance’s collection on the next flush.
method sqlalchemy.orm.AppenderQuery.suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
inherited from the HasSuffixes.suffix_with() method of HasSuffixes
Add one or more expressions following the statement as a whole.
This is used to support backend-specific suffix keywords on certain constructs.
E.g.:
stmt = (
    select(col1, col2)
    .cte()
    .suffix_with(
        "cycle empno set y_cycle to 1 default 0", dialect="oracle"
    )
)
Multiple suffixes can be specified by multiple calls to HasSuffixes.suffix_with().
Parameters:
• *suffixes – textual or ClauseElement construct which will be rendered following the target clause.
• dialect – Optional string dialect name which will limit rendering of this suffix to only that dialect.
method sqlalchemy.orm.AppenderQuery.with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') → Self
inherited from the HasHints.with_hint() method of HasHints
Add an indexing or other executional context hint for the given selectable to this Select or other selectable object.
Tip
The Select.with_hint() method adds hints that are specific to a single table to a statement, in a location that is dialect-specific. To add generic optimizer hints to the beginning of a statement ahead of the SELECT keyword such as for MySQL or Oracle Database, use the Select.prefix_with() method. To add optimizer hints to the end of a statement such as for PostgreSQL, use the Select.with_statement_hint() method.
The text of the hint is rendered in the appropriate location for the database backend in use, relative to the given Table or Alias passed as the selectable argument. The dialect implementation typically uses Python string substitution syntax with the token %(name)s to render the name of the table or alias. E.g. when using Oracle Database, the following:
select(mytable).with_hint(mytable, "index(%(name)s ix_mytable)")
Would render SQL as:
select /*+ index(mytable ix_mytable) */ from mytable
The dialect_name option will limit the rendering of a particular hint to a particular backend. Such as, to add hints for both Oracle Database and MSSql simultaneously:
select(mytable).with_hint(
    mytable, "index(%(name)s ix_mytable)", "oracle"
).with_hint(mytable, "WITH INDEX ix_mytable", "mssql")
See also
Select.with_statement_hint()
Select.prefix_with() - generic SELECT prefixing which also can suit some database-specific HINT syntaxes such as MySQL or Oracle Database optimizer hints
method sqlalchemy.orm.AppenderQuery.with_statement_hint(text: str, dialect_name: str = '*') → Self
inherited from the HasHints.with_statement_hint() method of HasHints
Add a statement hint to this Select or other selectable object.
Tip
Select.with_statement_hint() generally adds hints at the trailing end of a SELECT statement. To place dialect-specific hints such as optimizer hints at the front of the SELECT statement after the SELECT keyword, use the Select.prefix_with() method for an open-ended space, or for table-specific hints the Select.with_hint() may be used, which places hints in a dialect-specific location.
This method is similar to Select.with_hint() except that it does not require an individual table, and instead applies to the statement as a whole.
Hints here are specific to the backend database and may include directives such as isolation levels, file directives, fetch directives, etc.
See also
Select.with_hint()
Select.prefix_with() - generic SELECT prefixing which also can suit some database-specific HINT syntaxes such as MySQL or Oracle Database optimizer hints
class sqlalchemy.orm.DynamicMapped
Represent the ORM mapped attribute type for a “dynamic” relationship.
The DynamicMapped type annotation may be used in an Annotated Declarative Table mapping to indicate that the lazy="dynamic" loader strategy should be used for a particular relationship().
Legacy Feature
The “dynamic” lazy loader strategy is the legacy form of what is now the “write_only” strategy described in the section Write Only Relationships.
E.g.:
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    addresses: DynamicMapped[Address] = relationship(
        cascade="all,delete-orphan"
    )
See the section Dynamic Relationship Loaders for background.
Added in version 2.0.
See also
Dynamic Relationship Loaders - complete background
WriteOnlyMapped - fully 2.0 style version
Class signature
class sqlalchemy.orm.DynamicMapped (sqlalchemy.orm.base._MappedAnnotationBase)
Setting RaiseLoad
A “raise”-loaded relationship will raise an InvalidRequestError where the attribute would normally emit a lazy load:
class MyClass(Base):
    __tablename__ = "some_table"

    # 

    children: Mapped[List[MyRelatedClass]] = relationship(lazy="raise")
Above, attribute access on the children collection will raise an exception if it was not previously populated. This includes read access but for collections will also affect write access, as collections can’t be mutated without first loading them. The rationale for this is to ensure that an application is not emitting any unexpected lazy loads within a certain context. Rather than having to read through SQL logs to determine that all necessary attributes were eager loaded, the “raise” strategy will cause unloaded attributes to raise immediately if accessed. The raise strategy is also available on a query option basis using the raiseload() loader option.
See also
Preventing unwanted lazy loads using raiseload
Using Passive Deletes
An important aspect of collection management in SQLAlchemy is that when an object that refers to a collection is deleted, SQLAlchemy needs to consider the objects that are inside this collection. Those objects will need to be de-associated from the parent, which for a one-to-many collection would mean that foreign key columns are set to NULL, or based on cascade settings, may instead want to emit a DELETE for these rows.
The unit of work process only considers objects on a row-by-row basis, meaning a DELETE operation implies that all rows within a collection must be fully loaded into memory inside the flush process. This is not feasible for large collections, so we instead seek to rely upon the database’s own capability to update or delete the rows automatically using foreign key ON DELETE rules, instructing the unit of work to forego actually needing to load these rows in order to handle them. The unit of work can be instructed to work in this manner by configuring relationship.passive_deletes on the relationship() construct; the foreign key constraints in use must also be correctly configured.
For further detail on a complete “passive delete” configuration, see the section Using foreign key ON DELETE cascade with ORM relationships.


Collection Customization and API Details
The relationship() function defines a linkage between two classes. When the linkage defines a one-to-many or many-to-many relationship, it’s represented as a Python collection when objects are loaded and manipulated. This section presents additional information about collection configuration and techniques.
Customizing Collection Access
Mapping a one-to-many or many-to-many relationship results in a collection of values accessible through an attribute on the parent instance. The two common collection types for these are list and set, which in Declarative mappings that use Mapped is established by using the collection type within the Mapped container, as demonstrated in the Parent.children collection below where list is used:
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a list
    children: Mapped[List["Child"]] = relationship()


class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
Or for a set, illustrated in the same Parent.children collection:
from typing import Set
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a set
    children: Mapped[Set["Child"]] = relationship()


class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
Note
If using Python 3.7 or 3.8, annotations for collections need to use typing.List or typing.Set, e.g. Mapped[List["Child"]] or Mapped[Set["Child"]]; the list and set Python built-ins don’t yet support generic annotation in these Python versions, such as:
from typing import List


class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a List, Python 3.8 and earlier
    children: Mapped[List["Child"]] = relationship()
When using mappings without the Mapped annotation, such as when using imperative mappings or untyped Python code, as well as in a few special cases, the collection class for a relationship() can always be specified directly using the relationship.collection_class parameter:
# non-annotated mapping


class Parent(Base):
    __tablename__ = "parent"

    parent_id = mapped_column(Integer, primary_key=True)

    children = relationship("Child", collection_class=set)


class Child(Base):
    __tablename__ = "child"

    child_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent.id"))
In the absence of relationship.collection_class or Mapped, the default collection type is list.
Beyond list and set builtins, there is also support for two varieties of dictionary, described below at Dictionary Collections. There is also support for any arbitrary mutable sequence type can be set up as the target collection, with some additional configuration steps; this is described in the section Custom Collection Implementations.
Dictionary Collections
A little extra detail is needed when using a dictionary as a collection. This because objects are always loaded from the database as lists, and a key-generation strategy must be available to populate the dictionary correctly. The attribute_keyed_dict() function is by far the most common way to achieve a simple dictionary collection. It produces a dictionary class that will apply a particular attribute of the mapped class as a key. Below we map an Item class containing a dictionary of Note items keyed to the Note.keyword attribute. When using attribute_keyed_dict(), the Mapped annotation may be typed using the KeyFuncDict or just plain dict as illustrated in the following example. However, the relationship.collection_class parameter is required in this case so that the attribute_keyed_dict() may be appropriately parametrized:
from typing import Dict
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import attribute_keyed_dict
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("keyword"),
        cascade="all, delete-orphan",
    )


class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[Optional[str]]

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
Item.notes is then a dictionary:
 item = Item()
 item.notes["a"] = Note("a", "atext")
 item.notes.items()
{'a': <__main__.Note object at 0x2eaaf0>}
attribute_keyed_dict() will ensure that the .keyword attribute of each Note complies with the key in the dictionary. Such as, when assigning to Item.notes, the dictionary key we supply must match that of the actual Note object:
item = Item()
item.notes = {
    "a": Note("a", "atext"),
    "b": Note("b", "btext"),
}
The attribute which attribute_keyed_dict() uses as a key does not need to be mapped at all! Using a regular Python @property allows virtually any detail or combination of details about the object to be used as the key, as below when we establish it as a tuple of Note.keyword and the first ten letters of the Note.text field:
class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("note_key"),
        back_populates="item",
        cascade="all, delete-orphan",
    )


class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[str]

    item: Mapped["Item"] = relationship()

    @property
    def note_key(self):
        return (self.keyword, self.text[0:10])

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
Above we added a Note.item relationship, with a bi-directional relationship.back_populates configuration. Assigning to this reverse relationship, the Note is added to the Item.notes dictionary and the key is generated for us automatically:
 item = Item()
 n1 = Note("a", "atext")
 n1.item = item
 item.notes
{('a', 'atext'): <__main__.Note object at 0x2eaaf0>}
Other built-in dictionary types include column_keyed_dict(), which is almost like attribute_keyed_dict() except given the Column object directly:
from sqlalchemy.orm import column_keyed_dict


class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=column_keyed_dict(Note.__table__.c.keyword),
        cascade="all, delete-orphan",
    )
as well as mapped_collection() which is passed any callable function. Note that it’s usually easier to use attribute_keyed_dict() along with a @property as mentioned earlier:
from sqlalchemy.orm import mapped_collection


class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=mapped_collection(lambda note: note.text[0:10]),
        cascade="all, delete-orphan",
    )
Dictionary mappings are often combined with the “Association Proxy” extension to produce streamlined dictionary views. See Proxying to Dictionary Based Collections and Composite Association Proxies for examples.
Dealing with Key Mutations and back-populating for Dictionary collections
When using attribute_keyed_dict(), the “key” for the dictionary is taken from an attribute on the target object. Changes to this key are not tracked. This means that the key must be assigned towards when it is first used, and if the key changes, the collection will not be mutated. A typical example where this might be an issue is when relying upon backrefs to populate an attribute mapped collection. Given the following:
class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)

    bs: Mapped[Dict[str, "B"]] = relationship(
        collection_class=attribute_keyed_dict("data"),
        back_populates="a",
    )


class B(Base):
    __tablename__ = "b"

    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]

    a: Mapped["A"] = relationship(back_populates="bs")
Above, if we create a B() that refers to a specific A(), the back populates will then add the B() to the A.bs collection, however if the value of B.data is not set yet, the key will be None:
 a1 = A()
 b1 = B(a=a1)
 a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
Setting b1.data after the fact does not update the collection:
 b1.data = "the key"
 a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
This can also be seen if one attempts to set up B() in the constructor. The order of arguments changes the result:
 B(a=a1, data="the key")
<test3.B object at 0x7f7b10114280>
 a1.bs
{None: <test3.B object at 0x7f7b10114280>}
vs:
 B(data="the key", a=a1)
<test3.B object at 0x7f7b10114340>
 a1.bs
{'the key': <test3.B object at 0x7f7b10114340>}
If backrefs are being used in this way, ensure that attributes are populated in the correct order using an __init__ method.
An event handler such as the following may also be used to track changes in the collection as well:
from sqlalchemy import event
from sqlalchemy.orm import attributes


@event.listens_for(B.data, "set")
def set_item(obj, value, previous, initiator):
    if obj.a is not None:
        previous = None if previous == attributes.NO_VALUE else previous
        obj.a.bs[value] = obj
        obj.a.bs.pop(previous)
Custom Collection Implementations
You can use your own types for collections as well. In simple cases, inheriting from list or set, adding custom behavior, is all that’s needed. In other cases, special decorators are needed to tell SQLAlchemy more detail about how the collection operates.
Do I need a custom collection implementation?
In most cases not at all! The most common use cases for a “custom” collection is one that validates or marshals incoming values into a new form, such as a string that becomes a class instance, or one which goes a step beyond and represents the data internally in some fashion, presenting a “view” of that data on the outside of a different form.
For the first use case, the validates() decorator is by far the simplest way to intercept incoming values in all cases for the purposes of validation and simple marshaling. See Simple Validators for an example of this.
For the second use case, the Association Proxy extension is a well-tested, widely used system that provides a read/write “view” of a collection in terms of some attribute present on the target object. As the target attribute can be a @property that returns virtually anything, a wide array of “alternative” views of a collection can be constructed with just a few functions. This approach leaves the underlying mapped collection unaffected and avoids the need to carefully tailor collection behavior on a method-by-method basis.
Customized collections are useful when the collection needs to have special behaviors upon access or mutation operations that can’t otherwise be modeled externally to the collection. They can of course be combined with the above two approaches.
Collections in SQLAlchemy are transparently instrumented. Instrumentation means that normal operations on the collection are tracked and result in changes being written to the database at flush time. Additionally, collection operations can fire events which indicate some secondary operation must take place. Examples of a secondary operation include saving the child item in the parent’s Session (i.e. the save-update cascade), as well as synchronizing the state of a bi-directional relationship (i.e. a backref()).
The collections package understands the basic interface of lists, sets and dicts and will automatically apply instrumentation to those built-in types and their subclasses. Object-derived types that implement a basic collection interface are detected and instrumented via duck-typing:
class ListLike:
    def __init__(self):
        self.data = []

    def append(self, item):
        self.data.append(item)

    def remove(self, item):
        self.data.remove(item)

    def extend(self, items):
        self.data.extend(items)

    def __iter__(self):
        return iter(self.data)

    def foo(self):
        return "foo"
append, remove, and extend are known members of list, and will be instrumented automatically. __iter__ is not a mutator method and won’t be instrumented, and foo won’t be either.
Duck-typing (i.e. guesswork) isn’t rock-solid, of course, so you can be explicit about the interface you are implementing by providing an __emulates__ class attribute:
class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
This class looks similar to a Python list (i.e. “list-like”) as it has an append method, but the __emulates__ attribute forces it to be treated as a set. remove is known to be part of the set interface and will be instrumented.
But this class won’t work quite yet: a little glue is needed to adapt it for use by SQLAlchemy. The ORM needs to know which methods to use to append, remove and iterate over members of the collection. When using a type like list or set, the appropriate methods are well-known and used automatically when present. However the class above, which only roughly resembles a set, does not provide the expected add method, so we must indicate to the ORM the method that will instead take the place of the add method, in this case using a decorator @collection.appender; this is illustrated in the next section.
Annotating Custom Collections via Decorators
Decorators can be used to tag the individual methods the ORM needs to manage collections. Use them when your class doesn’t quite meet the regular interface for its container type, or when you otherwise would like to use a different method to get the job done.
from sqlalchemy.orm.collections import collection


class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    @collection.appender
    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
And that’s all that’s needed to complete the example. SQLAlchemy will add instances via the append method. remove and __iter__ are the default methods for sets and will be used for removing and iteration. Default methods can be changed as well:
from sqlalchemy.orm.collections import collection


class MyList(list):
    @collection.remover
    def zark(self, item):
        # do something special
        

    @collection.iterator
    def hey_use_this_instead_for_iteration(self): 
There is no requirement to be “list-like” or “set-like” at all. Collection classes can be any shape, so long as they have the append, remove and iterate interface marked for SQLAlchemy’s use. Append and remove methods will be called with a mapped entity as the single argument, and iterator methods are called with no arguments and must return an iterator.
Custom Dictionary-Based Collections
The KeyFuncDict class can be used as a base class for your custom types or as a mix-in to quickly add dict collection support to other classes. It uses a keying function to delegate to __setitem__ and __delitem__:
from sqlalchemy.orm.collections import KeyFuncDict


class MyNodeMap(KeyFuncDict):
    """Holds 'Node' objects, keyed by the 'name' attribute."""

    def __init__(self, *args, **kw):
        super().__init__(keyfunc=lambda node: node.name)
        dict.__init__(self, *args, **kw)
When subclassing KeyFuncDict, user-defined versions of __setitem__() or __delitem__() should be decorated with collection.internally_instrumented(), if they call down to those same methods on KeyFuncDict. This because the methods on KeyFuncDict are already instrumented - calling them from within an already instrumented call can cause events to be fired off repeatedly, or inappropriately, leading to internal state corruption in rare cases:
from sqlalchemy.orm.collections import KeyFuncDict, collection


class MyKeyFuncDict(KeyFuncDict):
    """Use @internally_instrumented when your methods
    call down to already-instrumented methods.

    """

    @collection.internally_instrumented
    def __setitem__(self, key, value, _sa_initiator=None):
        # do something with key, value
        super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

    @collection.internally_instrumented
    def __delitem__(self, key, _sa_initiator=None):
        # do something with key
        super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)
The ORM understands the dict interface just like lists and sets, and will automatically instrument all “dict-like” methods if you choose to subclass dict or provide dict-like collection behavior in a duck-typed class. You must decorate appender and remover methods, however- there are no compatible methods in the basic dictionary interface for SQLAlchemy to use by default. Iteration will go through values() unless otherwise decorated.
Instrumentation and Custom Types
Many custom types and existing library classes can be used as a entity collection type as-is without further ado. However, it is important to note that the instrumentation process will modify the type, adding decorators around methods automatically.
The decorations are lightweight and no-op outside of relationships, but they do add unneeded overhead when triggered elsewhere. When using a library class as a collection, it can be good practice to use the “trivial subclass” trick to restrict the decorations to just your usage in relationships. For example:
class MyAwesomeList(some.great.library.AwesomeList):
    pass


# relationship(, collection_class=MyAwesomeList)
The ORM uses this approach for built-ins, quietly substituting a trivial subclass when a list, set or dict is used directly.
Collection API
Object NameDescriptionattribute_keyed_dict(attr_name, *, [ignore_unpopulated_attribute])A dictionary-based collection type with attribute-based keying.attribute_mapped_collectionA dictionary-based collection type with attribute-based keying.column_keyed_dict(mapping_spec, *, [ignore_unpopulated_attribute])A dictionary-based collection type with column-based keying.column_mapped_collectionA dictionary-based collection type with column-based keying.keyfunc_mapping(keyfunc, *, [ignore_unpopulated_attribute])A dictionary-based collection type with arbitrary keying.KeyFuncDictBase for ORM mapped dictionary classes.mapped_collectionA dictionary-based collection type with arbitrary keying.MappedCollectionBase for ORM mapped dictionary classes.function sqlalchemy.orm.attribute_keyed_dict(attr_name: str, *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[Any, Any]]
A dictionary-based collection type with attribute-based keying.
Changed in version 2.0: Renamed attribute_mapped_collection to attribute_keyed_dict().
Returns a KeyFuncDict factory which will produce new dictionary keys based on the value of a particular named attribute on ORM mapped instances to be added to the dictionary.
Note
the value of the target attribute must be assigned with its value at the time that the object is being added to the dictionary collection. Additionally, changes to the key attribute are not tracked, which means the key in the dictionary is not automatically synchronized with the key value on the target object itself. See Dealing with Key Mutations and back-populating for Dictionary collections for further details.
See also
Dictionary Collections - background on use
Parameters:
• attr_name – string name of an ORM-mapped attribute on the mapped class, the value of which on a particular instance is to be used as the key for a new dictionary entry for that instance.
• ignore_unpopulated_attribute – 
if True, and the target attribute on an object is not populated at all, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the attribute being used for the dictionary key is determined that it was never populated with any value. The attribute_keyed_dict.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
function sqlalchemy.orm.column_keyed_dict(mapping_spec: Type[_KT] | Callable[[_KT], _VT], *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, _KT]]
A dictionary-based collection type with column-based keying.
Changed in version 2.0: Renamed column_mapped_collection to column_keyed_dict.
Returns a KeyFuncDict factory which will produce new dictionary keys based on the value of a particular Column-mapped attribute on ORM mapped instances to be added to the dictionary.
Note
the value of the target attribute must be assigned with its value at the time that the object is being added to the dictionary collection. Additionally, changes to the key attribute are not tracked, which means the key in the dictionary is not automatically synchronized with the key value on the target object itself. See Dealing with Key Mutations and back-populating for Dictionary collections for further details.
See also
Dictionary Collections - background on use
Parameters:
• mapping_spec – a Column object that is expected to be mapped by the target mapper to a particular attribute on the mapped class, the value of which on a particular instance is to be used as the key for a new dictionary entry for that instance.
• ignore_unpopulated_attribute – 
if True, and the mapped attribute indicated by the given Column target attribute on an object is not populated at all, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the attribute being used for the dictionary key is determined that it was never populated with any value. The column_keyed_dict.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
function sqlalchemy.orm.keyfunc_mapping(keyfunc: Callable[[Any], Any], *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, Any]]
A dictionary-based collection type with arbitrary keying.
Changed in version 2.0: Renamed mapped_collection to keyfunc_mapping().
Returns a KeyFuncDict factory with a keying function generated from keyfunc, a callable that takes an entity and returns a key value.
Note
the given keyfunc is called only once at the time that the target object is being added to the collection. Changes to the effective value returned by the function are not tracked.
See also
Dictionary Collections - background on use
Parameters:
• keyfunc – a callable that will be passed the ORM-mapped instance which should then generate a new key to use in the dictionary. If the value returned is LoaderCallableStatus.NO_VALUE, an error is raised.
• ignore_unpopulated_attribute – 
if True, and the callable returns LoaderCallableStatus.NO_VALUE for a particular instance, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the callable being used for the dictionary key returns LoaderCallableStatus.NO_VALUE, which in an ORM attribute context indicates an attribute that was never populated with any value. The mapped_collection.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
sqlalchemy.orm.attribute_mapped_collection = <function attribute_keyed_dict>
A dictionary-based collection type with attribute-based keying.
Changed in version 2.0: Renamed attribute_mapped_collection to attribute_keyed_dict().
Returns a KeyFuncDict factory which will produce new dictionary keys based on the value of a particular named attribute on ORM mapped instances to be added to the dictionary.
Note
the value of the target attribute must be assigned with its value at the time that the object is being added to the dictionary collection. Additionally, changes to the key attribute are not tracked, which means the key in the dictionary is not automatically synchronized with the key value on the target object itself. See Dealing with Key Mutations and back-populating for Dictionary collections for further details.
See also
Dictionary Collections - background on use
Parameters:
• attr_name – string name of an ORM-mapped attribute on the mapped class, the value of which on a particular instance is to be used as the key for a new dictionary entry for that instance.
• ignore_unpopulated_attribute – 
if True, and the target attribute on an object is not populated at all, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the attribute being used for the dictionary key is determined that it was never populated with any value. The attribute_keyed_dict.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
sqlalchemy.orm.column_mapped_collection = <function column_keyed_dict>
A dictionary-based collection type with column-based keying.
Changed in version 2.0: Renamed column_mapped_collection to column_keyed_dict.
Returns a KeyFuncDict factory which will produce new dictionary keys based on the value of a particular Column-mapped attribute on ORM mapped instances to be added to the dictionary.
Note
the value of the target attribute must be assigned with its value at the time that the object is being added to the dictionary collection. Additionally, changes to the key attribute are not tracked, which means the key in the dictionary is not automatically synchronized with the key value on the target object itself. See Dealing with Key Mutations and back-populating for Dictionary collections for further details.
See also
Dictionary Collections - background on use
Parameters:
• mapping_spec – a Column object that is expected to be mapped by the target mapper to a particular attribute on the mapped class, the value of which on a particular instance is to be used as the key for a new dictionary entry for that instance.
• ignore_unpopulated_attribute – 
if True, and the mapped attribute indicated by the given Column target attribute on an object is not populated at all, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the attribute being used for the dictionary key is determined that it was never populated with any value. The column_keyed_dict.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
sqlalchemy.orm.mapped_collection = <function keyfunc_mapping>
A dictionary-based collection type with arbitrary keying.
Changed in version 2.0: Renamed mapped_collection to keyfunc_mapping().
Returns a KeyFuncDict factory with a keying function generated from keyfunc, a callable that takes an entity and returns a key value.
Note
the given keyfunc is called only once at the time that the target object is being added to the collection. Changes to the effective value returned by the function are not tracked.
See also
Dictionary Collections - background on use
Parameters:
• keyfunc – a callable that will be passed the ORM-mapped instance which should then generate a new key to use in the dictionary. If the value returned is LoaderCallableStatus.NO_VALUE, an error is raised.
• ignore_unpopulated_attribute – 
if True, and the callable returns LoaderCallableStatus.NO_VALUE for a particular instance, the operation will be silently skipped. By default, an error is raised.
Added in version 2.0: an error is raised by default if the callable being used for the dictionary key returns LoaderCallableStatus.NO_VALUE, which in an ORM attribute context indicates an attribute that was never populated with any value. The mapped_collection.ignore_unpopulated_attribute parameter may be set which will instead indicate that this condition should be ignored, and the append operation silently skipped. This is in contrast to the behavior of the 1.x series which would erroneously populate the value in the dictionary with an arbitrary key value of None.
class sqlalchemy.orm.KeyFuncDict
Base for ORM mapped dictionary classes.
Extends the dict type with additional methods needed by SQLAlchemy ORM collection classes. Use of KeyFuncDict is most directly by using the attribute_keyed_dict() or column_keyed_dict() class factories. KeyFuncDict may also serve as the base for user-defined custom dictionary classes.
Changed in version 2.0: Renamed MappedCollection to KeyFuncDict.
See also
attribute_keyed_dict()
column_keyed_dict()
Dictionary Collections
Custom Collection Implementations
Members
__init__(), clear(), pop(), popitem(), remove(), set(), setdefault(), update()
Class signature
class sqlalchemy.orm.KeyFuncDict (builtins.dict, typing.Generic)
method sqlalchemy.orm.KeyFuncDict.__init__(keyfunc: Callable[[Any], Any], *dict_args: Any, ignore_unpopulated_attribute: bool = False) → None
Create a new collection with keying provided by keyfunc.
keyfunc may be any callable that takes an object and returns an object for use as a dictionary key.
The keyfunc will be called every time the ORM needs to add a member by value-only (such as when loading instances from the database) or remove a member. The usual cautions about dictionary keying apply- keyfunc(object) should return the same output for the life of the collection. Keying based on mutable properties can result in unreachable instances “lost” in the collection.
method sqlalchemy.orm.KeyFuncDict.clear()
Remove all items from the dict.
method sqlalchemy.orm.KeyFuncDict.pop(k[, d]) → v, remove specified key and return the corresponding value.
If the key is not found, return the default if given; otherwise, raise a KeyError.
method sqlalchemy.orm.KeyFuncDict.popitem()
Remove and return a (key, value) pair as a 2-tuple.
Pairs are returned in LIFO (last-in, first-out) order. Raises KeyError if the dict is empty.
method sqlalchemy.orm.KeyFuncDict.remove(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
Remove an item by value, consulting the keyfunc for the key.
method sqlalchemy.orm.KeyFuncDict.set(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
Add an item by value, consulting the keyfunc for the key.
method sqlalchemy.orm.KeyFuncDict.setdefault(key, default=None)
Insert key with a value of default if key is not in the dictionary.
Return the value for key if key is in the dictionary, else default.
method sqlalchemy.orm.KeyFuncDict.update([E, ]**F) → None.  Update D from mapping/iterable E and F.
If E is present and has a .keys() method, then does: for k in E.keys(): D[k] = E[k] If E is present and lacks a .keys() method, then does: for k, v in E: D[k] = v In either case, this is followed by: for k in F: D[k] = F[k]
sqlalchemy.orm.MappedCollection = <class 'sqlalchemy.orm.mapped_collection.KeyFuncDict'>
Base for ORM mapped dictionary classes.
Extends the dict type with additional methods needed by SQLAlchemy ORM collection classes. Use of KeyFuncDict is most directly by using the attribute_keyed_dict() or column_keyed_dict() class factories. KeyFuncDict may also serve as the base for user-defined custom dictionary classes.
Changed in version 2.0: Renamed MappedCollection to KeyFuncDict.
See also
attribute_keyed_dict()
column_keyed_dict()
Dictionary Collections
Custom Collection Implementations
Collection Internals
Object NameDescriptionbulk_replace(values, existing_adapter, new_adapter[, initiator])Load a new collection, firing events based on prior like membership.collectionDecorators for entity collection classes.collection_adapterReturn a callable object that fetches the given attribute(s) from its operand. After f = attrgetter(‘name’), the call f(r) returns r.name. After g = attrgetter(‘name’, ‘date’), the call g(r) returns (r.name, r.date). After h = attrgetter(‘name.first’, ‘name.last’), the call h(r) returns (r.name.first, r.name.last).CollectionAdapterBridges between the ORM and arbitrary Python collections.InstrumentedDictAn instrumented version of the built-in dict.InstrumentedListAn instrumented version of the built-in list.InstrumentedSetAn instrumented version of the built-in set.prepare_instrumentation(factory)Prepare a callable for future use as a collection class factory.function sqlalchemy.orm.collections.bulk_replace(values, existing_adapter, new_adapter, initiator=None)
Load a new collection, firing events based on prior like membership.
Appends instances in values onto the new_adapter. Events will be fired for any instance not present in the existing_adapter. Any instances in existing_adapter not present in values will have remove events fired upon them.
Parameters:
• values – An iterable of collection member instances
• existing_adapter – A CollectionAdapter of instances to be replaced
• new_adapter – An empty CollectionAdapter to load with values
class sqlalchemy.orm.collections.collection
Decorators for entity collection classes.
The decorators fall into two groups: annotations and interception recipes.
The annotating decorators (appender, remover, iterator, converter, internally_instrumented) indicate the method’s purpose and take no arguments. They are not written with parens:
@collection.appender
def append(self, append): 
The recipe decorators all require parens, even those that take no arguments:
Members
adds(), appender(), converter(), internally_instrumented(), iterator(), remover(), removes(), removes_return(), replaces()
@collection.adds("entity")
def insert(self, position, entity): 


@collection.removes_return()
def popitem(self): 
method sqlalchemy.orm.collections.collection.static adds(arg)
Mark the method as adding an entity to the collection.
Adds “add to collection” handling to the method. The decorator argument indicates which method argument holds the SQLAlchemy-relevant value. Arguments can be specified positionally (i.e. integer) or by name:
@collection.adds(1)
def push(self, item): 


@collection.adds("entity")
def do_stuff(self, thing, entity=None): 
method sqlalchemy.orm.collections.collection.static appender(fn)
Tag the method as the collection appender.
The appender method is called with one positional argument: the value to append. The method will be automatically decorated with ‘adds(1)’ if not already decorated:
@collection.appender
def add(self, append): 


# or, equivalently
@collection.appender
@collection.adds(1)
def add(self, append): 


# for mapping type, an 'append' may kick out a previous value
# that occupies that slot.  consider d['a'] = 'foo'- any previous
# value in d['a'] is discarded.
@collection.appender
@collection.replaces(1)
def add(self, entity):
    key = some_key_func(entity)
    previous = None
    if key in self:
        previous = self[key]
    self[key] = entity
    return previous
If the value to append is not allowed in the collection, you may raise an exception. Something to remember is that the appender will be called for each object mapped by a database query. If the database contains rows that violate your collection semantics, you will need to get creative to fix the problem, as access via the collection will not work.
If the appender method is internally instrumented, you must also receive the keyword argument ‘_sa_initiator’ and ensure its promulgation to collection events.
method sqlalchemy.orm.collections.collection.static converter(fn)
Tag the method as the collection converter.
Deprecated since version 1.3: The collection.converter() handler is deprecated and will be removed in a future release. Please refer to the bulk_replace listener interface in conjunction with the listen() function.
This optional method will be called when a collection is being replaced entirely, as in:
myobj.acollection = [newvalue1, newvalue2]
The converter method will receive the object being assigned and should return an iterable of values suitable for use by the appender method. A converter must not assign values or mutate the collection, its sole job is to adapt the value the user provides into an iterable of values for the ORM’s use.
The default converter implementation will use duck-typing to do the conversion. A dict-like collection will be convert into an iterable of dictionary values, and other types will simply be iterated:
@collection.converter
def convert(self, other): 
If the duck-typing of the object does not match the type of this collection, a TypeError is raised.
Supply an implementation of this method if you want to expand the range of possible types that can be assigned in bulk or perform validation on the values about to be assigned.
method sqlalchemy.orm.collections.collection.static internally_instrumented(fn)
Tag the method as instrumented.
This tag will prevent any decoration from being applied to the method. Use this if you are orchestrating your own calls to collection_adapter() in one of the basic SQLAlchemy interface methods, or to prevent an automatic ABC method decoration from wrapping your implementation:
# normally an 'extend' method on a list-like class would be
# automatically intercepted and re-implemented in terms of
# SQLAlchemy events and append().  your implementation will
# never be called, unless:
@collection.internally_instrumented
def extend(self, items): 
method sqlalchemy.orm.collections.collection.static iterator(fn)
Tag the method as the collection remover.
The iterator method is called with no arguments. It is expected to return an iterator over all collection members:
@collection.iterator
def __iter__(self): 
method sqlalchemy.orm.collections.collection.static remover(fn)
Tag the method as the collection remover.
The remover method is called with one positional argument: the value to remove. The method will be automatically decorated with removes_return() if not already decorated:
@collection.remover
def zap(self, entity): 


# or, equivalently
@collection.remover
@collection.removes_return()
def zap(self): 
If the value to remove is not present in the collection, you may raise an exception or return None to ignore the error.
If the remove method is internally instrumented, you must also receive the keyword argument ‘_sa_initiator’ and ensure its promulgation to collection events.
method sqlalchemy.orm.collections.collection.static removes(arg)
Mark the method as removing an entity in the collection.
Adds “remove from collection” handling to the method. The decorator argument indicates which method argument holds the SQLAlchemy-relevant value to be removed. Arguments can be specified positionally (i.e. integer) or by name:
@collection.removes(1)
def zap(self, item): 
For methods where the value to remove is not known at call-time, use collection.removes_return.
method sqlalchemy.orm.collections.collection.static removes_return()
Mark the method as removing an entity in the collection.
Adds “remove from collection” handling to the method. The return value of the method, if any, is considered the value to remove. The method arguments are not inspected:
@collection.removes_return()
def pop(self): 
For methods where the value to remove is known at call-time, use collection.remove.
method sqlalchemy.orm.collections.collection.static replaces(arg)
Mark the method as replacing an entity in the collection.
Adds “add to collection” and “remove from collection” handling to the method. The decorator argument indicates which method argument holds the SQLAlchemy-relevant value to be added, and return value, if any will be considered the value to remove.
Arguments can be specified positionally (i.e. integer) or by name:
@collection.replaces(2)
def __setitem__(self, index, item): 
sqlalchemy.orm.collections.collection_adapter = operator.attrgetter('_sa_adapter')
Return a callable object that fetches the given attribute(s) from its operand. After f = attrgetter(‘name’), the call f(r) returns r.name. After g = attrgetter(‘name’, ‘date’), the call g(r) returns (r.name, r.date). After h = attrgetter(‘name.first’, ‘name.last’), the call h(r) returns (r.name.first, r.name.last).
class sqlalchemy.orm.collections.CollectionAdapter
Bridges between the ORM and arbitrary Python collections.
Proxies base-level collection operations (append, remove, iterate) to the underlying Python collection, and emits add/remove events for entities entering or leaving the collection.
The ORM uses CollectionAdapter exclusively for interaction with entity collections.
class sqlalchemy.orm.collections.InstrumentedDict
An instrumented version of the built-in dict.
Class signature
class sqlalchemy.orm.collections.InstrumentedDict (builtins.dict, typing.Generic)
class sqlalchemy.orm.collections.InstrumentedList
An instrumented version of the built-in list.
Class signature
class sqlalchemy.orm.collections.InstrumentedList (builtins.list, typing.Generic)
class sqlalchemy.orm.collections.InstrumentedSet
An instrumented version of the built-in set.
Class signature
class sqlalchemy.orm.collections.InstrumentedSet (builtins.set, typing.Generic)
function sqlalchemy.orm.collections.prepare_instrumentation(factory: Type[Collection[Any]] | Callable[[], _AdaptedCollectionProtocol]) → Callable[[], _AdaptedCollectionProtocol]
Prepare a callable for future use as a collection class factory.
Given a collection class factory (either a type or no-arg callable), return another factory that will produce compatible instances when called.
This function is responsible for converting collection_class=list into the run-time behavior of collection_class=InstrumentedList.


Special Relationship Persistence Patterns
Rows that point to themselves / Mutually Dependent Rows
This is a very specific case where relationship() must perform an INSERT and a second UPDATE in order to properly populate a row (and vice versa an UPDATE and DELETE in order to delete without violating foreign key constraints). The two use cases are:
• A table contains a foreign key to itself, and a single row will have a foreign key value pointing to its own primary key.
• Two tables each contain a foreign key referencing the other table, with a row in each table referencing the other.
For example:
          user
---------------------------------
user_id    name   related_user_id
   1       'ed'          1
Or:
             widget                                                  entry
-------------------------------------------             ---------------------------------
widget_id     name        favorite_entry_id             entry_id      name      widget_id
   1       'somewidget'          5                         5       'someentry'     1
In the first case, a row points to itself. Technically, a database that uses sequences such as PostgreSQL or Oracle Database can INSERT the row at once using a previously generated value, but databases which rely upon autoincrement-style primary key identifiers cannot. The relationship() always assumes a “parent/child” model of row population during flush, so unless you are populating the primary key/foreign key columns directly, relationship() needs to use two statements.
In the second case, the “widget” row must be inserted before any referring “entry” rows, but then the “favorite_entry_id” column of that “widget” row cannot be set until the “entry” rows have been generated. In this case, it’s typically impossible to insert the “widget” and “entry” rows using just two INSERT statements; an UPDATE must be performed in order to keep foreign key constraints fulfilled. The exception is if the foreign keys are configured as “deferred until commit” (a feature some databases support) and if the identifiers were populated manually (again essentially bypassing relationship()).
To enable the usage of a supplementary UPDATE statement, we use the relationship.post_update option of relationship(). This specifies that the linkage between the two rows should be created using an UPDATE statement after both rows have been INSERTED; it also causes the rows to be de-associated with each other via UPDATE before a DELETE is emitted. The flag should be placed on just one of the relationships, preferably the many-to-one side. Below we illustrate a complete example, including two ForeignKey constructs:
from sqlalchemy import Integer, ForeignKey
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))


class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, primary_key=True)
    favorite_entry_id = mapped_column(
        Integer, ForeignKey("entry.entry_id", name="fk_favorite_entry")
    )
    name = mapped_column(String(50))

    entries = relationship(Entry, primaryjoin=widget_id == Entry.widget_id)
    favorite_entry = relationship(
        Entry, primaryjoin=favorite_entry_id == Entry.entry_id, post_update=True
    )
When a structure against the above configuration is flushed, the “widget” row will be INSERTed minus the “favorite_entry_id” value, then all the “entry” rows will be INSERTed referencing the parent “widget” row, and then an UPDATE statement will populate the “favorite_entry_id” column of the “widget” table (it’s one row at a time for the time being):
 w1 = Widget(name="somewidget")
 e1 = Entry(name="someentry")
 w1.favorite_entry = e1
 w1.entries = [e1]
 session.add_all([w1, e1])
 session.commit()
BEGIN (implicit)
INSERT INTO widget (favorite_entry_id, name) VALUES (?, ?)
(None, 'somewidget')
INSERT INTO entry (widget_id, name) VALUES (?, ?)
(1, 'someentry')
UPDATE widget SET favorite_entry_id=? WHERE widget.widget_id = ?
(1, 1)
COMMIT
An additional configuration we can specify is to supply a more comprehensive foreign key constraint on Widget, such that it’s guaranteed that favorite_entry_id refers to an Entry that also refers to this Widget. We can use a composite foreign key, as illustrated below:
from sqlalchemy import (
    Integer,
    ForeignKey,
    String,
    UniqueConstraint,
    ForeignKeyConstraint,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))
    __table_args__ = (UniqueConstraint("entry_id", "widget_id"),)


class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, autoincrement="ignore_fk", primary_key=True)
    favorite_entry_id = mapped_column(Integer)

    name = mapped_column(String(50))

    __table_args__ = (
        ForeignKeyConstraint(
            ["widget_id", "favorite_entry_id"],
            ["entry.widget_id", "entry.entry_id"],
            name="fk_favorite_entry",
        ),
    )

    entries = relationship(
        Entry, primaryjoin=widget_id == Entry.widget_id, foreign_keys=Entry.widget_id
    )
    favorite_entry = relationship(
        Entry,
        primaryjoin=favorite_entry_id == Entry.entry_id,
        foreign_keys=favorite_entry_id,
        post_update=True,
    )
The above mapping features a composite ForeignKeyConstraint bridging the widget_id and favorite_entry_id columns. To ensure that Widget.widget_id remains an “autoincrementing” column we specify Column.autoincrement to the value "ignore_fk" on Column, and additionally on each relationship() we must limit those columns considered as part of the foreign key for the purposes of joining and cross-population.
Mutable Primary Keys / Update Cascades
When the primary key of an entity changes, related items which reference the primary key must also be updated as well. For databases which enforce referential integrity, the best strategy is to use the database’s ON UPDATE CASCADE functionality in order to propagate primary key changes to referenced foreign keys - the values cannot be out of sync for any moment unless the constraints are marked as “deferrable”, that is, not enforced until the transaction completes.
It is highly recommended that an application which seeks to employ natural primary keys with mutable values to use the ON UPDATE CASCADE capabilities of the database. An example mapping which illustrates this is:
class User(Base):
    __tablename__ = "user"
    __table_args__ = {"mysql_engine": "InnoDB"}

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    addresses = relationship("Address")


class Address(Base):
    __tablename__ = "address"
    __table_args__ = {"mysql_engine": "InnoDB"}

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(
        String(50), ForeignKey("user.username", onupdate="cascade")
    )
Above, we illustrate onupdate="cascade" on the ForeignKey object, and we also illustrate the mysql_engine='InnoDB' setting which, on a MySQL backend, ensures that the InnoDB engine supporting referential integrity is used. When using SQLite, referential integrity should be enabled, using the configuration described at Foreign Key Support.
See also
Using foreign key ON DELETE cascade with ORM relationships - supporting ON DELETE CASCADE with relationships
mapper.passive_updates - similar feature on Mapper
Simulating limited ON UPDATE CASCADE without foreign key support
In those cases when a database that does not support referential integrity is used, and natural primary keys with mutable values are in play, SQLAlchemy offers a feature in order to allow propagation of primary key values to already-referenced foreign keys to a limited extent, by emitting an UPDATE statement against foreign key columns that immediately reference a primary key column whose value has changed. The primary platforms without referential integrity features are MySQL when the MyISAM storage engine is used, and SQLite when the PRAGMA foreign_keys=ON pragma is not used. Oracle Database also has no support for ON UPDATE CASCADE, but because it still enforces referential integrity, needs constraints to be marked as deferrable so that SQLAlchemy can emit UPDATE statements.
The feature is enabled by setting the relationship.passive_updates flag to False, most preferably on a one-to-many or many-to-many relationship(). When “updates” are no longer “passive” this indicates that SQLAlchemy will issue UPDATE statements individually for objects referenced in the collection referred to by the parent object with a changing primary key value. This also implies that collections will be fully loaded into memory if not already locally present.
Our previous mapping using passive_updates=False looks like:
class User(Base):
    __tablename__ = "user"

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    # passive_updates=False *only* needed if the database
    # does not implement ON UPDATE CASCADE
    addresses = relationship("Address", passive_updates=False)


class Address(Base):
    __tablename__ = "address"

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(String(50), ForeignKey("user.username"))
Key limitations of passive_updates=False include:
• it performs much more poorly than direct database ON UPDATE CASCADE, because it needs to fully pre-load affected collections using SELECT and also must emit UPDATE statements against those values, which it will attempt to run in “batches” but still runs on a per-row basis at the DBAPI level.
• the feature cannot “cascade” more than one level. That is, if mapping X has a foreign key which refers to the primary key of mapping Y, but then mapping Y’s primary key is itself a foreign key to mapping Z, passive_updates=False cannot cascade a change in primary key value from Z to X.
• Configuring passive_updates=False only on the many-to-one side of a relationship will not have a full effect, as the unit of work searches only through the current identity map for objects that may be referencing the one with a mutating primary key, not throughout the database.
As virtually all databases other than Oracle Database now support ON UPDATE CASCADE, it is highly recommended that traditional ON UPDATE CASCADE support be used in the case that natural and mutable primary key values are in use.


Using the legacy ‘backref’ relationship parameter
Note
The relationship.backref keyword should be considered legacy, and use of relationship.back_populates with explicit relationship() constructs should be preferred. Using individual relationship() constructs provides advantages including that both ORM mapped classes will include their attributes up front as the class is constructed, rather than as a deferred step, and configuration is more straightforward as all arguments are explicit. New PEP 484 features in SQLAlchemy 2.0 also take advantage of attributes being explicitly present in source code rather than using dynamic attribute generation.
See also
For general information about bidirectional relationships, see the following sections:
Working with ORM Related Objects - in the SQLAlchemy Unified Tutorial, presents an overview of bi-directional relationship configuration and behaviors using relationship.back_populates
Behavior of save-update cascade with bi-directional relationships - notes on bi-directional relationship() behavior regarding Session cascade behaviors.
relationship.back_populates
The relationship.backref keyword argument on the relationship() construct allows the automatic generation of a new relationship() that will be automatically be added to the ORM mapping for the related class. It will then be placed into a relationship.back_populates configuration against the current relationship() being configured, with both relationship() constructs referring to each other.
Starting with the following example:
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship("Address", backref="user")


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))
The above configuration establishes a collection of Address objects on User called User.addresses. It also establishes a .user attribute on Address which will refer to the parent User object. Using relationship.back_populates it’s equivalent to the following:
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship("Address", back_populates="user")


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

    user = relationship("User", back_populates="addresses")
The behavior of the User.addresses and Address.user relationships is that they now behave in a bi-directional way, indicating that changes on one side of the relationship impact the other. An example and discussion of this behavior is in the SQLAlchemy Unified Tutorial at Working with ORM Related Objects.
Backref Default Arguments
Since relationship.backref generates a whole new relationship(), the generation process by default will attempt to include corresponding arguments in the new relationship() that correspond to the original arguments. As an example, below is a relationship() that includes a custom join condition which also includes the relationship.backref keyword:
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        primaryjoin=(
            "and_(User.id==Address.user_id, Address.email.startswith('tony'))"
        ),
        backref="user",
    )


class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))
When the “backref” is generated, the relationship.primaryjoin condition is copied to the new relationship() as well:
 print(User.addresses.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'

 print(Address.user.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'

Other arguments that are transferrable include the relationship.secondary parameter that refers to a many-to-many association table, as well as the “join” arguments relationship.primaryjoin and relationship.secondaryjoin; “backref” is smart enough to know that these two arguments should also be “reversed” when generating the opposite side.
Specifying Backref Arguments
Lots of other arguments for a “backref” are not implicit, and include arguments like relationship.lazy, relationship.remote_side, relationship.cascade and relationship.cascade_backrefs. For this case we use the backref() function in place of a string; this will store a specific set of arguments that will be transferred to the new relationship() when generated:
# <other imports>
from sqlalchemy.orm import backref


class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        backref=backref("user", lazy="joined"),
    )
Where above, we placed a lazy="joined" directive only on the Address.user side, indicating that when a query against Address is made, a join to the User entity should be made automatically which will populate the .user attribute of each returned Address. The backref() function formatted the arguments we gave it into a form that is interpreted by the receiving relationship() as additional arguments to be applied to the new relationship it creates.


Relationships API
Object NameDescriptionbackref(name, **kwargs)When using the relationship.backref parameter, provides specific parameters to be used when the new relationship() is generated.dynamic_loader([argument], **kw)Construct a dynamically-loading mapper property.foreign(expr)Annotate a portion of a primaryjoin expression with a ‘foreign’ annotation.relationship([argument, secondary], *, [uselist, collection_class, primaryjoin, secondaryjoin, back_populates, order_by, backref, overlaps, post_update, cascade, viewonly, init, repr, default, default_factory, compare, kw_only, hash, lazy, passive_deletes, passive_updates, active_history, enable_typechecks, foreign_keys, remote_side, join_depth, comparator_factory, single_parent, innerjoin, distinct_target_key, load_on_pending, query_class, info, omit_join, sync_backref], **kw)Provide a relationship between two mapped classes.remote(expr)Annotate a portion of a primaryjoin expression with a ‘remote’ annotation.function sqlalchemy.orm.relationship(argument: _RelationshipArgumentType[Any] | None = None, secondary: _RelationshipSecondaryArgument | None = None, *, uselist: bool | None = None, collection_class: Type[Collection[Any]] | Callable[[], Collection[Any]] | None = None, primaryjoin: _RelationshipJoinConditionArgument | None = None, secondaryjoin: _RelationshipJoinConditionArgument | None = None, back_populates: str | None = None, order_by: _ORMOrderByArgument = False, backref: ORMBackrefArgument | None = None, overlaps: str | None = None, post_update: bool = False, cascade: str = 'save-update, merge', viewonly: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: _NoArg | _T = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, lazy: _LazyLoadArgumentType = 'select', passive_deletes: Literal['all'] | bool = False, passive_updates: bool = True, active_history: bool = False, enable_typechecks: bool = True, foreign_keys: _ORMColCollectionArgument | None = None, remote_side: _ORMColCollectionArgument | None = None, join_depth: int | None = None, comparator_factory: Type[RelationshipProperty.Comparator[Any]] | None = None, single_parent: bool = False, innerjoin: bool = False, distinct_target_key: bool | None = None, load_on_pending: bool = False, query_class: Type[Query[Any]] | None = None, info: _InfoType | None = None, omit_join: Literal[None, False] = None, sync_backref: bool | None = None, **kw: Any) → _RelationshipDeclared[Any]
Provide a relationship between two mapped classes.
This corresponds to a parent-child or associative table relationship. The constructed class is an instance of Relationship.
See also
Working with ORM Related Objects - tutorial introduction to relationship() in the SQLAlchemy Unified Tutorial
Relationship Configuration - narrative documentation
Parameters:
• argument – 
This parameter refers to the class that is to be related. It accepts several forms, including a direct reference to the target class itself, the Mapper instance for the target class, a Python callable / lambda that will return a reference to the class or Mapper when called, and finally a string name for the class, which will be resolved from the registry in use in order to locate the class, e.g.:
class SomeClass(Base):
    # 

    related = relationship("RelatedClass")
The relationship.argument may also be omitted from the relationship() construct entirely, and instead placed inside a Mapped annotation on the left side, which should include a Python collection type if the relationship is expected to be a collection, such as:
class SomeClass(Base):
    # 

    related_items: Mapped[List["RelatedItem"]] = relationship()
Or for a many-to-one or one-to-one relationship:
class SomeClass(Base):
    # 

    related_item: Mapped["RelatedItem"] = relationship()
•  See also
Defining Mapped Properties with Declarative - further detail on relationship configuration when using Declarative.
•  secondary – 
For a many-to-many relationship, specifies the intermediary table, and is typically an instance of Table. In less common circumstances, the argument may also be specified as an Alias construct, or even a Join construct.
relationship.secondary may also be passed as a callable function which is evaluated at mapper initialization time. When using Declarative, it may also be a string argument noting the name of a Table that is present in the MetaData collection associated with the parent-mapped Table.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
The relationship.secondary keyword argument is typically applied in the case where the intermediary Table is not otherwise expressed in any direct class mapping. If the “secondary” table is also explicitly mapped elsewhere (e.g. as in Association Object), one should consider applying the relationship.viewonly flag so that this relationship() is not used for persistence operations which may conflict with those of the association object pattern.
See also
Many To Many - Reference example of “many to many”.
Self-Referential Many-to-Many Relationship - Specifics on using many-to-many in a self-referential case.
Configuring Many-to-Many Relationships - Additional options when using Declarative.
Association Object - an alternative to relationship.secondary when composing association table relationships, allowing additional attributes to be specified on the association table.
Composite “Secondary” Joins - a lesser-used pattern which in some cases can enable complex relationship() SQL conditions to be used.
•  active_history=False – When True, indicates that the “previous” value for a many-to-one reference should be loaded when replaced, if not already loaded. Normally, history tracking logic for simple many-to-ones only needs to be aware of the “new” value in order to perform a flush. This flag is available for applications that make use of get_history() which also need to know the “previous” value of the attribute.
•  backref – 
A reference to a string relationship name, or a backref() construct, which will be used to automatically generate a new relationship() on the related class, which then refers to this one using a bi-directional relationship.back_populates configuration.
In modern Python, explicit use of relationship() with relationship.back_populates should be preferred, as it is more robust in terms of mapper configuration as well as more conceptually straightforward. It also integrates with new PEP 484 typing features introduced in SQLAlchemy 2.0 which is not possible with dynamically generated attributes.
See also
Using the legacy ‘backref’ relationship parameter - notes on using relationship.backref
Working with ORM Related Objects - in the SQLAlchemy Unified Tutorial, presents an overview of bi-directional relationship configuration and behaviors using relationship.back_populates
backref() - allows control over relationship() configuration when using relationship.backref.
•  back_populates – 
Indicates the name of a relationship() on the related class that will be synchronized with this one. It is usually expected that the relationship() on the related class also refer to this one. This allows objects on both sides of each relationship() to synchronize in-Python state changes and also provides directives to the unit of work flush process how changes along these relationships should be persisted.
See also
Working with ORM Related Objects - in the SQLAlchemy Unified Tutorial, presents an overview of bi-directional relationship configuration and behaviors.
Basic Relationship Patterns - includes many examples of relationship.back_populates.
relationship.backref - legacy form which allows more succinct configuration, but does not support explicit typing
•  overlaps – 
A string name or comma-delimited set of names of other relationships on either this mapper, a descendant mapper, or a target mapper with which this relationship may write to the same foreign keys upon persistence. The only effect this has is to eliminate the warning that this relationship will conflict with another upon persistence. This is used for such relationships that are truly capable of conflicting with each other on write, but the application will ensure that no such conflicts occur.
Added in version 1.4.
See also
relationship X will copy column Q to column P, which conflicts with relationship(s): ‘Y’ - usage example
•  cascade – 
A comma-separated list of cascade rules which determines how Session operations should be “cascaded” from parent to child. This defaults to False, which means the default cascade should be used - this default cascade is "save-update, merge".
The available cascades are save-update, merge, expunge, delete, delete-orphan, and refresh-expire. An additional option, all indicates shorthand for "save-update, merge, refresh-expire, expunge, delete", and is often used as in "all, delete-orphan" to indicate that related objects should follow along with the parent object in all cases, and be deleted when de-associated.
See also
Cascades - Full detail on each of the available cascade options.
•  cascade_backrefs=False – 
Legacy; this flag is always False.
Changed in version 2.0: “cascade_backrefs” functionality has been removed.
•  collection_class – 
A class or callable that returns a new list-holding object. will be used in place of a plain list for storing elements.
See also
Customizing Collection Access - Introductory documentation and examples.
•  comparator_factory – 
A class which extends Comparator which provides custom SQL clause generation for comparison operations.
See also
PropComparator - some detail on redefining comparators at this level.
Operator Customization - Brief intro to this feature.
•  distinct_target_key=None – 
Indicate if a “subquery” eager load should apply the DISTINCT keyword to the innermost SELECT statement. When left as None, the DISTINCT keyword will be applied in those cases when the target columns do not comprise the full primary key of the target table. When set to True, the DISTINCT keyword is applied to the innermost SELECT unconditionally.
It may be desirable to set this flag to False when the DISTINCT is reducing performance of the innermost subquery beyond that of what duplicate innermost rows may be causing.
See also
Relationship Loading Techniques - includes an introduction to subquery eager loading.
•  doc – Docstring which will be applied to the resulting descriptor.
•  foreign_keys – 
A list of columns which are to be used as “foreign key” columns, or columns which refer to the value in a remote column, within the context of this relationship() object’s relationship.primaryjoin condition. That is, if the relationship.primaryjoin condition of this relationship() is a.id == b.a_id, and the values in b.a_id are required to be present in a.id, then the “foreign key” column of this relationship() is b.a_id.
In normal cases, the relationship.foreign_keys parameter is not required. relationship() will automatically determine which columns in the relationship.primaryjoin condition are to be considered “foreign key” columns based on those Column objects that specify ForeignKey, or are otherwise listed as referencing columns in a ForeignKeyConstraint construct. relationship.foreign_keys is only needed when:
1. There is more than one way to construct a join from the local table to the remote table, as there are multiple foreign key references present. Setting foreign_keys will limit the relationship() to consider just those columns specified here as “foreign”.
2. The Table being mapped does not actually have ForeignKey or ForeignKeyConstraint constructs present, often because the table was reflected from a database that does not support foreign key reflection (MySQL MyISAM).
3. The relationship.primaryjoin argument is used to construct a non-standard join condition, which makes use of columns or expressions that do not normally refer to their “parent” column, such as a join condition expressed by a complex comparison using a SQL function.
The relationship() construct will raise informative error messages that suggest the use of the relationship.foreign_keys parameter when presented with an ambiguous condition. In typical cases, if relationship() doesn’t raise any exceptions, the relationship.foreign_keys parameter is usually not needed.
relationship.foreign_keys may also be passed as a callable function which is evaluated at mapper initialization time, and may be passed as a Python-evaluable string when using Declarative.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
See also
Handling Multiple Join Paths
Creating Custom Foreign Conditions
foreign() - allows direct annotation of the “foreign” columns within a relationship.primaryjoin condition.
•  info – Optional data dictionary which will be populated into the MapperProperty.info attribute of this object.
•  innerjoin=False – 
When True, joined eager loads will use an inner join to join against related tables instead of an outer join. The purpose of this option is generally one of performance, as inner joins generally perform better than outer joins.
This flag can be set to True when the relationship references an object via many-to-one using local foreign keys that are not nullable, or when the reference is one-to-one or a collection that is guaranteed to have one or at least one entry.
The option supports the same “nested” and “unnested” options as that of joinedload.innerjoin. See that flag for details on nested / unnested behaviors.
See also
joinedload.innerjoin - the option as specified by loader option, including detail on nesting behavior.
What Kind of Loading to Use ? - Discussion of some details of various loader options.
•  join_depth – 
When non-None, an integer value indicating how many levels deep “eager” loaders should join on a self-referring or cyclical relationship. The number counts how many times the same Mapper shall be present in the loading condition along a particular join branch. When left at its default of None, eager loaders will stop chaining when they encounter a the same target mapper which is already higher up in the chain. This option applies both to joined- and subquery- eager loaders.
See also
Configuring Self-Referential Eager Loading - Introductory documentation and examples.
•  lazy='select' – 
specifies How the related items should be loaded. Default value is select. Values include:
• select - items should be loaded lazily when the property is first accessed, using a separate SELECT statement, or identity map fetch for simple many-to-one references.
• immediate - items should be loaded as the parents are loaded, using a separate SELECT statement, or identity map fetch for simple many-to-one references.
• joined - items should be loaded “eagerly” in the same query as that of the parent, using a JOIN or LEFT OUTER JOIN. Whether the join is “outer” or not is determined by the relationship.innerjoin parameter.
• subquery - items should be loaded “eagerly” as the parents are loaded, using one additional SQL statement, which issues a JOIN to a subquery of the original statement, for each collection requested.
• selectin - items should be loaded “eagerly” as the parents are loaded, using one or more additional SQL statements, which issues a JOIN to the immediate parent object, specifying primary key identifiers using an IN clause.
• noload - no loading should occur at any time. The related collection will remain empty. The noload strategy is not recommended for general use. For a general use “never load” approach, see Write Only Relationships
• raise - lazy loading is disallowed; accessing the attribute, if its value were not already loaded via eager loading, will raise an InvalidRequestError. This strategy can be used when objects are to be detached from their attached Session after they are loaded.
• raise_on_sql - lazy loading that emits SQL is disallowed; accessing the attribute, if its value were not already loaded via eager loading, will raise an InvalidRequestError, if the lazy load needs to emit SQL. If the lazy load can pull the related value from the identity map or determine that it should be None, the value is loaded. This strategy can be used when objects will remain associated with the attached Session, however additional SELECT statements should be blocked.
• write_only - the attribute will be configured with a special “virtual collection” that may receive WriteOnlyCollection.add() and WriteOnlyCollection.remove() commands to add or remove individual objects, but will not under any circumstances load or iterate the full set of objects from the database directly. Instead, methods such as WriteOnlyCollection.select(), WriteOnlyCollection.insert(), WriteOnlyCollection.update() and WriteOnlyCollection.delete() are provided which generate SQL constructs that may be used to load and modify rows in bulk. Used for large collections that are never appropriate to load at once into memory.
The write_only loader style is configured automatically when the WriteOnlyMapped annotation is provided on the left hand side within a Declarative mapping. See the section Write Only Relationships for examples.
Added in version 2.0.
See also
Write Only Relationships - in the ORM Querying Guide
• dynamic - the attribute will return a pre-configured Query object for all read operations, onto which further filtering operations can be applied before iterating the results.
The dynamic loader style is configured automatically when the DynamicMapped annotation is provided on the left hand side within a Declarative mapping. See the section Dynamic Relationship Loaders for examples.
Legacy Feature
The “dynamic” lazy loader strategy is the legacy form of what is now the “write_only” strategy described in the section Write Only Relationships.
See also
Dynamic Relationship Loaders - in the ORM Querying Guide
Write Only Relationships - more generally useful approach for large collections that should not fully load into memory
• True - a synonym for ‘select’
• False - a synonym for ‘joined’
• None - a synonym for ‘noload’
See also
Relationship Loading Techniques - Full documentation on relationship loader configuration in the ORM Querying Guide.
•  load_on_pending=False – 
Indicates loading behavior for transient or pending parent objects.
When set to True, causes the lazy-loader to issue a query for a parent object that is not persistent, meaning it has never been flushed. This may take effect for a pending object when autoflush is disabled, or for a transient object that has been “attached” to a Session but is not part of its pending collection.
The relationship.load_on_pending flag does not improve behavior when the ORM is used normally - object references should be constructed at the object level, not at the foreign key level, so that they are present in an ordinary way before a flush proceeds. This flag is not not intended for general use.
See also
Session.enable_relationship_loading() - this method establishes “load on pending” behavior for the whole object, and also allows loading on objects that remain transient or detached.
•  order_by – 
Indicates the ordering that should be applied when loading these items. relationship.order_by is expected to refer to one of the Column objects to which the target class is mapped, or the attribute itself bound to the target class which refers to the column.
relationship.order_by may also be passed as a callable function which is evaluated at mapper initialization time, and may be passed as a Python-evaluable string when using Declarative.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
•  passive_deletes=False – 
Indicates loading behavior during delete operations.
A value of True indicates that unloaded child items should not be loaded during a delete operation on the parent. Normally, when a parent item is deleted, all child items are loaded so that they can either be marked as deleted, or have their foreign key to the parent set to NULL. Marking this flag as True usually implies an ON DELETE <CASCADE|SET NULL> rule is in place which will handle updating/deleting child rows on the database side.
Additionally, setting the flag to the string value ‘all’ will disable the “nulling out” of the child foreign keys, when the parent object is deleted and there is no delete or delete-orphan cascade enabled. This is typically used when a triggering or error raise scenario is in place on the database side. Note that the foreign key attributes on in-session child objects will not be changed after a flush occurs so this is a very special use-case setting. Additionally, the “nulling out” will still occur if the child object is de-associated with the parent.
See also
Using foreign key ON DELETE cascade with ORM relationships - Introductory documentation and examples.
•  passive_updates=True – 
Indicates the persistence behavior to take when a referenced primary key value changes in place, indicating that the referencing foreign key columns will also need their value changed.
When True, it is assumed that ON UPDATE CASCADE is configured on the foreign key in the database, and that the database will handle propagation of an UPDATE from a source column to dependent rows. When False, the SQLAlchemy relationship() construct will attempt to emit its own UPDATE statements to modify related targets. However note that SQLAlchemy cannot emit an UPDATE for more than one level of cascade. Also, setting this flag to False is not compatible in the case where the database is in fact enforcing referential integrity, unless those constraints are explicitly “deferred”, if the target backend supports it.
It is highly advised that an application which is employing mutable primary keys keeps passive_updates set to True, and instead uses the referential integrity features of the database itself in order to handle the change efficiently and fully.
See also
Mutable Primary Keys / Update Cascades - Introductory documentation and examples.
mapper.passive_updates - a similar flag which takes effect for joined-table inheritance mappings.
•  post_update – 
This indicates that the relationship should be handled by a second UPDATE statement after an INSERT or before a DELETE. This flag is used to handle saving bi-directional dependencies between two individual rows (i.e. each row references the other), where it would otherwise be impossible to INSERT or DELETE both rows fully since one row exists before the other. Use this flag when a particular mapping arrangement will incur two rows that are dependent on each other, such as a table that has a one-to-many relationship to a set of child rows, and also has a column that references a single child row within that list (i.e. both tables contain a foreign key to each other). If a flush operation returns an error that a “cyclical dependency” was detected, this is a cue that you might want to use relationship.post_update to “break” the cycle.
See also
Rows that point to themselves / Mutually Dependent Rows - Introductory documentation and examples.
•  primaryjoin – 
A SQL expression that will be used as the primary join of the child object against the parent object, or in a many-to-many relationship the join of the parent object to the association table. By default, this value is computed based on the foreign key relationships of the parent and child tables (or association table).
relationship.primaryjoin may also be passed as a callable function which is evaluated at mapper initialization time, and may be passed as a Python-evaluable string when using Declarative.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
See also
Specifying Alternate Join Conditions
•  remote_side – 
Used for self-referential relationships, indicates the column or list of columns that form the “remote side” of the relationship.
relationship.remote_side may also be passed as a callable function which is evaluated at mapper initialization time, and may be passed as a Python-evaluable string when using Declarative.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
See also
Adjacency List Relationships - in-depth explanation of how relationship.remote_side is used to configure self-referential relationships.
remote() - an annotation function that accomplishes the same purpose as relationship.remote_side, typically when a custom relationship.primaryjoin condition is used.
•  query_class – 
A Query subclass that will be used internally by the AppenderQuery returned by a “dynamic” relationship, that is, a relationship that specifies lazy="dynamic" or was otherwise constructed using the dynamic_loader() function.
See also
Dynamic Relationship Loaders - Introduction to “dynamic” relationship loaders.
•  secondaryjoin – 
A SQL expression that will be used as the join of an association table to the child object. By default, this value is computed based on the foreign key relationships of the association and child tables.
relationship.secondaryjoin may also be passed as a callable function which is evaluated at mapper initialization time, and may be passed as a Python-evaluable string when using Declarative.
Warning
When passed as a Python-evaluable string, the argument is interpreted using Python’s eval() function. DO NOT PASS UNTRUSTED INPUT TO THIS STRING. See Evaluation of relationship arguments for details on declarative evaluation of relationship() arguments.
See also
Specifying Alternate Join Conditions
•  single_parent – 
When True, installs a validator which will prevent objects from being associated with more than one parent at a time. This is used for many-to-one or many-to-many relationships that should be treated either as one-to-one or one-to-many. Its usage is optional, except for relationship() constructs which are many-to-one or many-to-many and also specify the delete-orphan cascade option. The relationship() construct itself will raise an error instructing when this option is required.
See also
Cascades - includes detail on when the relationship.single_parent flag may be appropriate.
•  uselist – 
A boolean that indicates if this property should be loaded as a list or a scalar. In most cases, this value is determined automatically by relationship() at mapper configuration time. When using explicit Mapped annotations, relationship.uselist may be derived from the whether or not the annotation within Mapped contains a collection class. Otherwise, relationship.uselist may be derived from the type and direction of the relationship - one to many forms a list, many to one forms a scalar, many to many is a list. If a scalar is desired where normally a list would be present, such as a bi-directional one-to-one relationship, use an appropriate Mapped annotation or set relationship.uselist to False.
The relationship.uselist flag is also available on an existing relationship() construct as a read-only attribute, which can be used to determine if this relationship() deals with collections or scalar attributes:
 User.addresses.property.uselist
True
• See also
One To One - Introduction to the “one to one” relationship pattern, which is typically when an alternate setting for relationship.uselist is involved.
• viewonly=False – 
When set to True, the relationship is used only for loading objects, and not for any persistence operation. A relationship() which specifies relationship.viewonly can work with a wider range of SQL operations within the relationship.primaryjoin condition, including operations that feature the use of a variety of comparison operators as well as SQL functions such as cast(). The relationship.viewonly flag is also of general use when defining any kind of relationship() that doesn’t represent the full set of related objects, to prevent modifications of the collection from resulting in persistence operations.
See also
Notes on using the viewonly relationship parameter - more details on best practices when using relationship.viewonly.
• sync_backref – 
A boolean that enables the events used to synchronize the in-Python attributes when this relationship is target of either relationship.backref or relationship.back_populates.
Defaults to None, which indicates that an automatic value should be selected based on the value of the relationship.viewonly flag. When left at its default, changes in state will be back-populated only if neither sides of a relationship is viewonly.
Added in version 1.3.17.
Changed in version 1.4: - A relationship that specifies relationship.viewonly automatically implies that relationship.sync_backref is False.
See also
relationship.viewonly
• omit_join – 
Allows manual control over the “selectin” automatic join optimization. Set to False to disable the “omit join” feature added in SQLAlchemy 1.3; or leave as None to leave automatic optimization in place.
Note
This flag may only be set to False. It is not necessary to set it to True as the “omit_join” optimization is automatically detected; if it is not detected, then the optimization is not supported.
Changed in version 1.3.11: setting omit_join to True will now emit a warning as this was not the intended use of this flag.
Added in version 1.3.
• init – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __init__() method as generated by the dataclass process.
• repr – Specific to Declarative Dataclass Mapping, specifies if the mapped attribute should be part of the __repr__() method as generated by the dataclass process.
• default_factory – Specific to Declarative Dataclass Mapping, specifies a default-value generation function that will take place as part of the __init__() method as generated by the dataclass process.
• compare – 
Specific to Declarative Dataclass Mapping, indicates if this field should be included in comparison operations when generating the __eq__() and __ne__() methods for the mapped class.
Added in version 2.0.0b4.
• kw_only – Specific to Declarative Dataclass Mapping, indicates if this field should be marked as keyword-only when generating the __init__().
• hash – 
Specific to Declarative Dataclass Mapping, controls if this field is included when generating the __hash__() method for the mapped class.
Added in version 2.0.36.
function sqlalchemy.orm.backref(name: str, **kwargs: Any) → ORMBackrefArgument
When using the relationship.backref parameter, provides specific parameters to be used when the new relationship() is generated.
E.g.:
"items": relationship(SomeItem, backref=backref("parent", lazy="subquery"))
The relationship.backref parameter is generally considered to be legacy; for modern applications, using explicit relationship() constructs linked together using the relationship.back_populates parameter should be preferred.
See also
Using the legacy ‘backref’ relationship parameter - background on backrefs
function sqlalchemy.orm.dynamic_loader(argument: _RelationshipArgumentType[Any] | None = None, **kw: Any) → RelationshipProperty[Any]
Construct a dynamically-loading mapper property.
This is essentially the same as using the lazy='dynamic' argument with relationship():
dynamic_loader(SomeClass)

# is the same as

relationship(SomeClass, lazy="dynamic")
See the section Dynamic Relationship Loaders for more details on dynamic loading.
function sqlalchemy.orm.foreign(expr: _CEA) → _CEA
Annotate a portion of a primaryjoin expression with a ‘foreign’ annotation.
See the section Creating Custom Foreign Conditions for a description of use.
See also
Creating Custom Foreign Conditions
remote()
function sqlalchemy.orm.remote(expr: _CEA) → _CEA
Annotate a portion of a primaryjoin expression with a ‘remote’ annotation.
See the section Creating Custom Foreign Conditions for a description of use.
See also
Creating Custom Foreign Conditions
foreign()



