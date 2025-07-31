ORM Querying Guide
This section provides an overview of emitting queries with the SQLAlchemy ORM using 2.0 style usage.
Readers of this section should be familiar with the SQLAlchemy overview at SQLAlchemy Unified Tutorial, and in particular most of the content here expands upon the content at Using SELECT Statements.
For users of SQLAlchemy 1.x
In the SQLAlchemy 2.x series, SQL SELECT statements for the ORM are constructed using the same select() construct as is used in Core, which is then invoked in terms of a Session using the Session.execute() method (as are the update() and delete() constructs now used for the ORM-Enabled INSERT, UPDATE, and DELETE statements feature). However, the legacy Query object, which performs these same steps as more of an “all-in-one” object, continues to remain available as a thin facade over this new system, to support applications that were built on the 1.x series without the need for wholesale replacement of all queries. For reference on this object, see the section Legacy Query API.
* Writing SELECT statements for ORM Mapped Classes
o Selecting ORM Entities and Attributes
* Selecting ORM Entities
* Selecting Multiple ORM Entities Simultaneously
* Selecting Individual Attributes
* Grouping Selected Attributes with Bundles
* Selecting ORM Aliases
* Getting ORM Results from Textual Statements
* Selecting Entities from Subqueries
* Selecting Entities from UNIONs and other set operations
o Joins
* Simple Relationship Joins
* Chaining Multiple Joins
* Joins to a Target Entity
* Joins to a Target with an ON Clause
* Combining Relationship with Custom ON Criteria
* Using Relationship to join between aliased targets
* Joining to Subqueries
* Joining to Subqueries along Relationship paths
* Subqueries that Refer to Multiple Entities
* Setting the leftmost FROM clause in a join
o Relationship WHERE Operators
* EXISTS forms: has() / any()
* Relationship Instance Comparison Operators
* Writing SELECT statements for Inheritance Mappings
o SELECTing from the base class vs. specific sub-classes
o Using selectin_polymorphic()
* Applying selectin_polymorphic() to an existing eager load
* Applying loader options to the subclasses loaded by selectin_polymorphic
* Configuring selectin_polymorphic() on mappers
o Using with_polymorphic()
* Filtering Subclass Attributes with with_polymorphic()
* Using aliasing with with_polymorphic
* Configuring with_polymorphic() on mappers
o Joining to specific sub-types or with_polymorphic() entities
* Eager Loading of Polymorphic Subtypes
o SELECT Statements for Single Inheritance Mappings
* Optimizing Attribute Loads for Single Inheritance
o Inheritance Loading API
* with_polymorphic()
* selectin_polymorphic()
* ORM-Enabled INSERT, UPDATE, and DELETE statements
o ORM Bulk INSERT Statements
* Getting new objects with RETURNING
* Using Heterogeneous Parameter Dictionaries
* Sending NULL values in ORM bulk INSERT statements
* Bulk INSERT for Joined Table Inheritance
* ORM Bulk Insert with SQL Expressions
* Legacy Session Bulk INSERT Methods
* ORM “upsert” Statements
o ORM Bulk UPDATE by Primary Key
* Disabling Bulk ORM Update by Primary Key for an UPDATE statement with multiple parameter sets
* Bulk UPDATE by Primary Key for Joined Table Inheritance
* Legacy Session Bulk UPDATE Methods
o ORM UPDATE and DELETE with Custom WHERE Criteria
* Important Notes and Caveats for ORM-Enabled Update and Delete
* Selecting a Synchronization Strategy
* Using RETURNING with UPDATE/DELETE and Custom WHERE Criteria
* UPDATE/DELETE with Custom WHERE Criteria for Joined Table Inheritance
* Legacy Query Methods
* Column Loading Options
o Limiting which Columns Load with Column Deferral
* Using load_only() to reduce loaded columns
* Using defer() to omit specific columns
* Using raiseload to prevent deferred column loads
o Configuring Column Deferral on Mappings
* Using deferred() for imperative mappers, mapped SQL expressions
* Using undefer() to “eagerly” load deferred columns
* Loading deferred columns in groups
* Undeferring by group with undefer_group()
* Undeferring on wildcards
* Configuring mapper-level “raiseload” behavior
o Loading Arbitrary SQL Expressions onto Objects
* Using with_expression() with UNIONs, other subqueries
o Column Loading API
* defer()
* deferred()
* query_expression()
* load_only()
* undefer()
* undefer_group()
* with_expression()
* Relationship Loading Techniques
o Summary of Relationship Loading Styles
o Configuring Loader Strategies at Mapping Time
o Relationship Loading with Loader Options
* Adding Criteria to loader options
* Specifying Sub-Options with Load.options()
o Lazy Loading
* Preventing unwanted lazy loads using raiseload
o Joined Eager Loading
* The Zen of Joined Eager Loading
o Select IN loading
o Subquery Eager Loading
o What Kind of Loading to Use ?
o Polymorphic Eager Loading
o Wildcard Loading Strategies
* Per-Entity Wildcard Loading Strategies
o Routing Explicit Joins/Statements into Eagerly Loaded Collections
* Using contains_eager() to load a custom-filtered collection result
o Relationship Loader API
* contains_eager()
* defaultload()
* immediateload()
* joinedload()
* lazyload()
* Load
* noload()
* raiseload()
* selectinload()
* subqueryload()
* ORM API Features for Querying
o ORM Loader Options
o ORM Execution Options
* Populate Existing
* Autoflush
* Fetching Large Result Sets with Yield Per
* Identity Token
* Legacy Query API
o The Query Object
* Query
o ORM-Specific Query Constructs


Writing SELECT statements for ORM Mapped Classes
About this Document
This section makes use of ORM mappings first illustrated in the SQLAlchemy Unified Tutorial, shown in the section Declaring Mapped Classes.
View the ORM setup for this page.
SELECT statements are produced by the select() function which returns a Select object. The entities and/or SQL expressions to return (i.e. the “columns” clause) are passed positionally to the function. From there, additional methods are used to generate the complete statement, such as the Select.where() method illustrated below:
from sqlalchemy import select
stmt = select(User).where(User.name == "spongebob")
Given a completed Select object, in order to execute it within the ORM to get rows back, the object is passed to Session.execute(), where a Result object is then returned:
result = session.execute(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.name = ?
[] ('spongebob',)
for user_obj in result.scalars():
    print(f"{user_obj.name} {user_obj.fullname}")
spongebob Spongebob Squarepants
Selecting ORM Entities and Attributes
The select() construct accepts ORM entities, including mapped classes as well as class-level attributes representing mapped columns, which are converted into ORM-annotated FromClause and ColumnElement elements at construction time.
A Select object that contains ORM-annotated entities is normally executed using a Session object, and not a Connection object, so that ORM-related features may take effect, including that instances of ORM-mapped objects may be returned. When using the Connection directly, result rows will only contain column-level data.
Selecting ORM Entities
Below we select from the User entity, producing a Select that selects from the mapped Table to which User is mapped:
result = session.execute(select(User).order_by(User.id))
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account ORDER BY user_account.id
[] ()
When selecting from ORM entities, the entity itself is returned in the result as a row with a single element, as opposed to a series of individual columns; for example above, the Result returns Row objects that have just a single element per row, that element holding onto a User object:
result.all()
[(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),),
 (User(id=2, name='sandy', fullname='Sandy Cheeks'),),
 (User(id=3, name='patrick', fullname='Patrick Star'),),
 (User(id=4, name='squidward', fullname='Squidward Tentacles'),),
 (User(id=5, name='ehkrabs', fullname='Eugene H. Krabs'),)]
When selecting a list of single-element rows containing ORM entities, it is typical to skip the generation of Row objects and instead receive ORM entities directly. This is most easily achieved by using the Session.scalars() method to execute, rather than the Session.execute() method, so that a ScalarResult object which yields single elements rather than rows is returned:
session.scalars(select(User).order_by(User.id)).all()
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account ORDER BY user_account.id
[] ()
[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
 User(id=2, name='sandy', fullname='Sandy Cheeks'),
 User(id=3, name='patrick', fullname='Patrick Star'),
 User(id=4, name='squidward', fullname='Squidward Tentacles'),
 User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]
Calling the Session.scalars() method is the equivalent to calling upon Session.execute() to receive a Result object, then calling upon Result.scalars() to receive a ScalarResult object.
Selecting Multiple ORM Entities Simultaneously
The select() function accepts any number of ORM classes and/or column expressions at once, including that multiple ORM classes may be requested. When SELECTing from multiple ORM classes, they are named in each result row based on their class name. In the example below, the result rows for a SELECT against User and Address will refer to them under the names User and Address:
stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
for row in session.execute(stmt):
    print(f"{row.User.name} {row.Address.email_address}")
SELECT user_account.id, user_account.name, user_account.fullname,
address.id AS id_1, address.user_id, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
ORDER BY user_account.id, address.id
[] ()
spongebob spongebob@sqlalchemy.org
sandy sandy@sqlalchemy.org
sandy squirrel@squirrelpower.org
patrick pat999@aol.com
squidward stentcl@sqlalchemy.org
If we wanted to assign different names to these entities in the rows, we would use the aliased() construct using the aliased.name parameter to alias them with an explicit name:
from sqlalchemy.orm import aliased
user_cls = aliased(User, name="user_cls")
email_cls = aliased(Address, name="email")
stmt = (
    select(user_cls, email_cls)
    .join(user_cls.addresses.of_type(email_cls))
    .order_by(user_cls.id, email_cls.id)
)
row = session.execute(stmt).first()
SELECT user_cls.id, user_cls.name, user_cls.fullname,
email.id AS id_1, email.user_id, email.email_address
FROM user_account AS user_cls JOIN address AS email
ON user_cls.id = email.user_id ORDER BY user_cls.id, email.id
[] ()
print(f"{row.user_cls.name} {row.email.email_address}")
spongebob spongebob@sqlalchemy.org
The aliased form above is discussed further at Using Relationship to join between aliased targets.
An existing Select construct may also have ORM classes and/or column expressions added to its columns clause using the Select.add_columns() method. We can produce the same statement as above using this form as well:
stmt = (
    select(User).join(User.addresses).add_columns(Address).order_by(User.id, Address.id)
)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname,
address.id AS id_1, address.user_id, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
ORDER BY user_account.id, address.id
Selecting Individual Attributes
The attributes on a mapped class, such as User.name and Address.email_address, can be used just like Column or other SQL expression objects when passed to select(). Creating a select() that is against specific columns will return Row objects, and not entities like User or Address objects. Each Row will have each column represented individually:
result = session.execute(
    select(User.name, Address.email_address)
    .join(User.addresses)
    .order_by(User.id, Address.id)
)
SELECT user_account.name, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
ORDER BY user_account.id, address.id
[] ()
The above statement returns Row objects with name and email_address columns, as illustrated in the runtime demonstration below:
for row in result:
    print(f"{row.name}  {row.email_address}")
spongebob  spongebob@sqlalchemy.org
sandy  sandy@sqlalchemy.org
sandy  squirrel@squirrelpower.org
patrick  pat999@aol.com
squidward  stentcl@sqlalchemy.org
Grouping Selected Attributes with Bundles
The Bundle construct is an extensible ORM-only construct that allows sets of column expressions to be grouped in result rows:
from sqlalchemy.orm import Bundle
stmt = select(
    Bundle("user", User.name, User.fullname),
    Bundle("email", Address.email_address),
).join_from(User, Address)
for row in session.execute(stmt):
    print(f"{row.user.name} {row.user.fullname} {row.email.email_address}")
SELECT user_account.name, user_account.fullname, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
[] ()
spongebob Spongebob Squarepants spongebob@sqlalchemy.org
sandy Sandy Cheeks sandy@sqlalchemy.org
sandy Sandy Cheeks squirrel@squirrelpower.org
patrick Patrick Star pat999@aol.com
squidward Squidward Tentacles stentcl@sqlalchemy.org
The Bundle is potentially useful for creating lightweight views and custom column groupings. Bundle may also be subclassed in order to return alternate data structures; see Bundle.create_row_processor() for an example.
See also
Bundle
Bundle.create_row_processor()
Selecting ORM Aliases
As discussed in the tutorial at Using Aliases, to create a SQL alias of an ORM entity is achieved using the aliased() construct against a mapped class:
from sqlalchemy.orm import aliased
u1 = aliased(User)
print(select(u1).order_by(u1.id))
SELECT user_account_1.id, user_account_1.name, user_account_1.fullname
FROM user_account AS user_account_1 ORDER BY user_account_1.id
As is the case when using Table.alias(), the SQL alias is anonymously named. For the case of selecting the entity from a row with an explicit name, the aliased.name parameter may be passed as well:
from sqlalchemy.orm import aliased
u1 = aliased(User, name="u1")
stmt = select(u1).order_by(u1.id)
row = session.execute(stmt).first()
SELECT u1.id, u1.name, u1.fullname
FROM user_account AS u1 ORDER BY u1.id
[] ()
print(f"{row.u1.name}")
spongebob
See also
The aliased construct is central for several use cases, including:
* making use of subqueries with the ORM; the sections Selecting Entities from Subqueries and Joining to Subqueries discuss this further.
* Controlling the name of an entity in a result set; see Selecting Multiple ORM Entities Simultaneously for an example
* Joining to the same ORM entity multiple times; see Using Relationship to join between aliased targets for an example.
Getting ORM Results from Textual Statements
The ORM supports loading of entities from SELECT statements that come from other sources. The typical use case is that of a textual SELECT statement, which in SQLAlchemy is represented using the text() construct. A text() construct can be augmented with information about the ORM-mapped columns that the statement would load; this can then be associated with the ORM entity itself so that ORM objects can be loaded based on this statement.
Given a textual SQL statement we’d like to load from:
from sqlalchemy import text
textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")
We can add column information to the statement by using the TextClause.columns() method; when this method is invoked, the TextClause object is converted into a TextualSelect object, which takes on a role that is comparable to the Select construct. The TextClause.columns() method is typically passed Column objects or equivalent, and in this case we can make use of the ORM-mapped attributes on the User class directly:
textual_sql = textual_sql.columns(User.id, User.name, User.fullname)
We now have an ORM-configured SQL construct that as given, can load the “id”, “name” and “fullname” columns separately. To use this SELECT statement as a source of complete User entities instead, we can link these columns to a regular ORM-enabled Select construct using the Select.from_statement() method:
orm_sql = select(User).from_statement(textual_sql)
for user_obj in session.execute(orm_sql).scalars():
    print(user_obj)
SELECT id, name, fullname FROM user_account ORDER BY id
[] ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
The same TextualSelect object can also be converted into a subquery using the TextualSelect.subquery() method, and linked to the User entity to it using the aliased() construct, in a similar manner as discussed below in Selecting Entities from Subqueries:
orm_subquery = aliased(User, textual_sql.subquery())
stmt = select(orm_subquery)
for user_obj in session.execute(stmt).scalars():
    print(user_obj)
SELECT anon_1.id, anon_1.name, anon_1.fullname
FROM (SELECT id, name, fullname FROM user_account ORDER BY id) AS anon_1
[] ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
The difference between using the TextualSelect directly with Select.from_statement() versus making use of aliased() is that in the former case, no subquery is produced in the resulting SQL. This can in some scenarios be advantageous from a performance or complexity perspective.
Selecting Entities from Subqueries
The aliased() construct discussed in the previous section can be used with any Subquery construct that comes from a method such as Select.subquery() to link ORM entities to the columns returned by that subquery; there must be a column correspondence relationship between the columns delivered by the subquery and the columns to which the entity is mapped, meaning, the subquery needs to be ultimately derived from those entities, such as in the example below:
inner_stmt = select(User).where(User.id < 7).order_by(User.id)
subq = inner_stmt.subquery()
aliased_user = aliased(User, subq)
stmt = select(aliased_user)
for user_obj in session.execute(stmt).scalars():
    print(user_obj)
 SELECT anon_1.id, anon_1.name, anon_1.fullname
FROM (SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
FROM user_account
WHERE user_account.id < ? ORDER BY user_account.id) AS anon_1
[generated in ] (7,)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
See also
ORM Entity Subqueries/CTEs - in the SQLAlchemy Unified Tutorial
Joining to Subqueries
Selecting Entities from UNIONs and other set operations
The union() and union_all() functions are the most common set operations, which along with other set operations such as except_(), intersect() and others deliver an object known as a CompoundSelect, which is composed of multiple Select constructs joined by a set-operation keyword. ORM entities may be selected from simple compound selects using the Select.from_statement() method illustrated previously at Getting ORM Results from Textual Statements. In this method, the UNION statement is the complete statement that will be rendered, no additional criteria can be added after Select.from_statement() is used:
from sqlalchemy import union_all
u = union_all(
    select(User).where(User.id < 2), select(User).where(User.id == 3)
).order_by(User.id)
stmt = select(User).from_statement(u)
for user_obj in session.execute(stmt).scalars():
    print(user_obj)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.id < ? UNION ALL SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.id = ? ORDER BY id
[generated in ] (2, 3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
A CompoundSelect construct can be more flexibly used within a query that can be further modified by organizing it into a subquery and linking it to an ORM entity using aliased(), as illustrated previously at Selecting Entities from Subqueries. In the example below, we first use CompoundSelect.subquery() to create a subquery of the UNION ALL statement, we then package that into the aliased() construct where it can be used like any other mapped entity in a select() construct, including that we can add filtering and order by criteria based on its exported columns:
subq = union_all(
    select(User).where(User.id < 2), select(User).where(User.id == 3)
).subquery()
user_alias = aliased(User, subq)
stmt = select(user_alias).order_by(user_alias.id)
for user_obj in session.execute(stmt).scalars():
    print(user_obj)
SELECT anon_1.id, anon_1.name, anon_1.fullname
FROM (SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
FROM user_account
WHERE user_account.id < ? UNION ALL SELECT user_account.id AS id, user_account.name AS name, user_account.fullname AS fullname
FROM user_account
WHERE user_account.id = ?) AS anon_1 ORDER BY anon_1.id
[generated in ] (2, 3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
See also
Selecting ORM Entities from Unions - in the SQLAlchemy Unified Tutorial
Joins
The Select.join() and Select.join_from() methods are used to construct SQL JOINs against a SELECT statement.
This section will detail ORM use cases for these methods. For a general overview of their use from a Core perspective, see Explicit FROM clauses and JOINs in the SQLAlchemy Unified Tutorial.
The usage of Select.join() in an ORM context for 2.0 style queries is mostly equivalent, minus legacy use cases, to the usage of the Query.join() method in 1.x style queries.
Simple Relationship Joins
Consider a mapping between two classes User and Address, with a relationship User.addresses representing a collection of Address objects associated with each User. The most common usage of Select.join() is to create a JOIN along this relationship, using the User.addresses attribute as an indicator for how this should occur:
stmt = select(User).join(User.addresses)
Where above, the call to Select.join() along User.addresses will result in SQL approximately equivalent to:
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
In the above example we refer to User.addresses as passed to Select.join() as the “on clause”, that is, it indicates how the “ON” portion of the JOIN should be constructed.
Tip
Note that using Select.join() to JOIN from one entity to another affects the FROM clause of the SELECT statement, but not the columns clause; the SELECT statement in this example will continue to return rows from only the User entity. To SELECT columns / entities from both User and Address at the same time, the Address entity must also be named in the select() function, or added to the Select construct afterwards using the Select.add_columns() method. See the section Selecting Multiple ORM Entities Simultaneously for examples of both of these forms.
Chaining Multiple Joins
To construct a chain of joins, multiple Select.join() calls may be used. The relationship-bound attribute implies both the left and right side of the join at once. Consider additional entities Order and Item, where the User.orders relationship refers to the Order entity, and the Order.items relationship refers to the Item entity, via an association table order_items. Two Select.join() calls will result in a JOIN first from User to Order, and a second from Order to Item. However, since Order.items is a many to many relationship, it results in two separate JOIN elements, for a total of three JOIN elements in the resulting SQL:
stmt = select(User).join(User.orders).join(Order.items)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
JOIN user_order ON user_account.id = user_order.user_id
JOIN order_items AS order_items_1 ON user_order.id = order_items_1.order_id
JOIN item ON item.id = order_items_1.item_id
The order in which each call to the Select.join() method is significant only to the degree that the “left” side of what we would like to join from needs to be present in the list of FROMs before we indicate a new target. Select.join() would not, for example, know how to join correctly if we were to specify select(User).join(Order.items).join(User.orders), and would raise an error. In correct practice, the Select.join() method is invoked in such a way that lines up with how we would want the JOIN clauses in SQL to be rendered, and each call should represent a clear link from what precedes it.
All of the elements that we target in the FROM clause remain available as potential points to continue joining FROM. We can continue to add other elements to join FROM the User entity above, for example adding on the User.addresses relationship to our chain of joins:
stmt = select(User).join(User.orders).join(Order.items).join(User.addresses)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
JOIN user_order ON user_account.id = user_order.user_id
JOIN order_items AS order_items_1 ON user_order.id = order_items_1.order_id
JOIN item ON item.id = order_items_1.item_id
JOIN address ON user_account.id = address.user_id
Joins to a Target Entity
A second form of Select.join() allows any mapped entity or core selectable construct as a target. In this usage, Select.join() will attempt to infer the ON clause for the JOIN, using the natural foreign key relationship between two entities:
stmt = select(User).join(Address)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
In the above calling form, Select.join() is called upon to infer the “on clause” automatically. This calling form will ultimately raise an error if either there are no ForeignKeyConstraint setup between the two mapped Table constructs, or if there are multiple ForeignKeyConstraint linkages between them such that the appropriate constraint to use is ambiguous.
Note
When making use of Select.join() or Select.join_from() without indicating an ON clause, ORM configured relationship() constructs are not taken into account. Only the configured ForeignKeyConstraint relationships between the entities at the level of the mapped Table objects are consulted when an attempt is made to infer an ON clause for the JOIN.
Joins to a Target with an ON Clause
The third calling form allows both the target entity as well as the ON clause to be passed explicitly. A example that includes a SQL expression as the ON clause is as follows:
stmt = select(User).join(Address, User.id == Address.user_id)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
The expression-based ON clause may also be a relationship()-bound attribute, in the same way it’s used in Simple Relationship Joins:
stmt = select(User).join(Address, User.addresses)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account JOIN address ON user_account.id = address.user_id
The above example seems redundant in that it indicates the target of Address in two different ways; however, the utility of this form becomes apparent when joining to aliased entities; see the section Using Relationship to join between aliased targets for an example.
Combining Relationship with Custom ON Criteria
The ON clause generated by the relationship() construct may be augmented with additional criteria. This is useful both for quick ways to limit the scope of a particular join over a relationship path, as well as for cases like configuring loader strategies such as joinedload() and selectinload(). The PropComparator.and_() method accepts a series of SQL expressions positionally that will be joined to the ON clause of the JOIN via AND. For example if we wanted to JOIN from User to Address but also limit the ON criteria to only certain email addresses:
stmt = select(User.fullname).join(
    User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
)
session.execute(stmt).all()
SELECT user_account.fullname
FROM user_account
JOIN address ON user_account.id = address.user_id AND address.email_address = ?
[] ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
See also
The PropComparator.and_() method also works with loader strategies such as joinedload() and selectinload(). See the section Adding Criteria to loader options.
Using Relationship to join between aliased targets
When constructing joins using relationship()-bound attributes to indicate the ON clause, the two-argument syntax illustrated in Joins to a Target with an ON Clause can be expanded to work with the aliased() construct, to indicate a SQL alias as the target of a join while still making use of the relationship()-bound attribute to indicate the ON clause, as in the example below, where the User entity is joined twice to two different aliased() constructs against the Address entity:
address_alias_1 = aliased(Address)
address_alias_2 = aliased(Address)
stmt = (
    select(User)
    .join(address_alias_1, User.addresses)
    .where(address_alias_1.email_address == "patrick@aol.com")
    .join(address_alias_2, User.addresses)
    .where(address_alias_2.email_address == "patrick@gmail.com")
)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
JOIN address AS address_1 ON user_account.id = address_1.user_id
JOIN address AS address_2 ON user_account.id = address_2.user_id
WHERE address_1.email_address = :email_address_1
AND address_2.email_address = :email_address_2
The same pattern may be expressed more succinctly using the modifier PropComparator.of_type(), which may be applied to the relationship()-bound attribute, passing along the target entity in order to indicate the target in one step. The example below uses PropComparator.of_type() to produce the same SQL statement as the one just illustrated:
print(
    select(User)
    .join(User.addresses.of_type(address_alias_1))
    .where(address_alias_1.email_address == "patrick@aol.com")
    .join(User.addresses.of_type(address_alias_2))
    .where(address_alias_2.email_address == "patrick@gmail.com")
)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
JOIN address AS address_1 ON user_account.id = address_1.user_id
JOIN address AS address_2 ON user_account.id = address_2.user_id
WHERE address_1.email_address = :email_address_1
AND address_2.email_address = :email_address_2
To make use of a relationship() to construct a join from an aliased entity, the attribute is available from the aliased() construct directly:
user_alias_1 = aliased(User)
print(select(user_alias_1.name).join(user_alias_1.addresses))
SELECT user_account_1.name
FROM user_account AS user_account_1
JOIN address ON user_account_1.id = address.user_id
Joining to Subqueries
The target of a join may be any “selectable” entity which includes subqueries. When using the ORM, it is typical that these targets are stated in terms of an aliased() construct, but this is not strictly required, particularly if the joined entity is not being returned in the results. For example, to join from the User entity to the Address entity, where the Address entity is represented as a row limited subquery, we first construct a Subquery object using Select.subquery(), which may then be used as the target of the Select.join() method:
subq = select(Address).where(Address.email_address == "pat999@aol.com").subquery()
stmt = select(User).join(subq, User.id == subq.c.user_id)
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
JOIN (SELECT address.id AS id,
address.user_id AS user_id, address.email_address AS email_address
FROM address
WHERE address.email_address = :email_address_1) AS anon_1
ON user_account.id = anon_1.user_id
The above SELECT statement when invoked via Session.execute() will return rows that contain User entities, but not Address entities. In order to include Address entities to the set of entities that would be returned in result sets, we construct an aliased() object against the Address entity and Subquery object. We also may wish to apply a name to the aliased() construct, such as "address" used below, so that we can refer to it by name in the result row:
address_subq = aliased(Address, subq, name="address")
stmt = select(User, address_subq).join(address_subq)
for row in session.execute(stmt):
    print(f"{row.User} {row.address}")
SELECT user_account.id, user_account.name, user_account.fullname,
anon_1.id AS id_1, anon_1.user_id, anon_1.email_address
FROM user_account
JOIN (SELECT address.id AS id,
address.user_id AS user_id, address.email_address AS email_address
FROM address
WHERE address.email_address = ?) AS anon_1 ON user_account.id = anon_1.user_id
[] ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
Joining to Subqueries along Relationship paths
The subquery form illustrated in the previous section may be expressed with more specificity using a relationship()-bound attribute using one of the forms indicated at Using Relationship to join between aliased targets. For example, to create the same join while ensuring the join is along that of a particular relationship(), we may use the PropComparator.of_type() method, passing the aliased() construct containing the Subquery object that’s the target of the join:
address_subq = aliased(Address, subq, name="address")
stmt = select(User, address_subq).join(User.addresses.of_type(address_subq))
for row in session.execute(stmt):
    print(f"{row.User} {row.address}")
SELECT user_account.id, user_account.name, user_account.fullname,
anon_1.id AS id_1, anon_1.user_id, anon_1.email_address
FROM user_account
JOIN (SELECT address.id AS id,
address.user_id AS user_id, address.email_address AS email_address
FROM address
WHERE address.email_address = ?) AS anon_1 ON user_account.id = anon_1.user_id
[] ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
Subqueries that Refer to Multiple Entities
A subquery that contains columns spanning more than one ORM entity may be applied to more than one aliased() construct at once, and used in the same Select construct in terms of each entity separately. The rendered SQL will continue to treat all such aliased() constructs as the same subquery, however from the ORM / Python perspective the different return values and object attributes can be referenced by using the appropriate aliased() construct.
Given for example a subquery that refers to both User and Address:
user_address_subq = (
    select(User.id, User.name, User.fullname, Address.id, Address.email_address)
    .join_from(User, Address)
    .where(Address.email_address.in_(["pat999@aol.com", "squirrel@squirrelpower.org"]))
    .subquery()
)
We can create aliased() constructs against both User and Address that each refer to the same object:
user_alias = aliased(User, user_address_subq, name="user")
address_alias = aliased(Address, user_address_subq, name="address")
A Select construct selecting from both entities will render the subquery once, but in a result-row context can return objects of both User and Address classes at the same time:
stmt = select(user_alias, address_alias).where(user_alias.name == "sandy")
for row in session.execute(stmt):
    print(f"{row.user} {row.address}")
SELECT anon_1.id, anon_1.name, anon_1.fullname, anon_1.id_1, anon_1.email_address
FROM (SELECT user_account.id AS id, user_account.name AS name,
user_account.fullname AS fullname, address.id AS id_1,
address.email_address AS email_address
FROM user_account JOIN address ON user_account.id = address.user_id
WHERE address.email_address IN (?, ?)) AS anon_1
WHERE anon_1.name = ?
[] ('pat999@aol.com', 'squirrel@squirrelpower.org', 'sandy')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='squirrel@squirrelpower.org')
Setting the leftmost FROM clause in a join
In cases where the left side of the current state of Select is not in line with what we want to join from, the Select.join_from() method may be used:
stmt = select(Address).join_from(User, User.addresses).where(User.name == "sandy")
print(stmt)
SELECT address.id, address.user_id, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
WHERE user_account.name = :name_1
The Select.join_from() method accepts two or three arguments, either in the form (<join from>, <onclause>), or (<join from>, <join to>, [<onclause>]):
stmt = select(Address).join_from(User, Address).where(User.name == "sandy")
print(stmt)
SELECT address.id, address.user_id, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
WHERE user_account.name = :name_1
To set up the initial FROM clause for a SELECT such that Select.join() can be used subsequent, the Select.select_from() method may also be used:
stmt = select(Address).select_from(User).join(Address).where(User.name == "sandy")
print(stmt)
SELECT address.id, address.user_id, address.email_address
FROM user_account JOIN address ON user_account.id = address.user_id
WHERE user_account.name = :name_1
Tip
The Select.select_from() method does not actually have the final say on the order of tables in the FROM clause. If the statement also refers to a Join construct that refers to existing tables in a different order, the Join construct takes precedence. When we use methods like Select.join() and Select.join_from(), these methods are ultimately creating such a Join object. Therefore we can see the contents of Select.select_from() being overridden in a case like this:
stmt = select(Address).select_from(User).join(Address.user).where(User.name == "sandy")
print(stmt)
SELECT address.id, address.user_id, address.email_address
FROM address JOIN user_account ON user_account.id = address.user_id
WHERE user_account.name = :name_1
Where above, we see that the FROM clause is address JOIN user_account, even though we stated select_from(User) first. Because of the .join(Address.user) method call, the statement is ultimately equivalent to the following:
from sqlalchemy.sql import join
>>>
user_table = User.__table__
address_table = Address.__table__
>>>
j = address_table.join(user_table, user_table.c.id == address_table.c.user_id)
stmt = (
    select(address_table)
    .select_from(user_table)
    .select_from(j)
    .where(user_table.c.name == "sandy")
)
print(stmt)
SELECT address.id, address.user_id, address.email_address
FROM address JOIN user_account ON user_account.id = address.user_id
WHERE user_account.name = :name_1
The Join construct above is added as another entry in the Select.select_from() list which supersedes the previous entry.
Relationship WHERE Operators
Besides the use of relationship() constructs within the Select.join() and Select.join_from() methods, relationship() also plays a role in helping to construct SQL expressions that are typically for use in the WHERE clause, using the Select.where() method.
EXISTS forms: has() / any()
The Exists construct was first introduced in the SQLAlchemy Unified Tutorial in the section EXISTS subqueries. This object is used to render the SQL EXISTS keyword in conjunction with a scalar subquery. The relationship() construct provides for some helper methods that may be used to generate some common EXISTS styles of queries in terms of the relationship.
For a one-to-many relationship such as User.addresses, an EXISTS against the address table that correlates back to the user_account table can be produced using PropComparator.any(). This method accepts an optional WHERE criteria to limit the rows matched by the subquery:
stmt = select(User.fullname).where(
    User.addresses.any(Address.email_address == "squirrel@squirrelpower.org")
)
session.execute(stmt).all()
SELECT user_account.fullname
FROM user_account
WHERE EXISTS (SELECT 1
FROM address
WHERE user_account.id = address.user_id AND address.email_address = ?)
[] ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
As EXISTS tends to be more efficient for negative lookups, a common query is to locate entities where there are no related entities present. This is succinct using a phrase such as ~User.addresses.any(), to select for User entities that have no related Address rows:
stmt = select(User.fullname).where(~User.addresses.any())
session.execute(stmt).all()
SELECT user_account.fullname
FROM user_account
WHERE NOT (EXISTS (SELECT 1
FROM address
WHERE user_account.id = address.user_id))
[] ()
[('Eugene H. Krabs',)]
The PropComparator.has() method works in mostly the same way as PropComparator.any(), except that it’s used for many-to-one relationships, such as if we wanted to locate all Address objects which belonged to “sandy”:
stmt = select(Address.email_address).where(Address.user.has(User.name == "sandy"))
session.execute(stmt).all()
SELECT address.email_address
FROM address
WHERE EXISTS (SELECT 1
FROM user_account
WHERE user_account.id = address.user_id AND user_account.name = ?)
[] ('sandy',)
[('sandy@sqlalchemy.org',), ('squirrel@squirrelpower.org',)]
Relationship Instance Comparison Operators
The relationship()-bound attribute also offers a few SQL construction implementations that are geared towards filtering a relationship()-bound attribute in terms of a specific instance of a related object, which can unpack the appropriate attribute values from a given persistent (or less commonly a detached) object instance and construct WHERE criteria in terms of the target relationship().
* many to one equals comparison - a specific object instance can be compared to many-to-one relationship, to select rows where the foreign key of the target entity matches the primary key value of the object given:
* user_obj = session.get(User, 1)
* SELECT 
* print(select(Address).where(Address.user == user_obj))
* SELECT address.id, address.user_id, address.email_address
* FROM address
* WHERE :param_1 = address.user_id
many to one not equals comparison - the not equals operator may also be used:
print(select(Address).where(Address.user != user_obj))
SELECT address.id, address.user_id, address.email_address
FROM address
WHERE address.user_id != :user_id_1 OR address.user_id IS NULL
object is contained in a one-to-many collection - this is essentially the one-to-many version of the “equals” comparison, select rows where the primary key equals the value of the foreign key in a related object:
address_obj = session.get(Address, 1)
SELECT 
print(select(User).where(User.addresses.contains(address_obj)))
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
WHERE user_account.id = :param_1
An object has a particular parent from a one-to-many perspective - the with_parent() function produces a comparison that returns rows which are referenced by a given parent, this is essentially the same as using the == operator with the many-to-one side:
from sqlalchemy.orm import with_parent
print(select(Address).where(with_parent(user_obj, User.addresses)))
SELECT address.id, address.user_id, address.email_address
FROM address
WHERE :param_1 = address.user_id


Writing SELECT statements for Inheritance Mappings
About this Document
This section makes use of ORM mappings configured using the ORM Inheritance feature, described at Mapping Class Inheritance Hierarchies. The emphasis will be on Joined Table Inheritance as this is the most intricate ORM querying case.
View the ORM setup for this page.
SELECTing from the base class vs. specific sub-classes
A SELECT statement constructed against a class in a joined inheritance hierarchy will query against the table to which the class is mapped, as well as any super-tables present, using JOIN to link them together. The query would then return objects that are of that requested type as well as any sub-types of the requested type, using the discriminator value in each row to determine the correct type. The query below is established against the Manager subclass of Employee, which then returns a result that will contain only objects of type Manager:
from sqlalchemy import select
stmt = select(Manager).order_by(Manager.id)
managers = session.scalars(stmt).all()
BEGIN (implicit)
SELECT manager.id, employee.id AS id_1, employee.name, employee.type, employee.company_id, manager.manager_name
FROM employee JOIN manager ON employee.id = manager.id ORDER BY manager.id
[] ()
print(managers)
[Manager('Mr. Krabs')]
When the SELECT statement is against the base class in the hierarchy, the default behavior is that only that class’ table will be included in the rendered SQL and JOIN will not be used. As in all cases, the discriminator column is used to distinguish between different requested sub-types, which then results in objects of any possible sub-type being returned. The objects returned will have attributes corresponding to the base table populated, and attributes corresponding to sub-tables will start in an un-loaded state, loading automatically when accessed. The loading of sub-attributes is configurable to be more “eager” in a variety of ways, discussed later in this section.
The example below creates a query against the Employee superclass. This indicates that objects of any type, including Manager, Engineer, and Employee, may be within the result set:
from sqlalchemy import select
stmt = select(Employee).order_by(Employee.id)
objects = session.scalars(stmt).all()
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type, employee.company_id
FROM employee ORDER BY employee.id
[] ()
print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
Above, the additional tables for Manager and Engineer were not included in the SELECT, which means that the returned objects will not yet contain data represented from those tables, in this example the .manager_name attribute of the Manager class as well as the .engineer_info attribute of the Engineer class. These attributes start out in the expired state, and will automatically populate themselves when first accessed using lazy loading:
mr_krabs = objects[0]
print(mr_krabs.manager_name)
SELECT manager.manager_name AS manager_manager_name
FROM manager
WHERE ? = manager.id
[] (1,)
Eugene H. Krabs
This lazy load behavior is not desirable if a large number of objects have been loaded, in the case that the consuming application will need to be accessing subclass-specific attributes, as this would be an example of the N plus one problem that emits additional SQL per row. This additional SQL can impact performance and also be incompatible with approaches such as using asyncio. Additionally, in our query for Employee objects, since the query is against the base table only, we did not have a way to add SQL criteria involving subclass-specific attributes in terms of Manager or Engineer. The next two sections detail two constructs that provide solutions to these two issues in different ways, the selectin_polymorphic() loader option and the with_polymorphic() entity construct.
Using selectin_polymorphic()
To address the issue of performance when accessing attributes on subclasses, the selectin_polymorphic() loader strategy may be used to eagerly load these additional attributes up front across many objects at once. This loader option works in a similar fashion as the selectinload() relationship loader strategy to emit an additional SELECT statement against each sub-table for objects loaded in the hierarchy, using IN to query for additional rows based on primary key.
selectin_polymorphic() accepts as its arguments the base entity that is being queried, followed by a sequence of subclasses of that entity for which their specific attributes should be loaded for incoming rows:
from sqlalchemy.orm import selectin_polymorphic
loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
The selectin_polymorphic() construct is then used as a loader option, passing it to the Select.options() method of Select. The example illustrates the use of selectin_polymorphic() to eagerly load columns local to both the Manager and Engineer subclasses:
from sqlalchemy.orm import selectin_polymorphic
loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
stmt = select(Employee).order_by(Employee.id).options(loader_opt)
objects = session.scalars(stmt).all()
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type, employee.company_id
FROM employee ORDER BY employee.id
[] ()
SELECT manager.id AS manager_id, employee.id AS employee_id,
employee.type AS employee_type, manager.manager_name AS manager_manager_name
FROM employee JOIN manager ON employee.id = manager.id
WHERE employee.id IN (?) ORDER BY employee.id
[] (1,)
SELECT engineer.id AS engineer_id, employee.id AS employee_id,
employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info
FROM employee JOIN engineer ON employee.id = engineer.id
WHERE employee.id IN (?, ?) ORDER BY employee.id
[] (2, 3)
print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
The above example illustrates two additional SELECT statements being emitted in order to eagerly fetch additional attributes such as Engineer.engineer_info as well as Manager.manager_name. We can now access these sub-attributes on the objects that were loaded without any additional SQL statements being emitted:
print(objects[0].manager_name)
Eugene H. Krabs
Tip
The selectin_polymorphic() loader option does not yet optimize for the fact that the base employee table does not need to be included in the second two “eager load” queries; hence in the example above we see a JOIN from employee to manager and engineer, even though columns from employee are already loaded. This is in contrast to the selectinload() relationship strategy which is more sophisticated in this regard and can factor out the JOIN when not needed.
Applying selectin_polymorphic() to an existing eager load
In addition to selectin_polymorphic() being specified as an option for a top-level entity loaded by a statement, we may also indicate selectin_polymorphic() on the target of an existing load. As our setup mapping includes a parent Company entity with a Company.employees relationship() referring to Employee entities, we may illustrate a SELECT against the Company entity that eagerly loads all Employee objects as well as all attributes on their subtypes as follows, by applying Load.selectin_polymorphic() as a chained loader option; in this form, the first argument is implicit from the previous loader option (in this case selectinload()), so we only indicate the additional target subclasses we wish to load:
from sqlalchemy.orm import selectinload
stmt = select(Company).options(
    selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
)
for company in session.scalars(stmt):
    print(f"company: {company.name}")
    print(f"employees: {company.employees}")
BEGIN (implicit)
SELECT company.id, company.name
FROM company
[] ()
SELECT employee.company_id AS employee_company_id, employee.id AS employee_id,
employee.name AS employee_name, employee.type AS employee_type
FROM employee
WHERE employee.company_id IN (?)
[] (1,)
SELECT manager.id AS manager_id, employee.id AS employee_id,
employee.type AS employee_type,
manager.manager_name AS manager_manager_name
FROM employee JOIN manager ON employee.id = manager.id
WHERE employee.id IN (?) ORDER BY employee.id
[] (1,)
SELECT engineer.id AS engineer_id, employee.id AS employee_id,
employee.type AS employee_type,
engineer.engineer_info AS engineer_engineer_info
FROM employee JOIN engineer ON employee.id = engineer.id
WHERE employee.id IN (?, ?) ORDER BY employee.id
[] (2, 3)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
See also
Eager Loading of Polymorphic Subtypes - illustrates the equivalent example as above using with_polymorphic() instead
Applying loader options to the subclasses loaded by selectin_polymorphic
The SELECT statements emitted by selectin_polymorphic() are themselves ORM statements, so we may also add other loader options (such as those documented at Relationship Loading Techniques) that refer to specific subclasses. These options should be applied as siblings to a selectin_polymorphic() option, that is, comma separated within select.options().
For example, if we considered that the Manager mapper had a one to many relationship to an entity called Paperwork, we could combine the use of selectin_polymorphic() and selectinload() to eagerly load this collection on all Manager objects, where the sub-attributes of Manager objects were also themselves eagerly loaded:
from sqlalchemy.orm import selectin_polymorphic
stmt = (
    select(Employee)
    .order_by(Employee.id)
    .options(
        selectin_polymorphic(Employee, [Manager, Engineer]),
        selectinload(Manager.paperwork),
    )
)
objects = session.scalars(stmt).all()
SELECT employee.id, employee.name, employee.type, employee.company_id
FROM employee ORDER BY employee.id
[] ()
SELECT manager.id AS manager_id, employee.id AS employee_id, employee.type AS employee_type, manager.manager_name AS manager_manager_name
FROM employee JOIN manager ON employee.id = manager.id
WHERE employee.id IN (?) ORDER BY employee.id
[] (1,)
SELECT paperwork.manager_id AS paperwork_manager_id, paperwork.id AS paperwork_id, paperwork.document_name AS paperwork_document_name
FROM paperwork
WHERE paperwork.manager_id IN (?)
[] (1,)
SELECT engineer.id AS engineer_id, employee.id AS employee_id, employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info
FROM employee JOIN engineer ON employee.id = engineer.id
WHERE employee.id IN (?, ?) ORDER BY employee.id
[] (2, 3)
print(objects[0])
Manager('Mr. Krabs')
print(objects[0].paperwork)
[Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
Applying loader options when selectin_polymorphic is itself a sub-option
Added in version 2.0.21.
The previous section illustrated selectin_polymorphic() and selectinload() used as sibling options, both used within a single call to select.options(). If the target entity is one that is already being loaded from a parent relationship, as in the example at Applying selectin_polymorphic() to an existing eager load, we can apply this “sibling” pattern using the Load.options() method that applies sub-options to a parent, as illustrated at Specifying Sub-Options with Load.options(). Below we combine the two examples to load Company.employees, also loading the attributes for the Manager and Engineer classes, as well as eagerly loading the `Manager.paperwork` attribute:
from sqlalchemy.orm import selectinload
stmt = select(Company).options(
    selectinload(Company.employees).options(
        selectin_polymorphic(Employee, [Manager, Engineer]),
        selectinload(Manager.paperwork),
    )
)
for company in session.scalars(stmt):
    print(f"company: {company.name}")
    for employee in company.employees:
        if isinstance(employee, Manager):
            print(f"manager: {employee.name} paperwork: {employee.paperwork}")
BEGIN (implicit)
SELECT company.id, company.name
FROM company
[] ()
SELECT employee.company_id AS employee_company_id, employee.id AS employee_id, employee.name AS employee_name, employee.type AS employee_type
FROM employee
WHERE employee.company_id IN (?)
[] (1,)
SELECT manager.id AS manager_id, employee.id AS employee_id, employee.type AS employee_type, manager.manager_name AS manager_manager_name
FROM employee JOIN manager ON employee.id = manager.id
WHERE employee.id IN (?) ORDER BY employee.id
[] (1,)
SELECT paperwork.manager_id AS paperwork_manager_id, paperwork.id AS paperwork_id, paperwork.document_name AS paperwork_document_name
FROM paperwork
WHERE paperwork.manager_id IN (?)
[] (1,)
SELECT engineer.id AS engineer_id, employee.id AS employee_id, employee.type AS employee_type, engineer.engineer_info AS engineer_engineer_info
FROM employee JOIN engineer ON employee.id = engineer.id
WHERE employee.id IN (?, ?) ORDER BY employee.id
[] (2, 3)
company: Krusty Krab
manager: Mr. Krabs paperwork: [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
Configuring selectin_polymorphic() on mappers
The behavior of selectin_polymorphic() may be configured on specific mappers so that it takes place by default, by using the Mapper.polymorphic_load parameter, using the value "selectin" on a per-subclass basis. The example below illustrates the use of this parameter within Engineer and Manager subclasses:
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "engineer",
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "manager",
    }
With the above mapping, SELECT statements against the Employee class will automatically assume the use of selectin_polymorphic(Employee, [Engineer, Manager]) as a loader option when the statement is emitted.
Using with_polymorphic()
In contrast to selectin_polymorphic() which affects only the loading of objects, the with_polymorphic() construct affects how the SQL query for a polymorphic structure is rendered, most commonly as a series of LEFT OUTER JOINs to each of the included sub-tables. This join structure is known as the polymorphic selectable. By providing for a view of several sub-tables at once, with_polymorphic() offers a means of writing a SELECT statement across several inherited classes at once with the ability to add filtering criteria based on individual sub-tables.
with_polymorphic() is essentially a special form of the aliased() construct. It accepts as its arguments a similar form to that of selectin_polymorphic(), which is the base entity that is being queried, followed by a sequence of subclasses of that entity for which their specific attributes should be loaded for incoming rows:
from sqlalchemy.orm import with_polymorphic
employee_poly = with_polymorphic(Employee, [Engineer, Manager])
In order to indicate that all subclasses should be part of the entity, with_polymorphic() will also accept the string "*", which may be passed in place of the sequence of classes to indicate all classes (note this is not yet supported by selectin_polymorphic()):
employee_poly = with_polymorphic(Employee, "*")
The example below illustrates the same operation as illustrated in the previous section, to load all columns for Manager and Engineer at once:
stmt = select(employee_poly).order_by(employee_poly.id)
objects = session.scalars(stmt).all()
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type, employee.company_id,
manager.id AS id_1, manager.manager_name, engineer.id AS id_2, engineer.engineer_info
FROM employee
LEFT OUTER JOIN manager ON employee.id = manager.id
LEFT OUTER JOIN engineer ON employee.id = engineer.id ORDER BY employee.id
[] ()
print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
As is the case with selectin_polymorphic(), attributes on subclasses are already loaded:
print(objects[0].manager_name)
Eugene H. Krabs
As the default selectable produced by with_polymorphic() uses LEFT OUTER JOIN, from a database point of view the query is not as well optimized as the approach that selectin_polymorphic() takes, with simple SELECT statements using only JOINs emitted on a per-table basis.
Filtering Subclass Attributes with with_polymorphic()
The with_polymorphic() construct makes available the attributes on the included subclass mappers, by including namespaces that allow references to subclasses. The employee_poly construct created in the previous section includes attributes named .Engineer and .Manager which provide the namespace for Engineer and Manager in terms of the polymorphic SELECT. In the example below, we can use the or_() construct to create criteria against both classes at once:
from sqlalchemy import or_
employee_poly = with_polymorphic(Employee, [Engineer, Manager])
stmt = (
    select(employee_poly)
    .where(
        or_(
            employee_poly.Manager.manager_name == "Eugene H. Krabs",
            employee_poly.Engineer.engineer_info
            == "Senior Customer Engagement Engineer",
        )
    )
    .order_by(employee_poly.id)
)
objects = session.scalars(stmt).all()
SELECT employee.id, employee.name, employee.type, employee.company_id, manager.id AS id_1,
manager.manager_name, engineer.id AS id_2, engineer.engineer_info
FROM employee
LEFT OUTER JOIN manager ON employee.id = manager.id
LEFT OUTER JOIN engineer ON employee.id = engineer.id
WHERE manager.manager_name = ? OR engineer.engineer_info = ?
ORDER BY employee.id
[] ('Eugene H. Krabs', 'Senior Customer Engagement Engineer')
print(objects)
[Manager('Mr. Krabs'), Engineer('Squidward')]
Using aliasing with with_polymorphic
The with_polymorphic() construct, as a special case of aliased(), also provides the basic feature that aliased() does, which is that of “aliasing” of the polymorphic selectable itself. Specifically this means two or more with_polymorphic() entities, referring to the same class hierarchy, can be used at once in a single statement.
To use this feature with a joined inheritance mapping, we typically want to pass two parameters, with_polymorphic.aliased as well as with_polymorphic.flat. The with_polymorphic.aliased parameter indicates that the polymorphic selectable should be referenced by an alias name that is unique to this construct. The with_polymorphic.flat parameter is specific to the default LEFT OUTER JOIN polymorphic selectable and indicates that a more optimized form of aliasing should be used in the statement.
To illustrate this feature, the example below emits a SELECT for two separate polymorphic entities, Employee joined with Engineer, and Employee joined with Manager. Since these two polymorphic entities will both be including the base employee table in their polymorphic selectable, aliasing must be applied in order to differentiate this table in its two different contexts. The two polymorphic entities are treated like two individual tables, and as such typically need to be joined with each other in some way, as illustrated below where the entities are joined on the company_id column along with some additional limiting criteria against the Employee / Manager entity:
manager_employee = with_polymorphic(Employee, [Manager], aliased=True, flat=True)
engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True, flat=True)
stmt = (
    select(manager_employee, engineer_employee)
    .join(
        engineer_employee,
        engineer_employee.company_id == manager_employee.company_id,
    )
    .where(
        or_(
            manager_employee.name == "Mr. Krabs",
            manager_employee.Manager.manager_name == "Eugene H. Krabs",
        )
    )
    .order_by(engineer_employee.name, manager_employee.name)
)
for manager, engineer in session.execute(stmt):
    print(f"{manager} {engineer}")
SELECT
employee_1.id, employee_1.name, employee_1.type, employee_1.company_id,
manager_1.id AS id_1, manager_1.manager_name,
employee_2.id AS id_2, employee_2.name AS name_1, employee_2.type AS type_1,
employee_2.company_id AS company_id_1, engineer_1.id AS id_3, engineer_1.engineer_info
FROM employee AS employee_1
LEFT OUTER JOIN manager AS manager_1 ON employee_1.id = manager_1.id
JOIN
   (employee AS employee_2 LEFT OUTER JOIN engineer AS engineer_1 ON employee_2.id = engineer_1.id)
ON employee_2.company_id = employee_1.company_id
WHERE employee_1.name = ? OR manager_1.manager_name = ?
ORDER BY employee_2.name, employee_1.name
[] ('Mr. Krabs', 'Eugene H. Krabs')
Manager('Mr. Krabs') Manager('Mr. Krabs')
Manager('Mr. Krabs') Engineer('SpongeBob')
Manager('Mr. Krabs') Engineer('Squidward')
In the above example, the behavior of with_polymorphic.flat is that the polymorphic selectables remain as a LEFT OUTER JOIN of their individual tables, which themselves are given anonymous alias names. There is also a right-nested JOIN produced.
When omitting the with_polymorphic.flat parameter, the usual behavior is that each polymorphic selectable is enclosed within a subquery, producing a more verbose form:
manager_employee = with_polymorphic(Employee, [Manager], aliased=True)
engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True)
stmt = (
    select(manager_employee, engineer_employee)
    .join(
        engineer_employee,
        engineer_employee.company_id == manager_employee.company_id,
    )
    .where(
        or_(
            manager_employee.name == "Mr. Krabs",
            manager_employee.Manager.manager_name == "Eugene H. Krabs",
        )
    )
    .order_by(engineer_employee.name, manager_employee.name)
)
print(stmt)
SELECT anon_1.employee_id, anon_1.employee_name, anon_1.employee_type,
anon_1.employee_company_id, anon_1.manager_id, anon_1.manager_manager_name, anon_2.employee_id AS employee_id_1,
anon_2.employee_name AS employee_name_1, anon_2.employee_type AS employee_type_1,
anon_2.employee_company_id AS employee_company_id_1, anon_2.engineer_id, anon_2.engineer_engineer_info
FROM
(SELECT employee.id AS employee_id, employee.name AS employee_name, employee.type AS employee_type,
employee.company_id AS employee_company_id,
manager.id AS manager_id, manager.manager_name AS manager_manager_name
FROM employee LEFT OUTER JOIN manager ON employee.id = manager.id) AS anon_1
JOIN
(SELECT employee.id AS employee_id, employee.name AS employee_name, employee.type AS employee_type,
employee.company_id AS employee_company_id, engineer.id AS engineer_id, engineer.engineer_info AS engineer_engineer_info
FROM employee LEFT OUTER JOIN engineer ON employee.id = engineer.id) AS anon_2
ON anon_2.employee_company_id = anon_1.employee_company_id
WHERE anon_1.employee_name = :employee_name_2 OR anon_1.manager_manager_name = :manager_manager_name_1
ORDER BY anon_2.employee_name, anon_1.employee_name
The above form historically has been more portable to backends that didn’t necessarily have support for right-nested JOINs, and it additionally may be appropriate when the “polymorphic selectable” used by with_polymorphic() is not a simple LEFT OUTER JOIN of tables, as is the case when using mappings such as concrete table inheritance mappings as well as when using alternative polymorphic selectables in general.
Configuring with_polymorphic() on mappers
As is the case with selectin_polymorphic(), the with_polymorphic() construct also supports a mapper-configured version which may be configured in two different ways, either on the base class using the mapper.with_polymorphic parameter, or in a more modern form using the Mapper.polymorphic_load parameter on a per-subclass basis, passing the value "inline".
Warning
For joined inheritance mappings, prefer explicit use of with_polymorphic() within queries, or for implicit eager subclass loading use Mapper.polymorphic_load with "selectin", instead of using the mapper-level mapper.with_polymorphic parameter described in this section. This parameter invokes complex heuristics intended to rewrite the FROM clauses within SELECT statements that can interfere with construction of more complex statements, particularly those with nested subqueries that refer to the same mapped entity.
For example, we may state our Employee mapping using Mapper.polymorphic_load as "inline" as below:
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "engineer",
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "manager",
    }
With the above mapping, SELECT statements against the Employee class will automatically assume the use of with_polymorphic(Employee, [Engineer, Manager]) as the primary entity when the statement is emitted:
print(select(Employee))
SELECT employee.id, employee.name, employee.type, engineer.id AS id_1,
engineer.engineer_info, manager.id AS id_2, manager.manager_name
FROM employee
LEFT OUTER JOIN engineer ON employee.id = engineer.id
LEFT OUTER JOIN manager ON employee.id = manager.id
When using mapper-level “with polymorphic”, queries can also refer to the subclass entities directly, where they implicitly represent the joined tables in the polymorphic query. Above, we can freely refer to Manager and Engineer directly against the default Employee entity:
print(
    select(Employee).where(
        or_(Manager.manager_name == "x", Engineer.engineer_info == "y")
    )
)
SELECT employee.id, employee.name, employee.type, engineer.id AS id_1,
engineer.engineer_info, manager.id AS id_2, manager.manager_name
FROM employee
LEFT OUTER JOIN engineer ON employee.id = engineer.id
LEFT OUTER JOIN manager ON employee.id = manager.id
WHERE manager.manager_name = :manager_name_1
OR engineer.engineer_info = :engineer_info_1
However, if we needed to refer to the Employee entity or its sub entities in separate, aliased contexts, we would again make direct use of with_polymorphic() to define these aliased entities as illustrated in Using aliasing with with_polymorphic.
For more centralized control over the polymorphic selectable, the more legacy form of mapper-level polymorphic control may be used which is the Mapper.with_polymorphic parameter, configured on the base class. This parameter accepts arguments that are comparable to the with_polymorphic() construct, however common use with a joined inheritance mapping is the plain asterisk, indicating all sub-tables should be LEFT OUTER JOINED, as in:
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "with_polymorphic": "*",
        "polymorphic_on": type,
    }


class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }


class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
Overall, the LEFT OUTER JOIN format used by with_polymorphic() and by options such as Mapper.with_polymorphic may be cumbersome from a SQL and database optimizer point of view; for general loading of subclass attributes in joined inheritance mappings, the selectin_polymorphic() approach, or its mapper level equivalent of setting Mapper.polymorphic_load to "selectin" should likely be preferred, making use of with_polymorphic() on a per-query basis only as needed.
Joining to specific sub-types or with_polymorphic() entities
As a with_polymorphic() entity is a special case of aliased(), in order to treat a polymorphic entity as the target of a join, specifically when using a relationship() construct as the ON clause, we use the same technique for regular aliases as detailed at Using Relationship to join between aliased targets, most succinctly using PropComparator.of_type(). In the example below we illustrate a join from the parent Company entity along the one-to-many relationship Company.employees, which is configured in the setup to link to Employee objects, using a with_polymorphic() entity as the target:
employee_plus_engineer = with_polymorphic(Employee, [Engineer])
stmt = (
    select(Company.name, employee_plus_engineer.name)
    .join(Company.employees.of_type(employee_plus_engineer))
    .where(
        or_(
            employee_plus_engineer.name == "SpongeBob",
            employee_plus_engineer.Engineer.engineer_info
            == "Senior Customer Engagement Engineer",
        )
    )
)
for company_name, emp_name in session.execute(stmt):
    print(f"{company_name} {emp_name}")
SELECT company.name, employee.name AS name_1
FROM company JOIN (employee LEFT OUTER JOIN engineer ON employee.id = engineer.id) ON company.id = employee.company_id
WHERE employee.name = ? OR engineer.engineer_info = ?
[] ('SpongeBob', 'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
More directly, PropComparator.of_type() is also used with inheritance mappings of any kind to limit a join along a relationship() to a particular sub-type of the relationship()’s target. The above query could be written strictly in terms of Engineer targets as follows:
stmt = (
    select(Company.name, Engineer.name)
    .join(Company.employees.of_type(Engineer))
    .where(
        or_(
            Engineer.name == "SpongeBob",
            Engineer.engineer_info == "Senior Customer Engagement Engineer",
        )
    )
)
for company_name, emp_name in session.execute(stmt):
    print(f"{company_name} {emp_name}")
SELECT company.name, employee.name AS name_1
FROM company JOIN (employee JOIN engineer ON employee.id = engineer.id) ON company.id = employee.company_id
WHERE employee.name = ? OR engineer.engineer_info = ?
[] ('SpongeBob', 'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
It can be observed above that joining to the Engineer target directly, rather than the “polymorphic selectable” of with_polymorphic(Employee, [Engineer]) has the useful characteristic of using an inner JOIN rather than a LEFT OUTER JOIN, which is generally more performant from a SQL optimizer point of view.
Eager Loading of Polymorphic Subtypes
The use of PropComparator.of_type() illustrated with the Select.join() method in the previous section may also be applied equivalently to relationship loader options, such as selectinload() and joinedload().
As a basic example, if we wished to load Company objects, and additionally eagerly load all elements of Company.employees using the with_polymorphic() construct against the full hierarchy, we may write:
all_employees = with_polymorphic(Employee, "*")
stmt = select(Company).options(selectinload(Company.employees.of_type(all_employees)))
for company in session.scalars(stmt):
    print(f"company: {company.name}")
    print(f"employees: {company.employees}")
SELECT company.id, company.name
FROM company
[] ()
SELECT employee.company_id AS employee_company_id, employee.id AS employee_id,
employee.name AS employee_name, employee.type AS employee_type, manager.id AS manager_id,
manager.manager_name AS manager_manager_name, engineer.id AS engineer_id,
engineer.engineer_info AS engineer_engineer_info
FROM employee
LEFT OUTER JOIN manager ON employee.id = manager.id
LEFT OUTER JOIN engineer ON employee.id = engineer.id
WHERE employee.company_id IN (?)
[] (1,)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
The above query may be compared directly to the selectin_polymorphic() version illustrated in the previous section Applying selectin_polymorphic() to an existing eager load.
See also
Applying selectin_polymorphic() to an existing eager load - illustrates the equivalent example as above using selectin_polymorphic() instead
SELECT Statements for Single Inheritance Mappings
Single Table Inheritance Setup
This section discusses single table inheritance, described at Single Table Inheritance, which uses a single table to represent multiple classes in a hierarchy.
View the ORM setup for this section.
In contrast to joined inheritance mappings, the construction of SELECT statements for single inheritance mappings tends to be simpler since for an all-single-inheritance hierarchy, there’s only one table.
Regardless of whether or not the inheritance hierarchy is all single-inheritance or has a mixture of joined and single inheritance, SELECT statements for single inheritance differentiate queries against the base class vs. a subclass by limiting the SELECT statement with additional WHERE criteria.
As an example, a query for the single-inheritance example mapping of Employee will load objects of type Manager, Engineer and Employee using a simple SELECT of the table:
stmt = select(Employee).order_by(Employee.id)
for obj in session.scalars(stmt):
    print(f"{obj}")
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type
FROM employee ORDER BY employee.id
[] ()
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
When a load is emitted for a specific subclass, additional criteria is added to the SELECT that limits the rows, such as below where a SELECT against the Engineer entity is performed:
stmt = select(Engineer).order_by(Engineer.id)
objects = session.scalars(stmt).all()
SELECT employee.id, employee.name, employee.type, employee.engineer_info
FROM employee
WHERE employee.type IN (?) ORDER BY employee.id
[] ('engineer',)
for obj in objects:
    print(f"{obj}")
Engineer('SpongeBob')
Engineer('Squidward')
Optimizing Attribute Loads for Single Inheritance
The default behavior of single inheritance mappings regarding how attributes on subclasses are SELECTed is similar to that of joined inheritance, in that subclass-specific attributes still emit a second SELECT by default. In the example below, a single Employee of type Manager is loaded, however since the requested class is Employee, the Manager.manager_name attribute is not present by default, and an additional SELECT is emitted when it’s accessed:
mr_krabs = session.scalars(select(Employee).where(Employee.name == "Mr. Krabs")).one()
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type
FROM employee
WHERE employee.name = ?
[] ('Mr. Krabs',)
mr_krabs.manager_name
SELECT employee.manager_name AS employee_manager_name
FROM employee
WHERE employee.id = ? AND employee.type IN (?)
[] (1, 'manager')
'Eugene H. Krabs'
To alter this behavior, the same general concepts used to eagerly load these additional attributes used in joined inheritance loading apply to single inheritance as well, including use of the selectin_polymorphic() option as well as the with_polymorphic() option, the latter of which simply includes the additional columns and from a SQL perspective is more efficient for single-inheritance mappers:
employees = with_polymorphic(Employee, "*")
stmt = select(employees).order_by(employees.id)
objects = session.scalars(stmt).all()
BEGIN (implicit)
SELECT employee.id, employee.name, employee.type,
employee.manager_name, employee.engineer_info
FROM employee ORDER BY employee.id
[] ()
for obj in objects:
    print(f"{obj}")
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
objects[0].manager_name
'Eugene H. Krabs'
Since the overhead of loading single-inheritance subclass mappings is usually minimal, it’s therefore recommended that single inheritance mappings include the Mapper.polymorphic_load parameter with a setting of "inline" for those subclasses where loading of their specific subclass attributes is expected to be common. An example illustrating the setup, modified to include this option, is below:
class Base(DeclarativeBase):
    pass
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    def __repr__(self):
        return f"{self.__class__.__name__}({self.name!r})"

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }
class Manager(Employee):
    manager_name: Mapped[str] = mapped_column(nullable=True)
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "polymorphic_load": "inline",
    }
class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "polymorphic_load": "inline",
    }
With the above mapping, the Manager and Engineer classes will have their columns included in SELECT statements against the Employee entity automatically:
print(select(Employee))
SELECT employee.id, employee.name, employee.type,
employee.manager_name, employee.engineer_info
FROM employee
Inheritance Loading API
Object Name
Description
selectin_polymorphic(base_cls, classes)
Indicate an eager load should take place for all attributes specific to a subclass.
with_polymorphic(base, classes[, selectable, flat, ])
Produce an AliasedClass construct which specifies columns for descendant mappers of the given base.
function sqlalchemy.orm.with_polymorphic(base: Type[_O] | Mapper[_O], classes: Literal['*'] | Iterable[Type[Any]], selectable: Literal[False, None] | FromClause = False, flat: bool = False, polymorphic_on: ColumnElement[Any] | None = None, aliased: bool = False, innerjoin: bool = False, adapt_on_names: bool = False, name: str | None = None, _use_mapper_path: bool = False) ? AliasedClass[_O]
Produce an AliasedClass construct which specifies columns for descendant mappers of the given base.
Using this method will ensure that each descendant mapper’s tables are included in the FROM clause, and will allow filter() criterion to be used against those tables. The resulting instances will also have those columns already loaded so that no “post fetch” of those columns will be required.
See also
Using with_polymorphic() - full discussion of with_polymorphic().
Parameters:
* base – Base class to be aliased.
* classes – a single class or mapper, or list of class/mappers, which inherit from the base class. Alternatively, it may also be the string '*', in which case all descending mapped classes will be added to the FROM clause.
* aliased – when True, the selectable will be aliased. For a JOIN, this means the JOIN will be SELECTed from inside of a subquery unless the with_polymorphic.flat flag is set to True, which is recommended for simpler use cases.
* flat – Boolean, will be passed through to the FromClause.alias() call so that aliases of Join objects will alias the individual tables inside the join, rather than creating a subquery. This is generally supported by all modern databases with regards to right-nested joins and generally produces more efficient queries. Setting this flag is recommended as long as the resulting SQL is functional.
* selectable – 
a table or subquery that will be used in place of the generated FROM clause. This argument is required if any of the desired classes use concrete table inheritance, since SQLAlchemy currently cannot generate UNIONs among tables automatically. If used, the selectable argument must represent the full set of tables and columns mapped by every mapped class. Otherwise, the unaccounted mapped columns will result in their table being appended directly to the FROM clause which will usually lead to incorrect results.
When left at its default value of False, the polymorphic selectable assigned to the base mapper is used for selecting rows. However, it may also be passed as None, which will bypass the configured polymorphic selectable and instead construct an ad-hoc selectable for the target classes given; for joined table inheritance this will be a join that includes all target mappers and their subclasses.
* polymorphic_on – a column to be used as the “discriminator” column for the given selectable. If not given, the polymorphic_on attribute of the base classes’ mapper will be used, if any. This is useful for mappings that don’t have polymorphic loading behavior by default.
* innerjoin – if True, an INNER JOIN will be used. This should only be specified if querying for one specific subtype only
* adapt_on_names – 
Passes through the aliased.adapt_on_names parameter to the aliased object. This may be useful in situations where the given selectable is not directly related to the existing mapped selectable.
Added in version 1.4.33.
* name – 
Name given to the generated AliasedClass.
Added in version 2.0.31.
function sqlalchemy.orm.selectin_polymorphic(base_cls: _EntityType[Any], classes: Iterable[Type[Any]]) ? _AbstractLoad
Indicate an eager load should take place for all attributes specific to a subclass.
This uses an additional SELECT with IN against all matched primary key values, and is the per-query analogue to the "selectin" setting on the mapper.polymorphic_load parameter.
Added in version 1.2.
See also
Using selectin_polymorphic()


ORM-Enabled INSERT, UPDATE, and DELETE statements
About this Document
This section makes use of ORM mappings first illustrated in the SQLAlchemy Unified Tutorial, shown in the section Declaring Mapped Classes, as well as inheritance mappings shown in the section Mapping Class Inheritance Hierarchies.
View the ORM setup for this page.
The Session.execute() method, in addition to handling ORM-enabled Select objects, can also accommodate ORM-enabled Insert, Update and Delete objects, in various ways which are each used to INSERT, UPDATE, or DELETE many database rows at once. There is also dialect-specific support for ORM-enabled “upserts”, which are INSERT statements that automatically make use of UPDATE for rows that already exist.
The following table summarizes the calling forms that are discussed in this document:
ORM Use Case
DML Construct Used
Data is passed using …
Supports RETURNING?
Supports Multi-Table Mappings?
ORM Bulk INSERT Statements
insert()
List of dictionaries to Session.execute.params
yes
yes
ORM Bulk Insert with SQL Expressions
insert()
Session.execute.params with Insert.values()
yes
yes
ORM Bulk Insert with Per Row SQL Expressions
insert()
List of dictionaries to Insert.values()
yes
no
ORM “upsert” Statements
insert()
List of dictionaries to Insert.values()
yes
no
ORM Bulk UPDATE by Primary Key
update()
List of dictionaries to Session.execute.params
no
yes
ORM UPDATE and DELETE with Custom WHERE Criteria
update(), delete()
keywords to Update.values()
yes
partial, with manual steps
ORM Bulk INSERT Statements
A insert() construct can be constructed in terms of an ORM class and passed to the Session.execute() method. A list of parameter dictionaries sent to the Session.execute.params parameter, separate from the Insert object itself, will invoke bulk INSERT mode for the statement, which essentially means the operation will optimize as much as possible for many rows:
from sqlalchemy import insert
session.execute(
    insert(User),
    [
        {"name": "spongebob", "fullname": "Spongebob Squarepants"},
        {"name": "sandy", "fullname": "Sandy Cheeks"},
        {"name": "patrick", "fullname": "Patrick Star"},
        {"name": "squidward", "fullname": "Squidward Tentacles"},
        {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ],
)
INSERT INTO user_account (name, fullname) VALUES (?, ?)
[] [('spongebob', 'Spongebob Squarepants'), ('sandy', 'Sandy Cheeks'), ('patrick', 'Patrick Star'),
('squidward', 'Squidward Tentacles'), ('ehkrabs', 'Eugene H. Krabs')]
<>
The parameter dictionaries contain key/value pairs which may correspond to ORM mapped attributes that line up with mapped Column or mapped_column() declarations, as well as with composite declarations. The keys should match the ORM mapped attribute name and not the actual database column name, if these two names happen to be different.
Changed in version 2.0: Passing an Insert construct to the Session.execute() method now invokes a “bulk insert”, which makes use of the same functionality as the legacy Session.bulk_insert_mappings() method. This is a behavior change compared to the 1.x series where the Insert would be interpreted in a Core-centric way, using column names for value keys; ORM attribute keys are now accepted. Core-style functionality is available by passing the execution option {"dml_strategy": "raw"} to the Session.execution_options parameter of Session.execute().
Getting new objects with RETURNING
The bulk ORM insert feature supports INSERT..RETURNING for selected backends, which can return a Result object that may yield individual columns back as well as fully constructed ORM objects corresponding to the newly generated records. INSERT..RETURNING requires the use of a backend that supports SQL RETURNING syntax as well as support for executemany with RETURNING; this feature is available with all SQLAlchemy-included backends with the exception of MySQL (MariaDB is included).
As an example, we can run the same statement as before, adding use of the UpdateBase.returning() method, passing the full User entity as what we’d like to return. Session.scalars() is used to allow iteration of User objects:
users = session.scalars(
    insert(User).returning(User),
    [
        {"name": "spongebob", "fullname": "Spongebob Squarepants"},
        {"name": "sandy", "fullname": "Sandy Cheeks"},
        {"name": "patrick", "fullname": "Patrick Star"},
        {"name": "squidward", "fullname": "Squidward Tentacles"},
        {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ],
)
INSERT INTO user_account (name, fullname)
VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
RETURNING id, name, fullname, species
[] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
'ehkrabs', 'Eugene H. Krabs')
print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
In the above example, the rendered SQL takes on the form used by the insertmanyvalues feature as requested by the SQLite backend, where individual parameter dictionaries are inlined into a single INSERT statement so that RETURNING may be used.
Changed in version 2.0: The ORM Session now interprets RETURNING clauses from Insert, Update, and even Delete constructs in an ORM context, meaning a mixture of column expressions and ORM mapped entities may be passed to the Insert.returning() method which will then be delivered in the way that ORM results are delivered from constructs such as Select, including that mapped entities will be delivered in the result as ORM mapped objects. Limited support for ORM loader options such as load_only() and selectinload() is also present.
Correlating RETURNING records with input data order
When using bulk INSERT with RETURNING, it’s important to note that most database backends provide no formal guarantee of the order in which the records from RETURNING are returned, including that there is no guarantee that their order will correspond to that of the input records. For applications that need to ensure RETURNING records can be correlated with input data, the additional parameter Insert.returning.sort_by_parameter_order may be specified, which depending on backend may use special INSERT forms that maintain a token which is used to reorder the returned rows appropriately, or in some cases, such as in the example below using the SQLite backend, the operation will INSERT one row at a time:
data = [
    {"name": "pearl", "fullname": "Pearl Krabs"},
    {"name": "plankton", "fullname": "Plankton"},
    {"name": "gary", "fullname": "Gary"},
]
user_ids = session.scalars(
    insert(User).returning(User.id, sort_by_parameter_order=True), data
)
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[(insertmanyvalues) 1/3 (ordered; batch not supported)] ('pearl', 'Pearl Krabs')
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[insertmanyvalues 2/3 (ordered; batch not supported)] ('plankton', 'Plankton')
INSERT INTO user_account (name, fullname) VALUES (?, ?) RETURNING id
[insertmanyvalues 3/3 (ordered; batch not supported)] ('gary', 'Gary')
for user_id, input_record in zip(user_ids, data):
    input_record["id"] = user_id
print(data)
[{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
{'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
{'name': 'gary', 'fullname': 'Gary', 'id': 8}]
Added in version 2.0.10: Added Insert.returning.sort_by_parameter_order which is implemented within the insertmanyvalues architecture.
See also
Correlating RETURNING rows to parameter sets - background on approaches taken to guarantee correspondence between input data and result rows without significant loss of performance
Using Heterogeneous Parameter Dictionaries
The ORM bulk insert feature supports lists of parameter dictionaries that are “heterogeneous”, which basically means “individual dictionaries can have different keys”. When this condition is detected, the ORM will break up the parameter dictionaries into groups corresponding to each set of keys and batch accordingly into separate INSERT statements:
users = session.scalars(
    insert(User).returning(User),
    [
        {
            "name": "spongebob",
            "fullname": "Spongebob Squarepants",
            "species": "Sea Sponge",
        },
        {"name": "sandy", "fullname": "Sandy Cheeks", "species": "Squirrel"},
        {"name": "patrick", "species": "Starfish"},
        {
            "name": "squidward",
            "fullname": "Squidward Tentacles",
            "species": "Squid",
        },
        {"name": "ehkrabs", "fullname": "Eugene H. Krabs", "species": "Crab"},
    ],
)
INSERT INTO user_account (name, fullname, species)
VALUES (?, ?, ?), (?, ?, ?) RETURNING id, name, fullname, species
[(insertmanyvalues) 1/1 (unordered)] ('spongebob', 'Spongebob Squarepants', 'Sea Sponge',
'sandy', 'Sandy Cheeks', 'Squirrel')
INSERT INTO user_account (name, species)
VALUES (?, ?) RETURNING id, name, fullname, species
[] ('patrick', 'Starfish')
INSERT INTO user_account (name, fullname, species)
VALUES (?, ?, ?), (?, ?, ?) RETURNING id, name, fullname, species
[(insertmanyvalues) 1/1 (unordered)] ('squidward', 'Squidward Tentacles',
'Squid', 'ehkrabs', 'Eugene H. Krabs', 'Crab')
In the above example, the five parameter dictionaries passed translated into three INSERT statements, grouped along the specific sets of keys in each dictionary while still maintaining row order, i.e. ("name", "fullname", "species"), ("name", "species"), ("name","fullname", "species").
Sending NULL values in ORM bulk INSERT statements
The bulk ORM insert feature draws upon a behavior that is also present in the legacy “bulk” insert behavior, as well as in the ORM unit of work overall, which is that rows which contain NULL values are INSERTed using a statement that does not refer to those columns; the rationale here is so that backends and schemas which contain server-side INSERT defaults that may be sensitive to the presence of a NULL value vs. no value present will produce a server side value as expected. This default behavior has the effect of breaking up the bulk inserted batches into more batches of fewer rows:
session.execute(
    insert(User),
    [
        {
            "name": "name_a",
            "fullname": "Employee A",
            "species": "Squid",
        },
        {
            "name": "name_b",
            "fullname": "Employee B",
            "species": "Squirrel",
        },
        {
            "name": "name_c",
            "fullname": "Employee C",
            "species": None,
        },
        {
            "name": "name_d",
            "fullname": "Employee D",
            "species": "Bluefish",
        },
    ],
)
INSERT INTO user_account (name, fullname, species) VALUES (?, ?, ?)
[] [('name_a', 'Employee A', 'Squid'), ('name_b', 'Employee B', 'Squirrel')]
INSERT INTO user_account (name, fullname) VALUES (?, ?)
[] ('name_c', 'Employee C')
INSERT INTO user_account (name, fullname, species) VALUES (?, ?, ?)
[] ('name_d', 'Employee D', 'Bluefish')

Above, the bulk INSERT of four rows is broken into three separate statements, the second statement reformatted to not refer to the NULL column for the single parameter dictionary that contains a None value. This default behavior may be undesirable when many rows in the dataset contain random NULL values, as it causes the “executemany” operation to be broken into a larger number of smaller operations; particularly when relying upon insertmanyvalues to reduce the overall number of statements, this can have a bigger performance impact.
To disable the handling of None values in the parameters into separate batches, pass the execution option render_nulls=True; this will cause all parameter dictionaries to be treated equivalently, assuming the same set of keys in each dictionary:
session.execute(
    insert(User).execution_options(render_nulls=True),
    [
        {
            "name": "name_a",
            "fullname": "Employee A",
            "species": "Squid",
        },
        {
            "name": "name_b",
            "fullname": "Employee B",
            "species": "Squirrel",
        },
        {
            "name": "name_c",
            "fullname": "Employee C",
            "species": None,
        },
        {
            "name": "name_d",
            "fullname": "Employee D",
            "species": "Bluefish",
        },
    ],
)
INSERT INTO user_account (name, fullname, species) VALUES (?, ?, ?)
[] [('name_a', 'Employee A', 'Squid'), ('name_b', 'Employee B', 'Squirrel'), ('name_c', 'Employee C', None), ('name_d', 'Employee D', 'Bluefish')]

Above, all parameter dictionaries are sent in a single INSERT batch, including the None value present in the third parameter dictionary.
Added in version 2.0.23: Added the render_nulls execution option which mirrors the behavior of the legacy Session.bulk_insert_mappings.render_nulls parameter.
Bulk INSERT for Joined Table Inheritance
ORM bulk insert builds upon the internal system that is used by the traditional unit of work system in order to emit INSERT statements. This means that for an ORM entity that is mapped to multiple tables, typically one which is mapped using joined table inheritance, the bulk INSERT operation will emit an INSERT statement for each table represented by the mapping, correctly transferring server-generated primary key values to the table rows that depend upon them. The RETURNING feature is also supported here, where the ORM will receive Result objects for each INSERT statement executed, and will then “horizontally splice” them together so that the returned rows include values for all columns inserted:
managers = session.scalars(
    insert(Manager).returning(Manager),
    [
        {"name": "sandy", "manager_name": "Sandy Cheeks"},
        {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
    ],
)
INSERT INTO employee (name, type) VALUES (?, ?) RETURNING id, name, type
[(insertmanyvalues) 1/2 (ordered; batch not supported)] ('sandy', 'manager')
INSERT INTO employee (name, type) VALUES (?, ?) RETURNING id, name, type
[insertmanyvalues 2/2 (ordered; batch not supported)] ('ehkrabs', 'manager')
INSERT INTO manager (id, manager_name) VALUES (?, ?), (?, ?) RETURNING id, manager_name, id AS id__1
[(insertmanyvalues) 1/1 (ordered)] (1, 'Sandy Cheeks', 2, 'Eugene H. Krabs')
Tip
Bulk INSERT of joined inheritance mappings requires that the ORM make use of the Insert.returning.sort_by_parameter_order parameter internally, so that it can correlate primary key values from RETURNING rows from the base table into the parameter sets being used to INSERT into the “sub” table, which is why the SQLite backend illustrated above transparently degrades to using non-batched statements. Background on this feature is at Correlating RETURNING rows to parameter sets.
ORM Bulk Insert with SQL Expressions
The ORM bulk insert feature supports the addition of a fixed set of parameters which may include SQL expressions to be applied to every target row. To achieve this, combine the use of the Insert.values() method, passing a dictionary of parameters that will be applied to all rows, with the usual bulk calling form by including a list of parameter dictionaries that contain individual row values when invoking Session.execute().
As an example, given an ORM mapping that includes a “timestamp” column:
import datetime


class LogRecord(Base):
    __tablename__ = "log_record"
    id: Mapped[int] = mapped_column(primary_key=True)
    message: Mapped[str]
    code: Mapped[str]
    timestamp: Mapped[datetime.datetime]
If we wanted to INSERT a series of LogRecord elements, each with a unique message field, however we would like to apply the SQL function now() to all rows, we can pass timestamp within Insert.values() and then pass the additional records using “bulk” mode:
from sqlalchemy import func
log_record_result = session.scalars(
    insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
    [
        {"message": "log message #1"},
        {"message": "log message #2"},
        {"message": "log message #3"},
        {"message": "log message #4"},
    ],
)
INSERT INTO log_record (message, code, timestamp)
VALUES (?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP),
(?, ?, CURRENT_TIMESTAMP), (?, ?, CURRENT_TIMESTAMP)
RETURNING id, message, code, timestamp
[(insertmanyvalues) 1/1 (unordered)] ('log message #1', 'SQLA', 'log message #2',
'SQLA', 'log message #3', 'SQLA', 'log message #4', 'SQLA')
print(log_record_result.all())
[LogRecord('log message #1', 'SQLA', datetime.datetime()),
 LogRecord('log message #2', 'SQLA', datetime.datetime()),
 LogRecord('log message #3', 'SQLA', datetime.datetime()),
 LogRecord('log message #4', 'SQLA', datetime.datetime())]
ORM Bulk Insert with Per Row SQL Expressions
The Insert.values() method itself accommodates a list of parameter dictionaries directly. When using the Insert construct in this way, without passing any list of parameter dictionaries to the Session.execute.params parameter, bulk ORM insert mode is not used, and instead the INSERT statement is rendered exactly as given and invoked exactly once. This mode of operation may be useful both for the case of passing SQL expressions on a per-row basis, and is also used when using “upsert” statements with the ORM, documented later in this chapter at ORM “upsert” Statements.
A contrived example of an INSERT that embeds per-row SQL expressions, and also demonstrates Insert.returning() in this form, is below:
from sqlalchemy import select
address_result = session.scalars(
    insert(Address)
    .values(
        [
            {
                "user_id": select(User.id).where(User.name == "sandy"),
                "email_address": "sandy@company.com",
            },
            {
                "user_id": select(User.id).where(User.name == "spongebob"),
                "email_address": "spongebob@company.com",
            },
            {
                "user_id": select(User.id).where(User.name == "patrick"),
                "email_address": "patrick@company.com",
            },
        ]
    )
    .returning(Address),
)
INSERT INTO address (user_id, email_address) VALUES
((SELECT user_account.id
FROM user_account
WHERE user_account.name = ?), ?), ((SELECT user_account.id
FROM user_account
WHERE user_account.name = ?), ?), ((SELECT user_account.id
FROM user_account
WHERE user_account.name = ?), ?) RETURNING id, user_id, email_address
[] ('sandy', 'sandy@company.com', 'spongebob', 'spongebob@company.com',
'patrick', 'patrick@company.com')
print(address_result.all())
[Address(email_address='sandy@company.com'),
 Address(email_address='spongebob@company.com'),
 Address(email_address='patrick@company.com')]
Because bulk ORM insert mode is not used above, the following features are not present:
* Joined table inheritance or other multi-table mappings are not supported, since that would require multiple INSERT statements.
* Heterogeneous parameter sets are not supported - each element in the VALUES set must have the same columns.
* Core-level scale optimizations such as the batching provided by insertmanyvalues are not available; statements will need to ensure the total number of parameters does not exceed limits imposed by the backing database.
For the above reasons, it is generally not recommended to use multiple parameter sets with Insert.values() with ORM INSERT statements unless there is a clear rationale, which is either that “upsert” is being used or there is a need to embed per-row SQL expressions in each parameter set.
See also
ORM “upsert” Statements
Legacy Session Bulk INSERT Methods
The Session includes legacy methods for performing “bulk” INSERT and UPDATE statements. These methods share implementations with the SQLAlchemy 2.0 versions of these features, described at ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key, however lack many features, namely RETURNING support as well as support for session-synchronization.
Code which makes use of Session.bulk_insert_mappings() for example can port code as follows, starting with this mappings example:
session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
The above is expressed using the new API as:
from sqlalchemy import insert

session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
See also
Legacy Session Bulk UPDATE Methods
ORM “upsert” Statements
Selected backends with SQLAlchemy may include dialect-specific Insert constructs which additionally have the ability to perform “upserts”, or INSERTs where an existing row in the parameter set is turned into an approximation of an UPDATE statement instead. By “existing row” , this may mean rows which share the same primary key value, or may refer to other indexed columns within the row that are considered to be unique; this is dependent on the capabilities of the backend in use.
The dialects included with SQLAlchemy that include dialect-specific “upsert” API features are:
* SQLite - using Insert documented at INSERT…ON CONFLICT (Upsert)
* PostgreSQL - using Insert documented at INSERT…ON CONFLICT (Upsert)
* MySQL/MariaDB - using Insert documented at INSERT…ON DUPLICATE KEY UPDATE (Upsert)
Users should review the above sections for background on proper construction of these objects; in particular, the “upsert” method typically needs to refer back to the original statement, so the statement is usually constructed in two separate steps.
Third party backends such as those mentioned at External Dialects may also feature similar constructs.
While SQLAlchemy does not yet have a backend-agnostic upsert construct, the above Insert variants are nonetheless ORM compatible in that they may be used in the same way as the Insert construct itself as documented at ORM Bulk Insert with Per Row SQL Expressions, that is, by embedding the desired rows to INSERT within the Insert.values() method. In the example below, the SQLite insert() function is used to generate an Insert construct that includes “ON CONFLICT DO UPDATE” support. The statement is then passed to Session.execute() where it proceeds normally, with the additional characteristic that the parameter dictionaries passed to Insert.values() are interpreted as ORM mapped attribute keys, rather than column names:
from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
stmt = sqlite_upsert(User).values(
    [
        {"name": "spongebob", "fullname": "Spongebob Squarepants"},
        {"name": "sandy", "fullname": "Sandy Cheeks"},
        {"name": "patrick", "fullname": "Patrick Star"},
        {"name": "squidward", "fullname": "Squidward Tentacles"},
        {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
    ]
)
stmt = stmt.on_conflict_do_update(
    index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
)
session.execute(stmt)
INSERT INTO user_account (name, fullname)
VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
ON CONFLICT (name) DO UPDATE SET fullname = excluded.fullname
[] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
'ehkrabs', 'Eugene H. Krabs')
<>
Using RETURNING with upsert statements
From the SQLAlchemy ORM’s point of view, upsert statements look like regular Insert constructs, which includes that Insert.returning() works with upsert statements in the same way as was demonstrated at ORM Bulk Insert with Per Row SQL Expressions, so that any column expression or relevant ORM entity class may be passed. Continuing from the example in the previous section:
result = session.scalars(
    stmt.returning(User), execution_options={"populate_existing": True}
)
INSERT INTO user_account (name, fullname)
VALUES (?, ?), (?, ?), (?, ?), (?, ?), (?, ?)
ON CONFLICT (name) DO UPDATE SET fullname = excluded.fullname
RETURNING id, name, fullname, species
[] ('spongebob', 'Spongebob Squarepants', 'sandy', 'Sandy Cheeks',
'patrick', 'Patrick Star', 'squidward', 'Squidward Tentacles',
'ehkrabs', 'Eugene H. Krabs')
print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
  User(name='sandy', fullname='Sandy Cheeks'),
  User(name='patrick', fullname='Patrick Star'),
  User(name='squidward', fullname='Squidward Tentacles'),
  User(name='ehkrabs', fullname='Eugene H. Krabs')]
The example above uses RETURNING to return ORM objects for each row inserted or upserted by the statement. The example also adds use of the Populate Existing execution option. This option indicates that User objects which are already present in the Session for rows that already exist should be refreshed with the data from the new row. For a pure Insert statement, this option is not significant, because every row produced is a brand new primary key identity. However when the Insert also includes “upsert” options, it may also be yielding results from rows that already exist and therefore may already have a primary key identity represented in the Session object’s identity map.
See also
Populate Existing
ORM Bulk UPDATE by Primary Key
The Update construct may be used with Session.execute() in a similar way as the Insert statement is used as described at ORM Bulk INSERT Statements, passing a list of many parameter dictionaries, each dictionary representing an individual row that corresponds to a single primary key value. This use should not be confused with a more common way to use Update statements with the ORM, using an explicit WHERE clause, which is documented at ORM UPDATE and DELETE with Custom WHERE Criteria.
For the “bulk” version of UPDATE, a update() construct is made in terms of an ORM class and passed to the Session.execute() method; the resulting Update object should have no values and typically no WHERE criteria, that is, the Update.values() method is not used, and the Update.where() is usually not used, but may be used in the unusual case that additional filtering criteria would be added.
Passing the Update construct along with a list of parameter dictionaries which each include a full primary key value will invoke bulk UPDATE by primary key mode for the statement, generating the appropriate WHERE criteria to match each row by primary key, and using executemany to run each parameter set against the UPDATE statement:
from sqlalchemy import update
session.execute(
    update(User),
    [
        {"id": 1, "fullname": "Spongebob Squarepants"},
        {"id": 3, "fullname": "Patrick Star"},
        {"id": 5, "fullname": "Eugene H. Krabs"},
    ],
)
UPDATE user_account SET fullname=? WHERE user_account.id = ?
[] [('Spongebob Squarepants', 1), ('Patrick Star', 3), ('Eugene H. Krabs', 5)]
<>
Note that each parameter dictionary must include a full primary key for each record, else an error is raised.
Like the bulk INSERT feature, heterogeneous parameter lists are supported here as well, where the parameters will be grouped into sub-batches of UPDATE runs.
Changed in version 2.0.11: Additional WHERE criteria can be combined with ORM Bulk UPDATE by Primary Key by using the Update.where() method to add additional criteria. However this criteria is always in addition to the WHERE criteria that’s already made present which includes primary key values.
The RETURNING feature is not available when using the “bulk UPDATE by primary key” feature; the list of multiple parameter dictionaries necessarily makes use of DBAPI executemany, which in its usual form does not typically support result rows.
Changed in version 2.0: Passing an Update construct to the Session.execute() method along with a list of parameter dictionaries now invokes a “bulk update”, which makes use of the same functionality as the legacy Session.bulk_update_mappings() method. This is a behavior change compared to the 1.x series where the Update would only be supported with explicit WHERE criteria and inline VALUES.
Disabling Bulk ORM Update by Primary Key for an UPDATE statement with multiple parameter sets
The ORM Bulk Update by Primary Key feature, which runs an UPDATE statement per record which includes WHERE criteria for each primary key value, is automatically used when:
1. the UPDATE statement given is against an ORM entity
2. the Session is used to execute the statement, and not a Core Connection
3. The parameters passed are a list of dictionaries.
In order to invoke an UPDATE statement without using “ORM Bulk Update by Primary Key”, invoke the statement against the Connection directly using the Session.connection() method to acquire the current Connection for the transaction:
from sqlalchemy import bindparam
session.connection().execute(
    update(User).where(User.name == bindparam("u_name")),
    [
        {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
        {"u_name": "patrick", "fullname": "Patrick Star"},
    ],
)
UPDATE user_account SET fullname=? WHERE user_account.name = ?
[] [('Spongebob Squarepants', 'spongebob'), ('Patrick Star', 'patrick')]
<>
See also
per-row ORM Bulk Update by Primary Key requires that records contain primary key values
Bulk UPDATE by Primary Key for Joined Table Inheritance
ORM bulk update has similar behavior to ORM bulk insert when using mappings with joined table inheritance; as described at Bulk INSERT for Joined Table Inheritance, the bulk UPDATE operation will emit an UPDATE statement for each table represented in the mapping, for which the given parameters include values to be updated (non-affected tables are skipped).
Example:
session.execute(
    update(Manager),
    [
        {
            "id": 1,
            "name": "scheeks",
            "manager_name": "Sandy Cheeks, President",
        },
        {
            "id": 2,
            "name": "eugene",
            "manager_name": "Eugene H. Krabs, VP Marketing",
        },
    ],
)
UPDATE employee SET name=? WHERE employee.id = ?
[] [('scheeks', 1), ('eugene', 2)]
UPDATE manager SET manager_name=? WHERE manager.id = ?
[] [('Sandy Cheeks, President', 1), ('Eugene H. Krabs, VP Marketing', 2)]
<>
Legacy Session Bulk UPDATE Methods
As discussed at Legacy Session Bulk INSERT Methods, the Session.bulk_update_mappings() method of Session is the legacy form of bulk update, which the ORM makes use of internally when interpreting a update() statement with primary key parameters given; however, when using the legacy version, features such as support for session-synchronization are not included.
The example below:
session.bulk_update_mappings(
    User,
    [
        {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
        {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
    ],
)
Is expressed using the new API as:
from sqlalchemy import update

session.execute(
    update(User),
    [
        {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
        {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
    ],
)
See also
Legacy Session Bulk INSERT Methods
ORM UPDATE and DELETE with Custom WHERE Criteria
The Update and Delete constructs, when constructed with custom WHERE criteria (that is, using the Update.where() and Delete.where() methods), may be invoked in an ORM context by passing them to Session.execute(), without using the Session.execute.params parameter. For Update, the values to be updated should be passed using Update.values().
This mode of use differs from the feature described previously at ORM Bulk UPDATE by Primary Key in that the ORM uses the given WHERE clause as is, rather than fixing the WHERE clause to be by primary key. This means that the single UPDATE or DELETE statement can affect many rows at once.
As an example, below an UPDATE is emitted that affects the “fullname” field of multiple rows
from sqlalchemy import update
stmt = (
    update(User)
    .where(User.name.in_(["squidward", "sandy"]))
    .values(fullname="Name starts with S")
)
session.execute(stmt)
UPDATE user_account SET fullname=? WHERE user_account.name IN (?, ?)
[] ('Name starts with S', 'squidward', 'sandy')
<>
For a DELETE, an example of deleting rows based on criteria:
from sqlalchemy import delete
stmt = delete(User).where(User.name.in_(["squidward", "sandy"]))
session.execute(stmt)
DELETE FROM user_account WHERE user_account.name IN (?, ?)
[] ('squidward', 'sandy')
<>
Warning
Please read the following section Important Notes and Caveats for ORM-Enabled Update and Delete for important notes regarding how the functionality of ORM-Enabled UPDATE and DELETE diverges from that of ORM unit of work features, such as using the Session.delete() method to delete individual objects.
Important Notes and Caveats for ORM-Enabled Update and Delete
The ORM-enabled UPDATE and DELETE features bypass ORM unit of work automation in favor of being able to emit a single UPDATE or DELETE statement that matches multiple rows at once without complexity.
* The operations do not offer in-Python cascading of relationships - it is assumed that ON UPDATE CASCADE and/or ON DELETE CASCADE is configured for any foreign key references which require it, otherwise the database may emit an integrity violation if foreign key references are being enforced. See the notes at Using foreign key ON DELETE cascade with ORM relationships for some examples.
* After the UPDATE or DELETE, dependent objects in the Session which were impacted by an ON UPDATE CASCADE or ON DELETE CASCADE on related tables, particularly objects that refer to rows that have now been deleted, may still reference those objects. This issue is resolved once the Session is expired, which normally occurs upon Session.commit() or can be forced by using Session.expire_all().
* ORM-enabled UPDATEs and DELETEs do not handle joined table inheritance automatically. See the section UPDATE/DELETE with Custom WHERE Criteria for Joined Table Inheritance for notes on how to work with joined-inheritance mappings.
* The WHERE criteria needed in order to limit the polymorphic identity to specific subclasses for single-table-inheritance mappings is included automatically . This only applies to a subclass mapper that has no table of its own.
* The with_loader_criteria() option is supported by ORM update and delete operations; criteria here will be added to that of the UPDATE or DELETE statement being emitted, as well as taken into account during the “synchronize” process.
* In order to intercept ORM-enabled UPDATE and DELETE operations with event handlers, use the SessionEvents.do_orm_execute() event.
Selecting a Synchronization Strategy
When making use of update() or delete() in conjunction with ORM-enabled execution using Session.execute(), additional ORM-specific functionality is present which will synchronize the state being changed by the statement with that of the objects that are currently present within the identity map of the Session. By “synchronize” we mean that UPDATEd attributes will be refreshed with the new value, or at the very least expired so that they will re-populate with their new value on next access, and DELETEd objects will be moved into the deleted state.
This synchronization is controllable as the “synchronization strategy”, which is passed as an string ORM execution option, typically by using the Session.execute.execution_options dictionary:
from sqlalchemy import update
stmt = (
    update(User).where(User.name == "squidward").values(fullname="Squidward Tentacles")
)
session.execute(stmt, execution_options={"synchronize_session": False})
UPDATE user_account SET fullname=? WHERE user_account.name = ?
[] ('Squidward Tentacles', 'squidward')
<>
The execution option may also be bundled with the statement itself using the Executable.execution_options() method:
from sqlalchemy import update
stmt = (
    update(User)
    .where(User.name == "squidward")
    .values(fullname="Squidward Tentacles")
    .execution_options(synchronize_session=False)
)
session.execute(stmt)
UPDATE user_account SET fullname=? WHERE user_account.name = ?
[] ('Squidward Tentacles', 'squidward')
<>
The following values for synchronize_session are supported:
* 'auto' - this is the default. The 'fetch' strategy will be used on backends that support RETURNING, which includes all SQLAlchemy-native drivers except for MySQL. If RETURNING is not supported, the 'evaluate' strategy will be used instead.
* 'fetch' - Retrieves the primary key identity of affected rows by either performing a SELECT before the UPDATE or DELETE, or by using RETURNING if the database supports it, so that in-memory objects which are affected by the operation can be refreshed with new values (updates) or expunged from the Session (deletes). This synchronization strategy may be used even if the given update() or delete() construct explicitly specifies entities or columns using UpdateBase.returning().
Changed in version 2.0: Explicit UpdateBase.returning() may be combined with the 'fetch' synchronization strategy when using ORM-enabled UPDATE and DELETE with WHERE criteria. The actual statement will contain the union of columns between that which the 'fetch' strategy requires and those which were requested.
* 'evaluate' - This indicates to evaluate the WHERE criteria given in the UPDATE or DELETE statement in Python, to locate matching objects within the Session. This approach does not add any SQL round trips to the operation, and in the absence of RETURNING support, may be more efficient. For UPDATE or DELETE statements with complex criteria, the 'evaluate' strategy may not be able to evaluate the expression in Python and will raise an error. If this occurs, use the 'fetch' strategy for the operation instead.
Tip
If a SQL expression makes use of custom operators using the Operators.op() or custom_op feature, the Operators.op.python_impl parameter may be used to indicate a Python function that will be used by the "evaluate" synchronization strategy.
Added in version 2.0.
Warning
The "evaluate" strategy should be avoided if an UPDATE operation is to run on a Session that has many objects which have been expired, because it will necessarily need to refresh objects in order to test them against the given WHERE criteria, which will emit a SELECT for each one. In this case, and particularly if the backend supports RETURNING, the "fetch" strategy should be preferred.
* False - don’t synchronize the session. This option may be useful for backends that don’t support RETURNING where the "evaluate" strategy is not able to be used. In this case, the state of objects in the Session is unchanged and will not automatically correspond to the UPDATE or DELETE statement that was emitted, if such objects that would normally correspond to the rows matched are present.
Using RETURNING with UPDATE/DELETE and Custom WHERE Criteria
The UpdateBase.returning() method is fully compatible with ORM-enabled UPDATE and DELETE with WHERE criteria. Full ORM objects and/or columns may be indicated for RETURNING:
from sqlalchemy import update
stmt = (
    update(User)
    .where(User.name == "squidward")
    .values(fullname="Squidward Tentacles")
    .returning(User)
)
result = session.scalars(stmt)
UPDATE user_account SET fullname=? WHERE user_account.name = ?
RETURNING id, name, fullname, species
[] ('Squidward Tentacles', 'squidward')
print(result.all())
[User(name='squidward', fullname='Squidward Tentacles')]
The support for RETURNING is also compatible with the fetch synchronization strategy, which also uses RETURNING. The ORM will organize the columns in RETURNING appropriately so that the synchronization proceeds as well as that the returned Result will contain the requested entities and SQL columns in their requested order.
Added in version 2.0: UpdateBase.returning() may be used for ORM enabled UPDATE and DELETE while still retaining full compatibility with the fetch synchronization strategy.
UPDATE/DELETE with Custom WHERE Criteria for Joined Table Inheritance
The UPDATE/DELETE with WHERE criteria feature, unlike the ORM Bulk UPDATE by Primary Key, only emits a single UPDATE or DELETE statement per call to Session.execute(). This means that when running an update() or delete() statement against a multi-table mapping, such as a subclass in a joined-table inheritance mapping, the statement must conform to the backend’s current capabilities, which may include that the backend does not support an UPDATE or DELETE statement that refers to multiple tables, or may have only limited support for this. This means that for mappings such as joined inheritance subclasses, the ORM version of the UPDATE/DELETE with WHERE criteria feature can only be used to a limited extent or not at all, depending on specifics.
The most straightforward way to emit a multi-row UPDATE statement for a joined-table subclass is to refer to the sub-table alone. This means the Update() construct should only refer to attributes that are local to the subclass table, as in the example below:
stmt = (
    update(Manager)
    .where(Manager.id == 1)
    .values(manager_name="Sandy Cheeks, President")
)
session.execute(stmt)
UPDATE manager SET manager_name=? WHERE manager.id = ?
[] ('Sandy Cheeks, President', 1)
<>
With the above form, a rudimentary way to refer to the base table in order to locate rows which will work on any SQL backend is so use a subquery:
stmt = (
    update(Manager)
    .where(
        Manager.id
        == select(Employee.id).where(Employee.name == "sandy").scalar_subquery()
    )
    .values(manager_name="Sandy Cheeks, President")
)
session.execute(stmt)
UPDATE manager SET manager_name=? WHERE manager.id = (SELECT employee.id
FROM employee
WHERE employee.name = ?) RETURNING id
[] ('Sandy Cheeks, President', 'sandy')
<>
For backends that support UPDATE…FROM, the subquery may be stated instead as additional plain WHERE criteria, however the criteria between the two tables must be stated explicitly in some way:
stmt = (
    update(Manager)
    .where(Manager.id == Employee.id, Employee.name == "sandy")
    .values(manager_name="Sandy Cheeks, President")
)
session.execute(stmt)
UPDATE manager SET manager_name=? FROM employee
WHERE manager.id = employee.id AND employee.name = ?
[] ('Sandy Cheeks, President', 'sandy')
<>
For a DELETE, it’s expected that rows in both the base table and the sub-table would be DELETEd at the same time. To DELETE many rows of joined inheritance objects without using cascading foreign keys, emit DELETE for each table individually:
from sqlalchemy import delete
session.execute(delete(Manager).where(Manager.id == 1))
DELETE FROM manager WHERE manager.id = ?
[] (1,)
<>
session.execute(delete(Employee).where(Employee.id == 1))
DELETE FROM employee WHERE employee.id = ?
[] (1,)
<>
Overall, normal unit of work processes should be preferred for updating and deleting rows for joined inheritance and other multi-table mappings, unless there is a performance rationale for using custom WHERE criteria.
Legacy Query Methods
The ORM enabled UPDATE/DELETE with WHERE feature was originally part of the now-legacy Query object, in the Query.update() and Query.delete() methods. These methods remain available and provide a subset of the same functionality as that described at ORM UPDATE and DELETE with Custom WHERE Criteria. The primary difference is that the legacy methods don’t provide for explicit RETURNING support.
See also
Query.update()
Query.delete()


Column Loading Options
About this Document
This section presents additional options regarding the loading of columns. The mappings used include columns that would store large string values for which we may want to limit when they are loaded.
View the ORM setup for this page. Some of the examples below will redefine the Book mapper to modify some of the column definitions.
Limiting which Columns Load with Column Deferral
Column deferral refers to ORM mapped columns that are omitted from a SELECT statement when objects of that type are queried. The general rationale here is performance, in cases where tables have seldom-used columns with potentially large data values, as fully loading these columns on every query may be time and/or memory intensive. SQLAlchemy ORM offers a variety of ways to control the loading of columns when entities are loaded.
Most examples in this section are illustrating ORM loader options. These are small constructs that are passed to the Select.options() method of the Select object, which are then consumed by the ORM when the object is compiled into a SQL string.
Using load_only() to reduce loaded columns
The load_only() loader option is the most expedient option to use when loading objects where it is known that only a small handful of columns will be accessed. This option accepts a variable number of class-bound attribute objects indicating those column-mapped attributes that should be loaded, where all other column-mapped attributes outside of the primary key will not be part of the columns fetched . In the example below, the Book class contains columns .title, .summary and .cover_photo. Using load_only() we can instruct the ORM to only load the .title and .summary columns up front:
from sqlalchemy import select
from sqlalchemy.orm import load_only
stmt = select(Book).options(load_only(Book.title, Book.summary))
books = session.scalars(stmt).all()
SELECT book.id, book.title, book.summary
FROM book
[] ()
for book in books:
    print(f"{book.title}  {book.summary}")
100 Years of Krabby Patties  some long summary
Sea Catch 22  another long summary
The Sea Grapes of Wrath  yet another summary
A Nut Like No Other  some long summary
Geodesic Domes: A Retrospective  another long summary
Rocketry for Squirrels  yet another summary
Above, the SELECT statement has omitted the .cover_photo column and included only .title and .summary, as well as the primary key column .id; the ORM will typically always fetch the primary key columns as these are required to establish the identity for the row.
Once loaded, the object will normally have lazy loading behavior applied to the remaining unloaded attributes, meaning that when any are first accessed, a SQL statement will be emitted within the current transaction in order to load the value. Below, accessing .cover_photo emits a SELECT statement to load its value:
img_data = books[0].cover_photo
SELECT book.cover_photo AS book_cover_photo
FROM book
WHERE book.id = ?
[] (1,)
Lazy loads are always emitted using the Session to which the object is in the persistent state. If the object is detached from any Session, the operation fails, raising an exception.
As an alternative to lazy loading on access, deferred columns may also be configured to raise an informative exception when accessed, regardless of their attachment state. When using the load_only() construct, this may be indicated using the load_only.raiseload parameter. See the section Using raiseload to prevent deferred column loads for background and examples.
Tip
as noted elsewhere, lazy loading is not available when using Asynchronous I/O (asyncio).
Using load_only() with multiple entities
load_only() limits itself to the single entity that is referred towards in its list of attributes (passing a list of attributes that span more than a single entity is currently disallowed). In the example below, the given load_only() option applies only to the Book entity. The User entity that’s also selected is not affected; within the resulting SELECT statement, all columns for user_account are present, whereas only book.id and book.title are present for the book table:
stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
print(stmt)
SELECT user_account.id, user_account.name, user_account.fullname,
book.id AS id_1, book.title
FROM user_account JOIN book ON user_account.id = book.owner_id
If we wanted to apply load_only() options to both User and Book, we would make use of two separate options:
stmt = (
    select(User, Book)
    .join_from(User, Book)
    .options(load_only(User.name), load_only(Book.title))
)
print(stmt)
SELECT user_account.id, user_account.name, book.id AS id_1, book.title
FROM user_account JOIN book ON user_account.id = book.owner_id
Using load_only() on related objects and collections
When using relationship loaders to control the loading of related objects, the Load.load_only() method of any relationship loader may be used to apply load_only() rules to columns on the sub-entity. In the example below, selectinload() is used to load the related books collection on each User object. By applying Load.load_only() to the resulting option object, when objects are loaded for the relationship, the SELECT emitted will only refer to the title column in addition to primary key column:
from sqlalchemy.orm import selectinload
stmt = select(User).options(selectinload(User.books).load_only(Book.title))
for user in session.scalars(stmt):
    print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
[] ()
SELECT book.owner_id AS book_owner_id, book.id AS book_id, book.title AS book_title
FROM book
WHERE book.owner_id IN (?, ?)
[] (1, 2)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
load_only() may also be applied to sub-entities without needing to state the style of loading to use for the relationship itself. If we didn’t want to change the default loading style of User.books but still apply load only rules to Book, we would link using the defaultload() option, which in this case will retain the default relationship loading style of "lazy", and applying our custom load_only() rule to the SELECT statement emitted for each User.books collection:
from sqlalchemy.orm import defaultload
stmt = select(User).options(defaultload(User.books).load_only(Book.title))
for user in session.scalars(stmt):
    print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
[] ()
SELECT book.id AS book_id, book.title AS book_title
FROM book
WHERE ? = book.owner_id
[] (1,)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
SELECT book.id AS book_id, book.title AS book_title
FROM book
WHERE ? = book.owner_id
[] (2,)
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
Using defer() to omit specific columns
The defer() loader option is a more fine grained alternative to load_only(), which allows a single specific column to be marked as “dont load”. In the example below, defer() is applied directly to the .cover_photo column, leaving the behavior of all other columns unchanged:
from sqlalchemy.orm import defer
stmt = select(Book).where(Book.owner_id == 2).options(defer(Book.cover_photo))
books = session.scalars(stmt).all()
SELECT book.id, book.owner_id, book.title, book.summary
FROM book
WHERE book.owner_id = ?
[] (2,)
for book in books:
    print(f"{book.title}: {book.summary}")
A Nut Like No Other: some long summary
Geodesic Domes: A Retrospective: another long summary
Rocketry for Squirrels: yet another summary
As is the case with load_only(), unloaded columns by default will load themselves when accessed using lazy loading:
img_data = books[0].cover_photo
SELECT book.cover_photo AS book_cover_photo
FROM book
WHERE book.id = ?
[] (4,)
Multiple defer() options may be used in one statement in order to mark several columns as deferred.
As is the case with load_only(), the defer() option also includes the ability to have a deferred attribute raise an exception on access rather than lazy loading. This is illustrated in the section Using raiseload to prevent deferred column loads.
Using raiseload to prevent deferred column loads
When using the load_only() or defer() loader options, attributes marked as deferred on an object have the default behavior that when first accessed, a SELECT statement will be emitted within the current transaction in order to load their value. It is often necessary to prevent this load from occurring, and instead raise an exception when the attribute is accessed, indicating that the need to query the database for this column was not expected. A typical scenario is an operation where objects are loaded with all the columns that are known to be required for the operation to proceed, which are then passed onto a view layer. Any further SQL operations that emit within the view layer should be caught, so that the up-front loading operation can be adjusted to accommodate for that additional data up front, rather than incurring additional lazy loading.
For this use case the defer() and load_only() options include a boolean parameter defer.raiseload, which when set to True will cause the affected attributes to raise on access. In the example below, the deferred column .cover_photo will disallow attribute access:
book = session.scalar(
    select(Book).options(defer(Book.cover_photo, raiseload=True)).where(Book.id == 4)
)
SELECT book.id, book.owner_id, book.title, book.summary
FROM book
WHERE book.id = ?
[] (4,)
book.cover_photo
Traceback (most recent call last):

sqlalchemy.exc.InvalidRequestError: 'Book.cover_photo' is not available due to raiseload=True
When using load_only() to name a specific set of non-deferred columns, raiseload behavior may be applied to the remaining columns using the load_only.raiseload parameter, which will be applied to all deferred attributes:
session.expunge_all()
book = session.scalar(
    select(Book).options(load_only(Book.title, raiseload=True)).where(Book.id == 5)
)
SELECT book.id, book.title
FROM book
WHERE book.id = ?
[] (5,)
book.summary
Traceback (most recent call last):

sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
Note
It is not yet possible to mix load_only() and defer() options which refer to the same entity together in one statement in order to change the raiseload behavior of certain attributes; currently, doing so will produce undefined loading behavior of attributes.
See also
The defer.raiseload feature is the column-level version of the same “raiseload” feature that’s available for relationships. For “raiseload” with relationships, see Preventing unwanted lazy loads using raiseload in the Relationship Loading Techniques section of this guide.
Configuring Column Deferral on Mappings
The functionality of defer() is available as a default behavior for mapped columns, as may be appropriate for columns that should not be loaded unconditionally on every query. To configure, use the mapped_column.deferred parameter of mapped_column(). The example below illustrates a mapping for Book which applies default column deferral to the summary and cover_photo columns:
class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    title: Mapped[str]
    summary: Mapped[str] = mapped_column(Text, deferred=True)
    cover_photo: Mapped[bytes] = mapped_column(LargeBinary, deferred=True)

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"
Using the above mapping, queries against Book will automatically not include the summary and cover_photo columns:
book = session.scalar(select(Book).where(Book.id == 2))
SELECT book.id, book.owner_id, book.title
FROM book
WHERE book.id = ?
[] (2,)
As is the case with all deferral, the default behavior when deferred attributes on the loaded object are first accessed is that they will lazy load their value:
img_data = book.cover_photo
SELECT book.cover_photo AS book_cover_photo
FROM book
WHERE book.id = ?
[] (2,)
As is the case with the defer() and load_only() loader options, mapper level deferral also includes an option for raiseload behavior to occur, rather than lazy loading, when no other options are present in a statement. This allows a mapping where certain columns will not load by default and will also never load lazily without explicit directives used in a statement. See the section Configuring mapper-level “raiseload” behavior for background on how to configure and use this behavior.
Using deferred() for imperative mappers, mapped SQL expressions
The deferred() function is the earlier, more general purpose “deferred column” mapping directive that precedes the introduction of the mapped_column() construct in SQLAlchemy.
deferred() is used when configuring ORM mappers, and accepts arbitrary SQL expressions or Column objects. As such it’s suitable to be used with non-declarative imperative mappings, passing it to the map_imperatively.properties dictionary:
from sqlalchemy import Blob
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Text
from sqlalchemy.orm import registry

mapper_registry = registry()

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(50)),
    Column("summary", Text),
    Column("cover_image", Blob),
)


class Book:
    pass


mapper_registry.map_imperatively(
    Book,
    book_table,
    properties={
        "summary": deferred(book_table.c.summary),
        "cover_image": deferred(book_table.c.cover_image),
    },
)
deferred() may also be used in place of column_property() when mapped SQL expressions should be loaded on a deferred basis:
from sqlalchemy.orm import deferred


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
See also
Using column_property - in the section SQL Expressions as Mapped Attributes
Applying Load, Persistence and Mapping Options for Imperative Table Columns - in the section Table Configuration with Declarative
Using undefer() to “eagerly” load deferred columns
With columns configured on mappings to defer by default, the undefer() option will cause any column that is normally deferred to be undeferred, that is, to load up front with all the other columns of the mapping. For example we may apply undefer() to the Book.summary column, which is indicated in the previous mapping as deferred:
from sqlalchemy.orm import undefer
book = session.scalar(select(Book).where(Book.id == 2).options(undefer(Book.summary)))
SELECT book.id, book.owner_id, book.title, book.summary
FROM book
WHERE book.id = ?
[] (2,)
The Book.summary column was now eagerly loaded, and may be accessed without additional SQL being emitted:
print(book.summary)
another long summary
Loading deferred columns in groups
Normally when a column is mapped with mapped_column(deferred=True), when the deferred attribute is accessed on an object, SQL will be emitted to load only that specific column and no others, even if the mapping has other columns that are also marked as deferred. In the common case that the deferred attribute is part of a group of attributes that should all load at once, rather than emitting SQL for each attribute individually, the mapped_column.deferred_group parameter may be used, which accepts an arbitrary string which will define a common group of columns to be undeferred:
class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    title: Mapped[str]
    summary: Mapped[str] = mapped_column(
        Text, deferred=True, deferred_group="book_attrs"
    )
    cover_photo: Mapped[bytes] = mapped_column(
        LargeBinary, deferred=True, deferred_group="book_attrs"
    )

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"
Using the above mapping, accessing either summary or cover_photo will load both columns at once using just one SELECT statement:
book = session.scalar(select(Book).where(Book.id == 2))
SELECT book.id, book.owner_id, book.title
FROM book
WHERE book.id = ?
[] (2,)
img_data, summary = book.cover_photo, book.summary
SELECT book.summary AS book_summary, book.cover_photo AS book_cover_photo
FROM book
WHERE book.id = ?
[] (2,)
Undeferring by group with undefer_group()
If deferred columns are configured with mapped_column.deferred_group as introduced in the preceding section, the entire group may be indicated to load eagerly using the undefer_group() option, passing the string name of the group to be eagerly loaded:
from sqlalchemy.orm import undefer_group
book = session.scalar(
    select(Book).where(Book.id == 2).options(undefer_group("book_attrs"))
)
SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
FROM book
WHERE book.id = ?
[] (2,)
Both summary and cover_photo are available without additional loads:
img_data, summary = book.cover_photo, book.summary
Undeferring on wildcards
Most ORM loader options accept a wildcard expression, indicated by "*", which indicates that the option should be applied to all relevant attributes. If a mapping has a series of deferred columns, all such columns can be undeferred at once, without using a group name, by indicating a wildcard:
book = session.scalar(select(Book).where(Book.id == 3).options(undefer("*")))
SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
FROM book
WHERE book.id = ?
[] (3,)
Configuring mapper-level “raiseload” behavior
The “raiseload” behavior first introduced at Using raiseload to prevent deferred column loads may also be applied as a default mapper-level behavior, using the mapped_column.deferred_raiseload parameter of mapped_column(). When using this parameter, the affected columns will raise on access in all cases unless explicitly “undeferred” using undefer() or load_only() at query time:
class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    title: Mapped[str]
    summary: Mapped[str] = mapped_column(Text, deferred=True, deferred_raiseload=True)
    cover_photo: Mapped[bytes] = mapped_column(
        LargeBinary, deferred=True, deferred_raiseload=True
    )

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"
Using the above mapping, the .summary and .cover_photo columns are by default not loadable:
book = session.scalar(select(Book).where(Book.id == 2))
SELECT book.id, book.owner_id, book.title
FROM book
WHERE book.id = ?
[] (2,)
book.summary
Traceback (most recent call last):

sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
Only by overriding their behavior at query time, typically using undefer() or undefer_group(), or less commonly defer(), may the attributes be loaded. The example below applies undefer('*') to undefer all attributes, also making use of Populate Existing to refresh the already-loaded object’s loader options:
book = session.scalar(
    select(Book)
    .where(Book.id == 2)
    .options(undefer("*"))
    .execution_options(populate_existing=True)
)
SELECT book.id, book.owner_id, book.title, book.summary, book.cover_photo
FROM book
WHERE book.id = ?
[] (2,)
book.summary
'another long summary'
Loading Arbitrary SQL Expressions onto Objects
As discussed Selecting ORM Entities and Attributes and elsewhere, the select() construct may be used to load arbitrary SQL expressions in a result set. Such as if we wanted to issue a query that loads User objects, but also includes a count of how many books each User owned, we could use func.count(Book.id) to add a “count” column to a query which includes a JOIN to Book as well as a GROUP BY owner id. This will yield Row objects that each contain two entries, one for User and one for func.count(Book.id):
from sqlalchemy import func
stmt = select(User, func.count(Book.id)).join_from(User, Book).group_by(Book.owner_id)
for user, book_count in session.execute(stmt):
    print(f"Username: {user.name}  Number of books: {book_count}")
SELECT user_account.id, user_account.name, user_account.fullname,
count(book.id) AS count_1
FROM user_account JOIN book ON user_account.id = book.owner_id
GROUP BY book.owner_id
[] ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
In the above example, the User entity and the “book count” SQL expression are returned separately. However, a popular use case is to produce a query that will yield User objects alone, which can be iterated for example using Session.scalars(), where the result of the func.count(Book.id) SQL expression is applied dynamically to each User entity. The end result would be similar to the case where an arbitrary SQL expression were mapped to the class using column_property(), except that the SQL expression can be modified at query time. For this use case SQLAlchemy provides the with_expression() loader option, which when combined with the mapper level query_expression() directive may produce this result.
To apply with_expression() to a query, the mapped class must have pre-configured an ORM mapped attribute using the query_expression() directive; this directive will produce an attribute on the mapped class that is suitable for receiving query-time SQL expressions. Below we add a new attribute User.book_count to User. This ORM mapped attribute is read-only and has no default value; accessing it on a loaded instance will normally produce None:
from sqlalchemy.orm import query_expression
class User(Base):
    __tablename__ = "user_account"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    book_count: Mapped[int] = query_expression()

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"
With the User.book_count attribute configured in our mapping, we may populate it with data from a SQL expression using the with_expression() loader option to apply a custom SQL expression to each User object as it’s loaded:
from sqlalchemy.orm import with_expression
stmt = (
    select(User)
    .join_from(User, Book)
    .group_by(Book.owner_id)
    .options(with_expression(User.book_count, func.count(Book.id)))
)
for user in session.scalars(stmt):
    print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT count(book.id) AS count_1, user_account.id, user_account.name,
user_account.fullname
FROM user_account JOIN book ON user_account.id = book.owner_id
GROUP BY book.owner_id
[] ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
Above, we moved our func.count(Book.id) expression out of the columns argument of the select() construct and into the with_expression() loader option. The ORM then considers this to be a special column load option that’s applied dynamically to the statement.
The query_expression() mapping has these caveats:
* On an object where with_expression() were not used to populate the attribute, the attribute on an object instance will have the value None, unless on the mapping the query_expression.default_expr parameter is set to a default SQL expression.
* The with_expression() value does not populate on an object that is already loaded, unless Populate Existing is used. The example below will not work, as the A object is already loaded:
* # load the first A
* obj = session.scalars(select(A).order_by(A.id)).first()
* 
* # load the same A with an option; expression will **not** be applied
* # to the already-loaded object
obj = session.scalars(select(A).options(with_expression(A.expr, some_expr))).first()
To ensure the attribute is re-loaded on an existing object, use the Populate Existing execution option to ensure all columns are re-populated:
obj = session.scalars(
    select(A)
    .options(with_expression(A.expr, some_expr))
    .execution_options(populate_existing=True)
).first()
*  The with_expression() SQL expression is lost when the object is expired. Once the object is expired, either via Session.expire() or via the expire_on_commit behavior of Session.commit(), the SQL expression and its value is no longer associated with the attribute and will return None on subsequent access.
*  with_expression(), as an object loading option, only takes effect on the outermost part of a query and only for a query against a full entity, and not for arbitrary column selects, within subqueries, or the elements of a compound statement such as a UNION. See the next section Using with_expression() with UNIONs, other subqueries for an example.
*  The mapped attribute cannot be applied to other parts of the query, such as the WHERE clause, the ORDER BY clause, and make use of the ad-hoc expression; that is, this won’t work:
# can't refer to A.expr elsewhere in the query
stmt = (
    select(A)
    .options(with_expression(A.expr, A.x + A.y))
    .filter(A.expr > 5)
    .order_by(A.expr)
)
The A.expr expression will resolve to NULL in the above WHERE clause and ORDER BY clause. To use the expression throughout the query, assign to a variable and use that:
# assign desired expression up front, then refer to that in
# the query
a_expr = A.x + A.y
stmt = (
    select(A)
    .options(with_expression(A.expr, a_expr))
    .filter(a_expr > 5)
    .order_by(a_expr)
)
See also
The with_expression() option is a special option used to apply SQL expressions to mapped classes dynamically at query time. For ordinary fixed SQL expressions configured on mappers, see the section SQL Expressions as Mapped Attributes.
Using with_expression() with UNIONs, other subqueries
The with_expression() construct is an ORM loader option, and as such may only be applied to the outermost level of a SELECT statement which is to load a particular ORM entity. It does not have any effect if used inside of a select() that will then be used as a subquery or as an element within a compound statement such as a UNION.
In order to use arbitrary SQL expressions in subqueries, normal Core-style means of adding expressions should be used. To assemble a subquery-derived expression onto the ORM entity’s query_expression() attributes, with_expression() is used at the top layer of ORM object loading, referencing the SQL expression within the subquery.
In the example below, two select() constructs are used against the ORM entity A with an additional SQL expression labeled in expr, and combined using union_all(). Then, at the topmost layer, the A entity is SELECTed from this UNION, using the querying technique described at Selecting Entities from UNIONs and other set operations, adding an option with with_expression() to extract this SQL expression onto newly loaded instances of A:
from sqlalchemy import union_all
s1 = (
    select(User, func.count(Book.id).label("book_count"))
    .join_from(User, Book)
    .where(User.name == "spongebob")
)
s2 = (
    select(User, func.count(Book.id).label("book_count"))
    .join_from(User, Book)
    .where(User.name == "sandy")
)
union_stmt = union_all(s1, s2)
orm_stmt = (
    select(User)
    .from_statement(union_stmt)
    .options(with_expression(User.book_count, union_stmt.selected_columns.book_count))
)
for user in session.scalars(orm_stmt):
    print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT user_account.id, user_account.name, user_account.fullname, count(book.id) AS book_count
FROM user_account JOIN book ON user_account.id = book.owner_id
WHERE user_account.name = ?
UNION ALL
SELECT user_account.id, user_account.name, user_account.fullname, count(book.id) AS book_count
FROM user_account JOIN book ON user_account.id = book.owner_id
WHERE user_account.name = ?
[] ('spongebob', 'sandy')
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
Column Loading API
Object Name
Description
defer(key, *addl_attrs, [raiseload])
Indicate that the given column-oriented attribute should be deferred, e.g. not loaded until accessed.
deferred(column, *additional_columns, [group, raiseload, comparator_factory, init, repr, default, default_factory, compare, kw_only, hash, active_history, expire_on_flush, info, doc])
Indicate a column-based mapped attribute that by default will not load unless accessed.
load_only(*attrs, [raiseload])
Indicate that for a particular entity, only the given list of column-based attribute names should be loaded; all others will be deferred.
query_expression([default_expr], *, [repr, compare, expire_on_flush, info, doc])
Indicate an attribute that populates from a query-time SQL expression.
undefer(key, *addl_attrs)
Indicate that the given column-oriented attribute should be undeferred, e.g. specified within the SELECT statement of the entity as a whole.
undefer_group(name)
Indicate that columns within the given deferred group name should be undeferred.
with_expression(key, expression)
Apply an ad-hoc SQL expression to a “deferred expression” attribute.
function sqlalchemy.orm.defer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) ? _AbstractLoad
Indicate that the given column-oriented attribute should be deferred, e.g. not loaded until accessed.
This function is part of the Load interface and supports both method-chained and standalone operation.
e.g.:
from sqlalchemy.orm import defer

session.query(MyClass).options(
    defer(MyClass.attribute_one), defer(MyClass.attribute_two)
)
To specify a deferred load of an attribute on a related class, the path can be specified one token at a time, specifying the loading style for each link along the chain. To leave the loading style for a link unchanged, use defaultload():
session.query(MyClass).options(
    defaultload(MyClass.someattr).defer(RelatedClass.some_column)
)
Multiple deferral options related to a relationship can be bundled at once using Load.options():
select(MyClass).options(
    defaultload(MyClass.someattr).options(
        defer(RelatedClass.some_column),
        defer(RelatedClass.some_other_column),
        defer(RelatedClass.another_column),
    )
)
Parameters:
* key – Attribute to be deferred.
* raiseload – raise InvalidRequestError rather than lazy loading a value when the deferred attribute is accessed. Used to prevent unwanted SQL from being emitted.
Added in version 1.4.
See also
Limiting which Columns Load with Column Deferral - in the ORM Querying Guide
load_only()
undefer()
function sqlalchemy.orm.deferred(column: _ORMColumnExprArgument[_T], *additional_columns: _ORMColumnExprArgument[Any], group: str | None = None, raiseload: bool = False, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, hash: _NoArg | bool | None = _NoArg.NO_ARG, active_history: bool = False, expire_on_flush: bool = True, info: _InfoType | None = None, doc: str | None = None) ? MappedSQLExpression[_T]
Indicate a column-based mapped attribute that by default will not load unless accessed.
When using mapped_column(), the same functionality as that of deferred() construct is provided by using the mapped_column.deferred parameter.
Parameters:
* *columns – columns to be mapped. This is typically a single Column object, however a collection is supported in order to support multiple columns mapped under the same attribute.
* raiseload – 
boolean, if True, indicates an exception should be raised if the load operation is to take place.
Added in version 1.4.
Additional arguments are the same as that of column_property().
See also
Using deferred() for imperative mappers, mapped SQL expressions
function sqlalchemy.orm.query_expression(default_expr: _ORMColumnExprArgument[_T] = <sqlalchemy.sql.elements.Null object>, *, repr: Union[_NoArg, bool] = _NoArg.NO_ARG, compare: Union[_NoArg, bool] = _NoArg.NO_ARG, expire_on_flush: bool = True, info: Optional[_InfoType] = None, doc: Optional[str] = None) ? MappedSQLExpression[_T]
Indicate an attribute that populates from a query-time SQL expression.
Parameters:
default_expr – Optional SQL expression object that will be used in all cases if not assigned later with with_expression().
Added in version 1.2.
See also
Loading Arbitrary SQL Expressions onto Objects - background and usage examples
function sqlalchemy.orm.load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) ? _AbstractLoad
Indicate that for a particular entity, only the given list of column-based attribute names should be loaded; all others will be deferred.
This function is part of the Load interface and supports both method-chained and standalone operation.
Example - given a class User, load only the name and fullname attributes:
session.query(User).options(load_only(User.name, User.fullname))
Example - given a relationship User.addresses -> Address, specify subquery loading for the User.addresses collection, but on each Address object load only the email_address attribute:
session.query(User).options(
    subqueryload(User.addresses).load_only(Address.email_address)
)
For a statement that has multiple entities, the lead entity can be specifically referred to using the Load constructor:
stmt = (
    select(User, Address)
    .join(User.addresses)
    .options(
        Load(User).load_only(User.name, User.fullname),
        Load(Address).load_only(Address.email_address),
    )
)
When used together with the populate_existing execution option only the attributes listed will be refreshed.
Parameters:
* *attrs – Attributes to be loaded, all others will be deferred.
* raiseload – 
raise InvalidRequestError rather than lazy loading a value when a deferred attribute is accessed. Used to prevent unwanted SQL from being emitted.
Added in version 2.0.
See also
Limiting which Columns Load with Column Deferral - in the ORM Querying Guide
Parameters:
* *attrs – Attributes to be loaded, all others will be deferred.
* raiseload – 
raise InvalidRequestError rather than lazy loading a value when a deferred attribute is accessed. Used to prevent unwanted SQL from being emitted.
Added in version 2.0.
function sqlalchemy.orm.undefer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any]) ? _AbstractLoad
Indicate that the given column-oriented attribute should be undeferred, e.g. specified within the SELECT statement of the entity as a whole.
The column being undeferred is typically set up on the mapping as a deferred() attribute.
This function is part of the Load interface and supports both method-chained and standalone operation.
Examples:
# undefer two columns
session.query(MyClass).options(
    undefer(MyClass.col1), undefer(MyClass.col2)
)

# undefer all columns specific to a single class using Load + *
session.query(MyClass, MyOtherClass).options(Load(MyClass).undefer("*"))

# undefer a column on a related object
select(MyClass).options(defaultload(MyClass.items).undefer(MyClass.text))
Parameters:
key – Attribute to be undeferred.
See also
Limiting which Columns Load with Column Deferral - in the ORM Querying Guide
defer()
undefer_group()
function sqlalchemy.orm.undefer_group(name: str) ? _AbstractLoad
Indicate that columns within the given deferred group name should be undeferred.
The columns being undeferred are set up on the mapping as deferred() attributes and include a “group” name.
E.g:
session.query(MyClass).options(undefer_group("large_attrs"))
To undefer a group of attributes on a related entity, the path can be spelled out using relationship loader options, such as defaultload():
select(MyClass).options(
    defaultload("someattr").undefer_group("large_attrs")
)
See also
Limiting which Columns Load with Column Deferral - in the ORM Querying Guide
defer()
undefer()
function sqlalchemy.orm.with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) ? _AbstractLoad
Apply an ad-hoc SQL expression to a “deferred expression” attribute.
This option is used in conjunction with the query_expression() mapper-level construct that indicates an attribute which should be the target of an ad-hoc SQL expression.
E.g.:
stmt = select(SomeClass).options(
    with_expression(SomeClass.x_y_expr, SomeClass.x + SomeClass.y)
)
Added in version 1.2.
Parameters:
* key – Attribute to be populated
* expr – SQL expression to be applied to the attribute.
See also
Loading Arbitrary SQL Expressions onto Objects - background and usage examples


Relationship Loading Techniques
About this Document
This section presents an in-depth view of how to load related objects. Readers should be familiar with Relationship Configuration and basic use.
Most examples here assume the “User/Address” mapping setup similar to the one illustrated at setup for selects.
A big part of SQLAlchemy is providing a wide range of control over how related objects get loaded when querying. By “related objects” we refer to collections or scalar associations configured on a mapper using relationship(). This behavior can be configured at mapper construction time using the relationship.lazy parameter to the relationship() function, as well as by using ORM loader options with the Select construct.
The loading of relationships falls into three categories; lazy loading, eager loading, and no loading. Lazy loading refers to objects that are returned from a query without the related objects loaded at first. When the given collection or reference is first accessed on a particular object, an additional SELECT statement is emitted such that the requested collection is loaded.
Eager loading refers to objects returned from a query with the related collection or scalar reference already loaded up front. The ORM achieves this either by augmenting the SELECT statement it would normally emit with a JOIN to load in related rows simultaneously, or by emitting additional SELECT statements after the primary one to load collections or scalar references at once.
“No” loading refers to the disabling of loading on a given relationship, either that the attribute is empty and is just never loaded, or that it raises an error when it is accessed, in order to guard against unwanted lazy loads.
Summary of Relationship Loading Styles
The primary forms of relationship loading are:
* lazy loading - available via lazy='select' or the lazyload() option, this is the form of loading that emits a SELECT statement at attribute access time to lazily load a related reference on a single object at a time. Lazy loading is the default loading style for all relationship() constructs that don’t otherwise indicate the relationship.lazy option. Lazy loading is detailed at Lazy Loading.
* select IN loading - available via lazy='selectin' or the selectinload() option, this form of loading emits a second (or more) SELECT statement which assembles the primary key identifiers of the parent objects into an IN clause, so that all members of related collections / scalar references are loaded at once by primary key. Select IN loading is detailed at Select IN loading.
* joined loading - available via lazy='joined' or the joinedload() option, this form of loading applies a JOIN to the given SELECT statement so that related rows are loaded in the same result set. Joined eager loading is detailed at Joined Eager Loading.
* raise loading - available via lazy='raise', lazy='raise_on_sql', or the raiseload() option, this form of loading is triggered at the same time a lazy load would normally occur, except it raises an ORM exception in order to guard against the application making unwanted lazy loads. An introduction to raise loading is at Preventing unwanted lazy loads using raiseload.
* subquery loading - available via lazy='subquery' or the subqueryload() option, this form of loading emits a second SELECT statement which re-states the original query embedded inside of a subquery, then JOINs that subquery to the related table to be loaded to load all members of related collections / scalar references at once. Subquery eager loading is detailed at Subquery Eager Loading.
* write only loading - available via lazy='write_only', or by annotating the left side of the Relationship object using the WriteOnlyMapped annotation. This collection-only loader style produces an alternative attribute instrumentation that never implicitly loads records from the database, instead only allowing WriteOnlyCollection.add(), WriteOnlyCollection.add_all() and WriteOnlyCollection.remove() methods. Querying the collection is performed by invoking a SELECT statement which is constructed using the WriteOnlyCollection.select() method. Write only loading is discussed at Write Only Relationships.
* dynamic loading - available via lazy='dynamic', or by annotating the left side of the Relationship object using the DynamicMapped annotation. This is a legacy collection-only loader style which produces a Query object when the collection is accessed, allowing custom SQL to be emitted against the collection’s contents. However, dynamic loaders will implicitly iterate the underlying collection in various circumstances which makes them less useful for managing truly large collections. Dynamic loaders are superseded by “write only” collections, which will prevent the underlying collection from being implicitly loaded under any circumstances. Dynamic loaders are discussed at Dynamic Relationship Loaders.
Configuring Loader Strategies at Mapping Time
The loader strategy for a particular relationship can be configured at mapping time to take place in all cases where an object of the mapped type is loaded, in the absence of any query-level options that modify it. This is configured using the relationship.lazy parameter to relationship(); common values for this parameter include select, selectin and joined.
The example below illustrates the relationship example at One To Many, configuring the Parent.children relationship to use Select IN loading when a SELECT statement for Parent objects is emitted:
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship


class Base(DeclarativeBase):
    pass


class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(lazy="selectin")


class Child(Base):
    __tablename__ = "child"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
Above, whenever a collection of Parent objects are loaded, each Parent will also have its children collection populated, using the "selectin" loader strategy that emits a second query.
The default value of the relationship.lazy argument is "select", which indicates Lazy Loading.
Relationship Loading with Loader Options
The other, and possibly more common way to configure loading strategies is to set them up on a per-query basis against specific attributes using the Select.options() method. Very detailed control over relationship loading is available using loader options; the most common are joinedload(), selectinload() and lazyload(). The option accepts a class-bound attribute referring to the specific class/attribute that should be targeted:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# set children to load lazily
stmt = select(Parent).options(lazyload(Parent.children))

from sqlalchemy.orm import joinedload

# set children to load eagerly with a join
stmt = select(Parent).options(joinedload(Parent.children))
The loader options can also be “chained” using method chaining to specify how loading should occur further levels deep:
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Parent).options(
    joinedload(Parent.children).subqueryload(Child.subelements)
)
Chained loader options can be applied against a “lazy” loaded collection. This means that when a collection or association is lazily loaded upon access, the specified option will then take effect:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
Above, the query will return Parent objects without the children collections loaded. When the children collection on a particular Parent object is first accessed, it will lazy load the related objects, but additionally apply eager loading to the subelements collection on each member of children.
Adding Criteria to loader options
The relationship attributes used to indicate loader options include the ability to add additional filtering criteria to the ON clause of the join that’s created, or to the WHERE criteria involved, depending on the loader strategy. This can be achieved using the PropComparator.and_() method which will pass through an option such that loaded results are limited to the given filter criteria:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))
When using limiting criteria, if a particular collection is already loaded it won’t be refreshed; to ensure the new criteria takes place, apply the Populate Existing execution option:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = (
    select(A)
    .options(lazyload(A.bs.and_(B.id > 5)))
    .execution_options(populate_existing=True)
)
In order to add filtering criteria to all occurrences of an entity throughout a query, regardless of loader strategy or where it occurs in the loading process, see the with_loader_criteria() function.
Added in version 1.4.
Specifying Sub-Options with Load.options()
Using method chaining, the loader style of each link in the path is explicitly stated. To navigate along a path without changing the existing loader style of a particular attribute, the defaultload() method/function may be used:
from sqlalchemy import select
from sqlalchemy.orm import defaultload

stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))
A similar approach can be used to specify multiple sub-options at once, using the Load.options() method:
from sqlalchemy import select
from sqlalchemy.orm import defaultload
from sqlalchemy.orm import joinedload

stmt = select(A).options(
    defaultload(A.atob).options(joinedload(B.btoc), joinedload(B.btod))
)
See also
Using load_only() on related objects and collections - illustrates examples of combining relationship and column-oriented loader options.
Note
The loader options applied to an object’s lazy-loaded collections are “sticky” to specific object instances, meaning they will persist upon collections loaded by that specific object for as long as it exists in memory. For example, given the previous example:
stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
if the children collection on a particular Parent object loaded by the above query is expired (such as when a Session object’s transaction is committed or rolled back, or Session.expire_all() is used), when the Parent.children collection is next accessed in order to re-load it, the Child.subelements collection will again be loaded using subquery eager loading. This stays the case even if the above Parent object is accessed from a subsequent query that specifies a different set of options. To change the options on an existing object without expunging it and re-loading, they must be set explicitly in conjunction using the Populate Existing execution option:
# change the options on Parent objects that were already loaded
stmt = (
    select(Parent)
    .execution_options(populate_existing=True)
    .options(lazyload(Parent.children).lazyload(Child.subelements))
    .all()
)
If the objects loaded above are fully cleared from the Session, such as due to garbage collection or that Session.expunge_all() were used, the “sticky” options will also be gone and the newly created objects will make use of new options if loaded again.
A future SQLAlchemy release may add more alternatives to manipulating the loader options on already-loaded objects.
Lazy Loading
By default, all inter-object relationships are lazy loading. The scalar or collection attribute associated with a relationship() contains a trigger which fires the first time the attribute is accessed. This trigger typically issues a SQL call at the point of access in order to load the related object or objects:
spongebob.addresses
SELECT
    addresses.id AS addresses_id,
    addresses.email_address AS addresses_email_address,
    addresses.user_id AS addresses_user_id
FROM addresses
WHERE ? = addresses.user_id
[5]
[<Address(u'spongebob@google.com')>, <Address(u'j25@yahoo.com')>]
The one case where SQL is not emitted is for a simple many-to-one relationship, when the related object can be identified by its primary key alone and that object is already present in the current Session. For this reason, while lazy loading can be expensive for related collections, in the case that one is loading lots of objects with simple many-to-ones against a relatively small set of possible target objects, lazy loading may be able to refer to these objects locally without emitting as many SELECT statements as there are parent objects.
This default behavior of “load upon attribute access” is known as “lazy” or “select” loading - the name “select” because a “SELECT” statement is typically emitted when the attribute is first accessed.
Lazy loading can be enabled for a given attribute that is normally configured in some other way using the lazyload() loader option:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# force lazy loading for an attribute that is set to
# load some other way normally
stmt = select(User).options(lazyload(User.addresses))
Preventing unwanted lazy loads using raiseload
The lazyload() strategy produces an effect that is one of the most common issues referred to in object relational mapping; the N plus one problem, which states that for any N objects loaded, accessing their lazy-loaded attributes means there will be N+1 SELECT statements emitted. In SQLAlchemy, the usual mitigation for the N+1 problem is to make use of its very capable eager load system. However, eager loading requires that the attributes which are to be loaded be specified with the Select up front. The problem of code that may access other attributes that were not eagerly loaded, where lazy loading is not desired, may be addressed using the raiseload() strategy; this loader strategy replaces the behavior of lazy loading with an informative error being raised:
from sqlalchemy import select
from sqlalchemy.orm import raiseload

stmt = select(User).options(raiseload(User.addresses))
Above, a User object loaded from the above query will not have the .addresses collection loaded; if some code later on attempts to access this attribute, an ORM exception is raised.
raiseload() may be used with a so-called “wildcard” specifier to indicate that all relationships should use this strategy. For example, to set up only one attribute as eager loading, and all the rest as raise:
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import raiseload

stmt = select(Order).options(joinedload(Order.items), raiseload("*"))
The above wildcard will apply to all relationships not just on Order besides items, but all those on the Item objects as well. To set up raiseload() for only the Order objects, specify a full path with Load:
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import Load

stmt = select(Order).options(joinedload(Order.items), Load(Order).raiseload("*"))
Conversely, to set up the raise for just the Item objects:
stmt = select(Order).options(joinedload(Order.items).raiseload("*"))
The raiseload() option applies only to relationship attributes. For column-oriented attributes, the defer() option supports the defer.raiseload option which works in the same way.
Tip
The “raiseload” strategies do not apply within the unit of work flush process. That means if the Session.flush() process needs to load a collection in order to finish its work, it will do so while bypassing any raiseload() directives.
See also
Wildcard Loading Strategies
Using raiseload to prevent deferred column loads
Joined Eager Loading
Joined eager loading is the oldest style of eager loading included with the SQLAlchemy ORM. It works by connecting a JOIN (by default a LEFT OUTER join) to the SELECT statement emitted, and populates the target scalar/collection from the same result set as that of the parent.
At the mapping level, this looks like:
class Address(Base):
    # 

    user: Mapped[User] = relationship(lazy="joined")
Joined eager loading is usually applied as an option to a query, rather than as a default loading option on the mapping, in particular when used for collections rather than many-to-one-references. This is achieved using the joinedload() loader option:
from sqlalchemy import select
from sqlalchemy.orm import joinedload
stmt = select(User).options(joinedload(User.addresses)).filter_by(name="spongebob")
spongebob = session.scalars(stmt).unique().all()
SELECT
    addresses_1.id AS addresses_1_id,
    addresses_1.email_address AS addresses_1_email_address,
    addresses_1.user_id AS addresses_1_user_id,
    users.id AS users_id, users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
LEFT OUTER JOIN addresses AS addresses_1
    ON users.id = addresses_1.user_id
WHERE users.name = ?
['spongebob']
Tip
When including joinedload() in reference to a one-to-many or many-to-many collection, the Result.unique() method must be applied to the returned result, which will uniquify the incoming rows by primary key that otherwise are multiplied out by the join. The ORM will raise an error if this is not present.
This is not automatic in modern SQLAlchemy, as it changes the behavior of the result set to return fewer ORM objects than the statement would normally return in terms of number of rows. Therefore SQLAlchemy keeps the use of Result.unique() explicit, so there’s no ambiguity that the returned objects are being uniqified on primary key.
The JOIN emitted by default is a LEFT OUTER JOIN, to allow for a lead object that does not refer to a related row. For an attribute that is guaranteed to have an element, such as a many-to-one reference to a related object where the referencing foreign key is NOT NULL, the query can be made more efficient by using an inner join; this is available at the mapping level via the relationship.innerjoin flag:
class Address(Base):
    # 

    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    user: Mapped[User] = relationship(lazy="joined", innerjoin=True)
At the query option level, via the joinedload.innerjoin flag:
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Address).options(joinedload(Address.user, innerjoin=True))
The JOIN will right-nest itself when applied in a chain that includes an OUTER JOIN:
from sqlalchemy import select
from sqlalchemy.orm import joinedload
stmt = select(User).options(
    joinedload(User.addresses).joinedload(Address.widgets, innerjoin=True)
)
results = session.scalars(stmt).unique().all()
SELECT
    widgets_1.id AS widgets_1_id,
    widgets_1.name AS widgets_1_name,
    addresses_1.id AS addresses_1_id,
    addresses_1.email_address AS addresses_1_email_address,
    addresses_1.user_id AS addresses_1_user_id,
    users.id AS users_id, users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
LEFT OUTER JOIN (
    addresses AS addresses_1 JOIN widgets AS widgets_1 ON
    addresses_1.widget_id = widgets_1.id
) ON users.id = addresses_1.user_id
Tip
If using database row locking techniques when emitting the SELECT, meaning the Select.with_for_update() method is being used to emit SELECT..FOR UPDATE, the joined table may be locked as well, depending on the behavior of the backend in use. It’s not recommended to use joined eager loading at the same time as SELECT..FOR UPDATE for this reason.
The Zen of Joined Eager Loading
Since joined eager loading seems to have many resemblances to the use of Select.join(), it often produces confusion as to when and how it should be used. It is critical to understand the distinction that while Select.join() is used to alter the results of a query, joinedload() goes through great lengths to not alter the results of the query, and instead hide the effects of the rendered join to only allow for related objects to be present.
The philosophy behind loader strategies is that any set of loading schemes can be applied to a particular query, and the results don’t change - only the number of SQL statements required to fully load related objects and collections changes. A particular query might start out using all lazy loads. After using it in context, it might be revealed that particular attributes or collections are always accessed, and that it would be more efficient to change the loader strategy for these. The strategy can be changed with no other modifications to the query, the results will remain identical, but fewer SQL statements would be emitted. In theory (and pretty much in practice), nothing you can do to the Select would make it load a different set of primary or related objects based on a change in loader strategy.
How joinedload() in particular achieves this result of not impacting entity rows returned in any way is that it creates an anonymous alias of the joins it adds to your query, so that they can’t be referenced by other parts of the query. For example, the query below uses joinedload() to create a LEFT OUTER JOIN from users to addresses, however the ORDER BY added against Address.email_address is not valid - the Address entity is not named in the query:
from sqlalchemy import select
from sqlalchemy.orm import joinedload
stmt = (
    select(User)
    .options(joinedload(User.addresses))
    .filter(User.name == "spongebob")
    .order_by(Address.email_address)
)
result = session.scalars(stmt).unique().all()
SELECT
    addresses_1.id AS addresses_1_id,
    addresses_1.email_address AS addresses_1_email_address,
    addresses_1.user_id AS addresses_1_user_id,
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
LEFT OUTER JOIN addresses AS addresses_1
    ON users.id = addresses_1.user_id
WHERE users.name = ?
ORDER BY addresses.email_address   <-- this part is wrong !
['spongebob']
Above, ORDER BY addresses.email_address is not valid since addresses is not in the FROM list. The correct way to load the User records and order by email address is to use Select.join():
from sqlalchemy import select
stmt = (
    select(User)
    .join(User.addresses)
    .filter(User.name == "spongebob")
    .order_by(Address.email_address)
)
result = session.scalars(stmt).unique().all()
SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
JOIN addresses ON users.id = addresses.user_id
WHERE users.name = ?
ORDER BY addresses.email_address
['spongebob']
The statement above is of course not the same as the previous one, in that the columns from addresses are not included in the result at all. We can add joinedload() back in, so that there are two joins - one is that which we are ordering on, the other is used anonymously to load the contents of the User.addresses collection:
stmt = (
    select(User)
    .join(User.addresses)
    .options(joinedload(User.addresses))
    .filter(User.name == "spongebob")
    .order_by(Address.email_address)
)
result = session.scalars(stmt).unique().all()
SELECT
    addresses_1.id AS addresses_1_id,
    addresses_1.email_address AS addresses_1_email_address,
    addresses_1.user_id AS addresses_1_user_id,
    users.id AS users_id, users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users JOIN addresses
    ON users.id = addresses.user_id
LEFT OUTER JOIN addresses AS addresses_1
    ON users.id = addresses_1.user_id
WHERE users.name = ?
ORDER BY addresses.email_address
['spongebob']
What we see above is that our usage of Select.join() is to supply JOIN clauses we’d like to use in subsequent query criterion, whereas our usage of joinedload() only concerns itself with the loading of the User.addresses collection, for each User in the result. In this case, the two joins most probably appear redundant - which they are. If we wanted to use just one JOIN for collection loading as well as ordering, we use the contains_eager() option, described in Routing Explicit Joins/Statements into Eagerly Loaded Collections below. But to see why joinedload() does what it does, consider if we were filtering on a particular Address:
stmt = (
    select(User)
    .join(User.addresses)
    .options(joinedload(User.addresses))
    .filter(User.name == "spongebob")
    .filter(Address.email_address == "someaddress@foo.com")
)
result = session.scalars(stmt).unique().all()
SELECT
    addresses_1.id AS addresses_1_id,
    addresses_1.email_address AS addresses_1_email_address,
    addresses_1.user_id AS addresses_1_user_id,
    users.id AS users_id, users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users JOIN addresses
    ON users.id = addresses.user_id
LEFT OUTER JOIN addresses AS addresses_1
    ON users.id = addresses_1.user_id
WHERE users.name = ? AND addresses.email_address = ?
['spongebob', 'someaddress@foo.com']
Above, we can see that the two JOINs have very different roles. One will match exactly one row, that of the join of User and Address where Address.email_address=='someaddress@foo.com'. The other LEFT OUTER JOIN will match all Address rows related to User, and is only used to populate the User.addresses collection, for those User objects that are returned.
By changing the usage of joinedload() to another style of loading, we can change how the collection is loaded completely independently of SQL used to retrieve the actual User rows we want. Below we change joinedload() into selectinload():
stmt = (
    select(User)
    .join(User.addresses)
    .options(selectinload(User.addresses))
    .filter(User.name == "spongebob")
    .filter(Address.email_address == "someaddress@foo.com")
)
result = session.scalars(stmt).all()
SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
JOIN addresses ON users.id = addresses.user_id
WHERE
    users.name = ?
    AND addresses.email_address = ?
['spongebob', 'someaddress@foo.com']
# selectinload() emits a SELECT in order
# to load all address records 
When using joined eager loading, if the query contains a modifier that impacts the rows returned externally to the joins, such as when using DISTINCT, LIMIT, OFFSET or equivalent, the completed statement is first wrapped inside a subquery, and the joins used specifically for joined eager loading are applied to the subquery. SQLAlchemy’s joined eager loading goes the extra mile, and then ten miles further, to absolutely ensure that it does not affect the end result of the query, only the way collections and related objects are loaded, no matter what the format of the query is.
See also
Routing Explicit Joins/Statements into Eagerly Loaded Collections - using contains_eager()
Select IN loading
In most cases, selectin loading is the most simple and efficient way to eagerly load collections of objects. The only scenario in which selectin eager loading is not feasible is when the model is using composite primary keys, and the backend database does not support tuples with IN, which currently includes SQL Server.
“Select IN” eager loading is provided using the "selectin" argument to relationship.lazy or by using the selectinload() loader option. This style of loading emits a SELECT that refers to the primary key values of the parent object, or in the case of a many-to-one relationship to the those of the child objects, inside of an IN clause, in order to load related associations:
from sqlalchemy import select
from sqlalchemy.orm import selectinload
stmt = (
    select(User)
    .options(selectinload(User.addresses))
    .filter(or_(User.name == "spongebob", User.name == "ed"))
)
result = session.scalars(stmt).all()
SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
WHERE users.name = ? OR users.name = ?
('spongebob', 'ed')
SELECT
    addresses.id AS addresses_id,
    addresses.email_address AS addresses_email_address,
    addresses.user_id AS addresses_user_id
FROM addresses
WHERE addresses.user_id IN (?, ?)
(5, 7)
Above, the second SELECT refers to addresses.user_id IN (5, 7), where the “5” and “7” are the primary key values for the previous two User objects loaded; after a batch of objects are completely loaded, their primary key values are injected into the IN clause for the second SELECT. Because the relationship between User and Address has a simple primary join condition and provides that the primary key values for User can be derived from Address.user_id, the statement has no joins or subqueries at all.
For simple many-to-one loads, a JOIN is also not needed as the foreign key value from the parent object is used:
from sqlalchemy import select
from sqlalchemy.orm import selectinload
stmt = select(Address).options(selectinload(Address.user))
result = session.scalars(stmt).all()
SELECT
    addresses.id AS addresses_id,
    addresses.email_address AS addresses_email_address,
    addresses.user_id AS addresses_user_id
    FROM addresses
SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
WHERE users.id IN (?, ?)
(1, 2)
Tip
by “simple” we mean that the relationship.primaryjoin condition expresses an equality comparison between the primary key of the “one” side and a straight foreign key of the “many” side, without any additional criteria.
Select IN loading also supports many-to-many relationships, where it currently will JOIN across all three tables to match rows from one side to the other.
Things to know about this kind of loading include:
* The strategy emits a SELECT for up to 500 parent primary key values at a time, as the primary keys are rendered into a large IN expression in the SQL statement. Some databases like Oracle Database have a hard limit on how large an IN expression can be, and overall the size of the SQL string shouldn’t be arbitrarily large.
* As “selectin” loading relies upon IN, for a mapping with composite primary keys, it must use the “tuple” form of IN, which looks like WHERE (table.column_a, table.column_b) IN ((?, ?), (?, ?), (?, ?)). This syntax is not currently supported on SQL Server and for SQLite requires at least version 3.15. There is no special logic in SQLAlchemy to check ahead of time which platforms support this syntax or not; if run against a non-supporting platform, the database will return an error immediately. An advantage to SQLAlchemy just running the SQL out for it to fail is that if a particular database does start supporting this syntax, it will work without any changes to SQLAlchemy (as was the case with SQLite).
Subquery Eager Loading
Legacy Feature
The subqueryload() eager loader is mostly legacy at this point, superseded by the selectinload() strategy which is of much simpler design, more flexible with features such as Yield Per, and emits more efficient SQL statements in most cases. As subqueryload() relies upon re-interpreting the original SELECT statement, it may fail to work efficiently when given very complex source queries.
subqueryload() may continue to be useful for the specific case of an eager loaded collection for objects that use composite primary keys, on the Microsoft SQL Server backend that continues to not have support for the “tuple IN” syntax.
Subquery loading is similar in operation to selectin eager loading, however the SELECT statement which is emitted is derived from the original statement, and has a more complex query structure as that of selectin eager loading.
Subquery eager loading is provided using the "subquery" argument to relationship.lazy or by using the subqueryload() loader option.
The operation of subquery eager loading is to emit a second SELECT statement for each relationship to be loaded, across all result objects at once. This SELECT statement refers to the original SELECT statement, wrapped inside of a subquery, so that we retrieve the same list of primary keys for the primary object being returned, then link that to the sum of all the collection members to load them at once:
from sqlalchemy import select
from sqlalchemy.orm import subqueryload
stmt = select(User).options(subqueryload(User.addresses)).filter_by(name="spongebob")
results = session.scalars(stmt).all()
SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
FROM users
WHERE users.name = ?
('spongebob',)
SELECT
    addresses.id AS addresses_id,
    addresses.email_address AS addresses_email_address,
    addresses.user_id AS addresses_user_id,
    anon_1.users_id AS anon_1_users_id
FROM (
    SELECT users.id AS users_id
    FROM users
    WHERE users.name = ?) AS anon_1
JOIN addresses ON anon_1.users_id = addresses.user_id
ORDER BY anon_1.users_id, addresses.id
('spongebob',)
Things to know about this kind of loading include:
* The SELECT statement emitted by the “subquery” loader strategy, unlike that of “selectin”, requires a subquery, and will inherit whatever performance limitations are present in the original query. The subquery itself may also incur performance penalties based on the specifics of the database in use.
* “subquery” loading imposes some special ordering requirements in order to work correctly. A query which makes use of subqueryload() in conjunction with a limiting modifier such as Select.limit(), or Select.offset() should always include Select.order_by() against unique column(s) such as the primary key, so that the additional queries emitted by subqueryload() include the same ordering as used by the parent query. Without it, there is a chance that the inner query could return the wrong rows:
* # incorrect, no ORDER BY
* stmt = select(User).options(subqueryload(User.addresses).limit(1))
* 
* # incorrect if User.name is not unique
* stmt = select(User).options(subqueryload(User.addresses)).order_by(User.name).limit(1)
* 
* # correct
* stmt = (
*     select(User)
*     .options(subqueryload(User.addresses))
*     .order_by(User.name, User.id)
*     .limit(1)
)
* See also
Why is ORDER BY recommended with LIMIT (especially with subqueryload())? - detailed example
* “subquery” loading also incurs additional performance / complexity issues when used on a many-levels-deep eager load, as subqueries will be nested repeatedly.
* “subquery” loading is not compatible with the “batched” loading supplied by Yield Per, both for collection and scalar relationships.
For the above reasons, the “selectin” strategy should be preferred over “subquery”.
See also
Select IN loading
What Kind of Loading to Use ?
Which type of loading to use typically comes down to optimizing the tradeoff between number of SQL executions, complexity of SQL emitted, and amount of data fetched.
One to Many / Many to Many Collection - The selectinload() is generally the best loading strategy to use. It emits an additional SELECT that uses as few tables as possible, leaving the original statement unaffected, and is most flexible for any kind of originating query. Its only major limitation is when using a table with composite primary keys on a backend that does not support “tuple IN”, which currently includes SQL Server and very old SQLite versions; all other included backends support it.
Many to One - The joinedload() strategy is the most general purpose strategy. In special cases, the immediateload() strategy may also be useful, if there are a very small number of potential related values, as this strategy will fetch the object from the local Session without emitting any SQL if the related object is already present.
Polymorphic Eager Loading
Specification of polymorphic options on a per-eager-load basis is supported. See the section Eager Loading of Polymorphic Subtypes for examples of the PropComparator.of_type() method in conjunction with the with_polymorphic() function.
Wildcard Loading Strategies
Each of joinedload(), subqueryload(), lazyload(), selectinload(), and raiseload() can be used to set the default style of relationship() loading for a particular query, affecting all relationship() -mapped attributes not otherwise specified in the statement. This feature is available by passing the string '*' as the argument to any of these options:
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(MyClass).options(lazyload("*"))
Above, the lazyload('*') option will supersede the lazy setting of all relationship() constructs in use for that query, with the exception of those that use lazy='write_only' or lazy='dynamic'.
If some relationships specify lazy='joined' or lazy='selectin', for example, using lazyload('*') will unilaterally cause all those relationships to use 'select' loading, e.g. emit a SELECT statement when each attribute is accessed.
The option does not supersede loader options stated in the query, such as joinedload(), selectinload(), etc. The query below will still use joined loading for the widget relationship:
from sqlalchemy import select
from sqlalchemy.orm import lazyload
from sqlalchemy.orm import joinedload

stmt = select(MyClass).options(lazyload("*"), joinedload(MyClass.widget))
While the instruction for joinedload() above will take place regardless of whether it appears before or after the lazyload() option, if multiple options that each included "*" were passed, the last one will take effect.
Per-Entity Wildcard Loading Strategies
A variant of the wildcard loader strategy is the ability to set the strategy on a per-entity basis. For example, if querying for User and Address, we can instruct all relationships on Address to use lazy loading, while leaving the loader strategies for User unaffected, by first applying the Load object, then specifying the * as a chained option:
from sqlalchemy import select
from sqlalchemy.orm import Load

stmt = select(User, Address).options(Load(Address).lazyload("*"))
Above, all relationships on Address will be set to a lazy load.
Routing Explicit Joins/Statements into Eagerly Loaded Collections
The behavior of joinedload() is such that joins are created automatically, using anonymous aliases as targets, the results of which are routed into collections and scalar references on loaded objects. It is often the case that a query already includes the necessary joins which represent a particular collection or scalar reference, and the joins added by the joinedload feature are redundant - yet you’d still like the collections/references to be populated.
For this SQLAlchemy supplies the contains_eager() option. This option is used in the same manner as the joinedload() option except it is assumed that the Select object will explicitly include the appropriate joins, typically using methods like Select.join(). Below, we specify a join between User and Address and additionally establish this as the basis for eager loading of User.addresses:
from sqlalchemy.orm import contains_eager

stmt = select(User).join(User.addresses).options(contains_eager(User.addresses))
If the “eager” portion of the statement is “aliased”, the path should be specified using PropComparator.of_type(), which allows the specific aliased() construct to be passed:
# use an alias of the Address entity
adalias = aliased(Address)

# construct a statement which expects the "addresses" results

stmt = (
    select(User)
    .outerjoin(User.addresses.of_type(adalias))
    .options(contains_eager(User.addresses.of_type(adalias)))
)

# get results normally
r = session.scalars(stmt).unique().all()
SELECT
    users.user_id AS users_user_id,
    users.user_name AS users_user_name,
    adalias.address_id AS adalias_address_id,
    adalias.user_id AS adalias_user_id,
    adalias.email_address AS adalias_email_address,
    (other columns)
FROM users
LEFT OUTER JOIN email_addresses AS email_addresses_1
ON users.user_id = email_addresses_1.user_id
The path given as the argument to contains_eager() needs to be a full path from the starting entity. For example if we were loading Users->orders->Order->items->Item, the option would be used as:
stmt = select(User).options(contains_eager(User.orders).contains_eager(Order.items))
Using contains_eager() to load a custom-filtered collection result
When we use contains_eager(), we are constructing ourselves the SQL that will be used to populate collections. From this, it naturally follows that we can opt to modify what values the collection is intended to store, by writing our SQL to load a subset of elements for collections or scalar attributes.
Tip
SQLAlchemy now has a much simpler way to do this, by allowing WHERE criteria to be added directly to loader options such as joinedload() and selectinload() using PropComparator.and_(). See the section Adding Criteria to loader options for examples.
The techniques described here still apply if the related collection is to be queried using SQL criteria or modifiers more complex than a simple WHERE clause.
As an example, we can load a User object and eagerly load only particular addresses into its .addresses collection by filtering the joined data, routing it using contains_eager(), also using Populate Existing to ensure any already-loaded collections are overwritten:
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address.like("%@aol.com"))
    .options(contains_eager(User.addresses))
    .execution_options(populate_existing=True)
)
The above query will load only User objects which contain at least Address object that contains the substring 'aol.com' in its email field; the User.addresses collection will contain only these Address entries, and not any other Address entries that are in fact associated with the collection.
Tip
In all cases, the SQLAlchemy ORM does not overwrite already loaded attributes and collections unless told to do so. As there is an identity map in use, it is often the case that an ORM query is returning objects that were in fact already present and loaded in memory. Therefore, when using contains_eager() to populate a collection in an alternate way, it is usually a good idea to use Populate Existing as illustrated above so that an already-loaded collection is refreshed with the new data. The populate_existing option will reset all attributes that were already present, including pending changes, so make sure all data is flushed before using it. Using the Session with its default behavior of autoflush is sufficient.
Note
The customized collection we load using contains_eager() is not “sticky”; that is, the next time this collection is loaded, it will be loaded with its usual default contents. The collection is subject to being reloaded if the object is expired, which occurs whenever the Session.commit(), Session.rollback() methods are used assuming default session settings, or the Session.expire_all() or Session.expire() methods are used.
See also
Adding Criteria to loader options - modern API allowing WHERE criteria directly within any relationship loader option
Relationship Loader API
Object Name
Description
contains_eager(*keys, **kw)
Indicate that the given attribute should be eagerly loaded from columns stated manually in the query.
defaultload(*keys)
Indicate an attribute should load using its predefined loader style.
immediateload(*keys, [recursion_depth])
Indicate that the given attribute should be loaded using an immediate load with a per-attribute SELECT statement.
joinedload(*keys, **kw)
Indicate that the given attribute should be loaded using joined eager loading.
lazyload(*keys)
Indicate that the given attribute should be loaded using “lazy” loading.
Load
Represents loader options which modify the state of a ORM-enabled Select or a legacy Query in order to affect how various mapped attributes are loaded.
noload(*keys)
Indicate that the given relationship attribute should remain unloaded.
raiseload(*keys, **kw)
Indicate that the given attribute should raise an error if accessed.
selectinload(*keys, [recursion_depth])
Indicate that the given attribute should be loaded using SELECT IN eager loading.
subqueryload(*keys)
Indicate that the given attribute should be loaded using subquery eager loading.
function sqlalchemy.orm.contains_eager(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) ? _AbstractLoad
Indicate that the given attribute should be eagerly loaded from columns stated manually in the query.
This function is part of the Load interface and supports both method-chained and standalone operation.
The option is used in conjunction with an explicit join that loads the desired rows, i.e.:
sess.query(Order).join(Order.user).options(contains_eager(Order.user))
The above query would join from the Order entity to its related User entity, and the returned Order objects would have the Order.user attribute pre-populated.
It may also be used for customizing the entries in an eagerly loaded collection; queries will normally want to use the Populate Existing execution option assuming the primary collection of parent objects may already have been loaded:
sess.query(User).join(User.addresses).filter(
    Address.email_address.like("%@aol.com")
).options(contains_eager(User.addresses)).populate_existing()
See the section Routing Explicit Joins/Statements into Eagerly Loaded Collections for complete usage details.
See also
Relationship Loading Techniques
Routing Explicit Joins/Statements into Eagerly Loaded Collections
function sqlalchemy.orm.defaultload(*keys: Literal['*'] | QueryableAttribute[Any]) ? _AbstractLoad
Indicate an attribute should load using its predefined loader style.
The behavior of this loading option is to not change the current loading style of the attribute, meaning that the previously configured one is used or, if no previous style was selected, the default loading will be used.
This method is used to link to other loader options further into a chain of attributes without altering the loader style of the links along the chain. For example, to set joined eager loading for an element of an element:
session.query(MyClass).options(
    defaultload(MyClass.someattribute).joinedload(
        MyOtherClass.someotherattribute
    )
)
defaultload() is also useful for setting column-level options on a related class, namely that of defer() and undefer():
session.scalars(
    select(MyClass).options(
        defaultload(MyClass.someattribute)
        .defer("some_column")
        .undefer("some_other_column")
    )
)
See also
Specifying Sub-Options with Load.options()
Load.options()
function sqlalchemy.orm.immediateload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) ? _AbstractLoad
Indicate that the given attribute should be loaded using an immediate load with a per-attribute SELECT statement.
The load is achieved using the “lazyloader” strategy and does not fire off any additional eager loaders.
The immediateload() option is superseded in general by the selectinload() option, which performs the same task more efficiently by emitting a SELECT for all loaded objects.
This function is part of the Load interface and supports both method-chained and standalone operation.
Parameters:
recursion_depth – 
optional int; when set to a positive integer in conjunction with a self-referential relationship, indicates “selectin” loading will continue that many levels deep automatically until no items are found.
Note
The immediateload.recursion_depth option currently supports only self-referential relationships. There is not yet an option to automatically traverse recursive structures with more than one relationship involved.
Warning
This parameter is new and experimental and should be treated as “alpha” status
Added in version 2.0: added immediateload.recursion_depth
See also
Relationship Loading Techniques
Select IN loading
function sqlalchemy.orm.joinedload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) ? _AbstractLoad
Indicate that the given attribute should be loaded using joined eager loading.
This function is part of the Load interface and supports both method-chained and standalone operation.
examples:
# joined-load the "orders" collection on "User"
select(User).options(joinedload(User.orders))

# joined-load Order.items and then Item.keywords
select(Order).options(joinedload(Order.items).joinedload(Item.keywords))

# lazily load Order.items, but when Items are loaded,
# joined-load the keywords collection
select(Order).options(lazyload(Order.items).joinedload(Item.keywords))
Parameters:
innerjoin – 
if True, indicates that the joined eager load should use an inner join instead of the default of left outer join:
select(Order).options(joinedload(Order.user, innerjoin=True))
In order to chain multiple eager joins together where some may be OUTER and others INNER, right-nested joins are used to link them:
select(A).options(
    joinedload(A.bs, innerjoin=False).joinedload(B.cs, innerjoin=True)
)
The above query, linking A.bs via “outer” join and B.cs via “inner” join would render the joins as “a LEFT OUTER JOIN (b JOIN c)”. When using older versions of SQLite (< 3.7.16), this form of JOIN is translated to use full subqueries as this syntax is otherwise not directly supported.
The innerjoin flag can also be stated with the term "unnested". This indicates that an INNER JOIN should be used, unless the join is linked to a LEFT OUTER JOIN to the left, in which case it will render as LEFT OUTER JOIN. For example, supposing A.bs is an outerjoin:
select(A).options(joinedload(A.bs).joinedload(B.cs, innerjoin="unnested"))
The above join will render as “a LEFT OUTER JOIN b LEFT OUTER JOIN c”, rather than as “a LEFT OUTER JOIN (b JOIN c)”.
Note
The “unnested” flag does not affect the JOIN rendered from a many-to-many association table, e.g. a table configured as relationship.secondary, to the target table; for correctness of results, these joins are always INNER and are therefore right-nested if linked to an OUTER join.
Note
The joins produced by joinedload() are anonymously aliased. The criteria by which the join proceeds cannot be modified, nor can the ORM-enabled Select or legacy Query refer to these joins in any way, including ordering. See The Zen of Joined Eager Loading for further detail.
To produce a specific SQL JOIN which is explicitly available, use Select.join() and Query.join(). To combine explicit JOINs with eager loading of collections, use contains_eager(); see Routing Explicit Joins/Statements into Eagerly Loaded Collections.
See also
Relationship Loading Techniques
Joined Eager Loading
function sqlalchemy.orm.lazyload(*keys: Literal['*'] | QueryableAttribute[Any]) ? _AbstractLoad
Indicate that the given attribute should be loaded using “lazy” loading.
This function is part of the Load interface and supports both method-chained and standalone operation.
See also
Relationship Loading Techniques
Lazy Loading
class sqlalchemy.orm.Load
Represents loader options which modify the state of a ORM-enabled Select or a legacy Query in order to affect how various mapped attributes are loaded.
The Load object is in most cases used implicitly behind the scenes when one makes use of a query option like joinedload(), defer(), or similar. It typically is not instantiated directly except for in some very specific cases.
See also
Per-Entity Wildcard Loading Strategies - illustrates an example where direct use of Load may be useful
Members
contains_eager(), defaultload(), defer(), get_children(), immediateload(), inherit_cache, joinedload(), lazyload(), load_only(), noload(), options(), process_compile_state(), process_compile_state_replaced_entities(), propagate_to_loaders, raiseload(), selectin_polymorphic(), selectinload(), subqueryload(), undefer(), undefer_group(), with_expression()
Class signature
class sqlalchemy.orm.Load (sqlalchemy.orm.strategy_options._AbstractLoad)
method sqlalchemy.orm.Load.contains_eager(attr: _AttrType, alias: _FromClauseArgument | None = None, _is_chain: bool = False, _propagate_to_loaders: bool = False) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.contains_eager method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the contains_eager() option applied.
See contains_eager() for usage examples.
method sqlalchemy.orm.Load.defaultload(attr: Literal['*'] | QueryableAttribute[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.defaultload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the defaultload() option applied.
See defaultload() for usage examples.
method sqlalchemy.orm.Load.defer(key: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.defer method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the defer() option applied.
See defer() for usage examples.
method sqlalchemy.orm.Load.get_children(*, omit_attrs: Tuple[str, ] = (), **kw: Any) ? Iterable[HasTraverseInternals]
inherited from the HasTraverseInternals.get_children() method of HasTraverseInternals
Return immediate child HasTraverseInternals elements of this HasTraverseInternals.
This is used for visit traversal.
**kw may contain flags that change the collection that is returned, for example to return a subset of items in order to cut down on larger traversals, or to return child items from a different context (such as schema-level collections instead of clause-level).
method sqlalchemy.orm.Load.immediateload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.immediateload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the immediateload() option applied.
See immediateload() for usage examples.
attribute sqlalchemy.orm.Load.inherit_cache: bool | None = None
inherited from the HasCacheKey.inherit_cache attribute of HasCacheKey
Indicate if this HasCacheKey instance should make use of the cache key generation scheme used by its immediate superclass.
The attribute defaults to None, which indicates that a construct has not yet taken into account whether or not its appropriate for it to participate in caching; this is functionally equivalent to setting the value to False, except that a warning is also emitted.
This flag can be set to True on a particular class, if the SQL that corresponds to the object does not change based on attributes which are local to this class, and not its superclass.
See also
Enabling Caching Support for Custom Constructs - General guideslines for setting the HasCacheKey.inherit_cache attribute for third-party or user defined SQL constructs.
method sqlalchemy.orm.Load.joinedload(attr: Literal['*'] | QueryableAttribute[Any], innerjoin: bool | None = None) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.joinedload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the joinedload() option applied.
See joinedload() for usage examples.
method sqlalchemy.orm.Load.lazyload(attr: Literal['*'] | QueryableAttribute[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.lazyload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the lazyload() option applied.
See lazyload() for usage examples.
method sqlalchemy.orm.Load.load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.load_only method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the load_only() option applied.
See load_only() for usage examples.
method sqlalchemy.orm.Load.noload(attr: Literal['*'] | QueryableAttribute[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.noload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the noload() option applied.
See noload() for usage examples.
method sqlalchemy.orm.Load.options(*opts: _AbstractLoad) ? Self
Apply a series of options as sub-options to this Load object.
E.g.:
query = session.query(Author)
query = query.options(
    joinedload(Author.book).options(
        load_only(Book.summary, Book.excerpt),
        joinedload(Book.citations).options(joinedload(Citation.author)),
    )
)
Parameters:
*opts – A series of loader option objects (ultimately Load objects) which should be applied to the path specified by this Load object.
Added in version 1.3.6.
See also
defaultload()
Specifying Sub-Options with Load.options()
method sqlalchemy.orm.Load.process_compile_state(compile_state: ORMCompileState) ? None
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state method of sqlalchemy.orm.strategy_options._AbstractLoad
Apply a modification to a given ORMCompileState.
This method is part of the implementation of a particular CompileStateOption and is only invoked internally when an ORM query is compiled.
method sqlalchemy.orm.Load.process_compile_state_replaced_entities(compile_state: ORMCompileState, mapper_entities: Sequence[_MapperEntity]) ? None
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state_replaced_entities method of sqlalchemy.orm.strategy_options._AbstractLoad
Apply a modification to a given ORMCompileState, given entities that were replaced by with_only_columns() or with_entities().
This method is part of the implementation of a particular CompileStateOption and is only invoked internally when an ORM query is compiled.
Added in version 1.4.19.
attribute sqlalchemy.orm.Load.propagate_to_loaders: bool
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.propagate_to_loaders attribute of sqlalchemy.orm.strategy_options._AbstractLoad
if True, indicate this option should be carried along to “secondary” SELECT statements that occur for relationship lazy loaders as well as attribute load / refresh operations.
method sqlalchemy.orm.Load.raiseload(attr: Literal['*'] | QueryableAttribute[Any], sql_only: bool = False) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.raiseload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the raiseload() option applied.
See raiseload() for usage examples.
method sqlalchemy.orm.Load.selectin_polymorphic(classes: Iterable[Type[Any]]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.selectin_polymorphic method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the selectin_polymorphic() option applied.
See selectin_polymorphic() for usage examples.
method sqlalchemy.orm.Load.selectinload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.selectinload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the selectinload() option applied.
See selectinload() for usage examples.
method sqlalchemy.orm.Load.subqueryload(attr: Literal['*'] | QueryableAttribute[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.subqueryload method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the subqueryload() option applied.
See subqueryload() for usage examples.
method sqlalchemy.orm.Load.undefer(key: Literal['*'] | QueryableAttribute[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.undefer method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the undefer() option applied.
See undefer() for usage examples.
method sqlalchemy.orm.Load.undefer_group(name: str) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.undefer_group method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the undefer_group() option applied.
See undefer_group() for usage examples.
method sqlalchemy.orm.Load.with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) ? Self
inherited from the sqlalchemy.orm.strategy_options._AbstractLoad.with_expression method of sqlalchemy.orm.strategy_options._AbstractLoad
Produce a new Load object with the with_expression() option applied.
See with_expression() for usage examples.
function sqlalchemy.orm.noload(*keys: Literal['*'] | QueryableAttribute[Any]) ? _AbstractLoad
Indicate that the given relationship attribute should remain unloaded.
The relationship attribute will return None when accessed without producing any loading effect.
This function is part of the Load interface and supports both method-chained and standalone operation.
noload() applies to relationship() attributes only.
Legacy Feature
The noload() option is legacy. As it forces collections to be empty, which invariably leads to non-intuitive and difficult to predict results. There are no legitimate uses for this option in modern SQLAlchemy.
See also
Relationship Loading Techniques
function sqlalchemy.orm.raiseload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) ? _AbstractLoad
Indicate that the given attribute should raise an error if accessed.
A relationship attribute configured with raiseload() will raise an InvalidRequestError upon access. The typical way this is useful is when an application is attempting to ensure that all relationship attributes that are accessed in a particular context would have been already loaded via eager loading. Instead of having to read through SQL logs to ensure lazy loads aren’t occurring, this strategy will cause them to raise immediately.
raiseload() applies to relationship() attributes only. In order to apply raise-on-SQL behavior to a column-based attribute, use the defer.raiseload parameter on the defer() loader option.
Parameters:
sql_only – if True, raise only if the lazy load would emit SQL, but not if it is only checking the identity map, or determining that the related value should just be None due to missing keys. When False, the strategy will raise for all varieties of relationship loading.
This function is part of the Load interface and supports both method-chained and standalone operation.
See also
Relationship Loading Techniques
Preventing unwanted lazy loads using raiseload
Using raiseload to prevent deferred column loads
function sqlalchemy.orm.selectinload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) ? _AbstractLoad
Indicate that the given attribute should be loaded using SELECT IN eager loading.
This function is part of the Load interface and supports both method-chained and standalone operation.
examples:
# selectin-load the "orders" collection on "User"
select(User).options(selectinload(User.orders))

# selectin-load Order.items and then Item.keywords
select(Order).options(
    selectinload(Order.items).selectinload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# selectin-load the keywords collection
select(Order).options(lazyload(Order.items).selectinload(Item.keywords))
Parameters:
recursion_depth – 
optional int; when set to a positive integer in conjunction with a self-referential relationship, indicates “selectin” loading will continue that many levels deep automatically until no items are found.
Note
The selectinload.recursion_depth option currently supports only self-referential relationships. There is not yet an option to automatically traverse recursive structures with more than one relationship involved.
Additionally, the selectinload.recursion_depth parameter is new and experimental and should be treated as “alpha” status for the 2.0 series.
Added in version 2.0: added selectinload.recursion_depth
See also
Relationship Loading Techniques
Select IN loading
function sqlalchemy.orm.subqueryload(*keys: Literal['*'] | QueryableAttribute[Any]) ? _AbstractLoad
Indicate that the given attribute should be loaded using subquery eager loading.
This function is part of the Load interface and supports both method-chained and standalone operation.
examples:
# subquery-load the "orders" collection on "User"
select(User).options(subqueryload(User.orders))

# subquery-load Order.items and then Item.keywords
select(Order).options(
    subqueryload(Order.items).subqueryload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# subquery-load the keywords collection
select(Order).options(lazyload(Order.items).subqueryload(Item.keywords))
See also
Relationship Loading Techniques
Subquery Eager Loading


ORM API Features for Querying
ORM Loader Options
Loader options are objects which, when passed to the Select.options() method of a Select object or similar SQL construct, affect the loading of both column and relationship-oriented attributes. The majority of loader options descend from the Load hierarchy. For a complete overview of using loader options, see the linked sections below.
See also
* Column Loading Options - details mapper and loading options that affect how column and SQL-expression mapped attributes are loaded
* Relationship Loading Techniques - details relationship and loading options that affect how relationship() mapped attributes are loaded
ORM Execution Options
ORM-level execution options are keyword options that may be associated with a statement execution using either the Session.execute.execution_options parameter, which is a dictionary argument accepted by Session methods such as Session.execute() and Session.scalars(), or by associating them directly with the statement to be invoked itself using the Executable.execution_options() method, which accepts them as arbitrary keyword arguments.
ORM-level options are distinct from the Core level execution options documented at Connection.execution_options(). It’s important to note that the ORM options discussed below are not compatible with Core level methods Connection.execution_options() or Engine.execution_options(); the options are ignored at this level, even if the Engine or Connection is associated with the Session in use.
Within this section, the Executable.execution_options() method style will be illustrated for examples.
Populate Existing
The populate_existing execution option ensures that, for all rows loaded, the corresponding instances in the Session will be fully refreshed – erasing any existing data within the objects (including pending changes) and replacing with the data loaded from the result.
Example use looks like:
stmt = select(User).execution_options(populate_existing=True)
result = session.execute(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account

Normally, ORM objects are only loaded once, and if they are matched up to the primary key in a subsequent result row, the row is not applied to the object. This is both to preserve pending, unflushed changes on the object as well as to avoid the overhead and complexity of refreshing data which is already there. The Session assumes a default working model of a highly isolated transaction, and to the degree that data is expected to change within the transaction outside of the local changes being made, those use cases would be handled using explicit steps such as this method.
Using populate_existing, any set of objects that matches a query can be refreshed, and it also allows control over relationship loader options. E.g. to refresh an instance while also refreshing a related set of objects:
stmt = (
    select(User)
    .where(User.name.in_(names))
    .execution_options(populate_existing=True)
    .options(selectinload(User.addresses))
)
# will refresh all matching User objects as well as the related
# Address objects
users = session.execute(stmt).scalars().all()
Another use case for populate_existing is in support of various attribute loading features that can change how an attribute is loaded on a per-query basis. Options for which this apply include:
* The with_expression() option
* The PropComparator.and_() method that can modify what a loader strategy loads
* The contains_eager() option
* The with_loader_criteria() option
* The load_only() option to select what attributes to refresh
The populate_existing execution option is equvialent to the Query.populate_existing() method in 1.x style ORM queries.
See also
I’m re-loading data with my Session but it isn’t seeing changes that I committed elsewhere - in Frequently Asked Questions
Refreshing / Expiring - in the ORM Session documentation
Autoflush
This option, when passed as False, will cause the Session to not invoke the “autoflush” step. It is equivalent to using the Session.no_autoflush context manager to disable autoflush:
stmt = select(User).execution_options(autoflush=False)
session.execute(stmt)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account

This option will also work on ORM-enabled Update and Delete queries.
The autoflush execution option is equvialent to the Query.autoflush() method in 1.x style ORM queries.
See also
Flushing
Fetching Large Result Sets with Yield Per
The yield_per execution option is an integer value which will cause the Result to buffer only a limited number of rows and/or ORM objects at a time, before making data available to the client.
Normally, the ORM will fetch all rows immediately, constructing ORM objects for each and assembling those objects into a single buffer, before passing this buffer to the Result object as a source of rows to be returned. The rationale for this behavior is to allow correct behavior for features such as joined eager loading, uniquifying of results, and the general case of result handling logic that relies upon the identity map maintaining a consistent state for every object in a result set as it is fetched.
The purpose of the yield_per option is to change this behavior so that the ORM result set is optimized for iteration through very large result sets (e.g. > 10K rows), where the user has determined that the above patterns don’t apply. When yield_per is used, the ORM will instead batch ORM results into sub-collections and yield rows from each sub-collection individually as the Result object is iterated, so that the Python interpreter doesn’t need to declare very large areas of memory which is both time consuming and leads to excessive memory use. The option affects both the way the database cursor is used as well as how the ORM constructs rows and objects to be passed to the Result.
Tip
From the above, it follows that the Result must be consumed in an iterable fashion, that is, using iteration such as for row in result or using partial row methods such as Result.fetchmany() or Result.partitions(). Calling Result.all() will defeat the purpose of using yield_per.
Using yield_per is equivalent to making use of both the Connection.execution_options.stream_results execution option, which selects for server side cursors to be used by the backend if supported, and the Result.yield_per() method on the returned Result object, which establishes a fixed size of rows to be fetched as well as a corresponding limit to how many ORM objects will be constructed at once.
Tip
yield_per is now available as a Core execution option as well, described in detail at Using Server Side Cursors (a.k.a. stream results). This section details the use of yield_per as an execution option with an ORM Session. The option behaves as similarly as possible in both contexts.
When used with the ORM, yield_per must be established either via the Executable.execution_options() method on the given statement or by passing it to the Session.execute.execution_options parameter of Session.execute() or other similar Session method such as Session.scalars(). Typical use for fetching ORM objects is illustrated below:
stmt = select(User).execution_options(yield_per=10)
for user_obj in session.scalars(stmt):
    print(user_obj)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
[] ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')

# rows continue 
The above code is equivalent to the example below, which uses Connection.execution_options.stream_results and Connection.execution_options.max_row_buffer Core-level execution options in conjunction with the Result.yield_per() method of Result:
# equivalent code
stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
for user_obj in session.scalars(stmt).yield_per(10):
    print(user_obj)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
[] ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')

# rows continue 
yield_per is also commonly used in combination with the Result.partitions() method, which will iterate rows in grouped partitions. The size of each partition defaults to the integer value passed to yield_per, as in the below example:
stmt = select(User).execution_options(yield_per=10)
for partition in session.scalars(stmt).partitions():
    for user_obj in partition:
        print(user_obj)
SELECT user_account.id, user_account.name, user_account.fullname
FROM user_account
[] ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')

# rows continue 
The yield_per execution option is not compatible with “subquery” eager loading loading or “joined” eager loading when using collections. It is potentially compatible with “select in” eager loading , provided the database driver supports multiple, independent cursors.
Additionally, the yield_per execution option is not compatible with the Result.unique() method; as this method relies upon storing a complete set of identities for all rows, it would necessarily defeat the purpose of using yield_per which is to handle an arbitrarily large number of rows.
Changed in version 1.4.6: An exception is raised when ORM rows are fetched from a Result object that makes use of the Result.unique() filter, at the same time as the yield_per execution option is used.
When using the legacy Query object with 1.x style ORM use, the Query.yield_per() method will have the same result as that of the yield_per execution option.
See also
Using Server Side Cursors (a.k.a. stream results)
Identity Token
Deep Alchemy
This option is an advanced-use feature mostly intended to be used with the Horizontal Sharding extension. For typical cases of loading objects with identical primary keys from different “shards” or partitions, consider using individual Session objects per shard first.
The “identity token” is an arbitrary value that can be associated within the identity key of newly loaded objects. This element exists first and foremost to support extensions which perform per-row “sharding”, where objects may be loaded from any number of replicas of a particular database table that nonetheless have overlapping primary key values. The primary consumer of “identity token” is the Horizontal Sharding extension, which supplies a general framework for persisting objects among multiple “shards” of a particular database table.
The identity_token execution option may be used on a per-query basis to directly affect this token. Using it directly, one can populate a Session with multiple instances of an object that have the same primary key and source table, but different “identities”.
One such example is to populate a Session with objects that come from same-named tables in different schemas, using the Translation of Schema Names feature which can affect the choice of schema within the scope of queries. Given a mapping as:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class MyTable(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
The default “schema” name for the class above is None, meaning, no schema qualification will be written into SQL statements. However, if we make use of Connection.execution_options.schema_translate_map, mapping None to an alternate schema, we can place instances of MyTable into two different schemas:
engine = create_engine(
    "postgresql+psycopg://scott:tiger@localhost/test",
)

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema"})
) as sess:
    sess.add(MyTable(name="this is schema one"))
    sess.commit()

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema_2"})
) as sess:
    sess.add(MyTable(name="this is schema two"))
    sess.commit()
The above two blocks create a Session object linked to a different schema translate map each time, and an instance of MyTable is persisted into both test_schema.my_table as well as test_schema_2.my_table.
The Session objects above are independent. If we wanted to persist both objects in one transaction, we would need to use the Horizontal Sharding extension to do this.
However, we can illustrate querying for these objects in one session as follows:
with Session(engine) as sess:
    obj1 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema"},
            identity_token="test_schema",
        )
    )
    obj2 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema_2"},
            identity_token="test_schema_2",
        )
    )
Both obj1 and obj2 are distinct from each other. However, they both refer to primary key id 1 for the MyTable class, yet are distinct. This is how the identity_token comes into play, which we can see in the inspection of each object, where we look at InstanceState.key to view the two distinct identity tokens:
from sqlalchemy import inspect
inspect(obj1).key
(<class '__main__.MyTable'>, (1,), 'test_schema')
inspect(obj2).key
(<class '__main__.MyTable'>, (1,), 'test_schema_2')
The above logic takes place automatically when using the Horizontal Sharding extension.
Added in version 2.0.0rc1: - added the identity_token ORM level execution option.
See also
Horizontal Sharding - in the ORM Examples section. See the script separate_schema_translates.py for a demonstration of the above use case using the full sharding API.
Inspecting entities and columns from ORM-enabled SELECT and DML statements
The select() construct, as well as the insert(), update() and delete() constructs (for the latter DML constructs, as of SQLAlchemy 1.4.33), all support the ability to inspect the entities in which these statements are created against, as well as the columns and datatypes that would be returned in a result set.
For a Select object, this information is available from the Select.column_descriptions attribute. This attribute operates in the same way as the legacy Query.column_descriptions attribute. The format returned is a list of dictionaries:
from pprint import pprint
user_alias = aliased(User, name="user2")
stmt = select(User, User.id, user_alias)
pprint(stmt.column_descriptions)
[{'aliased': False,
  'entity': <class 'User'>,
  'expr': <class 'User'>,
  'name': 'User',
  'type': <class 'User'>},
 {'aliased': False,
  'entity': <class 'User'>,
  'expr': <.InstrumentedAttribute object at >,
  'name': 'id',
  'type': Integer()},
 {'aliased': True,
  'entity': <AliasedClass ; User>,
  'expr': <AliasedClass ; User>,
  'name': 'user2',
  'type': <class 'User'>}]
When Select.column_descriptions is used with non-ORM objects such as plain Table or Column objects, the entries will contain basic information about individual columns returned in all cases:
stmt = select(user_table, address_table.c.id)
pprint(stmt.column_descriptions)
[{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
  'name': 'id',
  'type': Integer()},
 {'expr': Column('name', String(), table=<user_account>, nullable=False),
  'name': 'name',
  'type': String()},
 {'expr': Column('fullname', String(), table=<user_account>),
  'name': 'fullname',
  'type': String()},
 {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
  'name': 'id_1',
  'type': Integer()}]
Changed in version 1.4.33: The Select.column_descriptions attribute now returns a value when used against a Select that is not ORM-enabled. Previously, this would raise NotImplementedError.
For insert(), update() and delete() constructs, there are two separate attributes. One is UpdateBase.entity_description which returns information about the primary ORM entity and database table which the DML construct would be affecting:
from sqlalchemy import update
stmt = update(User).values(name="somename").returning(User.id)
pprint(stmt.entity_description)
{'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'table': Table('user_account', ),
 'type': <class 'User'>}
Tip
The UpdateBase.entity_description includes an entry "table" which is actually the table to be inserted, updated or deleted by the statement, which is not always the same as the SQL “selectable” to which the class may be mapped. For example, in a joined-table inheritance scenario, "table" will refer to the local table for the given entity.
The other is UpdateBase.returning_column_descriptions which delivers information about the columns present in the RETURNING collection in a manner roughly similar to that of Select.column_descriptions:
pprint(stmt.returning_column_descriptions)
[{'aliased': False,
  'entity': <class 'User'>,
  'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute >,
  'name': 'id',
  'type': Integer()}]
Added in version 1.4.33: Added the UpdateBase.entity_description and UpdateBase.returning_column_descriptions attributes.
Additional ORM API Constructs
Object Name
Description
aliased(element[, alias, name, flat, ])
Produce an alias of the given element, usually an AliasedClass instance.
AliasedClass
Represents an “aliased” form of a mapped class for usage with Query.
AliasedInsp
Provide an inspection interface for an AliasedClass object.
Bundle
A grouping of SQL expressions that are returned by a Query under one namespace.
join(left, right[, onclause, isouter, ])
Produce an inner join between left and right clauses.
outerjoin(left, right[, onclause, full])
Produce a left outer join between left and right clauses.
with_loader_criteria(entity_or_base, where_criteria[, loader_only, include_aliases, ])
Add additional WHERE criteria to the load for all occurrences of a particular entity.
with_parent(instance, prop[, from_entity])
Create filtering criterion that relates this query’s primary entity to the given related instance, using established relationship() configuration.
function sqlalchemy.orm.aliased(element: _EntityType[_O] | FromClause, alias: FromClause | None = None, name: str | None = None, flat: bool = False, adapt_on_names: bool = False) ? AliasedClass[_O] | FromClause | AliasedType[_O]
Produce an alias of the given element, usually an AliasedClass instance.
E.g.:
my_alias = aliased(MyClass)

stmt = select(MyClass, my_alias).filter(MyClass.id > my_alias.id)
result = session.execute(stmt)
The aliased() function is used to create an ad-hoc mapping of a mapped class to a new selectable. By default, a selectable is generated from the normally mapped selectable (typically a Table ) using the FromClause.alias() method. However, aliased() can also be used to link the class to a new select() statement. Also, the with_polymorphic() function is a variant of aliased() that is intended to specify a so-called “polymorphic selectable”, that corresponds to the union of several joined-inheritance subclasses at once.
For convenience, the aliased() function also accepts plain FromClause constructs, such as a Table or select() construct. In those cases, the FromClause.alias() method is called on the object and the new Alias object returned. The returned Alias is not ORM-mapped in this case.
See also
ORM Entity Aliases - in the SQLAlchemy Unified Tutorial
Selecting ORM Aliases - in the ORM Querying Guide
Parameters:
* element – element to be aliased. Is normally a mapped class, but for convenience can also be a FromClause element.
* alias – Optional selectable unit to map the element to. This is usually used to link the object to a subquery, and should be an aliased select construct as one would produce from the Query.subquery() method or the Select.subquery() or Select.alias() methods of the select() construct.
* name – optional string name to use for the alias, if not specified by the alias parameter. The name, among other things, forms the attribute name that will be accessible via tuples returned by a Query object. Not supported when creating aliases of Join objects.
* flat – 
Boolean, will be passed through to the FromClause.alias() call so that aliases of Join objects will alias the individual tables inside the join, rather than creating a subquery. This is generally supported by all modern databases with regards to right-nested joins and generally produces more efficient queries.
When aliased.flat is combined with aliased.name, the resulting joins will alias individual tables using a naming scheme similar to <prefix>_<tablename>. This naming scheme is for visibility / debugging purposes only and the specific scheme is subject to change without notice.
Added in version 2.0.32: added support for combining aliased.name with aliased.flat. Previously, this would raise NotImplementedError.
* adapt_on_names – 
if True, more liberal “matching” will be used when mapping the mapped columns of the ORM entity to those of the given selectable - a name-based match will be performed if the given selectable doesn’t otherwise have a column that corresponds to one on the entity. The use case for this is when associating an entity with some derived selectable such as one that uses aggregate functions:
class UnitPrice(Base):
    __tablename__ = "unit_price"
    
    unit_id = Column(Integer)
    price = Column(Numeric)


aggregated_unit_price = (
    Session.query(func.sum(UnitPrice.price).label("price"))
    .group_by(UnitPrice.unit_id)
    .subquery()
)

aggregated_unit_price = aliased(
    UnitPrice, alias=aggregated_unit_price, adapt_on_names=True
)
* Above, functions on aggregated_unit_price which refer to .price will return the func.sum(UnitPrice.price).label('price') column, as it is matched on the name “price”. Ordinarily, the “price” function wouldn’t have any “column correspondence” to the actual UnitPrice.price column as it is not a proxy of the original.
class sqlalchemy.orm.util.AliasedClass
Represents an “aliased” form of a mapped class for usage with Query.
The ORM equivalent of a alias() construct, this object mimics the mapped class using a __getattr__ scheme and maintains a reference to a real Alias object.
A primary purpose of AliasedClass is to serve as an alternate within a SQL statement generated by the ORM, such that an existing mapped entity can be used in multiple contexts. A simple example:
# find all pairs of users with the same name
user_alias = aliased(User)
session.query(User, user_alias).join(
    (user_alias, User.id > user_alias.id)
).filter(User.name == user_alias.name)
AliasedClass is also capable of mapping an existing mapped class to an entirely new selectable, provided this selectable is column- compatible with the existing mapped selectable, and it can also be configured in a mapping as the target of a relationship(). See the links below for examples.
The AliasedClass object is constructed typically using the aliased() function. It also is produced with additional configuration when using the with_polymorphic() function.
The resulting object is an instance of AliasedClass. This object implements an attribute scheme which produces the same attribute and method interface as the original mapped class, allowing AliasedClass to be compatible with any attribute technique which works on the original class, including hybrid attributes (see Hybrid Attributes).
The AliasedClass can be inspected for its underlying Mapper, aliased selectable, and other information using inspect():
from sqlalchemy import inspect

my_alias = aliased(MyClass)
insp = inspect(my_alias)
The resulting inspection object is an instance of AliasedInsp.
See also
aliased()
with_polymorphic()
Relationship to Aliased Class
Row-Limited Relationships with Window Functions
Class signature
class sqlalchemy.orm.AliasedClass (sqlalchemy.inspection.Inspectable, sqlalchemy.orm.ORMColumnsClauseRole)
class sqlalchemy.orm.util.AliasedInsp
Provide an inspection interface for an AliasedClass object.
The AliasedInsp object is returned given an AliasedClass using the inspect() function:
from sqlalchemy import inspect
from sqlalchemy.orm import aliased

my_alias = aliased(MyMappedClass)
insp = inspect(my_alias)
Attributes on AliasedInsp include:
* entity - the AliasedClass represented.
* mapper - the Mapper mapping the underlying class.
* selectable - the Alias construct which ultimately represents an aliased Table or Select construct.
* name - the name of the alias. Also is used as the attribute name when returned in a result tuple from Query.
* with_polymorphic_mappers - collection of Mapper objects indicating all those mappers expressed in the select construct for the AliasedClass.
* polymorphic_on - an alternate column or SQL expression which will be used as the “discriminator” for a polymorphic load.
See also
Runtime Inspection API
Class signature
class sqlalchemy.orm.AliasedInsp (sqlalchemy.orm.ORMEntityColumnsClauseRole, sqlalchemy.orm.ORMFromClauseRole, sqlalchemy.sql.cache_key.HasCacheKey, sqlalchemy.orm.base.InspectionAttr, sqlalchemy.util.langhelpers.MemoizedSlots, sqlalchemy.inspection.Inspectable, typing.Generic)
class sqlalchemy.orm.Bundle
A grouping of SQL expressions that are returned by a Query under one namespace.
The Bundle essentially allows nesting of the tuple-based results returned by a column-oriented Query object. It also is extensible via simple subclassing, where the primary capability to override is that of how the set of expressions should be returned, allowing post-processing as well as custom return types, without involving ORM identity-mapped classes.
See also
Grouping Selected Attributes with Bundles
Members
__init__(), c, columns, create_row_processor(), is_aliased_class, is_bundle, is_clause_element, is_mapper, label(), single_entity
Class signature
class sqlalchemy.orm.Bundle (sqlalchemy.orm.ORMColumnsClauseRole, sqlalchemy.sql.annotation.SupportsCloneAnnotations, sqlalchemy.sql.cache_key.MemoizedHasCacheKey, sqlalchemy.inspection.Inspectable, sqlalchemy.orm.base.InspectionAttr)
method sqlalchemy.orm.Bundle.__init__(name: str, *exprs: _ColumnExpressionArgument[Any], **kw: Any)
Construct a new Bundle.
e.g.:
bn = Bundle("mybundle", MyClass.x, MyClass.y)

for row in session.query(bn).filter(bn.c.x == 5).filter(bn.c.y == 4):
    print(row.mybundle.x, row.mybundle.y)
Parameters:
* name – name of the bundle.
* *exprs – columns or SQL expressions comprising the bundle.
* single_entity=False – if True, rows for this Bundle can be returned as a “single entity” outside of any enclosing tuple in the same manner as a mapped entity.
attribute sqlalchemy.orm.Bundle.c: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
An alias for Bundle.columns.
attribute sqlalchemy.orm.Bundle.columns: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
A namespace of SQL expressions referred to by this Bundle.
e.g.:
bn = Bundle("mybundle", MyClass.x, MyClass.y)

q = sess.query(bn).filter(bn.c.x == 5)
Nesting of bundles is also supported:
b1 = Bundle(
    "b1",
    Bundle("b2", MyClass.a, MyClass.b),
    Bundle("b3", MyClass.x, MyClass.y),
)

q = sess.query(b1).filter(b1.c.b2.c.a == 5).filter(b1.c.b3.c.y == 9)
See also
Bundle.c
method sqlalchemy.orm.Bundle.create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) ? Callable[[Row[Any]], Any]
Produce the “row processing” function for this Bundle.
May be overridden by subclasses to provide custom behaviors when results are fetched. The method is passed the statement object and a set of “row processor” functions at query execution time; these processor functions when given a result row will return the individual attribute value, which can then be adapted into any kind of return data structure.
The example below illustrates replacing the usual Row return structure with a straight Python dictionary:
from sqlalchemy.orm import Bundle


class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
        "Override create_row_processor to return values as dictionaries"

        def proc(row):
            return dict(zip(labels, (proc(row) for proc in procs)))

        return proc
A result from the above Bundle will return dictionary values:
bn = DictBundle("mybundle", MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == "d1"):
    print(row.mybundle["data1"], row.mybundle["data2"])
attribute sqlalchemy.orm.Bundle.is_aliased_class = False
True if this object is an instance of AliasedClass.
attribute sqlalchemy.orm.Bundle.is_bundle = True
True if this object is an instance of Bundle.
attribute sqlalchemy.orm.Bundle.is_clause_element = False
True if this object is an instance of ClauseElement.
attribute sqlalchemy.orm.Bundle.is_mapper = False
True if this object is an instance of Mapper.
method sqlalchemy.orm.Bundle.label(name)
Provide a copy of this Bundle passing a new label.
attribute sqlalchemy.orm.Bundle.single_entity = False
If True, queries for a single Bundle will be returned as a single entity, rather than an element within a keyed tuple.
function sqlalchemy.orm.with_loader_criteria(entity_or_base: _EntityType[Any], where_criteria: _ColumnExpressionArgument[bool] | Callable[[Any], _ColumnExpressionArgument[bool]], loader_only: bool = False, include_aliases: bool = False, propagate_to_loaders: bool = True, track_closure_variables: bool = True) ? LoaderCriteriaOption
Add additional WHERE criteria to the load for all occurrences of a particular entity.
Added in version 1.4.
The with_loader_criteria() option is intended to add limiting criteria to a particular kind of entity in a query, globally, meaning it will apply to the entity as it appears in the SELECT query as well as within any subqueries, join conditions, and relationship loads, including both eager and lazy loaders, without the need for it to be specified in any particular part of the query. The rendering logic uses the same system used by single table inheritance to ensure a certain discriminator is applied to a table.
E.g., using 2.0-style queries, we can limit the way the User.addresses collection is loaded, regardless of the kind of loading used:
from sqlalchemy.orm import with_loader_criteria

stmt = select(User).options(
    selectinload(User.addresses),
    with_loader_criteria(Address, Address.email_address != "foo"),
)
Above, the “selectinload” for User.addresses will apply the given filtering criteria to the WHERE clause.
Another example, where the filtering will be applied to the ON clause of the join, in this example using 1.x style queries:
q = (
    session.query(User)
    .outerjoin(User.addresses)
    .options(with_loader_criteria(Address, Address.email_address != "foo"))
)
The primary purpose of with_loader_criteria() is to use it in the SessionEvents.do_orm_execute() event handler to ensure that all occurrences of a particular entity are filtered in a certain way, such as filtering for access control roles. It also can be used to apply criteria to relationship loads. In the example below, we can apply a certain set of rules to all queries emitted by a particular Session:
session = Session(bind=engine)


@event.listens_for("do_orm_execute", session)
def _add_filtering_criteria(execute_state):

    if (
        execute_state.is_select
        and not execute_state.is_column_load
        and not execute_state.is_relationship_load
    ):
        execute_state.statement = execute_state.statement.options(
            with_loader_criteria(
                SecurityRole,
                lambda cls: cls.role.in_(["some_role"]),
                include_aliases=True,
            )
        )
In the above example, the SessionEvents.do_orm_execute() event will intercept all queries emitted using the Session. For those queries which are SELECT statements and are not attribute or relationship loads a custom with_loader_criteria() option is added to the query. The with_loader_criteria() option will be used in the given statement and will also be automatically propagated to all relationship loads that descend from this query.
The criteria argument given is a lambda that accepts a cls argument. The given class will expand to include all mapped subclass and need not itself be a mapped class.
Tip
When using with_loader_criteria() option in conjunction with the contains_eager() loader option, it’s important to note that with_loader_criteria() only affects the part of the query that determines what SQL is rendered in terms of the WHERE and FROM clauses. The contains_eager() option does not affect the rendering of the SELECT statement outside of the columns clause, so does not have any interaction with the with_loader_criteria() option. However, the way things “work” is that contains_eager() is meant to be used with a query that is already selecting from the additional entities in some way, where with_loader_criteria() can apply it’s additional criteria.
In the example below, assuming a mapping relationship as A -> A.bs -> B, the given with_loader_criteria() option will affect the way in which the JOIN is rendered:
stmt = (
    select(A)
    .join(A.bs)
    .options(contains_eager(A.bs), with_loader_criteria(B, B.flag == 1))
)
Above, the given with_loader_criteria() option will affect the ON clause of the JOIN that is specified by .join(A.bs), so is applied as expected. The contains_eager() option has the effect that columns from B are added to the columns clause:
SELECT
    b.id, b.a_id, b.data, b.flag,
    a.id AS id_1,
    a.data AS data_1
FROM a JOIN b ON a.id = b.a_id AND b.flag = :flag_1
The use of the contains_eager() option within the above statement has no effect on the behavior of the with_loader_criteria() option. If the contains_eager() option were omitted, the SQL would be the same as regards the FROM and WHERE clauses, where with_loader_criteria() continues to add its criteria to the ON clause of the JOIN. The addition of contains_eager() only affects the columns clause, in that additional columns against b are added which are then consumed by the ORM to produce B instances.
Warning
The use of a lambda inside of the call to with_loader_criteria() is only invoked once per unique class. Custom functions should not be invoked within this lambda. See Using Lambdas to add significant speed gains to statement production for an overview of the “lambda SQL” feature, which is for advanced use only.
Parameters:
* entity_or_base – a mapped class, or a class that is a super class of a particular set of mapped classes, to which the rule will apply.
* where_criteria – 
a Core SQL expression that applies limiting criteria. This may also be a “lambda:” or Python function that accepts a target class as an argument, when the given class is a base with many different mapped subclasses.
Note
To support pickling, use a module-level Python function to produce the SQL expression instead of a lambda or a fixed SQL expression, which tend to not be picklable.
* include_aliases – if True, apply the rule to aliased() constructs as well.
* propagate_to_loaders – 
defaults to True, apply to relationship loaders such as lazy loaders. This indicates that the option object itself including SQL expression is carried along with each loaded instance. Set to False to prevent the object from being assigned to individual instances.
See also
ORM Query Events - includes examples of using with_loader_criteria().
Adding global WHERE / ON criteria - basic example on how to combine with_loader_criteria() with the SessionEvents.do_orm_execute() event.
* track_closure_variables – 
when False, closure variables inside of a lambda expression will not be used as part of any cache key. This allows more complex expressions to be used inside of a lambda expression but requires that the lambda ensures it returns the identical SQL every time given a particular class.
Added in version 1.4.0b2.
function sqlalchemy.orm.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) ? _ORMJoin
Produce an inner join between left and right clauses.
join() is an extension to the core join interface provided by join(), where the left and right selectable may be not only core selectable objects such as Table, but also mapped classes or AliasedClass instances. The “on” clause can be a SQL expression or an ORM mapped attribute referencing a configured relationship().
join() is not commonly needed in modern usage, as its functionality is encapsulated within that of the Select.join() and Query.join() methods. which feature a significant amount of automation beyond join() by itself. Explicit use of join() with ORM-enabled SELECT statements involves use of the Select.select_from() method, as in:
from sqlalchemy.orm import join

stmt = (
    select(User)
    .select_from(join(User, Address, User.addresses))
    .filter(Address.email_address == "foo@bar.com")
)
In modern SQLAlchemy the above join can be written more succinctly as:
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address == "foo@bar.com")
)
Warning
using join() directly may not work properly with modern ORM options such as with_loader_criteria(). It is strongly recommended to use the idiomatic join patterns provided by methods such as Select.join() and Select.join_from() when creating ORM joins.
See also
Joins - in the ORM Querying Guide for background on idiomatic ORM join patterns
function sqlalchemy.orm.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) ? _ORMJoin
Produce a left outer join between left and right clauses.
This is the “outer join” version of the join() function, featuring the same behavior except that an OUTER JOIN is generated. See that function’s documentation for other usage details.
function sqlalchemy.orm.with_parent(instance: object, prop: attributes.QueryableAttribute[Any], from_entity: _EntityType[Any] | None = None) ? ColumnElement[bool]
Create filtering criterion that relates this query’s primary entity to the given related instance, using established relationship() configuration.
E.g.:
stmt = select(Address).where(with_parent(some_user, User.addresses))
The SQL rendered is the same as that rendered when a lazy loader would fire off from the given parent on that attribute, meaning that the appropriate state is taken from the parent object in Python without the need to render joins to the parent table in the rendered statement.
The given property may also make use of PropComparator.of_type() to indicate the left side of the criteria:
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(with_parent(u1, User.addresses.of_type(a2)))
The above use is equivalent to using the from_entity() argument:
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
    with_parent(u1, User.addresses, from_entity=a2)
)
Parameters:
* instance – An instance which has some relationship().
* property – Class-bound attribute, which indicates what relationship from the instance should be used to reconcile the parent/child relationship.
* from_entity – 
Entity in which to consider as the left side. This defaults to the “zero” entity of the Query itself.
Added in version 1.2.


Legacy Query API
About the Legacy Query API
This page contains the Python generated documentation for the Query construct, which for many years was the sole SQL interface when working with the SQLAlchemy ORM. As of version 2.0, an all new way of working is now the standard approach, where the same select() construct that works for Core works just as well for the ORM, providing a consistent interface for building queries.
For any application that is built on the SQLAlchemy ORM prior to the 2.0 API, the Query API will usually represents the vast majority of database access code within an application, and as such the majority of the Query API is not being removed from SQLAlchemy. The Query object behind the scenes now translates itself into a 2.0 style select() object when the Query object is executed, so it now is just a very thin adapter API.
For a guide to migrating an application based on Query to 2.0 style, see 2.0 Migration - ORM Usage.
For an introduction to writing SQL for ORM objects in the 2.0 style, start with the SQLAlchemy Unified Tutorial. Additional reference for 2.0 style querying is at ORM Querying Guide.
The Query Object
Query is produced in terms of a given Session, using the Session.query() method:
q = session.query(SomeMappedClass)
Following is the full interface for the Query object.
Object Name
Description
Query
ORM-level SQL construction object.
class sqlalchemy.orm.Query
ORM-level SQL construction object.
Legacy Feature
The ORM Query object is a legacy construct as of SQLAlchemy 2.0. See the notes at the top of Legacy Query API for an overview, including links to migration documentation.
Query objects are normally initially generated using the Session.query() method of Session, and in less common cases by instantiating the Query directly and associating with a Session using the Query.with_session() method.
Members
__init__(), add_column(), add_columns(), add_entity(), all(), apply_labels(), as_scalar(), autoflush(), column_descriptions, correlate(), count(), cte(), delete(), distinct(), enable_assertions(), enable_eagerloads(), except_(), except_all(), execution_options(), exists(), filter(), filter_by(), first(), from_statement(), get(), get_children(), get_execution_options(), get_label_style, group_by(), having(), instances(), intersect(), intersect_all(), is_single_entity, join(), label(), lazy_loaded_from, limit(), merge_result(), offset(), one(), one_or_none(), only_return_tuples(), options(), order_by(), outerjoin(), params(), populate_existing(), prefix_with(), reset_joinpoint(), scalar(), scalar_subquery(), select_from(), selectable, set_label_style(), slice(), statement, subquery(), suffix_with(), tuples(), union(), union_all(), update(), value(), values(), where(), whereclause, with_entities(), with_for_update(), with_hint(), with_labels(), with_parent(), with_session(), with_statement_hint(), with_transformation(), yield_per()
Class signature
class sqlalchemy.orm.Query (sqlalchemy.sql.expression._SelectFromElements, sqlalchemy.sql.annotation.SupportsCloneAnnotations, sqlalchemy.sql.expression.HasPrefixes, sqlalchemy.sql.expression.HasSuffixes, sqlalchemy.sql.expression.HasHints, sqlalchemy.event.registry.EventTarget, sqlalchemy.log.Identified, sqlalchemy.sql.expression.Generative, sqlalchemy.sql.expression.Executable, typing.Generic)
method sqlalchemy.orm.Query.__init__(entities: _ColumnsClauseArgument[Any] | Sequence[_ColumnsClauseArgument[Any]], session: Session | None = None)
Construct a Query directly.
E.g.:
q = Query([User, Address], session=some_session)
The above is equivalent to:
q = some_session.query(User, Address)
Parameters:
* entities – a sequence of entities and/or SQL expressions.
* session – a Session with which the Query will be associated. Optional; a Query can be associated with a Session generatively via the Query.with_session() method as well.
See also
Session.query()
Query.with_session()
method sqlalchemy.orm.Query.add_column(column: _ColumnExpressionArgument[Any]) ? Query[Any]
Add a column expression to the list of result columns to be returned.
Deprecated since version 1.4: Query.add_column() is deprecated and will be removed in a future release. Please use Query.add_columns()
method sqlalchemy.orm.Query.add_columns(*column: _ColumnExpressionArgument[Any]) ? Query[Any]
Add one or more column expressions to the list of result columns to be returned.
See also
Select.add_columns() - v2 comparable method.
method sqlalchemy.orm.Query.add_entity(entity: _EntityType[Any], alias: Alias | Subquery | None = None) ? Query[Any]
add a mapped entity to the list of result columns to be returned.
See also
Select.add_columns() - v2 comparable method.
method sqlalchemy.orm.Query.all() ? List[_T]
Return the results represented by this Query as a list.
This results in an execution of the underlying SQL statement.
Warning
The Query object, when asked to return either a sequence or iterator that consists of full ORM-mapped entities, will deduplicate entries based on primary key. See the FAQ for more details.
See also
My Query does not return the same number of objects as query.count() tells me - why?
See also
Result.all() - v2 comparable method.
Result.scalars() - v2 comparable method.
method sqlalchemy.orm.Query.apply_labels() ? Self
Deprecated since version 2.0: The Query.with_labels() and Query.apply_labels() method is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. Use set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL) instead. (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
method sqlalchemy.orm.Query.as_scalar() ? ScalarSelect[Any]
Return the full SELECT statement represented by this Query, converted to a scalar subquery.
Deprecated since version 1.4: The Query.as_scalar() method is deprecated and will be removed in a future release. Please refer to Query.scalar_subquery().
method sqlalchemy.orm.Query.autoflush(setting: bool) ? Self
Return a Query with a specific ‘autoflush’ setting.
As of SQLAlchemy 1.4, the Query.autoflush() method is equivalent to using the autoflush execution option at the ORM level. See the section Autoflush for further background on this option.
attribute sqlalchemy.orm.Query.column_descriptions
Return metadata about the columns which would be returned by this Query.
Format is a list of dictionaries:
user_alias = aliased(User, name="user2")
q = sess.query(User, User.id, user_alias)

# this expression:
q.column_descriptions

# would return:
[
    {
        "name": "User",
        "type": User,
        "aliased": False,
        "expr": User,
        "entity": User,
    },
    {
        "name": "id",
        "type": Integer(),
        "aliased": False,
        "expr": User.id,
        "entity": User,
    },
    {
        "name": "user2",
        "type": User,
        "aliased": True,
        "expr": user_alias,
        "entity": user_alias,
    },
]
See also
This API is available using 2.0 style queries as well, documented at:
* Inspecting entities and columns from ORM-enabled SELECT and DML statements
* Select.column_descriptions
method sqlalchemy.orm.Query.correlate(*fromclauses: Literal[None, False] | FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) ? Self
Return a Query construct which will correlate the given FROM clauses to that of an enclosing Query or select().
The method here accepts mapped classes, aliased() constructs, and Mapper constructs as arguments, which are resolved into expression constructs, in addition to appropriate expression constructs.
The correlation arguments are ultimately passed to Select.correlate() after coercion to expression constructs.
The correlation arguments take effect in such cases as when Query.from_self() is used, or when a subquery as returned by Query.subquery() is embedded in another select() construct.
See also
Select.correlate() - v2 equivalent method.
method sqlalchemy.orm.Query.count() ? int
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
method sqlalchemy.orm.Query.cte(name: str | None = None, recursive: bool = False, nesting: bool = False) ? CTE
Return the full SELECT statement represented by this Query represented as a common table expression (CTE).
Parameters and usage are the same as those of the SelectBase.cte() method; see that method for further details.
Here is the PostgreSQL WITH RECURSIVE example. Note that, in this example, the included_parts cte and the incl_alias alias of it are Core selectables, which means the columns are accessed via the .c. attribute. The parts_alias object is an aliased() instance of the Part entity, so column-mapped attributes are available directly:
from sqlalchemy.orm import aliased


class Part(Base):
    __tablename__ = "part"
    part = Column(String, primary_key=True)
    sub_part = Column(String, primary_key=True)
    quantity = Column(Integer)


included_parts = (
    session.query(Part.sub_part, Part.part, Part.quantity)
    .filter(Part.part == "our part")
    .cte(name="included_parts", recursive=True)
)

incl_alias = aliased(included_parts, name="pr")
parts_alias = aliased(Part, name="p")
included_parts = included_parts.union_all(
    session.query(
        parts_alias.sub_part, parts_alias.part, parts_alias.quantity
    ).filter(parts_alias.part == incl_alias.c.sub_part)
)

q = session.query(
    included_parts.c.sub_part,
    func.sum(included_parts.c.quantity).label("total_quantity"),
).group_by(included_parts.c.sub_part)
See also
Select.cte() - v2 equivalent method.
method sqlalchemy.orm.Query.delete(synchronize_session: SynchronizeSessionArgument = 'auto', delete_args: Dict[Any, Any] | None = None) ? int
Perform a DELETE with an arbitrary WHERE clause.
Deletes rows matched by this query from the database.
E.g.:
sess.query(User).filter(User.age == 25).delete(synchronize_session=False)

sess.query(User).filter(User.age == 25).delete(
    synchronize_session="evaluate"
)
Warning
See the section ORM-Enabled INSERT, UPDATE, and DELETE statements for important caveats and warnings, including limitations when using bulk UPDATE and DELETE with mapper inheritance configurations.
Parameters:
* synchronize_session – chooses the strategy to update the attributes on objects in the session. See the section ORM-Enabled INSERT, UPDATE, and DELETE statements for a discussion of these strategies.
* delete_args – 
Optional dictionary, if present will be passed to the underlying delete() construct as the **kw for the object. May be used to pass dialect-specific arguments such as mysql_limit.
Added in version 2.0.37.
Returns:
the count of rows matched as returned by the database’s “row count” feature.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
method sqlalchemy.orm.Query.distinct(*expr: _ColumnExpressionArgument[Any]) ? Self
Apply a DISTINCT to the query and return the newly resulting Query.
Note
The ORM-level distinct() call includes logic that will automatically add columns from the ORDER BY of the query to the columns clause of the SELECT statement, to satisfy the common need of the database backend that ORDER BY columns be part of the SELECT list when DISTINCT is used. These columns are not added to the list of columns actually fetched by the Query, however, so would not affect results. The columns are passed through when using the Query.statement accessor, however.
Deprecated since version 2.0: This logic is deprecated and will be removed in SQLAlchemy 2.0. See Using DISTINCT with additional columns, but only select the entity for a description of this use case in 2.0.
See also
Select.distinct() - v2 equivalent method.
Parameters:
*expr – 
optional column expressions. When present, the PostgreSQL dialect will render a DISTINCT ON (<expressions>) construct.
Deprecated since version 1.4: Using *expr in other dialects is deprecated and will raise CompileError in a future version.
method sqlalchemy.orm.Query.enable_assertions(value: bool) ? Self
Control whether assertions are generated.
When set to False, the returned Query will not assert its state before certain operations, including that LIMIT/OFFSET has not been applied when filter() is called, no criterion exists when get() is called, and no “from_statement()” exists when filter()/order_by()/group_by() etc. is called. This more permissive mode is used by custom Query subclasses to specify criterion or other modifiers outside of the usual usage patterns.
Care should be taken to ensure that the usage pattern is even possible. A statement applied by from_statement() will override any criterion set by filter() or order_by(), for example.
method sqlalchemy.orm.Query.enable_eagerloads(value: bool) ? Self
Control whether or not eager joins and subqueries are rendered.
When set to False, the returned Query will not render eager joins regardless of joinedload(), subqueryload() options or mapper-level lazy='joined'/lazy='subquery' configurations.
This is used primarily when nesting the Query’s statement into a subquery or other selectable, or when using Query.yield_per().
method sqlalchemy.orm.Query.except_(*q: Query) ? Self
Produce an EXCEPT of this Query against one or more queries.
Works the same way as Query.union(). See that method for usage examples.
See also
Select.except_() - v2 equivalent method.
method sqlalchemy.orm.Query.except_all(*q: Query) ? Self
Produce an EXCEPT ALL of this Query against one or more queries.
Works the same way as Query.union(). See that method for usage examples.
See also
Select.except_all() - v2 equivalent method.
method sqlalchemy.orm.Query.execution_options(**kwargs: Any) ? Self
Set non-SQL options which take effect during execution.
Options allowed here include all of those accepted by Connection.execution_options(), as well as a series of ORM specific options:
populate_existing=True - equivalent to using Query.populate_existing()
autoflush=True|False - equivalent to using Query.autoflush()
yield_per=<value> - equivalent to using Query.yield_per()
Note that the stream_results execution option is enabled automatically if the Query.yield_per() method or execution option is used.
Added in version 1.4: - added ORM options to Query.execution_options()
The execution options may also be specified on a per execution basis when using 2.0 style queries via the Session.execution_options parameter.
Warning
The Connection.execution_options.stream_results parameter should not be used at the level of individual ORM statement executions, as the Session will not track objects from different schema translate maps within a single session. For multiple schema translate maps within the scope of a single Session, see Horizontal Sharding.
See also
Using Server Side Cursors (a.k.a. stream results)
Query.get_execution_options()
Select.execution_options() - v2 equivalent method.
method sqlalchemy.orm.Query.exists() ? Exists
A convenience method that turns a query into an EXISTS subquery of the form EXISTS (SELECT 1 FROM … WHERE …).
e.g.:
q = session.query(User).filter(User.name == "fred")
session.query(q.exists())
Producing SQL similar to:
SELECT EXISTS (
    SELECT 1 FROM users WHERE users.name = :name_1
) AS anon_1
The EXISTS construct is usually used in the WHERE clause:
session.query(User.id).filter(q.exists()).scalar()
Note that some databases such as SQL Server don’t allow an EXISTS expression to be present in the columns clause of a SELECT. To select a simple boolean value based on the exists as a WHERE, use literal():
from sqlalchemy import literal

session.query(literal(True)).filter(q.exists()).scalar()
See also
Select.exists() - v2 comparable method.
method sqlalchemy.orm.Query.filter(*criterion: _ColumnExpressionArgument[bool]) ? Self
Apply the given filtering criterion to a copy of this Query, using SQL expressions.
e.g.:
session.query(MyClass).filter(MyClass.name == "some name")
Multiple criteria may be specified as comma separated; the effect is that they will be joined together using the and_() function:
session.query(MyClass).filter(MyClass.name == "some name", MyClass.id > 5)
The criterion is any SQL expression object applicable to the WHERE clause of a select. String expressions are coerced into SQL expression constructs via the text() construct.
See also
Query.filter_by() - filter on keyword expressions.
Select.where() - v2 equivalent method.
method sqlalchemy.orm.Query.filter_by(**kwargs: Any) ? Self
Apply the given filtering criterion to a copy of this Query, using keyword expressions.
e.g.:
session.query(MyClass).filter_by(name="some name")
Multiple criteria may be specified as comma separated; the effect is that they will be joined together using the and_() function:
session.query(MyClass).filter_by(name="some name", id=5)
The keyword expressions are extracted from the primary entity of the query, or the last entity that was the target of a call to Query.join().
See also
Query.filter() - filter on SQL expressions.
Select.filter_by() - v2 comparable method.
method sqlalchemy.orm.Query.first() ? _T | None
Return the first result of this Query or None if the result doesn’t contain any row.
first() applies a limit of one within the generated SQL, so that only one primary entity row is generated on the server side (note this may consist of multiple result rows if join-loaded collections are present).
Calling Query.first() results in an execution of the underlying query.
See also
Query.one()
Query.one_or_none()
Result.first() - v2 comparable method.
Result.scalars() - v2 comparable method.
method sqlalchemy.orm.Query.from_statement(statement: ExecutableReturnsRows) ? Self
Execute the given SELECT statement and return results.
This method bypasses all internal statement compilation, and the statement is executed without modification.
The statement is typically either a text() or select() construct, and should return the set of columns appropriate to the entity class represented by this Query.
See also
Select.from_statement() - v2 comparable method.
method sqlalchemy.orm.Query.get(ident: _PKIdentityArgument) ? Any | None
Return an instance based on the given primary key identifier, or None if not found.
Deprecated since version 2.0: The Query.get() method is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. The method is now available as Session.get() (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
E.g.:
my_user = session.query(User).get(5)

some_object = session.query(VersionedFoo).get((5, 10))

some_object = session.query(VersionedFoo).get({"id": 5, "version_id": 10})
Query.get() is special in that it provides direct access to the identity map of the owning Session. If the given primary key identifier is present in the local identity map, the object is returned directly from this collection and no SQL is emitted, unless the object has been marked fully expired. If not present, a SELECT is performed in order to locate the object.
Query.get() also will perform a check if the object is present in the identity map and marked as expired - a SELECT is emitted to refresh the object as well as to ensure that the row is still present. If not, ObjectDeletedError is raised.
Query.get() is only used to return a single mapped instance, not multiple instances or individual column constructs, and strictly on a single primary key value. The originating Query must be constructed in this way, i.e. against a single mapped entity, with no additional filtering criterion. Loading options via Query.options() may be applied however, and will be used if the object is not yet locally present.
Parameters:
ident – 
A scalar, tuple, or dictionary representing the primary key. For a composite (e.g. multiple column) primary key, a tuple or dictionary should be passed.
For a single-column primary key, the scalar calling form is typically the most expedient. If the primary key of a row is the value “5”, the call looks like:
my_object = query.get(5)
The tuple form contains primary key values typically in the order in which they correspond to the mapped Table object’s primary key columns, or if the Mapper.primary_key configuration parameter were used, in the order used for that parameter. For example, if the primary key of a row is represented by the integer digits “5, 10” the call would look like:
my_object = query.get((5, 10))
The dictionary form should include as keys the mapped attribute names corresponding to each element of the primary key. If the mapped class has the attributes id, version_id as the attributes which store the object’s primary key value, the call would look like:
my_object = query.get({"id": 5, "version_id": 10})
Added in version 1.3: the Query.get() method now optionally accepts a dictionary of attribute names to values in order to indicate a primary key identifier.
Returns:
The object instance, or None.
method sqlalchemy.orm.Query.get_children(*, omit_attrs: Tuple[str, ] = (), **kw: Any) ? Iterable[HasTraverseInternals]
inherited from the HasTraverseInternals.get_children() method of HasTraverseInternals
Return immediate child HasTraverseInternals elements of this HasTraverseInternals.
This is used for visit traversal.
**kw may contain flags that change the collection that is returned, for example to return a subset of items in order to cut down on larger traversals, or to return child items from a different context (such as schema-level collections instead of clause-level).
method sqlalchemy.orm.Query.get_execution_options() ? _ImmutableExecuteOptions
Get the non-SQL options which will take effect during execution.
Added in version 1.3.
See also
Query.execution_options()
Select.get_execution_options() - v2 comparable method.
attribute sqlalchemy.orm.Query.get_label_style
Retrieve the current label style.
Added in version 1.4.
See also
Select.get_label_style() - v2 equivalent method.
method sqlalchemy.orm.Query.group_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) ? Self
Apply one or more GROUP BY criterion to the query and return the newly resulting Query.
All existing GROUP BY settings can be suppressed by passing None - this will suppress any GROUP BY configured on mappers as well.
See also
These sections describe GROUP BY in terms of 2.0 style invocation but apply to Query as well:
Aggregate functions with GROUP BY / HAVING - in the SQLAlchemy Unified Tutorial
Ordering or Grouping by a Label - in the SQLAlchemy Unified Tutorial
Select.group_by() - v2 equivalent method.
method sqlalchemy.orm.Query.having(*having: _ColumnExpressionArgument[bool]) ? Self
Apply a HAVING criterion to the query and return the newly resulting Query.
Query.having() is used in conjunction with Query.group_by().
HAVING criterion makes it possible to use filters on aggregate functions like COUNT, SUM, AVG, MAX, and MIN, eg.:
q = (
    session.query(User.id)
    .join(User.addresses)
    .group_by(User.id)
    .having(func.count(Address.id) > 2)
)
See also
Select.having() - v2 equivalent method.
method sqlalchemy.orm.Query.instances(result_proxy: CursorResult[Any], context: QueryContext | None = None) ? Any
Return an ORM result given a CursorResult and QueryContext.
Deprecated since version 2.0: The Query.instances() method is deprecated and will be removed in a future release. Use the Select.from_statement() method or aliased() construct in conjunction with Session.execute() instead.
method sqlalchemy.orm.Query.intersect(*q: Query) ? Self
Produce an INTERSECT of this Query against one or more queries.
Works the same way as Query.union(). See that method for usage examples.
See also
Select.intersect() - v2 equivalent method.
method sqlalchemy.orm.Query.intersect_all(*q: Query) ? Self
Produce an INTERSECT ALL of this Query against one or more queries.
Works the same way as Query.union(). See that method for usage examples.
See also
Select.intersect_all() - v2 equivalent method.
attribute sqlalchemy.orm.Query.is_single_entity
Indicates if this Query returns tuples or single entities.
Returns True if this query returns a single entity for each instance in its result list, and False if this query returns a tuple of entities for each result.
Added in version 1.3.11.
See also
Query.only_return_tuples()
method sqlalchemy.orm.Query.join(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) ? Self
Create a SQL JOIN against this Query object’s criterion and apply generatively, returning the newly resulting Query.
Simple Relationship Joins
Consider a mapping between two classes User and Address, with a relationship User.addresses representing a collection of Address objects associated with each User. The most common usage of Query.join() is to create a JOIN along this relationship, using the User.addresses attribute as an indicator for how this should occur:
q = session.query(User).join(User.addresses)
Where above, the call to Query.join() along User.addresses will result in SQL approximately equivalent to:
SELECT user.id, user.name
FROM user JOIN address ON user.id = address.user_id
In the above example we refer to User.addresses as passed to Query.join() as the “on clause”, that is, it indicates how the “ON” portion of the JOIN should be constructed.
To construct a chain of joins, multiple Query.join() calls may be used. The relationship-bound attribute implies both the left and right side of the join at once:
q = (
    session.query(User)
    .join(User.orders)
    .join(Order.items)
    .join(Item.keywords)
)
Note
as seen in the above example, the order in which each call to the join() method occurs is important. Query would not, for example, know how to join correctly if we were to specify User, then Item, then Order, in our chain of joins; in such a case, depending on the arguments passed, it may raise an error that it doesn’t know how to join, or it may produce invalid SQL in which case the database will raise an error. In correct practice, the Query.join() method is invoked in such a way that lines up with how we would want the JOIN clauses in SQL to be rendered, and each call should represent a clear link from what precedes it.
Joins to a Target Entity or Selectable
A second form of Query.join() allows any mapped entity or core selectable construct as a target. In this usage, Query.join() will attempt to create a JOIN along the natural foreign key relationship between two entities:
q = session.query(User).join(Address)
In the above calling form, Query.join() is called upon to create the “on clause” automatically for us. This calling form will ultimately raise an error if either there are no foreign keys between the two entities, or if there are multiple foreign key linkages between the target entity and the entity or entities already present on the left side such that creating a join requires more information. Note that when indicating a join to a target without any ON clause, ORM configured relationships are not taken into account.
Joins to a Target with an ON Clause
The third calling form allows both the target entity as well as the ON clause to be passed explicitly. A example that includes a SQL expression as the ON clause is as follows:
q = session.query(User).join(Address, User.id == Address.user_id)
The above form may also use a relationship-bound attribute as the ON clause as well:
q = session.query(User).join(Address, User.addresses)
The above syntax can be useful for the case where we wish to join to an alias of a particular target entity. If we wanted to join to Address twice, it could be achieved using two aliases set up using the aliased() function:
a1 = aliased(Address)
a2 = aliased(Address)

q = (
    session.query(User)
    .join(a1, User.addresses)
    .join(a2, User.addresses)
    .filter(a1.email_address == "ed@foo.com")
    .filter(a2.email_address == "ed@bar.com")
)
The relationship-bound calling form can also specify a target entity using the PropComparator.of_type() method; a query equivalent to the one above would be:
a1 = aliased(Address)
a2 = aliased(Address)

q = (
    session.query(User)
    .join(User.addresses.of_type(a1))
    .join(User.addresses.of_type(a2))
    .filter(a1.email_address == "ed@foo.com")
    .filter(a2.email_address == "ed@bar.com")
)
Augmenting Built-in ON Clauses
As a substitute for providing a full custom ON condition for an existing relationship, the PropComparator.and_() function may be applied to a relationship attribute to augment additional criteria into the ON clause; the additional criteria will be combined with the default criteria using AND:
q = session.query(User).join(
    User.addresses.and_(Address.email_address != "foo@bar.com")
)
Added in version 1.4.
Joining to Tables and Subqueries
The target of a join may also be any table or SELECT statement, which may be related to a target entity or not. Use the appropriate .subquery() method in order to make a subquery out of a query:
subq = (
    session.query(Address)
    .filter(Address.email_address == "ed@foo.com")
    .subquery()
)


q = session.query(User).join(subq, User.id == subq.c.user_id)
Joining to a subquery in terms of a specific relationship and/or target entity may be achieved by linking the subquery to the entity using aliased():
subq = (
    session.query(Address)
    .filter(Address.email_address == "ed@foo.com")
    .subquery()
)

address_subq = aliased(Address, subq)

q = session.query(User).join(User.addresses.of_type(address_subq))
Controlling what to Join From
In cases where the left side of the current state of Query is not in line with what we want to join from, the Query.select_from() method may be used:
q = (
    session.query(Address)
    .select_from(User)
    .join(User.addresses)
    .filter(User.name == "ed")
)
Which will produce SQL similar to:
SELECT address.* FROM user
    JOIN address ON user.id=address.user_id
    WHERE user.name = :name_1
See also
Select.join() - v2 equivalent method.
Parameters:
* *props – Incoming arguments for Query.join(), the props collection in modern use should be considered to be a one or two argument form, either as a single “target” entity or ORM attribute-bound relationship, or as a target entity plus an “on clause” which may be a SQL expression or ORM attribute-bound relationship.
* isouter=False – If True, the join used will be a left outer join, just as if the Query.outerjoin() method were called.
* full=False – render FULL OUTER JOIN; implies isouter.
method sqlalchemy.orm.Query.label(name: str | None) ? Label[Any]
Return the full SELECT statement represented by this Query, converted to a scalar subquery with a label of the given name.
See also
Select.label() - v2 comparable method.
attribute sqlalchemy.orm.Query.lazy_loaded_from
An InstanceState that is using this Query for a lazy load operation.
Deprecated since version 1.4: This attribute should be viewed via the ORMExecuteState.lazy_loaded_from attribute, within the context of the SessionEvents.do_orm_execute() event.
See also
ORMExecuteState.lazy_loaded_from
method sqlalchemy.orm.Query.limit(limit: _LimitOffsetType) ? Self
Apply a LIMIT to the query and return the newly resulting Query.
See also
Select.limit() - v2 equivalent method.
method sqlalchemy.orm.Query.merge_result(iterator: FrozenResult[Any] | Iterable[Sequence[Any]] | Iterable[object], load: bool = True) ? FrozenResult[Any] | Iterable[Any]
Merge a result into this Query object’s Session.
Deprecated since version 2.0: The Query.merge_result() method is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. The method is superseded by the merge_frozen_result() function. (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
Given an iterator returned by a Query of the same structure as this one, return an identical iterator of results, with all mapped instances merged into the session using Session.merge(). This is an optimized method which will merge all mapped instances, preserving the structure of the result rows and unmapped columns with less method overhead than that of calling Session.merge() explicitly for each value.
The structure of the results is determined based on the column list of this Query - if these do not correspond, unchecked errors will occur.
The ‘load’ argument is the same as that of Session.merge().
For an example of how Query.merge_result() is used, see the source code for the example Dogpile Caching, where Query.merge_result() is used to efficiently restore state from a cache back into a target Session.
method sqlalchemy.orm.Query.offset(offset: _LimitOffsetType) ? Self
Apply an OFFSET to the query and return the newly resulting Query.
See also
Select.offset() - v2 equivalent method.
method sqlalchemy.orm.Query.one() ? _T
Return exactly one result or raise an exception.
Raises NoResultFound if the query selects no rows. Raises MultipleResultsFound if multiple object identities are returned, or if multiple rows are returned for a query that returns only scalar values as opposed to full identity-mapped entities.
Calling one() results in an execution of the underlying query.
See also
Query.first()
Query.one_or_none()
Result.one() - v2 comparable method.
Result.scalar_one() - v2 comparable method.
method sqlalchemy.orm.Query.one_or_none() ? _T | None
Return at most one result or raise an exception.
Returns None if the query selects no rows. Raises sqlalchemy.orm.exc.MultipleResultsFound if multiple object identities are returned, or if multiple rows are returned for a query that returns only scalar values as opposed to full identity-mapped entities.
Calling Query.one_or_none() results in an execution of the underlying query.
See also
Query.first()
Query.one()
Result.one_or_none() - v2 comparable method.
Result.scalar_one_or_none() - v2 comparable method.
method sqlalchemy.orm.Query.only_return_tuples(value: bool) ? Query
When set to True, the query results will always be a Row object.
This can change a query that normally returns a single entity as a scalar to return a Row result in all cases.
See also
Query.tuples() - returns tuples, but also at the typing level will type results as Tuple.
Query.is_single_entity()
Result.tuples() - v2 comparable method.
method sqlalchemy.orm.Query.options(*args: ExecutableOption) ? Self
Return a new Query object, applying the given list of mapper options.
Most supplied options regard changing how column- and relationship-mapped attributes are loaded.
See also
Column Loading Options
Relationship Loading with Loader Options
method sqlalchemy.orm.Query.order_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) ? Self
Apply one or more ORDER BY criteria to the query and return the newly resulting Query.
e.g.:
q = session.query(Entity).order_by(Entity.id, Entity.name)
Calling this method multiple times is equivalent to calling it once with all the clauses concatenated. All existing ORDER BY criteria may be cancelled by passing None by itself. New ORDER BY criteria may then be added by invoking Query.order_by() again, e.g.:
# will erase all ORDER BY and ORDER BY new_col alone
q = q.order_by(None).order_by(new_col)
See also
These sections describe ORDER BY in terms of 2.0 style invocation but apply to Query as well:
ORDER BY - in the SQLAlchemy Unified Tutorial
Ordering or Grouping by a Label - in the SQLAlchemy Unified Tutorial
Select.order_by() - v2 equivalent method.
method sqlalchemy.orm.Query.outerjoin(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) ? Self
Create a left outer join against this Query object’s criterion and apply generatively, returning the newly resulting Query.
Usage is the same as the join() method.
See also
Select.outerjoin() - v2 equivalent method.
method sqlalchemy.orm.Query.params(_Query__params: Dict[str, Any] | None = None, **kw: Any) ? Self
Add values for bind parameters which may have been specified in filter().
Parameters may be specified using **kwargs, or optionally a single dictionary as the first positional argument. The reason for both is that **kwargs is convenient, however some parameter dictionaries contain unicode keys in which case **kwargs cannot be used.
method sqlalchemy.orm.Query.populate_existing() ? Self
Return a Query that will expire and refresh all instances as they are loaded, or reused from the current Session.
As of SQLAlchemy 1.4, the Query.populate_existing() method is equivalent to using the populate_existing execution option at the ORM level. See the section Populate Existing for further background on this option.
method sqlalchemy.orm.Query.prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') ? Self
inherited from the HasPrefixes.prefix_with() method of HasPrefixes
Add one or more expressions following the statement keyword, i.e. SELECT, INSERT, UPDATE, or DELETE. Generative.
This is used to support backend-specific prefix keywords such as those provided by MySQL.
E.g.:
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with("/*+ BKA(t1) */", dialect="mysql")
Multiple prefixes can be specified by multiple calls to HasPrefixes.prefix_with().
Parameters:
* *prefixes – textual or ClauseElement construct which will be rendered following the INSERT, UPDATE, or DELETE keyword.
* dialect – optional string dialect name which will limit rendering of this prefix to only that dialect.
method sqlalchemy.orm.Query.reset_joinpoint() ? Self
Return a new Query, where the “join point” has been reset back to the base FROM entities of the query.
This method is usually used in conjunction with the aliased=True feature of the Query.join() method. See the example in Query.join() for how this is used.
method sqlalchemy.orm.Query.scalar() ? Any
Return the first element of the first result or None if no rows present. If multiple rows are returned, raises MultipleResultsFound.
session.query(Item).scalar()
<Item>
session.query(Item.id).scalar()
1
session.query(Item.id).filter(Item.id < 0).scalar()
None
session.query(Item.id, Item.name).scalar()
1
session.query(func.count(Parent.id)).scalar()
20
This results in an execution of the underlying query.
See also
Result.scalar() - v2 comparable method.
method sqlalchemy.orm.Query.scalar_subquery() ? ScalarSelect[Any]
Return the full SELECT statement represented by this Query, converted to a scalar subquery.
Analogous to SelectBase.scalar_subquery().
Changed in version 1.4: The Query.scalar_subquery() method replaces the Query.as_scalar() method.
See also
Select.scalar_subquery() - v2 comparable method.
method sqlalchemy.orm.Query.select_from(*from_obj: FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) ? Self
Set the FROM clause of this Query explicitly.
Query.select_from() is often used in conjunction with Query.join() in order to control which entity is selected from on the “left” side of the join.
The entity or selectable object here effectively replaces the “left edge” of any calls to Query.join(), when no joinpoint is otherwise established - usually, the default “join point” is the leftmost entity in the Query object’s list of entities to be selected.
A typical example:
q = (
    session.query(Address)
    .select_from(User)
    .join(User.addresses)
    .filter(User.name == "ed")
)
Which produces SQL equivalent to:
SELECT address.* FROM user
JOIN address ON user.id=address.user_id
WHERE user.name = :name_1
Parameters:
*from_obj – collection of one or more entities to apply to the FROM clause. Entities can be mapped classes, AliasedClass objects, Mapper objects as well as core FromClause elements like subqueries.
See also
Query.join()
Query.select_entity_from()
Select.select_from() - v2 equivalent method.
attribute sqlalchemy.orm.Query.selectable
Return the Select object emitted by this Query.
Used for inspect() compatibility, this is equivalent to:
query.enable_eagerloads(False).with_labels().statement
method sqlalchemy.orm.Query.set_label_style(style: SelectLabelStyle) ? Self
Apply column labels to the return value of Query.statement.
Indicates that this Query’s statement accessor should return a SELECT statement that applies labels to all columns in the form <tablename>_<columnname>; this is commonly used to disambiguate columns from multiple tables which have the same name.
When the Query actually issues SQL to load rows, it always uses column labeling.
Note
The Query.set_label_style() method only applies the output of Query.statement, and not to any of the result-row invoking systems of Query itself, e.g. Query.first(), Query.all(), etc. To execute a query using Query.set_label_style(), invoke the Query.statement using Session.execute():
result = session.execute(
    query.set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL).statement
)
Added in version 1.4.
See also
Select.set_label_style() - v2 equivalent method.
method sqlalchemy.orm.Query.slice(start: int, stop: int) ? Self
Computes the “slice” of the Query represented by the given indices and returns the resulting Query.
The start and stop indices behave like the argument to Python’s built-in range() function. This method provides an alternative to using LIMIT/OFFSET to get a slice of the query.
For example,
session.query(User).order_by(User.id).slice(1, 3)
renders as
SELECT users.id AS users_id,
       users.name AS users_name
FROM users ORDER BY users.id
LIMIT ? OFFSET ?
(2, 1)
See also
Query.limit()
Query.offset()
Select.slice() - v2 equivalent method.
attribute sqlalchemy.orm.Query.statement
The full SELECT statement represented by this Query.
The statement by default will not have disambiguating labels applied to the construct unless with_labels(True) is called first.
method sqlalchemy.orm.Query.subquery(name: str | None = None, with_labels: bool = False, reduce_columns: bool = False) ? Subquery
Return the full SELECT statement represented by this Query, embedded within an Alias.
Eager JOIN generation within the query is disabled.
See also
Select.subquery() - v2 comparable method.
Parameters:
* name – string name to be assigned as the alias; this is passed through to FromClause.alias(). If None, a name will be deterministically generated at compile time.
* with_labels – if True, with_labels() will be called on the Query first to apply table-qualified labels to all columns.
* reduce_columns – if True, Select.reduce_columns() will be called on the resulting select() construct, to remove same-named columns where one also refers to the other via foreign key or WHERE clause equivalence.
method sqlalchemy.orm.Query.suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') ? Self
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
* *suffixes – textual or ClauseElement construct which will be rendered following the target clause.
* dialect – Optional string dialect name which will limit rendering of this suffix to only that dialect.
method sqlalchemy.orm.Query.tuples() ? Query
return a tuple-typed form of this Query.
This method invokes the Query.only_return_tuples() method with a value of True, which by itself ensures that this Query will always return Row objects, even if the query is made against a single entity. It then also at the typing level will return a “typed” query, if possible, that will type result rows as Tuple objects with typed elements.
This method can be compared to the Result.tuples() method, which returns “self”, but from a typing perspective returns an object that will yield typed Tuple objects for results. Typing takes effect only if this Query object is a typed query object already.
Added in version 2.0.
See also
Result.tuples() - v2 equivalent method.
method sqlalchemy.orm.Query.union(*q: Query) ? Self
Produce a UNION of this Query against one or more queries.
e.g.:
q1 = sess.query(SomeClass).filter(SomeClass.foo == "bar")
q2 = sess.query(SomeClass).filter(SomeClass.bar == "foo")

q3 = q1.union(q2)
The method accepts multiple Query objects so as to control the level of nesting. A series of union() calls such as:
x.union(y).union(z).all()
will nest on each union(), and produces:
SELECT * FROM (SELECT * FROM (SELECT * FROM X UNION
                SELECT * FROM y) UNION SELECT * FROM Z)
Whereas:
x.union(y, z).all()
produces:
SELECT * FROM (SELECT * FROM X UNION SELECT * FROM y UNION
                SELECT * FROM Z)
Note that many database backends do not allow ORDER BY to be rendered on a query called within UNION, EXCEPT, etc. To disable all ORDER BY clauses including those configured on mappers, issue query.order_by(None) - the resulting Query object will not render ORDER BY within its SELECT statement.
See also
Select.union() - v2 equivalent method.
method sqlalchemy.orm.Query.union_all(*q: Query) ? Self
Produce a UNION ALL of this Query against one or more queries.
Works the same way as Query.union(). See that method for usage examples.
See also
Select.union_all() - v2 equivalent method.
method sqlalchemy.orm.Query.update(values: Dict[_DMLColumnArgument, Any], synchronize_session: SynchronizeSessionArgument = 'auto', update_args: Dict[Any, Any] | None = None) ? int
Perform an UPDATE with an arbitrary WHERE clause.
Updates rows matched by this query in the database.
E.g.:
sess.query(User).filter(User.age == 25).update(
    {User.age: User.age - 10}, synchronize_session=False
)

sess.query(User).filter(User.age == 25).update(
    {"age": User.age - 10}, synchronize_session="evaluate"
)
Warning
See the section ORM-Enabled INSERT, UPDATE, and DELETE statements for important caveats and warnings, including limitations when using arbitrary UPDATE and DELETE with mapper inheritance configurations.
Parameters:
* values – a dictionary with attributes names, or alternatively mapped attributes or SQL expressions, as keys, and literal values or sql expressions as values. If parameter-ordered mode is desired, the values can be passed as a list of 2-tuples; this requires that the update.preserve_parameter_order flag is passed to the Query.update.update_args dictionary as well.
* synchronize_session – chooses the strategy to update the attributes on objects in the session. See the section ORM-Enabled INSERT, UPDATE, and DELETE statements for a discussion of these strategies.
* update_args – Optional dictionary, if present will be passed to the underlying update() construct as the **kw for the object. May be used to pass dialect-specific arguments such as mysql_limit, as well as other special arguments such as update.preserve_parameter_order.
Returns:
the count of rows matched as returned by the database’s “row count” feature.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
method sqlalchemy.orm.Query.value(column: _ColumnExpressionArgument[Any]) ? Any
Return a scalar result corresponding to the given column expression.
Deprecated since version 1.4: Query.value() is deprecated and will be removed in a future release. Please use Query.with_entities() in combination with Query.scalar()
method sqlalchemy.orm.Query.values(*columns: _ColumnsClauseArgument[Any]) ? Iterable[Any]
Return an iterator yielding result tuples corresponding to the given list of columns
Deprecated since version 1.4: Query.values() is deprecated and will be removed in a future release. Please use Query.with_entities()
method sqlalchemy.orm.Query.where(*criterion: _ColumnExpressionArgument[bool]) ? Self
A synonym for Query.filter().
Added in version 1.4.
See also
Select.where() - v2 equivalent method.
attribute sqlalchemy.orm.Query.whereclause
A readonly attribute which returns the current WHERE criterion for this Query.
This returned value is a SQL expression construct, or None if no criterion has been established.
See also
Select.whereclause - v2 equivalent property.
method sqlalchemy.orm.Query.with_entities(*entities: _ColumnsClauseArgument[Any], **_Query__kw: Any) ? Query[Any]
Return a new Query replacing the SELECT list with the given entities.
e.g.:
# Users, filtered on some arbitrary criterion
# and then ordered by related email address
q = (
    session.query(User)
    .join(User.address)
    .filter(User.name.like("%ed%"))
    .order_by(Address.email)
)

# given *only* User.id==5, Address.email, and 'q', what
# would the *next* User in the result be ?
subq = (
    q.with_entities(Address.email)
    .order_by(None)
    .filter(User.id == 5)
    .subquery()
)
q = q.join((subq, subq.c.email < Address.email)).limit(1)
See also
Select.with_only_columns() - v2 comparable method.
method sqlalchemy.orm.Query.with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) ? Self
return a new Query with the specified options for the FOR UPDATE clause.
The behavior of this method is identical to that of GenerativeSelect.with_for_update(). When called with no arguments, the resulting SELECT statement will have a FOR UPDATE clause appended. When additional arguments are specified, backend-specific options such as FOR UPDATE NOWAIT or LOCK IN SHARE MODE can take effect.
E.g.:
q = (
    sess.query(User)
    .populate_existing()
    .with_for_update(nowait=True, of=User)
)
The above query on a PostgreSQL backend will render like:
SELECT users.id AS users_id FROM users FOR UPDATE OF users NOWAIT
Warning
Using with_for_update in the context of eager loading relationships is not officially supported or recommended by SQLAlchemy and may not work with certain queries on various database backends. When with_for_update is successfully used with a query that involves joinedload(), SQLAlchemy will attempt to emit SQL that locks all involved tables.
Note
It is generally a good idea to combine the use of the Query.populate_existing() method when using the Query.with_for_update() method. The purpose of Query.populate_existing() is to force all the data read from the SELECT to be populated into the ORM objects returned, even if these objects are already in the identity map.
See also
GenerativeSelect.with_for_update() - Core level method with full argument and behavioral description.
Query.populate_existing() - overwrites attributes of objects already loaded in the identity map.
method sqlalchemy.orm.Query.with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') ? Self
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
method sqlalchemy.orm.Query.with_labels() ? Self
Deprecated since version 2.0: The Query.with_labels() and Query.apply_labels() method is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. Use set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL) instead. (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
method sqlalchemy.orm.Query.with_parent(instance: object, property: attributes.QueryableAttribute[Any] | None = None, from_entity: _ExternalEntityType[Any] | None = None) ? Self
Add filtering criterion that relates the given instance to a child object or collection, using its attribute state as well as an established relationship() configuration.
Deprecated since version 2.0: The Query.with_parent() method is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. Use the with_parent() standalone construct. (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
The method uses the with_parent() function to generate the clause, the result of which is passed to Query.filter().
Parameters are the same as with_parent(), with the exception that the given property can be None, in which case a search is performed against this Query object’s target mapper.
Parameters:
* instance – An instance which has some relationship().
* property – Class bound attribute which indicates what relationship from the instance should be used to reconcile the parent/child relationship.
* from_entity – Entity in which to consider as the left side. This defaults to the “zero” entity of the Query itself.
method sqlalchemy.orm.Query.with_session(session: Session) ? Self
Return a Query that will use the given Session.
While the Query object is normally instantiated using the Session.query() method, it is legal to build the Query directly without necessarily using a Session. Such a Query object, or any Query already associated with a different Session, can produce a new Query object associated with a target session using this method:
from sqlalchemy.orm import Query

query = Query([MyClass]).filter(MyClass.id == 5)

result = query.with_session(my_session).one()
method sqlalchemy.orm.Query.with_statement_hint(text: str, dialect_name: str = '*') ? Self
inherited from the HasHints.with_statement_hint() method of HasHints
Add a statement hint to this Select or other selectable object.
Tip
Select.with_statement_hint() generally adds hints at the trailing end of a SELECT statement. To place dialect-specific hints such as optimizer hints at the front of the SELECT statement after the SELECT keyword, use the Select.prefix_with() method for an open-ended space, or for table-specific hints the Select.with_hint() may be used, which places hints in a dialect-specific location.
This method is similar to Select.with_hint() except that it does not require an individual table, and instead applies to the statement as a whole.
Hints here are specific to the backend database and may include directives such as isolation levels, file directives, fetch directives, etc.
See also
Select.with_hint()
Select.prefix_with() - generic SELECT prefixing which also can suit some database-specific HINT syntaxes such as MySQL or Oracle Database optimizer hints
method sqlalchemy.orm.Query.with_transformation(fn: Callable[[Query], Query]) ? Query
Return a new Query object transformed by the given function.
E.g.:
def filter_something(criterion):
    def transform(q):
        return q.filter(criterion)

    return transform


q = q.with_transformation(filter_something(x == 5))
This allows ad-hoc recipes to be created for Query objects.
method sqlalchemy.orm.Query.yield_per(count: int) ? Self
Yield only count rows at a time.
The purpose of this method is when fetching very large result sets (> 10K rows), to batch results in sub-collections and yield them out partially, so that the Python interpreter doesn’t need to declare very large areas of memory which is both time consuming and leads to excessive memory use. The performance from fetching hundreds of thousands of rows can often double when a suitable yield-per setting (e.g. approximately 1000) is used, even with DBAPIs that buffer rows (which are most).
As of SQLAlchemy 1.4, the Query.yield_per() method is equivalent to using the yield_per execution option at the ORM level. See the section Fetching Large Result Sets with Yield Per for further background on this option.
See also
Fetching Large Result Sets with Yield Per


ORM-Specific Query Constructs


Using the Session
The declarative base and ORM mapping functions described at ORM Mapped Class Configuration are the primary configurational interface for the ORM. Once mappings are configured, the primary usage interface for persistence operations is the Session.
* Session Basics
o What does the Session do ?
o Basics of Using a Session
* Opening and Closing a Session
* Framing out a begin / commit / rollback block
* Using a sessionmaker
* Querying
* Adding New or Existing Items
* Deleting
* Flushing
* Get by Primary Key
* Expiring / Refreshing
* UPDATE and DELETE with arbitrary WHERE clause
* Auto Begin
* Committing
* Rolling Back
* Closing
o Session Frequently Asked Questions
* When do I make a sessionmaker?
* When do I construct a Session, when do I commit it, and when do I close it?
* Is the Session a cache?
* How can I get the Session for a certain object?
* Is the Session thread-safe? Is AsyncSession safe to share in concurrent tasks?
* State Management
o Quickie Intro to Object States
* Getting the Current State of an Object
o Session Attributes
o Session Referencing Behavior
o Merging
* Merge Tips
o Expunging
o Refreshing / Expiring
* What Actually Loads
* When to Expire or Refresh
* Cascades
o save-update
* Behavior of save-update cascade with bi-directional relationships
o delete
* Using delete cascade with many-to-many relationships
* Using foreign key ON DELETE cascade with ORM relationships
* Using foreign key ON DELETE with many-to-many relationships
o delete-orphan
o merge
o refresh-expire
o expunge
o Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships
* Transactions and Connection Management
o Managing Transactions
* Using SAVEPOINT
* Session-level vs. Engine level transaction control
* Explicit Begin
* Enabling Two-Phase Commit
* Setting Transaction Isolation Levels / DBAPI AUTOCOMMIT
* Tracking Transaction State with Events
o Joining a Session into an External Transaction (such as for test suites)
* Additional Persistence Techniques
o Embedding SQL Insert/Update Expressions into a Flush
o Using SQL Expressions with Sessions
o Forcing NULL on a column with a default
o Fetching Server-Generated Defaults
* Case 1: non primary key, RETURNING or equivalent is supported
* Case 2: Table includes trigger-generated values which are not compatible with RETURNING
* Case 3: non primary key, RETURNING or equivalent is not supported or not needed
* Case 4: primary key, RETURNING or equivalent is supported
* Case 5: primary key, RETURNING or equivalent is not supported
* Notes on eagerly fetching client invoked SQL expressions used for INSERT or UPDATE
o Using INSERT, UPDATE and ON CONFLICT (i.e. upsert) to return ORM Objects
* Using PostgreSQL ON CONFLICT with RETURNING to return upserted ORM objects
o Partitioning Strategies (e.g. multiple database backends per Session)
* Simple Vertical Partitioning
* Coordination of Transactions for a multiple-engine Session
* Custom Vertical Partitioning
* Horizontal Partitioning
o Bulk Operations
* Contextual/Thread-local Sessions
o Implicit Method Access
o Thread-Local Scope
o Using Thread-Local Scope with Web Applications
o Using Custom Created Scopes
o Contextual Session API
* scoped_session
* ScopedRegistry
* ThreadLocalRegistry
* QueryPropertyDescriptor
* Tracking queries, object and Session Changes with Events
o Execute Events
* Basic Query Interception
* Adding global WHERE / ON criteria
* Re-Executing Statements
o Persistence Events
* before_flush()
* after_flush()
* after_flush_postexec()
* Mapper-level Flush Events
o Object Lifecycle Events
* Transient
* Transient to Pending
* Pending to Persistent
* Pending to Transient
* Loaded as Persistent
* Persistent to Transient
* Persistent to Deleted
* Deleted to Detached
* Persistent to Detached
* Detached to Persistent
* Deleted to Persistent
o Transaction Events
o Attribute Change Events
* Session API
o Session and sessionmaker()
* sessionmaker
* ORMExecuteState
* Session
* SessionTransaction
* SessionTransactionOrigin
o Session Utilities
* close_all_sessions()
* make_transient()
* make_transient_to_detached()
* object_session()
* was_deleted()
o Attribute and State Management Utilities
* object_state()
* del_attribute()
* get_attribute()
* get_history()
* init_collection()
* flag_modified()
* flag_dirty()
* instance_state()
* is_instrumented()
* set_attribute()
* set_committed_value()
* History


Session Basics
What does the Session do ?
In the most general sense, the Session establishes all conversations with the database and represents a “holding zone” for all the objects which you’ve loaded or associated with it during its lifespan. It provides the interface where SELECT and other queries are made that will return and modify ORM-mapped objects. The ORM objects themselves are maintained inside the Session, inside a structure called the identity map - a data structure that maintains unique copies of each object, where “unique” means “only one object with a particular primary key”.
The Session in its most common pattern of use begins in a mostly stateless form. Once queries are issued or other objects are persisted with it, it requests a connection resource from an Engine that is associated with the Session, and then establishes a transaction on that connection. This transaction remains in effect until the Session is instructed to commit or roll back the transaction. When the transaction ends, the connection resource associated with the Engine is released to the connection pool managed by the engine. A new transaction then starts with a new connection checkout.
The ORM objects maintained by a Session are instrumented such that whenever an attribute or a collection is modified in the Python program, a change event is generated which is recorded by the Session. Whenever the database is about to be queried, or when the transaction is about to be committed, the Session first flushes all pending changes stored in memory to the database. This is known as the unit of work pattern.
When using a Session, it’s useful to consider the ORM mapped objects that it maintains as proxy objects to database rows, which are local to the transaction being held by the Session. In order to maintain the state on the objects as matching what’s actually in the database, there are a variety of events that will cause objects to re-access the database in order to keep synchronized. It is possible to “detach” objects from a Session, and to continue using them, though this practice has its caveats. It’s intended that usually, you’d re-associate detached objects with another Session when you want to work with them again, so that they can resume their normal task of representing database state.
Basics of Using a Session
The most basic Session use patterns are presented here.
Opening and Closing a Session
The Session may be constructed on its own or by using the sessionmaker class. It typically is passed a single Engine as a source of connectivity up front. A typical use may look like:
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# create session and add objects
with Session(engine) as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
Above, the Session is instantiated with an Engine associated with a particular database URL. It is then used in a Python context manager (i.e. with: statement) so that it is automatically closed at the end of the block; this is equivalent to calling the Session.close() method.
The call to Session.commit() is optional, and is only needed if the work we’ve done with the Session includes new data to be persisted to the database. If we were only issuing SELECT calls and did not need to write any changes, then the call to Session.commit() would be unnecessary.
Note
Note that after Session.commit() is called, either explicitly or when using a context manager, all objects associated with the Session are expired, meaning their contents are erased to be re-loaded within the next transaction. If these objects are instead detached, they will be non-functional until re-associated with a new Session, unless the Session.expire_on_commit parameter is used to disable this behavior. See the section Committing for more detail.
Framing out a begin / commit / rollback block
We may also enclose the Session.commit() call and the overall “framing” of the transaction within a context manager for those cases where we will be committing data to the database. By “framing” we mean that if all operations succeed, the Session.commit() method will be called, but if any exceptions are raised, the Session.rollback() method will be called so that the transaction is rolled back immediately, before propagating the exception outward. In Python this is most fundamentally expressed using a try: / except: / else: block such as:
# verbose version of what a context manager will do
with Session(engine) as session:
    session.begin()
    try:
        session.add(some_object)
        session.add(some_other_object)
    except:
        session.rollback()
        raise
    else:
        session.commit()
The long-form sequence of operations illustrated above can be achieved more succinctly by making use of the SessionTransaction object returned by the Session.begin() method, which provides a context manager interface for the same sequence of operations:
# create session and add objects
with Session(engine) as session:
    with session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
More succinctly, the two contexts may be combined:
# create session and add objects
with Session(engine) as session, session.begin():
    session.add(some_object)
    session.add(some_other_object)
# inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
Using a sessionmaker
The purpose of sessionmaker is to provide a factory for Session objects with a fixed configuration. As it is typical that an application will have an Engine object in module scope, the sessionmaker can provide a factory for Session objects that are constructed against this engine:
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources, typically in module scope
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() without needing to pass the
# engine each time
with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
# closes the session
The sessionmaker is analogous to the Engine as a module-level factory for function-level sessions / connections. As such it also has its own sessionmaker.begin() method, analogous to Engine.begin(), which returns a Session object and also maintains a begin/commit/rollback block:
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() and include begin()/commit()/rollback()
# at once
with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits the transaction, closes the session
Where above, the Session will both have its transaction committed as well as that the Session will be closed, when the above with: block ends.
When you write your application, the sessionmaker factory should be scoped the same as the Engine object created by create_engine(), which is typically at module-level or global scope. As these objects are both factories, they can be used by any number of functions and threads simultaneously.
See also
sessionmaker
Session
Querying
The primary means of querying is to make use of the select() construct to create a Select object, which is then executed to return a result using methods such as Session.execute() and Session.scalars(). Results are then returned in terms of Result objects, including sub-variants such as ScalarResult.
A complete guide to SQLAlchemy ORM querying can be found at ORM Querying Guide. Some brief examples follow:
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # query for ``User`` objects
    statement = select(User).filter_by(name="ed")

    # list of ``User`` objects
    user_obj = session.scalars(statement).all()

    # query for individual columns
    statement = select(User.name, User.fullname)

    # list of Row objects
    rows = session.execute(statement).all()
Changed in version 2.0: “2.0” style querying is now standard. See 2.0 Migration - ORM Usage for migration notes from the 1.x series.
See also
ORM Querying Guide
Adding New or Existing Items
Session.add() is used to place instances in the session. For transient (i.e. brand new) instances, this will have the effect of an INSERT taking place for those instances upon the next flush. For instances which are persistent (i.e. were loaded by this session), they are already present and do not need to be added. Instances which are detached (i.e. have been removed from a session) may be re-associated with a session using this method:
user1 = User(name="user1")
user2 = User(name="user2")
session.add(user1)
session.add(user2)

session.commit()  # write changes to the database
To add a list of items to the session at once, use Session.add_all():
session.add_all([item1, item2, item3])
The Session.add() operation cascades along the save-update cascade. For more details see the section Cascades.
Deleting
The Session.delete() method places an instance into the Session’s list of objects to be marked as deleted:
# mark two objects to be deleted
session.delete(obj1)
session.delete(obj2)

# commit (or flush)
session.commit()
Session.delete() marks an object for deletion, which will result in a DELETE statement emitted for each primary key affected. Before the pending deletes are flushed, objects marked by “delete” are present in the Session.deleted collection. After the DELETE, they are expunged from the Session, which becomes permanent after the transaction is committed.
There are various important behaviors related to the Session.delete() operation, particularly in how relationships to other objects and collections are handled. There’s more information on how this works in the section Cascades, but in general the rules are:
* Rows that correspond to mapped objects that are related to a deleted object via the relationship() directive are not deleted by default. If those objects have a foreign key constraint back to the row being deleted, those columns are set to NULL. This will cause a constraint violation if the columns are non-nullable.
* To change the “SET NULL” into a DELETE of a related object’s row, use the delete cascade on the relationship().
* Rows that are in tables linked as “many-to-many” tables, via the relationship.secondary parameter, are deleted in all cases when the object they refer to is deleted.
* When related objects include a foreign key constraint back to the object being deleted, and the related collections to which they belong are not currently loaded into memory, the unit of work will emit a SELECT to fetch all related rows, so that their primary key values can be used to emit either UPDATE or DELETE statements on those related rows. In this way, the ORM without further instruction will perform the function of ON DELETE CASCADE, even if this is configured on Core ForeignKeyConstraint objects.
* The relationship.passive_deletes parameter can be used to tune this behavior and rely upon “ON DELETE CASCADE” more naturally; when set to True, this SELECT operation will no longer take place, however rows that are locally present will still be subject to explicit SET NULL or DELETE. Setting relationship.passive_deletes to the string "all" will disable all related object update/delete.
* When the DELETE occurs for an object marked for deletion, the object is not automatically removed from collections or object references that refer to it. When the Session is expired, these collections may be loaded again so that the object is no longer present. However, it is preferable that instead of using Session.delete() for these objects, the object should instead be removed from its collection and then delete-orphan should be used so that it is deleted as a secondary effect of that collection removal. See the section Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships for an example of this.
See also
delete - describes “delete cascade”, which marks related objects for deletion when a lead object is deleted.
delete-orphan - describes “delete orphan cascade”, which marks related objects for deletion when they are de-associated from their lead object.
Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships - important background on Session.delete() as involves relationships being refreshed in memory.
Flushing
When the Session is used with its default configuration, the flush step is nearly always done transparently. Specifically, the flush occurs before any individual SQL statement is issued as a result of a Query or a 2.0-style Session.execute() call, as well as within the Session.commit() call before the transaction is committed. It also occurs before a SAVEPOINT is issued when Session.begin_nested() is used.
A Session flush can be forced at any time by calling the Session.flush() method:
session.flush()
The flush which occurs automatically within the scope of certain methods is known as autoflush. Autoflush is defined as a configurable, automatic flush call which occurs at the beginning of methods including:
* Session.execute() and other SQL-executing methods, when used against ORM-enabled SQL constructs, such as select() objects that refer to ORM entities and/or ORM-mapped attributes
* When a Query is invoked to send SQL to the database
* Within the Session.merge() method before querying the database
* When objects are refreshed
* When ORM lazy load operations occur against unloaded object attributes.
There are also points at which flushes occur unconditionally; these points are within key transactional boundaries which include:
* Within the process of the Session.commit() method
* When Session.begin_nested() is called
* When the Session.prepare() 2PC method is used.
The autoflush behavior, as applied to the previous list of items, can be disabled by constructing a Session or sessionmaker passing the Session.autoflush parameter as False:
Session = sessionmaker(autoflush=False)
Additionally, autoflush can be temporarily disabled within the flow of using a Session using the Session.no_autoflush context manager:
with mysession.no_autoflush:
    mysession.add(some_object)
    mysession.flush()
To reiterate: The flush process always occurs when transactional methods such as Session.commit() and Session.begin_nested() are called, regardless of any “autoflush” settings, when the Session has remaining pending changes to process.
As the Session only invokes SQL to the database within the context of a DBAPI transaction, all “flush” operations themselves only occur within a database transaction (subject to the isolation level of the database transaction), provided that the DBAPI is not in driver level autocommit mode. This means that assuming the database connection is providing for atomicity within its transactional settings, if any individual DML statement inside the flush fails, the entire operation will be rolled back.
When a failure occurs within a flush, in order to continue using that same Session, an explicit call to Session.rollback() is required after a flush fails, even though the underlying transaction will have been rolled back already (even if the database driver is technically in driver-level autocommit mode). This is so that the overall nesting pattern of so-called “subtransactions” is consistently maintained. The FAQ section “This Session’s transaction has been rolled back due to a previous exception during flush.” (or similar) contains a more detailed description of this behavior.
See also
“This Session’s transaction has been rolled back due to a previous exception during flush.” (or similar) - further background on why Session.rollback() must be called when a flush fails.
Get by Primary Key
As the Session makes use of an identity map which refers to current in-memory objects by primary key, the Session.get() method is provided as a means of locating objects by primary key, first looking within the current identity map and then querying the database for non present values. Such as, to locate a User entity with primary key identity (5, ):
my_user = session.get(User, 5)
The Session.get() also includes calling forms for composite primary key values, which may be passed as tuples or dictionaries, as well as additional parameters which allow for specific loader and execution options. See Session.get() for the complete parameter list.
See also
Session.get()
Expiring / Refreshing
An important consideration that will often come up when using the Session is that of dealing with the state that is present on objects that have been loaded from the database, in terms of keeping them synchronized with the current state of the transaction. The SQLAlchemy ORM is based around the concept of an identity map such that when an object is “loaded” from a SQL query, there will be a unique Python object instance maintained corresponding to a particular database identity. This means if we emit two separate queries, each for the same row, and get a mapped object back, the two queries will have returned the same Python object:
u1 = session.scalars(select(User).where(User.id == 5)).one()
u2 = session.scalars(select(User).where(User.id == 5)).one()
u1 is u2
True
Following from this, when the ORM gets rows back from a query, it will skip the population of attributes for an object that’s already loaded. The design assumption here is to assume a transaction that’s perfectly isolated, and then to the degree that the transaction isn’t isolated, the application can take steps on an as-needed basis to refresh objects from the database transaction. The FAQ entry at I’m re-loading data with my Session but it isn’t seeing changes that I committed elsewhere discusses this concept in more detail.
When an ORM mapped object is loaded into memory, there are three general ways to refresh its contents with new data from the current transaction:
* the expire() method - the Session.expire() method will erase the contents of selected or all attributes of an object, such that they will be loaded from the database when they are next accessed, e.g. using a lazy loading pattern:
* session.expire(u1)
u1.some_attribute  # <-- lazy loads from the transaction
the refresh() method - closely related is the Session.refresh() method, which does everything the Session.expire() method does but also emits one or more SQL queries immediately to actually refresh the contents of the object:
session.refresh(u1)  # <-- emits a SQL query
u1.some_attribute  # <-- is refreshed from the transaction
the populate_existing() method or execution option - This is now an execution option documented at Populate Existing; in legacy form it’s found on the Query object as the Query.populate_existing() method. This operation in either form indicates that objects being returned from a query should be unconditionally re-populated from their contents in the database:
u2 = session.scalars(
    select(User).where(User.id == 5).execution_options(populate_existing=True)
).one()
Further discussion on the refresh / expire concept can be found at Refreshing / Expiring.
See also
Refreshing / Expiring
I’m re-loading data with my Session but it isn’t seeing changes that I committed elsewhere
UPDATE and DELETE with arbitrary WHERE clause
SQLAlchemy 2.0 includes enhanced capabilities for emitting several varieties of ORM-enabled INSERT, UPDATE and DELETE statements. See the document at ORM-Enabled INSERT, UPDATE, and DELETE statements for documentation.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
ORM UPDATE and DELETE with Custom WHERE Criteria
Auto Begin
The Session object features a behavior known as autobegin. This indicates that the Session will internally consider itself to be in a “transactional” state as soon as any work is performed with the Session, either involving modifications to the internal state of the Session with regards to object state changes, or with operations that require database connectivity.
When the Session is first constructed, there’s no transactional state present. The transactional state is begun automatically, when a method such as Session.add() or Session.execute() is invoked, or similarly if a Query is executed to return results (which ultimately uses Session.execute()), or if an attribute is modified on a persistent object.
The transactional state can be checked by accessing the Session.in_transaction() method, which returns True or False indicating if the “autobegin” step has proceeded. While not normally needed, the Session.get_transaction() method will return the actual SessionTransaction object that represents this transactional state.
The transactional state of the Session may also be started explicitly, by invoking the Session.begin() method. When this method is called, the Session is placed into the “transactional” state unconditionally. Session.begin() may be used as a context manager as described at Framing out a begin / commit / rollback block.
Disabling Autobegin to Prevent Implicit Transactions
The “autobegin” behavior may be disabled using the Session.autobegin parameter set to False. By using this parameter, a Session will require that the Session.begin() method is called explicitly. Upon construction, as well as after any of the Session.rollback(), Session.commit(), or Session.close() methods are called, the Session won’t implicitly begin any new transactions and will raise an error if an attempt to use the Session is made without first calling Session.begin():
with Session(engine, autobegin=False) as session:
    session.begin()  # <-- required, else InvalidRequestError raised on next call

    session.add(User(name="u1"))
    session.commit()

    session.begin()  # <-- required, else InvalidRequestError raised on next call

    u1 = session.scalar(select(User).filter_by(name="u1"))
Added in version 2.0: Added Session.autobegin, allowing “autobegin” behavior to be disabled
Committing
Session.commit() is used to commit the current transaction. At its core this indicates that it emits COMMIT on all current database connections that have a transaction in progress; from a DBAPI perspective this means the connection.commit() DBAPI method is invoked on each DBAPI connection.
When there is no transaction in place for the Session, indicating that no operations were invoked on this Session since the previous call to Session.commit(), the method will begin and commit an internal-only “logical” transaction, that does not normally affect the database unless pending flush changes were detected, but will still invoke event handlers and object expiration rules.
The Session.commit() operation unconditionally issues Session.flush() before emitting COMMIT on relevant database connections. If no pending changes are detected, then no SQL is emitted to the database. This behavior is not configurable and is not affected by the Session.autoflush parameter.
Subsequent to that, assuming the Session is bound to an Engine, Session.commit() will then COMMIT the actual database transaction that is in place, if one was started. After the commit, the Connection object associated with that transaction is closed, causing its underlying DBAPI connection to be released back to the connection pool associated with the Engine to which the Session is bound.
For a Session that’s bound to multiple engines (e.g. as described at Partitioning Strategies), the same COMMIT steps will proceed for each Engine / Connection that is in play within the “logical” transaction being committed. These database transactions are uncoordinated with each other unless two-phase features are enabled.
Other connection-interaction patterns are available as well, by binding the Session to a Connection directly; in this case, it’s assumed that an externally-managed transaction is present, and a real COMMIT will not be emitted automatically in this case; see the section Joining a Session into an External Transaction (such as for test suites) for background on this pattern.
Finally, all objects within the Session are expired as the transaction is closed out. This is so that when the instances are next accessed, either through attribute access or by them being present in the result of a SELECT, they receive the most recent state. This behavior may be controlled by the Session.expire_on_commit flag, which may be set to False when this behavior is undesirable.
See also
Auto Begin
Rolling Back
Session.rollback() rolls back the current transaction, if any. When there is no transaction in place, the method passes silently.
With a default configured session, the post-rollback state of the session, subsequent to a transaction having been begun either via autobegin or by calling the Session.begin() method explicitly, is as follows:
* Database transactions are rolled back. For a Session bound to a single Engine, this means ROLLBACK is emitted for at most a single Connection that’s currently in use. For Session objects bound to multiple Engine objects, ROLLBACK is emitted for all Connection objects that were checked out.
* Database connections are released. This follows the same connection-related behavior noted in Committing, where Connection objects obtained from Engine objects are closed, causing the DBAPI connections to be released to the connection pool within the Engine. New connections are checked out from the Engine if and when a new transaction begins.
* For a Session that’s bound directly to a Connection as described at Joining a Session into an External Transaction (such as for test suites), rollback behavior on this Connection would follow the behavior specified by the Session.join_transaction_mode parameter, which could involve rolling back savepoints or emitting a real ROLLBACK.
* Objects which were initially in the pending state when they were added to the Session within the lifespan of the transaction are expunged, corresponding to their INSERT statement being rolled back. The state of their attributes remains unchanged.
* Objects which were marked as deleted within the lifespan of the transaction are promoted back to the persistent state, corresponding to their DELETE statement being rolled back. Note that if those objects were first pending within the transaction, that operation takes precedence instead.
* All objects not expunged are fully expired - this is regardless of the Session.expire_on_commit setting.
With that state understood, the Session may safely continue usage after a rollback occurs.
Changed in version 1.4: The Session object now features deferred “begin” behavior, as described in autobegin. If no transaction is begun, methods like Session.commit() and Session.rollback() have no effect. This behavior would not have been observed prior to 1.4 as under non-autocommit mode, a transaction would always be implicitly present.
When a Session.flush() fails, typically for reasons like primary key, foreign key, or “not nullable” constraint violations, a ROLLBACK is issued automatically (it’s currently not possible for a flush to continue after a partial failure). However, the Session goes into a state known as “inactive” at this point, and the calling application must always call the Session.rollback() method explicitly so that the Session can go back into a usable state (it can also be simply closed and discarded). See the FAQ entry at “This Session’s transaction has been rolled back due to a previous exception during flush.” (or similar) for further discussion.
See also
Auto Begin
Closing
The Session.close() method issues a Session.expunge_all() which removes all ORM-mapped objects from the session, and releases any transactional/connection resources from the Engine object(s) to which it is bound. When connections are returned to the connection pool, transactional state is rolled back as well.
By default, when the Session is closed, it is essentially in the original state as when it was first constructed, and may be used again. In this sense, the Session.close() method is more like a “reset” back to the clean state and not as much like a “database close” method. In this mode of operation the method Session.reset() is an alias to Session.close() and behaves in the same way.
The default behavior of Session.close() can be changed by setting the parameter Session.close_resets_only to False, indicating that the Session cannot be reused after the method Session.close() has been called. In this mode of operation the Session.reset() method will allow multiple “reset” of the session, behaving like Session.close() when Session.close_resets_only is set to True.
Added in version 2.0.22.
It’s recommended that the scope of a Session be limited by a call to Session.close() at the end, especially if the Session.commit() or Session.rollback() methods are not used. The Session may be used as a context manager to ensure that Session.close() is called:
with Session(engine) as session:
    result = session.execute(select(User))

# closes session automatically
Changed in version 1.4: The Session object features deferred “begin” behavior, as described in autobegin. no longer immediately begins a new transaction after the Session.close() method is called.
Session Frequently Asked Questions
By this point, many users already have questions about sessions. This section presents a mini-FAQ (note that we have also a real FAQ) of the most basic issues one is presented with when using a Session.
When do I make a sessionmaker?
Just one time, somewhere in your application’s global scope. It should be looked upon as part of your application’s configuration. If your application has three .py files in a package, you could, for example, place the sessionmaker line in your __init__.py file; from that point on your other modules say “from mypackage import Session”. That way, everyone else just uses Session(), and the configuration of that session is controlled by that central point.
If your application starts up, does imports, but does not know what database it’s going to be connecting to, you can bind the Session at the “class” level to the engine later on, using sessionmaker.configure().
In the examples in this section, we will frequently show the sessionmaker being created right above the line where we actually invoke Session. But that’s just for example’s sake! In reality, the sessionmaker would be somewhere at the module level. The calls to instantiate Session would then be placed at the point in the application where database conversations begin.
When do I construct a Session, when do I commit it, and when do I close it?
tl;dr;
1. As a general rule, keep the lifecycle of the session separate and external from functions and objects that access and/or manipulate database data. This will greatly help with achieving a predictable and consistent transactional scope.
2. Make sure you have a clear notion of where transactions begin and end, and keep transactions short, meaning, they end at the series of a sequence of operations, instead of being held open indefinitely.
A Session is typically constructed at the beginning of a logical operation where database access is potentially anticipated.
The Session, whenever it is used to talk to the database, begins a database transaction as soon as it starts communicating. This transaction remains in progress until the Session is rolled back, committed, or closed. The Session will begin a new transaction if it is used again, subsequent to the previous transaction ending; from this it follows that the Session is capable of having a lifespan across many transactions, though only one at a time. We refer to these two concepts as transaction scope and session scope.
It’s usually not very hard to determine the best points at which to begin and end the scope of a Session, though the wide variety of application architectures possible can introduce challenging situations.
Some sample scenarios include:
* Web applications. In this case, it’s best to make use of the SQLAlchemy integrations provided by the web framework in use. Or otherwise, the basic pattern is create a Session at the start of a web request, call the Session.commit() method at the end of web requests that do POST, PUT, or DELETE, and then close the session at the end of web request. It’s also usually a good idea to set Session.expire_on_commit to False so that subsequent access to objects that came from a Session within the view layer do not need to emit new SQL queries to refresh the objects, if the transaction has been committed already.
* A background daemon which spawns off child forks would want to create a Session local to each child process, work with that Session through the life of the “job” that the fork is handling, then tear it down when the job is completed.
* For a command-line script, the application would create a single, global Session that is established when the program begins to do its work, and commits it right as the program is completing its task.
* For a GUI interface-driven application, the scope of the Session may best be within the scope of a user-generated event, such as a button push. Or, the scope may correspond to explicit user interaction, such as the user “opening” a series of records, then “saving” them.
As a general rule, the application should manage the lifecycle of the session externally to functions that deal with specific data. This is a fundamental separation of concerns which keeps data-specific operations agnostic of the context in which they access and manipulate that data.
E.g. don’t do this:
### this is the **wrong way to do it** ###


class ThingOne:
    def go(self):
        session = Session()
        try:
            session.execute(update(FooBar).values(x=5))
            session.commit()
        except:
            session.rollback()
            raise


class ThingTwo:
    def go(self):
        session = Session()
        try:
            session.execute(update(Widget).values(q=18))
            session.commit()
        except:
            session.rollback()
            raise


def run_my_program():
    ThingOne().go()
    ThingTwo().go()
Keep the lifecycle of the session (and usually the transaction) separate and external. The example below illustrates how this might look, and additionally makes use of a Python context manager (i.e. the with: keyword) in order to manage the scope of the Session and its transaction automatically:
### this is a **better** (but not the only) way to do it ###


class ThingOne:
    def go(self, session):
        session.execute(update(FooBar).values(x=5))


class ThingTwo:
    def go(self, session):
        session.execute(update(Widget).values(q=18))


def run_my_program():
    with Session() as session:
        with session.begin():
            ThingOne().go(session)
            ThingTwo().go(session)
Changed in version 1.4: The Session may be used as a context manager without the use of external helper functions.
Is the Session a cache?
Yeee…no. It’s somewhat used as a cache, in that it implements the identity map pattern, and stores objects keyed to their primary key. However, it doesn’t do any kind of query caching. This means, if you say session.scalars(select(Foo).filter_by(name='bar')), even if Foo(name='bar') is right there, in the identity map, the session has no idea about that. It has to issue SQL to the database, get the rows back, and then when it sees the primary key in the row, then it can look in the local identity map and see that the object is already there. It’s only when you say query.get({some primary key}) that the Session doesn’t have to issue a query.
Additionally, the Session stores object instances using a weak reference by default. This also defeats the purpose of using the Session as a cache.
The Session is not designed to be a global object from which everyone consults as a “registry” of objects. That’s more the job of a second level cache. SQLAlchemy provides a pattern for implementing second level caching using dogpile.cache, via the Dogpile Caching example.
How can I get the Session for a certain object?
Use the Session.object_session() classmethod available on Session:
session = Session.object_session(someobject)
The newer Runtime Inspection API system can also be used:
from sqlalchemy import inspect

session = inspect(someobject).session
Is the Session thread-safe? Is AsyncSession safe to share in concurrent tasks?
The Session is a mutable, stateful object that represents a single database transaction. An instance of Session therefore cannot be shared among concurrent threads or asyncio tasks without careful synchronization. The Session is intended to be used in a non-concurrent fashion, that is, a particular instance of Session should be used in only one thread or task at a time.
When using the AsyncSession object from SQLAlchemy’s asyncio extension, this object is only a thin proxy on top of a Session, and the same rules apply; it is an unsynchronized, mutable, stateful object, so it is not safe to use a single instance of AsyncSession in multiple asyncio tasks at once.
An instance of Session or AsyncSession represents a single logical database transaction, referencing only a single Connection at a time for a particular Engine or AsyncEngine to which the object is bound (note that these objects both support being bound to multiple engines at once, however in this case there will still be only one connection per engine in play within the scope of a transaction).
A database connection within a transaction is also a stateful object that is intended to be operated upon in a non-concurrent, sequential fashion. Commands are issued on the connection in a sequence, which are handled by the database server in the exact order in which they are emitted. As the Session emits commands upon this connection and receives results, the Session itself is transitioning through internal state changes that align with the state of commands and data present on this connection; states which include if a transaction were begun, committed, or rolled back, what SAVEPOINTs if any are in play, as well as fine-grained synchronization of the state of individual database rows with local ORM-mapped objects.
When designing database applications for concurrency, the appropriate model is that each concurrent task / thread works with its own database transaction. This is why when discussing the issue of database concurrency, the standard terminology used is multiple, concurrent transactions. Within traditional RDMS there is no analogue for a single database transaction that is receiving and processing multiple commands concurrently.
The concurrency model for SQLAlchemy’s Session and AsyncSession is therefore Session per thread, AsyncSession per task. An application that uses multiple threads, or multiple tasks in asyncio such as when using an API like asyncio.gather() would want to ensure that each thread has its own Session, each asyncio task has its own AsyncSession.
The best way to ensure this use is by using the standard context manager pattern locally within the top level Python function that is inside the thread or task, which will ensure the lifespan of the Session or AsyncSession is maintained within a local scope.
For applications that benefit from having a “global” Session where it’s not an option to pass the Session object to specific functions and methods which require it, the scoped_session approach can provide for a “thread local” Session object; see the section Contextual/Thread-local Sessions for background. Within the asyncio context, the async_scoped_session object is the asyncio analogue for scoped_session, however is more challenging to configure as it requires a custom “context” function.


State Management
Quickie Intro to Object States
It’s helpful to know the states which an instance can have within a session:
* Transient - an instance that’s not in a session, and is not saved to the database; i.e. it has no database identity. The only relationship such an object has to the ORM is that its class has a Mapper associated with it.
* Pending - when you Session.add() a transient instance, it becomes pending. It still wasn’t actually flushed to the database yet, but it will be when the next flush occurs.
* Persistent - An instance which is present in the session and has a record in the database. You get persistent instances by either flushing so that the pending instances become persistent, or by querying the database for existing instances (or moving persistent instances from other sessions into your local session).
* Deleted - An instance which has been deleted within a flush, but the transaction has not yet completed. Objects in this state are essentially in the opposite of “pending” state; when the session’s transaction is committed, the object will move to the detached state. Alternatively, when the session’s transaction is rolled back, a deleted object moves back to the persistent state.
* Detached - an instance which corresponds, or previously corresponded, to a record in the database, but is not currently in any session. The detached object will contain a database identity marker, however because it is not associated with a session, it is unknown whether or not this database identity actually exists in a target database. Detached objects are safe to use normally, except that they have no ability to load unloaded attributes or attributes that were previously marked as “expired”.
For a deeper dive into all possible state transitions, see the section Object Lifecycle Events which describes each transition as well as how to programmatically track each one.
Getting the Current State of an Object
The actual state of any mapped object can be viewed at any time using the inspect() function on a mapped instance; this function will return the corresponding InstanceState object which manages the internal ORM state for the object. InstanceState provides, among other accessors, boolean attributes indicating the persistence state of the object, including:
* InstanceState.transient
* InstanceState.pending
* InstanceState.persistent
* InstanceState.deleted
* InstanceState.detached
E.g.:
from sqlalchemy import inspect
insp = inspect(my_object)
insp.persistent
True
See also
Inspection of Mapped Instances - further examples of InstanceState
Session Attributes
The Session itself acts somewhat like a set-like collection. All items present may be accessed using the iterator interface:
for obj in session:
    print(obj)
And presence may be tested for using regular “contains” semantics:
if obj in session:
    print("Object is present")
The session is also keeping track of all newly created (i.e. pending) objects, all objects which have had changes since they were last loaded or saved (i.e. “dirty”), and everything that’s been marked as deleted:
# pending objects recently added to the Session
session.new

# persistent objects which currently have changes detected
# (this collection is now created on the fly each time the property is called)
session.dirty

# persistent objects that have been marked as deleted via session.delete(obj)
session.deleted

# dictionary of all persistent objects, keyed on their
# identity key
session.identity_map
(Documentation: Session.new, Session.dirty, Session.deleted, Session.identity_map).
Session Referencing Behavior
Objects within the session are weakly referenced. This means that when they are dereferenced in the outside application, they fall out of scope from within the Session as well and are subject to garbage collection by the Python interpreter. The exceptions to this include objects which are pending, objects which are marked as deleted, or persistent objects which have pending changes on them. After a full flush, these collections are all empty, and all objects are again weakly referenced.
To cause objects in the Session to remain strongly referenced, usually a simple approach is all that’s needed. Examples of externally managed strong-referencing behavior include loading objects into a local dictionary keyed to their primary key, or into lists or sets for the span of time that they need to remain referenced. These collections can be associated with a Session, if desired, by placing them into the Session.info dictionary.
An event based approach is also feasible. A simple recipe that provides “strong referencing” behavior for all objects as they remain within the persistent state is as follows:
from sqlalchemy import event


def strong_reference_session(session):
    @event.listens_for(session, "pending_to_persistent")
    @event.listens_for(session, "deleted_to_persistent")
    @event.listens_for(session, "detached_to_persistent")
    @event.listens_for(session, "loaded_as_persistent")
    def strong_ref_object(sess, instance):
        if "refs" not in sess.info:
            sess.info["refs"] = refs = set()
        else:
            refs = sess.info["refs"]

        refs.add(instance)

    @event.listens_for(session, "persistent_to_detached")
    @event.listens_for(session, "persistent_to_deleted")
    @event.listens_for(session, "persistent_to_transient")
    def deref_object(sess, instance):
        sess.info["refs"].discard(instance)
Above, we intercept the SessionEvents.pending_to_persistent(), SessionEvents.detached_to_persistent(), SessionEvents.deleted_to_persistent() and SessionEvents.loaded_as_persistent() event hooks in order to intercept objects as they enter the persistent transition, and the SessionEvents.persistent_to_detached() and SessionEvents.persistent_to_deleted() hooks to intercept objects as they leave the persistent state.
The above function may be called for any Session in order to provide strong-referencing behavior on a per-Session basis:
from sqlalchemy.orm import Session

my_session = Session()
strong_reference_session(my_session)
It may also be called for any sessionmaker:
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()
strong_reference_session(maker)
Merging
Session.merge() transfers state from an outside object into a new or already existing instance within a session. It also reconciles the incoming data against the state of the database, producing a history stream which will be applied towards the next flush, or alternatively can be made to produce a simple “transfer” of state without producing change history or accessing the database. Usage is as follows:
merged_object = session.merge(existing_object)
When given an instance, it follows these steps:
* It examines the primary key of the instance. If it’s present, it attempts to locate that instance in the local identity map. If the load=True flag is left at its default, it also checks the database for this primary key if not located locally.
* If the given instance has no primary key, or if no instance can be found with the primary key given, a new instance is created.
* The state of the given instance is then copied onto the located/newly created instance. For attribute values which are present on the source instance, the value is transferred to the target instance. For attribute values that aren’t present on the source instance, the corresponding attribute on the target instance is expired from memory, which discards any locally present value from the target instance for that attribute, but no direct modification is made to the database-persisted value for that attribute.
If the load=True flag is left at its default, this copy process emits events and will load the target object’s unloaded collections for each attribute present on the source object, so that the incoming state can be reconciled against what’s present in the database. If load is passed as False, the incoming data is “stamped” directly without producing any history.
* The operation is cascaded to related objects and collections, as indicated by the merge cascade (see Cascades).
* The new instance is returned.
With Session.merge(), the given “source” instance is not modified nor is it associated with the target Session, and remains available to be merged with any number of other Session objects. Session.merge() is useful for taking the state of any kind of object structure without regard for its origins or current session associations and copying its state into a new session. Here’s some examples:
* An application which reads an object structure from a file and wishes to save it to the database might parse the file, build up the structure, and then use Session.merge() to save it to the database, ensuring that the data within the file is used to formulate the primary key of each element of the structure. Later, when the file has changed, the same process can be re-run, producing a slightly different object structure, which can then be merged in again, and the Session will automatically update the database to reflect those changes, loading each object from the database by primary key and then updating its state with the new state given.
* An application is storing objects in an in-memory cache, shared by many Session objects simultaneously. Session.merge() is used each time an object is retrieved from the cache to create a local copy of it in each Session which requests it. The cached object remains detached; only its state is moved into copies of itself that are local to individual Session objects.
In the caching use case, it’s common to use the load=False flag to remove the overhead of reconciling the object’s state with the database. There’s also a “bulk” version of Session.merge() called Query.merge_result() that was designed to work with cache-extended Query objects - see the section Dogpile Caching.
* An application wants to transfer the state of a series of objects into a Session maintained by a worker thread or other concurrent system. Session.merge() makes a copy of each object to be placed into this new Session. At the end of the operation, the parent thread/process maintains the objects it started with, and the thread/worker can proceed with local copies of those objects.
In the “transfer between threads/processes” use case, the application may want to use the load=False flag as well to avoid overhead and redundant SQL queries as the data is transferred.
Merge Tips
Session.merge() is an extremely useful method for many purposes. However, it deals with the intricate border between objects that are transient/detached and those that are persistent, as well as the automated transference of state. The wide variety of scenarios that can present themselves here often require a more careful approach to the state of objects. Common problems with merge usually involve some unexpected state regarding the object being passed to Session.merge().
Lets use the canonical example of the User and Address objects:
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    addresses = relationship("Address", backref="user")


class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String(50), nullable=False)
    user_id = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
Assume a User object with one Address, already persistent:
u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
session.add(u1)
session.commit()
We now create a1, an object outside the session, which we’d like to merge on top of the existing Address:
existing_a1 = u1.addresses[0]
a1 = Address(id=existing_a1.id)
A surprise would occur if we said this:
a1.user = u1
a1 = session.merge(a1)
session.commit()
sqlalchemy.orm.exc.FlushError: New instance <Address at 0x1298f50>
with identity key (<class '__main__.Address'>, (1,)) conflicts with
persistent instance <Address at 0x12a25d0>
Why is that ? We weren’t careful with our cascades. The assignment of a1.user to a persistent object cascaded to the backref of User.addresses and made our a1 object pending, as though we had added it. Now we have two Address objects in the session:
a1 = Address()
a1.user = u1
a1 in session
True
existing_a1 in session
True
a1 is existing_a1
False
Above, our a1 is already pending in the session. The subsequent Session.merge() operation essentially does nothing. Cascade can be configured via the relationship.cascade option on relationship(), although in this case it would mean removing the save-update cascade from the User.addresses relationship - and usually, that behavior is extremely convenient. The solution here would usually be to not assign a1.user to an object already persistent in the target session.
The cascade_backrefs=False option of relationship() will also prevent the Address from being added to the session via the a1.user = u1 assignment.
Further detail on cascade operation is at Cascades.
Another example of unexpected state:
a1 = Address(id=existing_a1.id, user_id=u1.id)
a1.user = None
a1 = session.merge(a1)
session.commit()
sqlalchemy.exc.IntegrityError: (IntegrityError) address.user_id
may not be NULL
Above, the assignment of user takes precedence over the foreign key assignment of user_id, with the end result that None is applied to user_id, causing a failure.
Most Session.merge() issues can be examined by first checking - is the object prematurely in the session ?
a1 = Address(id=existing_a1, user_id=user.id)
assert a1 not in session
a1 = session.merge(a1)
Or is there state on the object that we don’t want ? Examining __dict__ is a quick way to check:
a1 = Address(id=existing_a1, user_id=user.id)
a1.user
a1.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x1298d10>,
    'user_id': 1,
    'id': 1,
    'user': None}
# we don't want user=None merged, remove it
del a1.user
a1 = session.merge(a1)
# success
session.commit()
Expunging
Expunge removes an object from the Session, sending persistent instances to the detached state, and pending instances to the transient state:
session.expunge(obj1)
To remove all items, call Session.expunge_all() (this method was formerly known as clear()).
Refreshing / Expiring
Expiring means that the database-persisted data held inside a series of object attributes is erased, in such a way that when those attributes are next accessed, a SQL query is emitted which will refresh that data from the database.
When we talk about expiration of data we are usually talking about an object that is in the persistent state. For example, if we load an object as follows:
user = session.scalars(select(User).filter_by(name="user1").limit(1)).first()
The above User object is persistent, and has a series of attributes present; if we were to look inside its __dict__, we’d see that state loaded:
user.__dict__
{
  'id': 1, 'name': u'user1',
  '_sa_instance_state': <>,
}
where id and name refer to those columns in the database. _sa_instance_state is a non-database-persisted value used by SQLAlchemy internally (it refers to the InstanceState for the instance. While not directly relevant to this section, if we want to get at it, we should use the inspect() function to access it).
At this point, the state in our User object matches that of the loaded database row. But upon expiring the object using a method such as Session.expire(), we see that the state is removed:
session.expire(user)
user.__dict__
{'_sa_instance_state': <>}
We see that while the internal “state” still hangs around, the values which correspond to the id and name columns are gone. If we were to access one of these columns and are watching SQL, we’d see this:
print(user.name)
SELECT user.id AS user_id, user.name AS user_name
FROM user
WHERE user.id = ?
(1,)
user1
Above, upon accessing the expired attribute user.name, the ORM initiated a lazy load to retrieve the most recent state from the database, by emitting a SELECT for the user row to which this user refers. Afterwards, the __dict__ is again populated:
user.__dict__
{
  'id': 1, 'name': u'user1',
  '_sa_instance_state': <>,
}
Note
While we are peeking inside of __dict__ in order to see a bit of what SQLAlchemy does with object attributes, we should not modify the contents of __dict__ directly, at least as far as those attributes which the SQLAlchemy ORM is maintaining (other attributes outside of SQLA’s realm are fine). This is because SQLAlchemy uses descriptors in order to track the changes we make to an object, and when we modify __dict__ directly, the ORM won’t be able to track that we changed something.
Another key behavior of both Session.expire() and Session.refresh() is that all un-flushed changes on an object are discarded. That is, if we were to modify an attribute on our User:
user.name = "user2"
but then we call Session.expire() without first calling Session.flush(), our pending value of 'user2' is discarded:
session.expire(user)
user.name
'user1'
The Session.expire() method can be used to mark as “expired” all ORM-mapped attributes for an instance:
# expire all ORM-mapped attributes on obj1
session.expire(obj1)
it can also be passed a list of string attribute names, referring to specific attributes to be marked as expired:
# expire only attributes obj1.attr1, obj1.attr2
session.expire(obj1, ["attr1", "attr2"])
The Session.expire_all() method allows us to essentially call Session.expire() on all objects contained within the Session at once:
session.expire_all()
The Session.refresh() method has a similar interface, but instead of expiring, it emits an immediate SELECT for the object’s row immediately:
# reload all attributes on obj1
session.refresh(obj1)
Session.refresh() also accepts a list of string attribute names, but unlike Session.expire(), expects at least one name to be that of a column-mapped attribute:
# reload obj1.attr1, obj1.attr2
session.refresh(obj1, ["attr1", "attr2"])
Tip
An alternative method of refreshing which is often more flexible is to use the Populate Existing feature of the ORM, available for 2.0 style queries with select() as well as from the Query.populate_existing() method of Query within 1.x style queries. Using this execution option, all of the ORM objects returned in the result set of the statement will be refreshed with data from the database:
stmt = (
    select(User)
    .execution_options(populate_existing=True)
    .where((User.name.in_(["a", "b", "c"])))
)
for user in session.execute(stmt).scalars():
    print(user)  # will be refreshed for those columns that came back from the query
See Populate Existing for further detail.
What Actually Loads
The SELECT statement that’s emitted when an object marked with Session.expire() or loaded with Session.refresh() varies based on several factors, including:
* The load of expired attributes is triggered from column-mapped attributes only. While any kind of attribute can be marked as expired, including a relationship() - mapped attribute, accessing an expired relationship() attribute will emit a load only for that attribute, using standard relationship-oriented lazy loading. Column-oriented attributes, even if expired, will not load as part of this operation, and instead will load when any column-oriented attribute is accessed.
* relationship()- mapped attributes will not load in response to expired column-based attributes being accessed.
* Regarding relationships, Session.refresh() is more restrictive than Session.expire() with regards to attributes that aren’t column-mapped. Calling Session.refresh() and passing a list of names that only includes relationship-mapped attributes will actually raise an error. In any case, non-eager-loading relationship() attributes will not be included in any refresh operation.
* relationship() attributes configured as “eager loading” via the relationship.lazy parameter will load in the case of Session.refresh(), if either no attribute names are specified, or if their names are included in the list of attributes to be refreshed.
* Attributes that are configured as deferred() will not normally load, during either the expired-attribute load or during a refresh. An unloaded attribute that’s deferred() instead loads on its own when directly accessed, or if part of a “group” of deferred attributes where an unloaded attribute in that group is accessed.
* For expired attributes that are loaded on access, a joined-inheritance table mapping will emit a SELECT that typically only includes those tables for which unloaded attributes are present. The action here is sophisticated enough to load only the parent or child table, for example, if the subset of columns that were originally expired encompass only one or the other of those tables.
* When Session.refresh() is used on a joined-inheritance table mapping, the SELECT emitted will resemble that of when Session.query() is used on the target object’s class. This is typically all those tables that are set up as part of the mapping.
When to Expire or Refresh
The Session uses the expiration feature automatically whenever the transaction referred to by the session ends. Meaning, whenever Session.commit() or Session.rollback() is called, all objects within the Session are expired, using a feature equivalent to that of the Session.expire_all() method. The rationale is that the end of a transaction is a demarcating point at which there is no more context available in order to know what the current state of the database is, as any number of other transactions may be affecting it. Only when a new transaction starts can we again have access to the current state of the database, at which point any number of changes may have occurred.
Transaction Isolation
Of course, most databases are capable of handling multiple transactions at once, even involving the same rows of data. When a relational database handles multiple transactions involving the same tables or rows, this is when the isolation aspect of the database comes into play. The isolation behavior of different databases varies considerably and even on a single database can be configured to behave in different ways (via the so-called isolation level setting). In that sense, the Session can’t fully predict when the same SELECT statement, emitted a second time, will definitely return the data we already have, or will return new data. So as a best guess, it assumes that within the scope of a transaction, unless it is known that a SQL expression has been emitted to modify a particular row, there’s no need to refresh a row unless explicitly told to do so.
The Session.expire() and Session.refresh() methods are used in those cases when one wants to force an object to re-load its data from the database, in those cases when it is known that the current state of data is possibly stale. Reasons for this might include:
* some SQL has been emitted within the transaction outside of the scope of the ORM’s object handling, such as if a Table.update() construct were emitted using the Session.execute() method;
* if the application is attempting to acquire data that is known to have been modified in a concurrent transaction, and it is also known that the isolation rules in effect allow this data to be visible.
The second bullet has the important caveat that “it is also known that the isolation rules in effect allow this data to be visible.” This means that it cannot be assumed that an UPDATE that happened on another database connection will yet be visible here locally; in many cases, it will not. This is why if one wishes to use Session.expire() or Session.refresh() in order to view data between ongoing transactions, an understanding of the isolation behavior in effect is essential.
See also
Session.expire()
Session.expire_all()
Session.refresh()
Populate Existing - allows any ORM query to refresh objects as they would be loaded normally, refreshing all matching objects in the identity map against the results of a SELECT statement.
isolation - glossary explanation of isolation which includes links to Wikipedia.
The SQLAlchemy Session In-Depth - a video + slides with an in-depth discussion of the object lifecycle including the role of data expiration.


Cascades
Mappers support the concept of configurable cascade behavior on relationship() constructs. This refers to how operations performed on a “parent” object relative to a particular Session should be propagated to items referred to by that relationship (e.g. “child” objects), and is affected by the relationship.cascade option.
The default behavior of cascade is limited to cascades of the so-called save-update and merge settings. The typical “alternative” setting for cascade is to add the delete and delete-orphan options; these settings are appropriate for related objects which only exist as long as they are attached to their parent, and are otherwise deleted.
Cascade behavior is configured using the relationship.cascade option on relationship():
class Order(Base):
    __tablename__ = "order"

    items = relationship("Item", cascade="all, delete-orphan")
    customer = relationship("User", cascade="save-update")
To set cascades on a backref, the same flag can be used with the backref() function, which ultimately feeds its arguments back into relationship():
class Item(Base):
    __tablename__ = "item"

    order = relationship(
        "Order", backref=backref("items", cascade="all, delete-orphan")
    )
The Origins of Cascade
SQLAlchemy’s notion of cascading behavior on relationships, as well as the options to configure them, are primarily derived from the similar feature in the Hibernate ORM; Hibernate refers to “cascade” in a few places such as in Example: Parent/Child. If cascades are confusing, we’ll refer to their conclusion, stating “The sections we have just covered can be a bit confusing. However, in practice, it all works out nicely.”
The default value of relationship.cascade is save-update, merge. The typical alternative setting for this parameter is either all or more commonly all, delete-orphan. The all symbol is a synonym for save-update, merge, refresh-expire, expunge, delete, and using it in conjunction with delete-orphan indicates that the child object should follow along with its parent in all cases, and be deleted once it is no longer associated with that parent.
Warning
The all cascade option implies the refresh-expire cascade setting which may not be desirable when using the Asynchronous I/O (asyncio) extension, as it will expire related objects more aggressively than is typically appropriate in an explicit IO context. See the notes at Preventing Implicit IO when Using AsyncSession for further background.
The list of available values which can be specified for the relationship.cascade parameter are described in the following subsections.
save-update
save-update cascade indicates that when an object is placed into a Session via Session.add(), all the objects associated with it via this relationship() should also be added to that same Session. Suppose we have an object user1 with two related objects address1, address2:
user1 = User()
address1, address2 = Address(), Address()
user1.addresses = [address1, address2]
If we add user1 to a Session, it will also add address1, address2 implicitly:
sess = Session()
sess.add(user1)
address1 in sess
True
save-update cascade also affects attribute operations for objects that are already present in a Session. If we add a third object, address3 to the user1.addresses collection, it becomes part of the state of that Session:
address3 = Address()
user1.addresses.append(address3)
address3 in sess
True
A save-update cascade can exhibit surprising behavior when removing an item from a collection or de-associating an object from a scalar attribute. In some cases, the orphaned objects may still be pulled into the ex-parent’s Session; this is so that the flush process may handle that related object appropriately. This case usually only arises if an object is removed from one Session and added to another:
user1 = sess1.scalars(select(User).filter_by(id=1)).first()
address1 = user1.addresses[0]
sess1.close()  # user1, address1 no longer associated with sess1
user1.addresses.remove(address1)  # address1 no longer associated with user1
sess2 = Session()
sess2.add(user1)  # but it still gets added to the new session,
address1 in sess2  # because it's still "pending" for flush
True
The save-update cascade is on by default, and is typically taken for granted; it simplifies code by allowing a single call to Session.add() to register an entire structure of objects within that Session at once. While it can be disabled, there is usually not a need to do so.
Behavior of save-update cascade with bi-directional relationships
The save-update cascade takes place uni-directionally in the context of a bi-directional relationship, i.e. when using the relationship.back_populates or relationship.backref parameters to create two separate relationship() objects which refer to each other.
An object that’s not associated with a Session, when assigned to an attribute or collection on a parent object that is associated with a Session, will be automatically added to that same Session. However, the same operation in reverse will not have this effect; an object that’s not associated with a Session, upon which a child object that is associated with a Session is assigned, will not result in an automatic addition of that parent object to the Session. The overall subject of this behavior is known as “cascade backrefs”, and represents a change in behavior that was standardized as of SQLAlchemy 2.0.
To illustrate, given a mapping of Order objects which relate bi-directionally to a series of Item objects via relationships Order.items and Item.order:
mapper_registry.map_imperatively(
    Order,
    order_table,
    properties={"items": relationship(Item, back_populates="order")},
)

mapper_registry.map_imperatively(
    Item,
    item_table,
    properties={"order": relationship(Order, back_populates="items")},
)
If an Order is already associated with a Session, and an Item object is then created and appended to the Order.items collection of that Order, the Item will be automatically cascaded into that same Session:
o1 = Order()
session.add(o1)
o1 in session
True

i1 = Item()
o1.items.append(i1)
o1 is i1.order
True
i1 in session
True
Above, the bidirectional nature of Order.items and Item.order means that appending to Order.items also assigns to Item.order. At the same time, the save-update cascade allowed for the Item object to be added to the same Session which the parent Order was already associated.
However, if the operation above is performed in the reverse direction, where Item.order is assigned rather than appending directly to Order.item, the cascade operation into the Session will not take place automatically, even though the object assignments Order.items and Item.order will be in the same state as in the previous example:
o1 = Order()
session.add(o1)
o1 in session
True

i1 = Item()
i1.order = o1
i1 in order.items
True
i1 in session
False
In the above case, after the Item object is created and all the desired state is set upon it, it should then be added to the Session explicitly:
session.add(i1)
In older versions of SQLAlchemy, the save-update cascade would occur bidirectionally in all cases. It was then made optional using an option known as cascade_backrefs. Finally, in SQLAlchemy 1.4 the old behavior was deprecated and the cascade_backrefs option was removed in SQLAlchemy 2.0. The rationale is that users generally do not find it intuitive that assigning to an attribute on an object, illustrated above as the assignment of i1.order = o1, would alter the persistence state of that object i1 such that it’s now pending within a Session, and there would frequently be subsequent issues where autoflush would prematurely flush the object and cause errors, in those cases where the given object was still being constructed and wasn’t in a ready state to be flushed. The option to select between uni-directional and bi-directional behvaiors was also removed, as this option created two slightly different ways of working, adding to the overall learning curve of the ORM as well as to the documentation and user support burden.
See also
cascade_backrefs behavior deprecated for removal in 2.0 - background on the change in behavior for “cascade backrefs”
delete
The delete cascade indicates that when a “parent” object is marked for deletion, its related “child” objects should also be marked for deletion. If for example we have a relationship User.addresses with delete cascade configured:
class User(Base):
    # 

    addresses = relationship("Address", cascade="all, delete")
If using the above mapping, we have a User object and two related Address objects:
user1 = sess1.scalars(select(User).filter_by(id=1)).first()
address1, address2 = user1.addresses
If we mark user1 for deletion, after the flush operation proceeds, address1 and address2 will also be deleted:
sess.delete(user1)
sess.commit()
DELETE FROM address WHERE address.id = ?
((1,), (2,))
DELETE FROM user WHERE user.id = ?
(1,)
COMMIT
Alternatively, if our User.addresses relationship does not have delete cascade, SQLAlchemy’s default behavior is to instead de-associate address1 and address2 from user1 by setting their foreign key reference to NULL. Using a mapping as follows:
class User(Base):
    # 

    addresses = relationship("Address")
Upon deletion of a parent User object, the rows in address are not deleted, but are instead de-associated:
sess.delete(user1)
sess.commit()
UPDATE address SET user_id=? WHERE address.id = ?
(None, 1)
UPDATE address SET user_id=? WHERE address.id = ?
(None, 2)
DELETE FROM user WHERE user.id = ?
(1,)
COMMIT
delete cascade on one-to-many relationships is often combined with delete-orphan cascade, which will emit a DELETE for the related row if the “child” object is deassociated from the parent. The combination of delete and delete-orphan cascade covers both situations where SQLAlchemy has to decide between setting a foreign key column to NULL versus deleting the row entirely.
The feature by default works completely independently of database-configured FOREIGN KEY constraints that may themselves configure CASCADE behavior. In order to integrate more efficiently with this configuration, additional directives described at Using foreign key ON DELETE cascade with ORM relationships should be used.
Warning
Note that the ORM’s “delete” and “delete-orphan” behavior applies only to the use of the Session.delete() method to mark individual ORM instances for deletion within the unit of work process. It does not apply to “bulk” deletes, which would be emitted using the delete() construct as illustrated at ORM UPDATE and DELETE with Custom WHERE Criteria. See Important Notes and Caveats for ORM-Enabled Update and Delete for additional background.
See also
Using foreign key ON DELETE cascade with ORM relationships
Using delete cascade with many-to-many relationships
delete-orphan
Using delete cascade with many-to-many relationships
The cascade="all, delete" option works equally well with a many-to-many relationship, one that uses relationship.secondary to indicate an association table. When a parent object is deleted, and therefore de-associated with its related objects, the unit of work process will normally delete rows from the association table, but leave the related objects intact. When combined with cascade="all, delete", additional DELETE statements will take place for the child rows themselves.
The following example adapts that of Many To Many to illustrate the cascade="all, delete" setting on one side of the association:
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id")),
    Column("right_id", Integer, ForeignKey("right.id")),
)


class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )


class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
    )
Above, when a Parent object is marked for deletion using Session.delete(), the flush process will as usual delete the associated rows from the association table, however per cascade rules it will also delete all related Child rows.
Warning
If the above cascade="all, delete" setting were configured on both relationships, then the cascade action would continue cascading through all Parent and Child objects, loading each children and parents collection encountered and deleting everything that’s connected. It is typically not desirable for “delete” cascade to be configured bidirectionally.
See also
Deleting Rows from the Many to Many Table
Using foreign key ON DELETE with many-to-many relationships
Using foreign key ON DELETE cascade with ORM relationships
The behavior of SQLAlchemy’s “delete” cascade overlaps with the ON DELETE feature of a database FOREIGN KEY constraint. SQLAlchemy allows configuration of these schema-level DDL behaviors using the ForeignKey and ForeignKeyConstraint constructs; usage of these objects in conjunction with Table metadata is described at ON UPDATE and ON DELETE.
In order to use ON DELETE foreign key cascades in conjunction with relationship(), it’s important to note first and foremost that the relationship.cascade setting must still be configured to match the desired “delete” or “set null” behavior (using delete cascade or leaving it omitted), so that whether the ORM or the database level constraints will handle the task of actually modifying the data in the database, the ORM will still be able to appropriately track the state of locally present objects that may be affected.
There is then an additional option on relationship() which indicates the degree to which the ORM should try to run DELETE/UPDATE operations on related rows itself, vs. how much it should rely upon expecting the database-side FOREIGN KEY constraint cascade to handle the task; this is the relationship.passive_deletes parameter and it accepts options False (the default), True and "all".
The most typical example is that where child rows are to be deleted when parent rows are deleted, and that ON DELETE CASCADE is configured on the relevant FOREIGN KEY constraint as well:
class Parent(Base):
    __tablename__ = "parent"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        back_populates="parent",
        cascade="all, delete",
        passive_deletes=True,
    )


class Child(Base):
    __tablename__ = "child"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("parent.id", ondelete="CASCADE"))
    parent = relationship("Parent", back_populates="children")
The behavior of the above configuration when a parent row is deleted is as follows:
1. The application calls session.delete(my_parent), where my_parent is an instance of Parent.
2. When the Session next flushes changes to the database, all of the currently loaded items within the my_parent.children collection are deleted by the ORM, meaning a DELETE statement is emitted for each record.
3. If the my_parent.children collection is unloaded, then no DELETE statements are emitted. If the relationship.passive_deletes flag were not set on this relationship(), then a SELECT statement for unloaded Child objects would have been emitted.
4. A DELETE statement is then emitted for the my_parent row itself.
5. The database-level ON DELETE CASCADE setting ensures that all rows in child which refer to the affected row in parent are also deleted.
6. The Parent instance referred to by my_parent, as well as all instances of Child that were related to this object and were loaded (i.e. step 2 above took place), are de-associated from the Session.
Note
To use “ON DELETE CASCADE”, the underlying database engine must support FOREIGN KEY constraints and they must be enforcing:
* When using MySQL, an appropriate storage engine must be selected. See CREATE TABLE arguments including Storage Engines for details.
* When using SQLite, foreign key support must be enabled explicitly. See Foreign Key Support for details.
Notes on Passive Deletes
It is important to note the differences between the ORM and the relational database’s notion of “cascade” as well as how they integrate:
* A database level ON DELETE cascade is configured effectively on the many-to-one side of the relationship; that is, we configure it relative to the FOREIGN KEY constraint that is the “many” side of a relationship. At the ORM level, this direction is reversed. SQLAlchemy handles the deletion of “child” objects relative to a “parent” from the “parent” side, which means that delete and delete-orphan cascade are configured on the one-to-many side.
* Database level foreign keys with no ON DELETE setting are often used to prevent a parent row from being removed, as it would necessarily leave an unhandled related row present. If this behavior is desired in a one-to-many relationship, SQLAlchemy’s default behavior of setting a foreign key to NULL can be caught in one of two ways:
o The easiest and most common is just to set the foreign-key-holding column to NOT NULL at the database schema level. An attempt by SQLAlchemy to set the column to NULL will fail with a simple NOT NULL constraint exception.
o The other, more special case way is to set the relationship.passive_deletes flag to the string "all". This has the effect of entirely disabling SQLAlchemy’s behavior of setting the foreign key column to NULL, and a DELETE will be emitted for the parent row without any affect on the child row, even if the child row is present in memory. This may be desirable in the case when database-level foreign key triggers, either special ON DELETE settings or otherwise, need to be activated in all cases when a parent row is deleted.
* Database level ON DELETE cascade is generally much more efficient than relying upon the “cascade” delete feature of SQLAlchemy. The database can chain a series of cascade operations across many relationships at once; e.g. if row A is deleted, all the related rows in table B can be deleted, and all the C rows related to each of those B rows, and on and on, all within the scope of a single DELETE statement. SQLAlchemy on the other hand, in order to support the cascading delete operation fully, has to individually load each related collection in order to target all rows that then may have further related collections. That is, SQLAlchemy isn’t sophisticated enough to emit a DELETE for all those related rows at once within this context.
* SQLAlchemy doesn’t need to be this sophisticated, as we instead provide smooth integration with the database’s own ON DELETE functionality, by using the relationship.passive_deletes option in conjunction with properly configured foreign key constraints. Under this behavior, SQLAlchemy only emits DELETE for those rows that are already locally present in the Session; for any collections that are unloaded, it leaves them to the database to handle, rather than emitting a SELECT for them. The section Using foreign key ON DELETE cascade with ORM relationships provides an example of this use.
* While database-level ON DELETE functionality works only on the “many” side of a relationship, SQLAlchemy’s “delete” cascade has limited ability to operate in the reverse direction as well, meaning it can be configured on the “many” side to delete an object on the “one” side when the reference on the “many” side is deleted. However this can easily result in constraint violations if there are other objects referring to this “one” side from the “many”, so it typically is only useful when a relationship is in fact a “one to one”. The relationship.single_parent flag should be used to establish an in-Python assertion for this case.
Using foreign key ON DELETE with many-to-many relationships
As described at Using delete cascade with many-to-many relationships, “delete” cascade works for many-to-many relationships as well. To make use of ON DELETE CASCADE foreign keys in conjunction with many to many, FOREIGN KEY directives are configured on the association table. These directives can handle the task of automatically deleting from the association table, but cannot accommodate the automatic deletion of the related objects themselves.
In this case, the relationship.passive_deletes directive can save us some additional SELECT statements during a delete operation but there are still some collections that the ORM will continue to load, in order to locate affected child objects and handle them correctly.
Note
Hypothetical optimizations to this could include a single DELETE statement against all parent-associated rows of the association table at once, then use RETURNING to locate affected related child rows, however this is not currently part of the ORM unit of work implementation.
In this configuration, we configure ON DELETE CASCADE on both foreign key constraints of the association table. We configure cascade="all, delete" on the parent->child side of the relationship, and we can then configure passive_deletes=True on the other side of the bidirectional relationship as illustrated below:
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id", ondelete="CASCADE")),
    Column("right_id", Integer, ForeignKey("right.id", ondelete="CASCADE")),
)


class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )


class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
        passive_deletes=True,
    )
Using the above configuration, the deletion of a Parent object proceeds as follows:
1. A Parent object is marked for deletion using Session.delete().
2. When the flush occurs, if the Parent.children collection is not loaded, the ORM will first emit a SELECT statement in order to load the Child objects that correspond to Parent.children.
3. It will then then emit DELETE statements for the rows in association which correspond to that parent row.
4. for each Child object affected by this immediate deletion, because passive_deletes=True is configured, the unit of work will not need to try to emit SELECT statements for each Child.parents collection as it is assumed the corresponding rows in association will be deleted.
5. DELETE statements are then emitted for each Child object that was loaded from Parent.children.
delete-orphan
delete-orphan cascade adds behavior to the delete cascade, such that a child object will be marked for deletion when it is de-associated from the parent, not just when the parent is marked for deletion. This is a common feature when dealing with a related object that is “owned” by its parent, with a NOT NULL foreign key, so that removal of the item from the parent collection results in its deletion.
delete-orphan cascade implies that each child object can only have one parent at a time, and in the vast majority of cases is configured only on a one-to-many relationship. For the much less common case of setting it on a many-to-one or many-to-many relationship, the “many” side can be forced to allow only a single object at a time by configuring the relationship.single_parent argument, which establishes Python-side validation that ensures the object is associated with only one parent at a time, however this greatly limits the functionality of the “many” relationship and is usually not what’s desired.
See also
For relationship <relationship>, delete-orphan cascade is normally configured only on the “one” side of a one-to-many relationship, and not on the “many” side of a many-to-one or many-to-many relationship. - background on a common error scenario involving delete-orphan cascade.
merge
merge cascade indicates that the Session.merge() operation should be propagated from a parent that’s the subject of the Session.merge() call down to referred objects. This cascade is also on by default.
refresh-expire
refresh-expire is an uncommon option, indicating that the Session.expire() operation should be propagated from a parent down to referred objects. When using Session.refresh(), the referred objects are expired only, but not actually refreshed.
expunge
expunge cascade indicates that when the parent object is removed from the Session using Session.expunge(), the operation should be propagated down to referred objects.
Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships
The ORM in general never modifies the contents of a collection or scalar relationship during the flush process. This means, if your class has a relationship() that refers to a collection of objects, or a reference to a single object such as many-to-one, the contents of this attribute will not be modified when the flush process occurs. Instead, it is expected that the Session would eventually be expired, either through the expire-on-commit behavior of Session.commit() or through explicit use of Session.expire(). At that point, any referenced object or collection associated with that Session will be cleared and will re-load itself upon next access.
A common confusion that arises regarding this behavior involves the use of the Session.delete() method. When Session.delete() is invoked upon an object and the Session is flushed, the row is deleted from the database. Rows that refer to the target row via foreign key, assuming they are tracked using a relationship() between the two mapped object types, will also see their foreign key attributes UPDATED to null, or if delete cascade is set up, the related rows will be deleted as well. However, even though rows related to the deleted object might be themselves modified as well, no changes occur to relationship-bound collections or object references on the objects involved in the operation within the scope of the flush itself. This means if the object was a member of a related collection, it will still be present on the Python side until that collection is expired. Similarly, if the object were referenced via many-to-one or one-to-one from another object, that reference will remain present on that object until the object is expired as well.
Below, we illustrate that after an Address object is marked for deletion, it’s still present in the collection associated with the parent User, even after a flush:
address = user.addresses[1]
session.delete(address)
session.flush()
address in user.addresses
True
When the above session is committed, all attributes are expired. The next access of user.addresses will re-load the collection, revealing the desired state:
session.commit()
address in user.addresses
False
There is a recipe for intercepting Session.delete() and invoking this expiration automatically; see ExpireRelationshipOnFKChange for this. However, the usual practice of deleting items within collections is to forego the usage of Session.delete() directly, and instead use cascade behavior to automatically invoke the deletion as a result of removing the object from the parent collection. The delete-orphan cascade accomplishes this, as illustrated in the example below:
class User(Base):
    __tablename__ = "user"

    # 

    addresses = relationship("Address", cascade="all, delete-orphan")


# 

del user.addresses[1]
session.flush()
Where above, upon removing the Address object from the User.addresses collection, the delete-orphan cascade has the effect of marking the Address object for deletion in the same way as passing it to Session.delete().
The delete-orphan cascade can also be applied to a many-to-one or one-to-one relationship, so that when an object is de-associated from its parent, it is also automatically marked for deletion. Using delete-orphan cascade on a many-to-one or one-to-one requires an additional flag relationship.single_parent which invokes an assertion that this related object is not to shared with any other parent simultaneously:
class User(Base):
    # 

    preference = relationship(
        "Preference", cascade="all, delete-orphan", single_parent=True
    )
Above, if a hypothetical Preference object is removed from a User, it will be deleted on flush:
some_user.preference = None
session.flush()  # will delete the Preference object
See also
Cascades for detail on cascades.


Transactions and Connection Management
Managing Transactions
Changed in version 1.4: Session transaction management has been revised to be clearer and easier to use. In particular, it now features “autobegin” operation, which means the point at which a transaction begins may be controlled, without using the legacy “autocommit” mode.
The Session tracks the state of a single “virtual” transaction at a time, using an object called SessionTransaction. This object then makes use of the underlying Engine or engines to which the Session object is bound in order to start real connection-level transactions using the Connection object as needed.
This “virtual” transaction is created automatically when needed, or can alternatively be started using the Session.begin() method. To as great a degree as possible, Python context manager use is supported both at the level of creating Session objects as well as to maintain the scope of the SessionTransaction.
Below, assume we start with a Session:
from sqlalchemy.orm import Session

session = Session(engine)
We can now run operations within a demarcated transaction using a context manager:
with session.begin():
    session.add(some_object())
    session.add(some_other_object())
# commits transaction at the end, or rolls back if there
# was an exception raised
At the end of the above context, assuming no exceptions were raised, any pending objects will be flushed to the database and the database transaction will be committed. If an exception was raised within the above block, then the transaction would be rolled back. In both cases, the above Session subsequent to exiting the block is ready to be used in subsequent transactions.
The Session.begin() method is optional, and the Session may also be used in a commit-as-you-go approach, where it will begin transactions automatically as needed; these only need be committed or rolled back:
session = Session(engine)

session.add(some_object())
session.add(some_other_object())

session.commit()  # commits

# will automatically begin again
result = session.execute(text("< some select statement >"))
session.add_all([more_objects, ])
session.commit()  # commits

session.add(still_another_object)
session.flush()  # flush still_another_object
session.rollback()  # rolls back still_another_object
The Session itself features a Session.close() method. If the Session is begun within a transaction that has not yet been committed or rolled back, this method will cancel (i.e. rollback) that transaction, and also expunge all objects contained within the Session object’s state. If the Session is being used in such a way that a call to Session.commit() or Session.rollback() is not guaranteed (e.g. not within a context manager or similar), the close method may be used to ensure all resources are released:
# expunges all objects, releases all transactions unconditionally
# (with rollback), releases all database connections back to their
# engines
session.close()
Finally, the session construction / close process can itself be run via context manager. This is the best way to ensure that the scope of a Session object’s use is scoped within a fixed block. Illustrated via the Session constructor first:
with Session(engine) as session:
    session.add(some_object())
    session.add(some_other_object())

    session.commit()  # commits

    session.add(still_another_object)
    session.flush()  # flush still_another_object

    session.commit()  # commits

    result = session.execute(text("<some SELECT statement>"))

# remaining transactional state from the .execute() call is
# discarded
Similarly, the sessionmaker can be used in the same way:
Session = sessionmaker(engine)

with Session() as session:
    with session.begin():
        session.add(some_object)
    # commits

# closes the Session
sessionmaker itself includes a sessionmaker.begin() method to allow both operations to take place at once:
with Session.begin() as session:
    session.add(some_object)
Using SAVEPOINT
SAVEPOINT transactions, if supported by the underlying engine, may be delineated using the Session.begin_nested() method:
Session = sessionmaker()

with Session.begin() as session:
    session.add(u1)
    session.add(u2)

    nested = session.begin_nested()  # establish a savepoint
    session.add(u3)
    nested.rollback()  # rolls back u3, keeps u1 and u2

# commits u1 and u2
Each time Session.begin_nested() is called, a new “BEGIN SAVEPOINT” command is emitted to the database within the scope of the current database transaction (starting one if not already in progress), and an object of type SessionTransaction is returned, which represents a handle to this SAVEPOINT. When the .commit() method on this object is called, “RELEASE SAVEPOINT” is emitted to the database, and if instead the .rollback() method is called, “ROLLBACK TO SAVEPOINT” is emitted. The enclosing database transaction remains in progress.
Session.begin_nested() is typically used as a context manager where specific per-instance errors may be caught, in conjunction with a rollback emitted for that portion of the transaction’s state, without rolling back the whole transaction, as in the example below:
for record in records:
    try:
        with session.begin_nested():
            session.merge(record)
    except:
        print("Skipped record %s" % record)
session.commit()
When the context manager yielded by Session.begin_nested() completes, it “commits” the savepoint, which includes the usual behavior of flushing all pending state. When an error is raised, the savepoint is rolled back and the state of the Session local to the objects that were changed is expired.
This pattern is ideal for situations such as using PostgreSQL and catching IntegrityError to detect duplicate rows; PostgreSQL normally aborts the entire transaction when such an error is raised, however when using SAVEPOINT, the outer transaction is maintained. In the example below a list of data is persisted into the database, with the occasional “duplicate primary key” record skipped, without rolling back the entire operation:
from sqlalchemy import exc

with session.begin():
    for record in records:
        try:
            with session.begin_nested():
                obj = SomeRecord(id=record["identifier"], name=record["name"])
                session.add(obj)
        except exc.IntegrityError:
            print(f"Skipped record {record} - row already exists")
When Session.begin_nested() is called, the Session first flushes all currently pending state to the database; this occurs unconditionally, regardless of the value of the Session.autoflush parameter which normally may be used to disable automatic flush. The rationale for this behavior is so that when a rollback on this nested transaction occurs, the Session may expire any in-memory state that was created within the scope of the SAVEPOINT, while ensuring that when those expired objects are refreshed, the state of the object graph prior to the beginning of the SAVEPOINT will be available to re-load from the database.
In modern versions of SQLAlchemy, when a SAVEPOINT initiated by Session.begin_nested() is rolled back, in-memory object state that was modified since the SAVEPOINT was created is expired, however other object state that was not altered since the SAVEPOINT began is maintained. This is so that subsequent operations can continue to make use of the otherwise unaffected data without the need for refreshing it from the database.
See also
Connection.begin_nested() - Core SAVEPOINT API
Session-level vs. Engine level transaction control
The Connection in Core and _session.Session in ORM feature equivalent transactional semantics, both at the level of the sessionmaker vs. the Engine, as well as the Session vs. the Connection. The following sections detail these scenarios based on the following scheme:
ORM                                           Core
-----------------------------------------     -----------------------------------
sessionmaker                                  Engine
Session                                       Connection
sessionmaker.begin()                          Engine.begin()
some_session.commit()                         some_connection.commit()
with some_sessionmaker() as session:          with some_engine.connect() as conn:
with some_sessionmaker.begin() as session:    with some_engine.begin() as conn:
with some_session.begin_nested() as sp:       with some_connection.begin_nested() as sp:
Commit as you go
Both Session and Connection feature Connection.commit() and Connection.rollback() methods. Using SQLAlchemy 2.0-style operation, these methods affect the outermost transaction in all cases. For the Session, it is assumed that Session.autobegin is left at its default value of True.
Engine:
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.connect() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    conn.commit()
Session:
Session = sessionmaker(engine)

with Session() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    session.commit()
Begin Once
Both sessionmaker and Engine feature a Engine.begin() method that will both procure a new object with which to execute SQL statements (the Session and Connection, respectively) and then return a context manager that will maintain a begin/commit/rollback context for that object.
Engine:
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
# commits and closes automatically
Session:
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
# commits and closes automatically
Nested Transaction
When using a SAVEPOINT via the Session.begin_nested() or Connection.begin_nested() methods, the transaction object returned must be used to commit or rollback the SAVEPOINT. Calling the Session.commit() or Connection.commit() methods will always commit the outermost transaction; this is a SQLAlchemy 2.0 specific behavior that is reversed from the 1.x series.
Engine:
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    savepoint = conn.begin_nested()
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    savepoint.commit()  # or rollback

# commits automatically
Session:
Session = sessionmaker(engine)

with Session.begin() as session:
    savepoint = session.begin_nested()
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    savepoint.commit()  # or rollback
# commits automatically
Explicit Begin
The Session features “autobegin” behavior, meaning that as soon as operations begin to take place, it ensures a SessionTransaction is present to track ongoing operations. This transaction is completed when Session.commit() is called.
It is often desirable, particularly in framework integrations, to control the point at which the “begin” operation occurs. To suit this, the Session uses an “autobegin” strategy, such that the Session.begin() method may be called directly for a Session that has not already had a transaction begun:
Session = sessionmaker(bind=engine)
session = Session()
session.begin()
try:
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
    session.commit()
except:
    session.rollback()
    raise
The above pattern is more idiomatically invoked using a context manager:
Session = sessionmaker(bind=engine)
session = Session()
with session.begin():
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
The Session.begin() method and the session’s “autobegin” process use the same sequence of steps to begin the transaction. This includes that the SessionEvents.after_transaction_create() event is invoked when it occurs; this hook is used by frameworks in order to integrate their own transactional processes with that of the ORM Session.
Enabling Two-Phase Commit
For backends which support two-phase operation (currently MySQL and PostgreSQL), the session can be instructed to use two-phase commit semantics. This will coordinate the committing of transactions across databases so that the transaction is either committed or rolled back in all databases. You can also Session.prepare() the session for interacting with transactions not managed by SQLAlchemy. To use two phase transactions set the flag twophase=True on the session:
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker(twophase=True)

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()

# .work with accounts and users

# commit.  session will issue a flush to all DBs, and a prepare step to all DBs,
# before committing both transactions
session.commit()
Setting Transaction Isolation Levels / DBAPI AUTOCOMMIT
Most DBAPIs support the concept of configurable transaction isolation levels. These are traditionally the four levels “READ UNCOMMITTED”, “READ COMMITTED”, “REPEATABLE READ” and “SERIALIZABLE”. These are usually applied to a DBAPI connection before it begins a new transaction, noting that most DBAPIs will begin this transaction implicitly when SQL statements are first emitted.
DBAPIs that support isolation levels also usually support the concept of true “autocommit”, which means that the DBAPI connection itself will be placed into a non-transactional autocommit mode. This usually means that the typical DBAPI behavior of emitting “BEGIN” to the database automatically no longer occurs, but it may also include other directives. When using this mode, the DBAPI does not use a transaction under any circumstances. SQLAlchemy methods like .begin(), .commit() and .rollback() pass silently.
SQLAlchemy’s dialects support settable isolation modes on a per-Engine or per-Connection basis, using flags at both the create_engine() level as well as at the Connection.execution_options() level.
When using the ORM Session, it acts as a facade for engines and connections, but does not expose transaction isolation directly. So in order to affect transaction isolation level, we need to act upon the Engine or Connection as appropriate.
See also
Setting Transaction Isolation Levels including DBAPI Autocommit - be sure to review how isolation levels work at the level of the SQLAlchemy Connection object as well.
Setting Isolation For A Sessionmaker / Engine Wide
To set up a Session or sessionmaker with a specific isolation level globally, the first technique is that an Engine can be constructed against a specific isolation level in all cases, which is then used as the source of connectivity for a Session and/or sessionmaker:
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    isolation_level="REPEATABLE READ",
)

Session = sessionmaker(eng)
Another option, useful if there are to be two engines with different isolation levels at once, is to use the Engine.execution_options() method, which will produce a shallow copy of the original Engine which shares the same connection pool as the parent engine. This is often preferable when operations will be separated into “transactional” and “autocommit” operations:
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

transactional_session = sessionmaker(eng)
autocommit_session = sessionmaker(autocommit_engine)
Above, both “eng” and "autocommit_engine" share the same dialect and connection pool. However the “AUTOCOMMIT” mode will be set upon connections when they are acquired from the autocommit_engine. The two sessionmaker objects “transactional_session” and “autocommit_session" then inherit these characteristics when they work with database connections.
The “autocommit_session” continues to have transactional semantics, including that Session.commit() and Session.rollback() still consider themselves to be “committing” and “rolling back” objects, however the transaction will be silently absent. For this reason, it is typical, though not strictly required, that a Session with AUTOCOMMIT isolation be used in a read-only fashion, that is:
with autocommit_session() as session:
    some_objects = session.execute(text("<statement>"))
    some_other_objects = session.execute(text("<statement>"))

# closes connection
Setting Isolation for Individual Sessions
When we make a new Session, either using the constructor directly or when we call upon the callable produced by a sessionmaker, we can pass the bind argument directly, overriding the pre-existing bind. We can for example create our Session from a default sessionmaker and pass an engine set for autocommit:
plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

# will normally use plain_engine
Session = sessionmaker(plain_engine)

# make a specific Session that will use the "autocommit" engine
with Session(bind=autocommit_engine) as session:
    # work with session
    
For the case where the Session or sessionmaker is configured with multiple “binds”, we can either re-specify the binds argument fully, or if we want to only replace specific binds, we can use the Session.bind_mapper() or Session.bind_table() methods:
with Session() as session:
    session.bind_mapper(User, autocommit_engine)
Setting Isolation for Individual Transactions
A key caveat regarding isolation level is that the setting cannot be safely modified on a Connection where a transaction has already started. Databases cannot change the isolation level of a transaction in progress, and some DBAPIs and SQLAlchemy dialects have inconsistent behaviors in this area.
Therefore it is preferable to use a Session that is up front bound to an engine with the desired isolation level. However, the isolation level on a per-connection basis can be affected by using the Session.connection() method at the start of a transaction:
from sqlalchemy.orm import Session

# assume session just constructed
sess = Session(bind=engine)

# call connection() with options before any other operations proceed.
# this will procure a new connection from the bound engine and begin a real
# database transaction.
sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

# work with session in SERIALIZABLE isolation level

# commit transaction.  the connection is released
# and reverted to its previous isolation level.
sess.commit()

# subsequent to commit() above, a new transaction may be begun if desired,
# which will proceed with the previous default isolation level unless
# it is set again.
Above, we first produce a Session using either the constructor or a sessionmaker. Then we explicitly set up the start of a database-level transaction by calling upon Session.connection(), which provides for execution options that will be passed to the connection before the database-level transaction is begun. The transaction proceeds with this selected isolation level. When the transaction completes, the isolation level is reset on the connection to its default before the connection is returned to the connection pool.
The Session.begin() method may also be used to begin the Session level transaction; calling upon Session.connection() subsequent to that call may be used to set up the per-connection-transaction isolation level:
sess = Session(bind=engine)

with sess.begin():
    # call connection() with options before any other operations proceed.
    # this will procure a new connection from the bound engine and begin a
    # real database transaction.
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # work with session in SERIALIZABLE isolation level

# outside the block, the transaction has been committed.  the connection is
# released and reverted to its previous isolation level.
Tracking Transaction State with Events
See the section Transaction Events for an overview of the available event hooks for session transaction state changes.
Joining a Session into an External Transaction (such as for test suites)
If a Connection is being used which is already in a transactional state (i.e. has a Transaction established), a Session can be made to participate within that transaction by just binding the Session to that Connection. The usual rationale for this is a test suite that allows ORM code to work freely with a Session, including the ability to call Session.commit(), where afterwards the entire database interaction is rolled back.
Changed in version 2.0: The “join into an external transaction” recipe is newly improved again in 2.0; event handlers to “reset” the nested transaction are no longer required.
The recipe works by establishing a Connection within a transaction and optionally a SAVEPOINT, then passing it to a Session as the “bind”; the Session.join_transaction_mode parameter is passed with the setting "create_savepoint", which indicates that new SAVEPOINTs should be created in order to implement BEGIN/COMMIT/ROLLBACK for the Session, which will leave the external transaction in the same state in which it was passed.
When the test tears down, the external transaction is rolled back so that any data changes throughout the test are reverted:
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
from unittest import TestCase

# global application scope.  create Session class, engine
Session = sessionmaker()

engine = create_engine("postgresql+psycopg2://")


class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def test_something(self):
        # use the session in tests.

        self.session.add(Foo())
        self.session.commit()

    def test_something_with_rollbacks(self):
        self.session.add(Bar())
        self.session.flush()
        self.session.rollback()

        self.session.add(Foo())
        self.session.commit()

    def tearDown(self):
        self.session.close()

        # rollback - everything that happened with the
        # Session above (including calls to commit())
        # is rolled back.
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
The above recipe is part of SQLAlchemy’s own CI to ensure that it remains working as expected.


Additional Persistence Techniques
Embedding SQL Insert/Update Expressions into a Flush
This feature allows the value of a database column to be set to a SQL expression instead of a literal value. It’s especially useful for atomic updates, calling stored procedures, etc. All you do is assign an expression to an attribute:
class SomeClass(Base):
    __tablename__ = "some_table"

    # 

    value = mapped_column(Integer)


someobject = session.get(SomeClass, 5)

# set 'value' attribute to a SQL expression adding one
someobject.value = SomeClass.value + 1

# issues "UPDATE some_table SET value=value+1"
session.commit()
This technique works both for INSERT and UPDATE statements. After the flush/commit operation, the value attribute on someobject above is expired, so that when next accessed the newly generated value will be loaded from the database.
The feature also has conditional support to work in conjunction with primary key columns. For backends that have RETURNING support (including Oracle Database, SQL Server, MariaDB 10.5, SQLite 3.35) a SQL expression may be assigned to a primary key column as well. This allows both the SQL expression to be evaluated, as well as allows any server side triggers that modify the primary key value on INSERT, to be successfully retrieved by the ORM as part of the object’s primary key:
class Foo(Base):
    __tablename__ = "foo"
    pk = mapped_column(Integer, primary_key=True)
    bar = mapped_column(Integer)


e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)

session = Session(e)

foo = Foo(pk=sql.select(sql.func.coalesce(sql.func.max(Foo.pk) + 1, 1)))
session.add(foo)
session.commit()
On PostgreSQL, the above Session will emit the following INSERT:
INSERT INTO foo (foopk, bar) VALUES
((SELECT coalesce(max(foo.foopk) + %(max_1)s, %(coalesce_2)s) AS coalesce_1
FROM foo), %(bar)s) RETURNING foo.foopk
Added in version 1.3: SQL expressions can now be passed to a primary key column during an ORM flush; if the database supports RETURNING, or if pysqlite is in use, the ORM will be able to retrieve the server-generated value as the value of the primary key attribute.
Using SQL Expressions with Sessions
SQL expressions and strings can be executed via the Session within its transactional context. This is most easily accomplished using the Session.execute() method, which returns a CursorResult in the same manner as an Engine or Connection:
Session = sessionmaker(bind=engine)
session = Session()

# execute a string statement
result = session.execute(text("select * from table where id=:id"), {"id": 7})

# execute a SQL expression construct
result = session.execute(select(mytable).where(mytable.c.id == 7))
The current Connection held by the Session is accessible using the Session.connection() method:
connection = session.connection()
The examples above deal with a Session that’s bound to a single Engine or Connection. To execute statements using a Session which is bound either to multiple engines, or none at all (i.e. relies upon bound metadata), both Session.execute() and Session.connection() accept a dictionary of bind arguments Session.execute.bind_arguments which may include “mapper” which is passed a mapped class or Mapper instance, which is used to locate the proper context for the desired engine:
Session = sessionmaker()
session = Session()

# need to specify mapper or class when executing
result = session.execute(
    text("select * from table where id=:id"),
    {"id": 7},
    bind_arguments={"mapper": MyMappedClass},
)

result = session.execute(
    select(mytable).where(mytable.c.id == 7), bind_arguments={"mapper": MyMappedClass}
)

connection = session.connection(MyMappedClass)
Changed in version 1.4: the mapper and clause arguments to Session.execute() are now passed as part of a dictionary sent as the Session.execute.bind_arguments parameter. The previous arguments are still accepted however this usage is deprecated.
Forcing NULL on a column with a default
The ORM considers any attribute that was never set on an object as a “default” case; the attribute will be omitted from the INSERT statement:
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True)


obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the NULL value
Omitting a column from the INSERT means that the column will have the NULL value set, unless the column has a default set up, in which case the default value will be persisted. This holds true both from a pure SQL perspective with server-side defaults, as well as the behavior of SQLAlchemy’s insert behavior with both client-side and server-side defaults:
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")


obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the value 'default'
However, in the ORM, even if one assigns the Python value None explicitly to the object, this is treated the same as though the value were never assigned:
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")


obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM still omits it from the statement and the
# database will still persist this as the value 'default'
The above operation will persist into the data column the server default value of "default" and not SQL NULL, even though None was passed; this is a long-standing behavior of the ORM that many applications hold as an assumption.
So what if we want to actually put NULL into this column, even though the column has a default value? There are two approaches. One is that on a per-instance level, we assign the attribute using the null SQL construct:
from sqlalchemy import null

obj = MyObject(id=1, data=null())
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set as null();
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
The null SQL construct always translates into the SQL NULL value being directly present in the target INSERT statement.
If we’d like to be able to use the Python value None and have this also be persisted as NULL despite the presence of column defaults, we can configure this for the ORM using a Core-level modifier TypeEngine.evaluates_none(), which indicates a type where the ORM should treat the value None the same as any other value and pass it through, rather than omitting it as a “missing” value:
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(
        String(50).evaluates_none(),  # indicate that None should always be passed
        nullable=True,
        server_default="default",
    )


obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
Evaluating None
The TypeEngine.evaluates_none() modifier is primarily intended to signal a type where the Python value “None” is significant, the primary example being a JSON type which may want to persist the JSON null value rather than SQL NULL. We are slightly repurposing it here in order to signal to the ORM that we’d like None to be passed into the type whenever present, even though no special type-level behaviors are assigned to it.
Fetching Server-Generated Defaults
As introduced in the sections Server-invoked DDL-Explicit Default Expressions and Marking Implicitly Generated Values, timestamps, and Triggered Columns, the Core supports the notion of database columns for which the database itself generates a value upon INSERT and in less common cases upon UPDATE statements. The ORM features support for such columns regarding being able to fetch these newly generated values upon flush. This behavior is required in the case of primary key columns that are generated by the server, since the ORM has to know the primary key of an object once it is persisted.
In the vast majority of cases, primary key columns that have their value generated automatically by the database are simple integer columns, which are implemented by the database as either a so-called “autoincrement” column, or from a sequence associated with the column. Every database dialect within SQLAlchemy Core supports a method of retrieving these primary key values which is often native to the Python DBAPI, and in general this process is automatic. There is more documentation regarding this at Column.autoincrement.
For server-generating columns that are not primary key columns or that are not simple autoincrementing integer columns, the ORM requires that these columns are marked with an appropriate server_default directive that allows the ORM to retrieve this value. Not all methods are supported on all backends, however, so care must be taken to use the appropriate method. The two questions to be answered are, 1. is this column part of the primary key or not, and 2. does the database support RETURNING or an equivalent, such as “OUTPUT inserted”; these are SQL phrases which return a server-generated value at the same time as the INSERT or UPDATE statement is invoked. RETURNING is currently supported by PostgreSQL, Oracle Database, MariaDB 10.5, SQLite 3.35, and SQL Server.
Case 1: non primary key, RETURNING or equivalent is supported
In this case, columns should be marked as FetchedValue or with an explicit Column.server_default. The ORM will automatically add these columns to the RETURNING clause when performing INSERT statements, assuming the Mapper.eager_defaults parameter is set to True, or if left at its default setting of "auto", for dialects that support both RETURNING as well as insertmanyvalues:
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    # server-side SQL date function generates a new timestamp
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # some other server-side function not named here, such as a trigger,
    # populates a value into this column during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # set eager defaults to True.  This is usually optional, as if the
    # backend supports RETURNING + insertmanyvalues, eager defaults
    # will take place regardless on INSERT
    __mapper_args__ = {"eager_defaults": True}
Above, an INSERT statement that does not specify explicit values for “timestamp” or “special_identifier” from the client side will include the “timestamp” and “special_identifier” columns within the RETURNING clause so they are available immediately. On the PostgreSQL database, an INSERT for the above table will look like:
INSERT INTO my_table DEFAULT VALUES RETURNING my_table.id, my_table.timestamp, my_table.special_identifier
Changed in version 2.0.0rc1: The Mapper.eager_defaults parameter now defaults to a new setting "auto", which will automatically make use of RETURNING to fetch server-generated default values on INSERT if the backing database supports both RETURNING as well as insertmanyvalues.
Note
The "auto" value for Mapper.eager_defaults only applies to INSERT statements. UPDATE statements will not use RETURNING, even if available, unless Mapper.eager_defaults is set to True. This is because there is no equivalent “insertmanyvalues” feature for UPDATE, so UPDATE RETURNING will require that UPDATE statements are emitted individually for each row being UPDATEd.
Case 2: Table includes trigger-generated values which are not compatible with RETURNING
The "auto" setting of Mapper.eager_defaults means that a backend that supports RETURNING will usually make use of RETURNING with INSERT statements in order to retrieve newly generated default values. However there are limitations of server-generated values that are generated using triggers, such that RETURNING can’t be used:
* SQL Server does not allow RETURNING to be used in an INSERT statement to retrieve a trigger-generated value; the statement will fail.
* SQLite has limitations in combining the use of RETURNING with triggers, such that the RETURNING clause will not have the INSERTed value available
* Other backends may have limitations with RETURNING in conjunction with triggers, or other kinds of server-generated values.
To disable the use of RETURNING for such values, including not just for server generated default values but also to ensure that the ORM will never use RETURNING with a particular table, specify Table.implicit_returning as False for the mapped Table. Using a Declarative mapping this looks like:
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str] = mapped_column(String(50))

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # disable all use of RETURNING for the table
    __table_args__ = {"implicit_returning": False}
On SQL Server with the pyodbc driver, an INSERT for the above table will not use RETURNING and will use the SQL Server scope_identity() function to retrieve the newly generated primary key value:
INSERT INTO my_table (data) VALUES (?); select scope_identity()
See also
INSERT behavior - background on the SQL Server dialect’s methods of fetching newly generated primary key values
Case 3: non primary key, RETURNING or equivalent is not supported or not needed
This case is the same as case 1 above, except we typically don’t want to use Mapper.eager_defaults, as its current implementation in the absence of RETURNING support is to emit a SELECT-per-row, which is not performant. Therefore the parameter is omitted in the mapping below:
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())
After a record with the above mapping is INSERTed on a backend that does not include RETURNING or “insertmanyvalues” support, the “timestamp” and “special_identifier” columns will remain empty, and will be fetched via a second SELECT statement when they are first accessed after the flush, e.g. they are marked as “expired”.
If the Mapper.eager_defaults is explicitly provided with a value of True, and the backend database does not support RETURNING or an equivalent, the ORM will emit a SELECT statement immediately following the INSERT statement in order to fetch newly generated values; the ORM does not currently have the ability to SELECT many newly inserted rows in batch if RETURNING was not available. This is usually undesirable as it adds additional SELECT statements to the flush process that may not be needed. Using the above mapping with the Mapper.eager_defaults flag set to True against MySQL (not MariaDB) results in SQL like this upon flush:
INSERT INTO my_table () VALUES ()

-- when eager_defaults **is** used, but RETURNING is not supported
SELECT my_table.timestamp AS my_table_timestamp, my_table.special_identifier AS my_table_special_identifier
FROM my_table WHERE my_table.id = %s
A future release of SQLAlchemy may seek to improve the efficiency of eager defaults in the abcense of RETURNING to batch many rows within a single SELECT statement.
Case 4: primary key, RETURNING or equivalent is supported
A primary key column with a server-generated value must be fetched immediately upon INSERT; the ORM can only access rows for which it has a primary key value, so if the primary key is generated by the server, the ORM needs a way to retrieve that new value immediately upon INSERT.
As mentioned above, for integer “autoincrement” columns, as well as columns marked with Identity and special constructs such as PostgreSQL SERIAL, these types are handled automatically by the Core; databases include functions for fetching the “last inserted id” where RETURNING is not supported, and where RETURNING is supported SQLAlchemy will use that.
For example, using Oracle Database with a column marked as Identity, RETURNING is used automatically to fetch the new primary key value:
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Identity(), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
The INSERT for a model as above on Oracle Database looks like:
INSERT INTO my_table (data) VALUES (:data) RETURNING my_table.id INTO :ret_0
SQLAlchemy renders an INSERT for the “data” field, but only includes “id” in the RETURNING clause, so that server-side generation for “id” will take place and the new value will be returned immediately.
For non-integer values generated by server side functions or triggers, as well as for integer values that come from constructs outside the table itself, including explicit sequences and triggers, the server default generation must be marked in the table metadata. Using Oracle Database as the example again, we can illustrate a similar table as above naming an explicit sequence using the Sequence construct:
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Sequence("my_oracle_seq"), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
An INSERT for this version of the model on Oracle Database would look like:
INSERT INTO my_table (id, data) VALUES (my_oracle_seq.nextval, :data) RETURNING my_table.id INTO :ret_0
Where above, SQLAlchemy renders my_sequence.nextval for the primary key column so that it is used for new primary key generation, and also uses RETURNING to get the new value back immediately.
If the source of data is not represented by a simple SQL function or Sequence, such as when using triggers or database-specific datatypes that produce new values, the presence of a value-generating default may be indicated by using FetchedValue within the column definition. Below is a model that uses a SQL Server TIMESTAMP column as the primary key; on SQL Server, this datatype generates new values automatically, so this is indicated in the table metadata by indicating FetchedValue for the Column.server_default parameter:
class MySQLServerModel(Base):
    __tablename__ = "my_table"

    timestamp: Mapped[datetime.datetime] = mapped_column(
        TIMESTAMP(), server_default=FetchedValue(), primary_key=True
    )
    data: Mapped[str] = mapped_column(String(50))
An INSERT for the above table on SQL Server looks like:
INSERT INTO my_table (data) OUTPUT inserted.timestamp VALUES (?)
Case 5: primary key, RETURNING or equivalent is not supported
In this area we are generating rows for a database such as MySQL where some means of generating a default is occurring on the server, but is outside of the database’s usual autoincrement routine. In this case, we have to make sure SQLAlchemy can “pre-execute” the default, which means it has to be an explicit SQL expression.
Note
This section will illustrate multiple recipes involving datetime values for MySQL, since the datetime datatypes on this backend has additional idiosyncratic requirements that are useful to illustrate. Keep in mind however that MySQL requires an explicit “pre-executed” default generator for any auto-generated datatype used as the primary key other than the usual single-column autoincrementing integer value.
MySQL with DateTime primary key
Using the example of a DateTime column for MySQL, we add an explicit pre-execute-supported default using the “NOW()” SQL function:
class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(DateTime(), default=func.now(), primary_key=True)
Where above, we select the “NOW()” function to deliver a datetime value to the column. The SQL generated by the above is:
SELECT now() AS anon_1
INSERT INTO my_table (timestamp) VALUES (%s)
('2018-08-09 13:08:46',)
MySQL with TIMESTAMP primary key
When using the TIMESTAMP datatype with MySQL, MySQL ordinarily associates a server-side default with this datatype automatically. However when we use one as a primary key, the Core cannot retrieve the newly generated value unless we execute the function ourselves. As TIMESTAMP on MySQL actually stores a binary value, we need to add an additional “CAST” to our usage of “NOW()” so that we retrieve a binary value that can be persisted into the column:
from sqlalchemy import cast, Binary


class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(
        TIMESTAMP(), default=cast(func.now(), Binary), primary_key=True
    )
Above, in addition to selecting the “NOW()” function, we additionally make use of the Binary datatype in conjunction with cast() so that the returned value is binary. SQL rendered from the above within an INSERT looks like:
SELECT CAST(now() AS BINARY) AS anon_1
INSERT INTO my_table (timestamp) VALUES (%s)
(b'2018-08-09 13:08:46',)
See also
Column INSERT/UPDATE Defaults
Notes on eagerly fetching client invoked SQL expressions used for INSERT or UPDATE
The preceding examples indicate the use of Column.server_default to create tables that include default-generation functions within their DDL.
SQLAlchemy also supports non-DDL server side defaults, as documented at Client-Invoked SQL Expressions; these “client invoked SQL expressions” are set up using the Column.default and Column.onupdate parameters.
These SQL expressions currently are subject to the same limitations within the ORM as occurs for true server-side defaults; they won’t be eagerly fetched with RETURNING when Mapper.eager_defaults is set to "auto" or True unless the FetchedValue directive is associated with the Column, even though these expressions are not DDL server defaults and are actively rendered by SQLAlchemy itself. This limitation may be addressed in future SQLAlchemy releases.
The FetchedValue construct can be applied to Column.server_default or Column.server_onupdate at the same time that a SQL expression is used with Column.default and Column.onupdate, such as in the example below where the func.now() construct is used as a client-invoked SQL expression for Column.default and Column.onupdate. In order for the behavior of Mapper.eager_defaults to include that it fetches these values using RETURNING when available, Column.server_default and Column.server_onupdate are used with FetchedValue to ensure that the fetch occurs:
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    created = mapped_column(
        DateTime(), default=func.now(), server_default=FetchedValue()
    )
    updated = mapped_column(
        DateTime(),
        onupdate=func.now(),
        server_default=FetchedValue(),
        server_onupdate=FetchedValue(),
    )

    __mapper_args__ = {"eager_defaults": True}
With a mapping similar to the above, the SQL rendered by the ORM for INSERT and UPDATE will include created and updated in the RETURNING clause:
INSERT INTO my_table (created) VALUES (now()) RETURNING my_table.id, my_table.created, my_table.updated

UPDATE my_table SET updated=now() WHERE my_table.id = %(my_table_id)s RETURNING my_table.updated
Using INSERT, UPDATE and ON CONFLICT (i.e. upsert) to return ORM Objects
SQLAlchemy 2.0 includes enhanced capabilities for emitting several varieties of ORM-enabled INSERT, UPDATE, and upsert statements. See the document at ORM-Enabled INSERT, UPDATE, and DELETE statements for documentation. For upsert, see ORM “upsert” Statements.
Using PostgreSQL ON CONFLICT with RETURNING to return upserted ORM objects
This section has moved to ORM “upsert” Statements.
Partitioning Strategies (e.g. multiple database backends per Session)
Simple Vertical Partitioning
Vertical partitioning places different classes, class hierarchies, or mapped tables, across multiple databases, by configuring the Session with the Session.binds argument. This argument receives a dictionary that contains any combination of ORM-mapped classes, arbitrary classes within a mapped hierarchy (such as declarative base classes or mixins), Table objects, and Mapper objects as keys, which then refer typically to Engine or less typically Connection objects as targets. The dictionary is consulted whenever the Session needs to emit SQL on behalf of a particular kind of mapped class in order to locate the appropriate source of database connectivity:
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker()

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()
Above, SQL operations against either class will make usage of the Engine linked to that class. The functionality is comprehensive across both read and write operations; a Query that is against entities mapped to engine1 (determined by looking at the first entity in the list of items requested) will make use of engine1 to run the query. A flush operation will make use of both engines on a per-class basis as it flushes objects of type User and Account.
In the more common case, there are typically base or mixin classes that can be used to distinguish between operations that are destined for different database connections. The Session.binds argument can accommodate any arbitrary Python class as a key, which will be used if it is found to be in the __mro__ (Python method resolution order) for a particular mapped class. Supposing two declarative bases are representing two different database connections:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Session


class BaseA(DeclarativeBase):
    pass


class BaseB(DeclarativeBase):
    pass


class User(BaseA): 


class Address(BaseA): 


class GameInfo(BaseB): 


class GameStats(BaseB): 


Session = sessionmaker()

# all User/Address operations will be on engine 1, all
# Game operations will be on engine 2
Session.configure(binds={BaseA: engine1, BaseB: engine2})
Above, classes which descend from BaseA and BaseB will have their SQL operations routed to one of two engines based on which superclass they descend from, if any. In the case of a class that descends from more than one “bound” superclass, the superclass that is highest in the target class’ hierarchy will be chosen to represent which engine should be used.
See also
Session.binds
Coordination of Transactions for a multiple-engine Session
One caveat to using multiple bound engines is in the case where a commit operation may fail on one backend after the commit has succeeded on another. This is an inconsistency problem that in relational databases is solved using a “two phase transaction”, which adds an additional “prepare” step to the commit sequence that allows for multiple databases to agree to commit before actually completing the transaction.
Due to limited support within DBAPIs, SQLAlchemy has limited support for two- phase transactions across backends. Most typically, it is known to work well with the PostgreSQL backend and to a lesser extent with the MySQL backend. However, the Session is fully capable of taking advantage of the two phase transaction feature when the backend supports it, by setting the Session.use_twophase flag within sessionmaker or Session. See Enabling Two-Phase Commit for an example.
Custom Vertical Partitioning
More comprehensive rule-based class-level partitioning can be built by overriding the Session.get_bind() method. Below we illustrate a custom Session which delivers the following rules:
1. Flush operations, as well as bulk “update” and “delete” operations, are delivered to the engine named leader.
2. Operations on objects that subclass MyOtherClass all occur on the other engine.
3. Read operations for all other classes occur on a random choice of the follower1 or follower2 database.
engines = {
    "leader": create_engine("sqlite:///leader.db"),
    "other": create_engine("sqlite:///other.db"),
    "follower1": create_engine("sqlite:///follower1.db"),
    "follower2": create_engine("sqlite:///follower2.db"),
}

from sqlalchemy.sql import Update, Delete
from sqlalchemy.orm import Session, sessionmaker
import random


class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None):
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines["other"]
        elif self._flushing or isinstance(clause, (Update, Delete)):
            # NOTE: this is for example, however in practice reader/writer
            # splits are likely more straightforward by using two distinct
            # Sessions at the top of a "reader" or "writer" operation.
            # See note below
            return engines["leader"]
        else:
            return engines[random.choice(["follower1", "follower2"])]
The above Session class is plugged in using the class_ argument to sessionmaker:
Session = sessionmaker(class_=RoutingSession)
This approach can be combined with multiple MetaData objects, using an approach such as that of using the declarative __abstract__ keyword, described at __abstract__.
Note
While the above example illustrates routing of specific SQL statements to a so-called “leader” or “follower” database based on whether or not the statement expects to write data, this is likely not a practical approach, as it leads to uncoordinated transaction behavior between reading and writing within the same operation. In practice, it’s likely best to construct the Session up front as a “reader” or “writer” session, based on the overall operation / transaction that’s proceeding. That way, an operation that will be writing data will also emit its read-queries within the same transaction scope. See the example at Setting Isolation For A Sessionmaker / Engine Wide for a recipe that sets up one sessionmaker for “read only” operations using autocommit connections, and another for “write” operations which will include DML / COMMIT.
See also
Django-style Database Routers in SQLAlchemy - blog post on a more comprehensive example of Session.get_bind()
Horizontal Partitioning
Horizontal partitioning partitions the rows of a single table (or a set of tables) across multiple databases. The SQLAlchemy Session contains support for this concept, however to use it fully requires that Session and Query subclasses are used. A basic version of these subclasses are available in the Horizontal Sharding ORM extension. An example of use is at: Horizontal Sharding.
Bulk Operations
Legacy Feature
SQLAlchemy 2.0 has integrated the Session “bulk insert” and “bulk update” capabilities into 2.0 style Session.execute() method, making direct use of Insert and Update constructs. See the document at ORM-Enabled INSERT, UPDATE, and DELETE statements for documentation, including Legacy Session Bulk INSERT Methods which illustrates migration from the older methods to the new methods.


Contextual/Thread-local Sessions
Recall from the section When do I construct a Session, when do I commit it, and when do I close it?, the concept of “session scopes” was introduced, with an emphasis on web applications and the practice of linking the scope of a Session with that of a web request. Most modern web frameworks include integration tools so that the scope of the Session can be managed automatically, and these tools should be used as they are available.
SQLAlchemy includes its own helper object, which helps with the establishment of user-defined Session scopes. It is also used by third-party integration systems to help construct their integration schemes.
The object is the scoped_session object, and it represents a registry of Session objects. If you’re not familiar with the registry pattern, a good introduction can be found in Patterns of Enterprise Architecture.
Warning
The scoped_session registry by default uses a Python threading.local() in order to track Session instances. This is not necessarily compatible with all application servers, particularly those which make use of greenlets or other alternative forms of concurrency control, which may lead to race conditions (e.g. randomly occurring failures) when used in moderate to high concurrency scenarios. Please read Thread-Local Scope and Using Thread-Local Scope with Web Applications below to more fully understand the implications of using threading.local() to track Session objects and consider more explicit means of scoping when using application servers which are not based on traditional threads.
Note
The scoped_session object is a very popular and useful object used by many SQLAlchemy applications. However, it is important to note that it presents only one approach to the issue of Session management. If you’re new to SQLAlchemy, and especially if the term “thread-local variable” seems strange to you, we recommend that if possible you familiarize first with an off-the-shelf integration system such as Flask-SQLAlchemy or zope.sqlalchemy.
A scoped_session is constructed by calling it, passing it a factory which can create new Session objects. A factory is just something that produces a new object when called, and in the case of Session, the most common factory is the sessionmaker, introduced earlier in this section. Below we illustrate this usage:
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker

session_factory = sessionmaker(bind=some_engine)
Session = scoped_session(session_factory)
The scoped_session object we’ve created will now call upon the sessionmaker when we “call” the registry:
some_session = Session()
Above, some_session is an instance of Session, which we can now use to talk to the database. This same Session is also present within the scoped_session registry we’ve created. If we call upon the registry a second time, we get back the same Session:
some_other_session = Session()
some_session is some_other_session
True
This pattern allows disparate sections of the application to call upon a global scoped_session, so that all those areas may share the same session without the need to pass it explicitly. The Session we’ve established in our registry will remain, until we explicitly tell our registry to dispose of it, by calling scoped_session.remove():
Session.remove()
The scoped_session.remove() method first calls Session.close() on the current Session, which has the effect of releasing any connection/transactional resources owned by the Session first, then discarding the Session itself. “Releasing” here means that connections are returned to their connection pool and any transactional state is rolled back, ultimately using the rollback() method of the underlying DBAPI connection.
At this point, the scoped_session object is “empty”, and will create a new Session when called again. As illustrated below, this is not the same Session we had before:
new_session = Session()
new_session is some_session
False
The above series of steps illustrates the idea of the “registry” pattern in a nutshell. With that basic idea in hand, we can discuss some of the details of how this pattern proceeds.
Implicit Method Access
The job of the scoped_session is simple; hold onto a Session for all who ask for it. As a means of producing more transparent access to this Session, the scoped_session also includes proxy behavior, meaning that the registry itself can be treated just like a Session directly; when methods are called on this object, they are proxied to the underlying Session being maintained by the registry:
Session = scoped_session(some_factory)

# equivalent to:
#
# session = Session()
# print(session.scalars(select(MyClass)).all())
#
print(Session.scalars(select(MyClass)).all())
The above code accomplishes the same task as that of acquiring the current Session by calling upon the registry, then using that Session.
Thread-Local Scope
Users who are familiar with multithreaded programming will note that representing anything as a global variable is usually a bad idea, as it implies that the global object will be accessed by many threads concurrently. The Session object is entirely designed to be used in a non-concurrent fashion, which in terms of multithreading means “only in one thread at a time”. So our above example of scoped_session usage, where the same Session object is maintained across multiple calls, suggests that some process needs to be in place such that multiple calls across many threads don’t actually get a handle to the same session. We call this notion thread local storage, which means, a special object is used that will maintain a distinct object per each application thread. Python provides this via the threading.local() construct. The scoped_session object by default uses this object as storage, so that a single Session is maintained for all who call upon the scoped_session registry, but only within the scope of a single thread. Callers who call upon the registry in a different thread get a Session instance that is local to that other thread.
Using this technique, the scoped_session provides a quick and relatively simple (if one is familiar with thread-local storage) way of providing a single, global object in an application that is safe to be called upon from multiple threads.
The scoped_session.remove() method, as always, removes the current Session associated with the thread, if any. However, one advantage of the threading.local() object is that if the application thread itself ends, the “storage” for that thread is also garbage collected. So it is in fact “safe” to use thread local scope with an application that spawns and tears down threads, without the need to call scoped_session.remove(). However, the scope of transactions themselves, i.e. ending them via Session.commit() or Session.rollback(), will usually still be something that must be explicitly arranged for at the appropriate time, unless the application actually ties the lifespan of a thread to the lifespan of a transaction.
Using Thread-Local Scope with Web Applications
As discussed in the section When do I construct a Session, when do I commit it, and when do I close it?, a web application is architected around the concept of a web request, and integrating such an application with the Session usually implies that the Session will be associated with that request. As it turns out, most Python web frameworks, with notable exceptions such as the asynchronous frameworks Twisted and Tornado, use threads in a simple way, such that a particular web request is received, processed, and completed within the scope of a single worker thread. When the request ends, the worker thread is released to a pool of workers where it is available to handle another request.
This simple correspondence of web request and thread means that to associate a Session with a thread implies it is also associated with the web request running within that thread, and vice versa, provided that the Session is created only after the web request begins and torn down just before the web request ends. So it is a common practice to use scoped_session as a quick way to integrate the Session with a web application. The sequence diagram below illustrates this flow:
Web Server          Web Framework        SQLAlchemy ORM Code
--------------      --------------       ------------------------------
startup        ->   Web framework        # Session registry is established
                    initializes          Session = scoped_session(sessionmaker())

incoming
web request    ->   web request     ->   # The registry is *optionally*
                    starts               # called upon explicitly to create
                                         # a Session local to the thread and/or request
                                         Session()

                                         # the Session registry can otherwise
                                         # be used at any time, creating the
                                         # request-local Session() if not present,
                                         # or returning the existing one
                                         Session.execute(select(MyClass)) # 

                                         Session.add(some_object) # 

                                         # if data was modified, commit the
                                         # transaction
                                         Session.commit()

                    web request ends  -> # the registry is instructed to
                                         # remove the Session
                                         Session.remove()

                    sends output      <-
outgoing web    <-
response
Using the above flow, the process of integrating the Session with the web application has exactly two requirements:
1. Create a single scoped_session registry when the web application first starts, ensuring that this object is accessible by the rest of the application.
2. Ensure that scoped_session.remove() is called when the web request ends, usually by integrating with the web framework’s event system to establish an “on request end” event.
As noted earlier, the above pattern is just one potential way to integrate a Session with a web framework, one which in particular makes the significant assumption that the web framework associates web requests with application threads. It is however strongly recommended that the integration tools provided with the web framework itself be used, if available, instead of scoped_session.
In particular, while using a thread local can be convenient, it is preferable that the Session be associated directly with the request, rather than with the current thread. The next section on custom scopes details a more advanced configuration which can combine the usage of scoped_session with direct request based scope, or any kind of scope.
Using Custom Created Scopes
The scoped_session object’s default behavior of “thread local” scope is only one of many options on how to “scope” a Session. A custom scope can be defined based on any existing system of getting at “the current thing we are working with”.
Suppose a web framework defines a library function get_current_request(). An application built using this framework can call this function at any time, and the result will be some kind of Request object that represents the current request being processed. If the Request object is hashable, then this function can be easily integrated with scoped_session to associate the Session with the request. Below we illustrate this in conjunction with a hypothetical event marker provided by the web framework on_request_end, which allows code to be invoked whenever a request ends:
from my_web_framework import get_current_request, on_request_end
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(bind=some_engine), scopefunc=get_current_request)


@on_request_end
def remove_session(req):
    Session.remove()
Above, we instantiate scoped_session in the usual way, except that we pass our request-returning function as the “scopefunc”. This instructs scoped_session to use this function to generate a dictionary key whenever the registry is called upon to return the current Session. In this case it is particularly important that we ensure a reliable “remove” system is implemented, as this dictionary is not otherwise self-managed.
Contextual Session API
Object Name
Description
QueryPropertyDescriptor
Describes the type applied to a class-level scoped_session.query_property() attribute.
scoped_session
Provides scoped management of Session objects.
ScopedRegistry
A Registry that can store one or multiple instances of a single class on the basis of a “scope” function.
ThreadLocalRegistry
A ScopedRegistry that uses a threading.local() variable for storage.
class sqlalchemy.orm.scoped_session
Provides scoped management of Session objects.
See Contextual/Thread-local Sessions for a tutorial.
Note
When using Asynchronous I/O (asyncio), the async-compatible async_scoped_session class should be used in place of scoped_session.
Members
__call__(), __init__(), add(), add_all(), autoflush, begin(), begin_nested(), bind, bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), configure(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_one(), identity_key(), identity_map, info, is_active, is_modified(), merge(), new, no_autoflush, object_session(), query(), query_property(), refresh(), remove(), reset(), rollback(), scalar(), scalars(), session_factory
Class signature
class sqlalchemy.orm.scoped_session (typing.Generic)
method sqlalchemy.orm.scoped_session.__call__(**kw: Any) ? _S
Return the current Session, creating it using the scoped_session.session_factory if not present.
Parameters:
**kw – Keyword arguments will be passed to the scoped_session.session_factory callable, if an existing Session is not present. If the Session is present and keyword arguments have been passed, InvalidRequestError is raised.
method sqlalchemy.orm.scoped_session.__init__(session_factory: sessionmaker[_S], scopefunc: Callable[[], Any] | None = None)
Construct a new scoped_session.
Parameters:
* session_factory – a factory to create new Session instances. This is usually, but not necessarily, an instance of sessionmaker.
* scopefunc – optional function which defines the current scope. If not passed, the scoped_session object assumes “thread-local” scope, and will use a Python threading.local() in order to maintain the current Session. If passed, the function should return a hashable token; this token will be used as the key in a dictionary in order to store and retrieve the current Session.
method sqlalchemy.orm.scoped_session.add(instance: object, _warn: bool = True) ? None
Place an object into this Session.
Proxied for the Session class on behalf of the scoped_session class.
Objects that are in the transient state when passed to the Session.add() method will move to the pending state, until the next flush, at which point they will move to the persistent state.
Objects that are in the detached state when passed to the Session.add() method will move to the persistent state directly.
If the transaction used by the Session is rolled back, objects which were transient when they were passed to Session.add() will be moved back to the transient state, and will no longer be present within this Session.
See also
Session.add_all()
Adding New or Existing Items - at Basics of Using a Session
method sqlalchemy.orm.scoped_session.add_all(instances: Iterable[object]) ? None
Add the given collection of instances to this Session.
Proxied for the Session class on behalf of the scoped_session class.
See the documentation for Session.add() for a general behavioral description.
See also
Session.add()
Adding New or Existing Items - at Basics of Using a Session
attribute sqlalchemy.orm.scoped_session.autoflush
Proxy for the Session.autoflush attribute on behalf of the scoped_session class.
method sqlalchemy.orm.scoped_session.begin(nested: bool = False) ? SessionTransaction
Begin a transaction, or nested transaction, on this Session, if one is not already begun.
Proxied for the Session class on behalf of the scoped_session class.
The Session object features autobegin behavior, so that normally it is not necessary to call the Session.begin() method explicitly. However, it may be used in order to control the scope of when the transactional state is begun.
When used to begin the outermost transaction, an error is raised if this Session is already inside of a transaction.
Parameters:
nested – if True, begins a SAVEPOINT transaction and is equivalent to calling Session.begin_nested(). For documentation on SAVEPOINT transactions, please see Using SAVEPOINT.
Returns:
the SessionTransaction object. Note that SessionTransaction acts as a Python context manager, allowing Session.begin() to be used in a “with” block. See Explicit Begin for an example.
See also
Auto Begin
Managing Transactions
Session.begin_nested()
method sqlalchemy.orm.scoped_session.begin_nested() ? SessionTransaction
Begin a “nested” transaction on this Session, e.g. SAVEPOINT.
Proxied for the Session class on behalf of the scoped_session class.
The target database(s) and associated drivers must support SQL SAVEPOINT for this method to function correctly.
For documentation on SAVEPOINT transactions, please see Using SAVEPOINT.
Returns:
the SessionTransaction object. Note that SessionTransaction acts as a context manager, allowing Session.begin_nested() to be used in a “with” block. See Using SAVEPOINT for a usage example.
See also
Using SAVEPOINT
Serializable isolation / Savepoints / Transactional DDL - special workarounds required with the SQLite driver in order for SAVEPOINT to work correctly. For asyncio use cases, see the section Serializable isolation / Savepoints / Transactional DDL (asyncio version).
attribute sqlalchemy.orm.scoped_session.bind
Proxy for the Session.bind attribute on behalf of the scoped_session class.
method sqlalchemy.orm.scoped_session.bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) ? None
Perform a bulk insert of the given list of mapping dictionaries.
Proxied for the Session class on behalf of the scoped_session class.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key. The 2.0 API shares implementation details with this method and adds new features as well.
Parameters:
* mapper – a mapped class, or the actual Mapper object, representing the single kind of object represented within the mapping list.
* mappings – a sequence of dictionaries, each one containing the state of the mapped row to be inserted, in terms of the attribute names on the mapped class. If the mapping refers to multiple tables, such as a joined-inheritance mapping, each dictionary must contain all keys to be populated into all tables.
* return_defaults – 
when True, the INSERT process will be altered to ensure that newly generated primary key values will be fetched. The rationale for this parameter is typically to enable Joined Table Inheritance mappings to be bulk inserted.
Note
for backends that don’t support RETURNING, the Session.bulk_insert_mappings.return_defaults parameter can significantly decrease performance as INSERT statements can no longer be batched. See “Insert Many Values” Behavior for INSERT statements for background on which backends are affected.
* render_nulls – 
When True, a value of None will result in a NULL value being included in the INSERT statement, rather than the column being omitted from the INSERT. This allows all the rows being INSERTed to have the identical set of columns which allows the full set of rows to be batched to the DBAPI. Normally, each column-set that contains a different combination of NULL values than the previous row must omit a different series of columns from the rendered INSERT statement, which means it must be emitted as a separate statement. By passing this flag, the full set of rows are guaranteed to be batchable into one batch; the cost however is that server-side defaults which are invoked by an omitted column will be skipped, so care must be taken to ensure that these are not necessary.
Warning
When this flag is set, server side default SQL values will not be invoked for those columns that are inserted as NULL; the NULL value will be sent explicitly. Care must be taken to ensure that no server-side default functions need to be invoked for the operation as a whole.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_save_objects()
Session.bulk_update_mappings()
method sqlalchemy.orm.scoped_session.bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) ? None
Perform a bulk save of the given list of objects.
Proxied for the Session class on behalf of the scoped_session class.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key.
For general INSERT and UPDATE of existing ORM mapped objects, prefer standard unit of work data management patterns, introduced in the SQLAlchemy Unified Tutorial at Data Manipulation with the ORM. SQLAlchemy 2.0 now uses “Insert Many Values” Behavior for INSERT statements with modern dialects which solves previous issues of bulk INSERT slowness.
Parameters:
* objects – 
a sequence of mapped object instances. The mapped objects are persisted as is, and are not associated with the Session afterwards.
For each object, whether the object is sent as an INSERT or an UPDATE is dependent on the same rules used by the Session in traditional operation; if the object has the InstanceState.key attribute set, then the object is assumed to be “detached” and will result in an UPDATE. Otherwise, an INSERT is used.
In the case of an UPDATE, statements are grouped based on which attributes have changed, and are thus to be the subject of each SET clause. If update_changed_only is False, then all attributes present within each object are applied to the UPDATE statement, which may help in allowing the statements to be grouped together into a larger executemany(), and will also reduce the overhead of checking history on attributes.
* return_defaults – when True, rows that are missing values which generate defaults, namely integer primary key defaults and sequences, will be inserted one at a time, so that the primary key value is available. In particular this will allow joined-inheritance and other multi-table mappings to insert correctly without the need to provide primary key values ahead of time; however, Session.bulk_save_objects.return_defaults greatly reduces the performance gains of the method overall. It is strongly advised to please use the standard Session.add_all() approach.
* update_changed_only – when True, UPDATE statements are rendered based on those attributes in each state that have logged changes. When False, all attributes present are rendered into the SET clause with the exception of primary key attributes.
* preserve_order – when True, the order of inserts and updates matches exactly the order in which the objects are given. When False, common types of objects are grouped into inserts and updates, to allow for more batching opportunities.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_insert_mappings()
Session.bulk_update_mappings()
method sqlalchemy.orm.scoped_session.bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) ? None
Perform a bulk update of the given list of mapping dictionaries.
Proxied for the Session class on behalf of the scoped_session class.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key. The 2.0 API shares implementation details with this method and adds new features as well.
Parameters:
* mapper – a mapped class, or the actual Mapper object, representing the single kind of object represented within the mapping list.
* mappings – a sequence of dictionaries, each one containing the state of the mapped row to be updated, in terms of the attribute names on the mapped class. If the mapping refers to multiple tables, such as a joined-inheritance mapping, each dictionary may contain keys corresponding to all tables. All those keys which are present and are not part of the primary key are applied to the SET clause of the UPDATE statement; the primary key values, which are required, are applied to the WHERE clause.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_insert_mappings()
Session.bulk_save_objects()
method sqlalchemy.orm.scoped_session.close() ? None
Close out the transactional resources and ORM objects used by this Session.
Proxied for the Session class on behalf of the scoped_session class.
This expunges all ORM objects associated with this Session, ends any transaction in progress and releases any Connection objects which this Session itself has checked out from associated Engine objects. The operation then leaves the Session in a state which it may be used again.
Tip
In the default running mode the Session.close() method does not prevent the Session from being used again. The Session itself does not actually have a distinct “closed” state; it merely means the Session will release all database connections and ORM objects.
Setting the parameter Session.close_resets_only to False will instead make the close final, meaning that any further action on the session will be forbidden.
Changed in version 1.4: The Session.close() method does not immediately create a new SessionTransaction object; instead, the new SessionTransaction is created only if the Session is used again for a database operation.
See also
Closing - detail on the semantics of Session.close() and Session.reset().
Session.reset() - a similar method that behaves like close() with the parameter Session.close_resets_only set to True.
classmethod sqlalchemy.orm.scoped_session.close_all() ? None
Close all sessions in memory.
Proxied for the Session class on behalf of the scoped_session class.
Deprecated since version 1.3: The Session.close_all() method is deprecated and will be removed in a future release. Please refer to close_all_sessions().
method sqlalchemy.orm.scoped_session.commit() ? None
Flush pending changes and commit the current transaction.
Proxied for the Session class on behalf of the scoped_session class.
When the COMMIT operation is complete, all objects are fully expired, erasing their internal contents, which will be automatically re-loaded when the objects are next accessed. In the interim, these objects are in an expired state and will not function if they are detached from the Session. Additionally, this re-load operation is not supported when using asyncio-oriented APIs. The Session.expire_on_commit parameter may be used to disable this behavior.
When there is no transaction in place for the Session, indicating that no operations were invoked on this Session since the previous call to Session.commit(), the method will begin and commit an internal-only “logical” transaction, that does not normally affect the database unless pending flush changes were detected, but will still invoke event handlers and object expiration rules.
The outermost database transaction is committed unconditionally, automatically releasing any SAVEPOINTs in effect.
See also
Committing
Managing Transactions
Preventing Implicit IO when Using AsyncSession
method sqlalchemy.orm.scoped_session.configure(**kwargs: Any) ? None
reconfigure the sessionmaker used by this scoped_session.
See sessionmaker.configure().
method sqlalchemy.orm.scoped_session.connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) ? Connection
Return a Connection object corresponding to this Session object’s transactional state.
Proxied for the Session class on behalf of the scoped_session class.
Either the Connection corresponding to the current transaction is returned, or if no transaction is in progress, a new one is begun and the Connection returned (note that no transactional state is established with the DBAPI until the first SQL statement is emitted).
Ambiguity in multi-bind or unbound Session objects can be resolved through any of the optional keyword arguments. This ultimately makes usage of the get_bind() method for resolution.
Parameters:
* bind_arguments – dictionary of bind arguments. May include “mapper”, “bind”, “clause”, other custom arguments that are passed to Session.get_bind().
* execution_options – 
a dictionary of execution options that will be passed to Connection.execution_options(), when the connection is first procured only. If the connection is already present within the Session, a warning is emitted and the arguments are ignored.
See also
Setting Transaction Isolation Levels / DBAPI AUTOCOMMIT
method sqlalchemy.orm.scoped_session.delete(instance: object) ? None
Mark an instance as deleted.
Proxied for the Session class on behalf of the scoped_session class.
The object is assumed to be either persistent or detached when passed; after the method is called, the object will remain in the persistent state until the next flush proceeds. During this time, the object will also be a member of the Session.deleted collection.
When the next flush proceeds, the object will move to the deleted state, indicating a DELETE statement was emitted for its row within the current transaction. When the transaction is successfully committed, the deleted object is moved to the detached state and is no longer present within this Session.
See also
Deleting - at Basics of Using a Session
attribute sqlalchemy.orm.scoped_session.deleted
The set of all instances marked as ‘deleted’ within this Session
Proxied for the Session class on behalf of the scoped_session class.
attribute sqlalchemy.orm.scoped_session.dirty
The set of all persistent instances considered dirty.
Proxied for the Session class on behalf of the scoped_session class.
E.g.:
some_mapped_object in session.dirty
Instances are considered dirty when they were modified but not deleted.
Note that this ‘dirty’ calculation is ‘optimistic’; most attribute-setting or collection modification operations will mark an instance as ‘dirty’ and place it in this set, even if there is no net change to the attribute’s value. At flush time, the value of each attribute is compared to its previously saved value, and if there’s no net change, no SQL operation will occur (this is a more expensive operation so it’s only done at flush time).
To check if an instance has actionable net changes to its attributes, use the Session.is_modified() method.
method sqlalchemy.orm.scoped_session.execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) ? Result[Any]
Execute a SQL expression construct.
Proxied for the Session class on behalf of the scoped_session class.
Returns a Result object representing results of the statement execution.
E.g.:
from sqlalchemy import select

result = session.execute(select(User).where(User.id == 5))
The API contract of Session.execute() is similar to that of Connection.execute(), the 2.0 style version of Connection.
Changed in version 1.4: the Session.execute() method is now the primary point of ORM statement execution when using 2.0 style ORM usage.
Parameters:
* statement – An executable statement (i.e. an Executable expression such as select()).
* params – Optional dictionary, or list of dictionaries, containing bound parameter values. If a single dictionary, single-row execution occurs; if a list of dictionaries, an “executemany” will be invoked. The keys in each dictionary must correspond to parameter names present in the statement.
* execution_options – 
optional dictionary of execution options, which will be associated with the statement execution. This dictionary can provide a subset of the options that are accepted by Connection.execution_options(), and may also provide additional options understood only in an ORM context.
See also
ORM Execution Options - ORM-specific execution options
* bind_arguments – dictionary of additional arguments to determine the bind. May include “mapper”, “bind”, or other custom arguments. Contents of this dictionary are passed to the Session.get_bind() method.
Returns:
a Result object.
method sqlalchemy.orm.scoped_session.expire(instance: object, attribute_names: Iterable[str] | None = None) ? None
Expire the attributes on an instance.
Proxied for the Session class on behalf of the scoped_session class.
Marks the attributes of an instance as out of date. When an expired attribute is next accessed, a query will be issued to the Session object’s current transactional context in order to load all expired attributes for the given instance. Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction.
To expire all objects in the Session simultaneously, use Session.expire_all().
The Session object’s default behavior is to expire all state whenever the Session.rollback() or Session.commit() methods are called, so that new state can be loaded for the new transaction. For this reason, calling Session.expire() only makes sense for the specific case that a non-ORM SQL statement was emitted in the current transaction.
Parameters:
* instance – The instance to be refreshed.
* attribute_names – optional list of string attribute names indicating a subset of attributes to be expired.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.refresh()
Query.populate_existing()
method sqlalchemy.orm.scoped_session.expire_all() ? None
Expires all persistent instances within this Session.
Proxied for the Session class on behalf of the scoped_session class.
When any attributes on a persistent instance is next accessed, a query will be issued using the Session object’s current transactional context in order to load all expired attributes for the given instance. Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction.
To expire individual objects and individual attributes on those objects, use Session.expire().
The Session object’s default behavior is to expire all state whenever the Session.rollback() or Session.commit() methods are called, so that new state can be loaded for the new transaction. For this reason, calling Session.expire_all() is not usually needed, assuming the transaction is isolated.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.refresh()
Query.populate_existing()
method sqlalchemy.orm.scoped_session.expunge(instance: object) ? None
Remove the instance from this Session.
Proxied for the Session class on behalf of the scoped_session class.
This will free all internal references to the instance. Cascading will be applied according to the expunge cascade rule.
method sqlalchemy.orm.scoped_session.expunge_all() ? None
Remove all object instances from this Session.
Proxied for the Session class on behalf of the scoped_session class.
This is equivalent to calling expunge(obj) on all objects in this Session.
method sqlalchemy.orm.scoped_session.flush(objects: Sequence[Any] | None = None) ? None
Flush all the object changes to the database.
Proxied for the Session class on behalf of the scoped_session class.
Writes out all pending object creations, deletions and modifications to the database as INSERTs, DELETEs, UPDATEs, etc. Operations are automatically ordered by the Session’s unit of work dependency solver.
Database operations will be issued in the current transactional context and do not affect the state of the transaction, unless an error occurs, in which case the entire transaction is rolled back. You may flush() as often as you like within a transaction to move changes from Python to the database’s transaction buffer.
Parameters:
objects – 
Optional; restricts the flush operation to operate only on elements that are in the given collection.
This feature is for an extremely narrow set of use cases where particular objects may need to be operated upon before the full flush() occurs. It is not intended for general use.
method sqlalchemy.orm.scoped_session.get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) ? _O | None
Return an instance based on the given primary key identifier, or None if not found.
Proxied for the Session class on behalf of the scoped_session class.
E.g.:
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(VersionedFoo, {"id": 5, "version_id": 10})
Added in version 1.4: Added Session.get(), which is moved from the now legacy Query.get() method.
Session.get() is special in that it provides direct access to the identity map of the Session. If the given primary key identifier is present in the local identity map, the object is returned directly from this collection and no SQL is emitted, unless the object has been marked fully expired. If not present, a SELECT is performed in order to locate the object.
Session.get() also will perform a check if the object is present in the identity map and marked as expired - a SELECT is emitted to refresh the object as well as to ensure that the row is still present. If not, ObjectDeletedError is raised.
Parameters:
* entity – a mapped class or Mapper indicating the type of entity to be loaded.
* ident – 
A scalar, tuple, or dictionary representing the primary key. For a composite (e.g. multiple column) primary key, a tuple or dictionary should be passed.
For a single-column primary key, the scalar calling form is typically the most expedient. If the primary key of a row is the value “5”, the call looks like:
my_object = session.get(SomeClass, 5)
The tuple form contains primary key values typically in the order in which they correspond to the mapped Table object’s primary key columns, or if the Mapper.primary_key configuration parameter were used, in the order used for that parameter. For example, if the primary key of a row is represented by the integer digits “5, 10” the call would look like:
my_object = session.get(SomeClass, (5, 10))
The dictionary form should include as keys the mapped attribute names corresponding to each element of the primary key. If the mapped class has the attributes id, version_id as the attributes which store the object’s primary key value, the call would look like:
my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
* 
* options – optional sequence of loader options which will be applied to the query, if one is emitted.
* populate_existing – causes the method to unconditionally emit a SQL query and refresh the object with the newly loaded data, regardless of whether or not the object is already present.
* with_for_update – optional boolean True indicating FOR UPDATE should be used, or may be a dictionary containing flags to indicate a more specific set of FOR UPDATE flags for the SELECT; flags should match the parameters of Query.with_for_update(). Supersedes the Session.refresh.lockmode parameter.
* execution_options – 
optional dictionary of execution options, which will be associated with the query execution if one is emitted. This dictionary can provide a subset of the options that are accepted by Connection.execution_options(), and may also provide additional options understood only in an ORM context.
Added in version 1.4.29.
See also
ORM Execution Options - ORM-specific execution options
* bind_arguments – 
dictionary of additional arguments to determine the bind. May include “mapper”, “bind”, or other custom arguments. Contents of this dictionary are passed to the Session.get_bind() method.
Returns:
The object instance, or None.
method sqlalchemy.orm.scoped_session.get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) ? Engine | Connection
Return a “bind” to which this Session is bound.
Proxied for the Session class on behalf of the scoped_session class.
The “bind” is usually an instance of Engine, except in the case where the Session has been explicitly bound directly to a Connection.
For a multiply-bound or unbound Session, the mapper or clause arguments are used to determine the appropriate bind to return.
Note that the “mapper” argument is usually present when Session.get_bind() is called via an ORM operation such as a Session.query(), each individual INSERT/UPDATE/DELETE operation within a Session.flush(), call, etc.
The order of resolution is:
1. if mapper given and Session.binds is present, locate a bind based first on the mapper in use, then on the mapped class in use, then on any base classes that are present in the __mro__ of the mapped class, from more specific superclasses to more general.
2. if clause given and Session.binds is present, locate a bind based on Table objects found in the given clause present in Session.binds.
3. if Session.binds is present, return that.
4. if clause given, attempt to return a bind linked to the MetaData ultimately associated with the clause.
5. if mapper given, attempt to return a bind linked to the MetaData ultimately associated with the Table or other selectable to which the mapper is mapped.
6. No bind can be found, UnboundExecutionError is raised.
Note that the Session.get_bind() method can be overridden on a user-defined subclass of Session to provide any kind of bind resolution scheme. See the example at Custom Vertical Partitioning.
Parameters:
* mapper – Optional mapped class or corresponding Mapper instance. The bind can be derived from a Mapper first by consulting the “binds” map associated with this Session, and secondly by consulting the MetaData associated with the Table to which the Mapper is mapped for a bind.
* clause – A ClauseElement (i.e. select(), text(), etc.). If the mapper argument is not present or could not produce a bind, the given expression construct will be searched for a bound element, typically a Table associated with bound MetaData.
See also
Partitioning Strategies (e.g. multiple database backends per Session)
Session.binds
Session.bind_mapper()
Session.bind_table()
method sqlalchemy.orm.scoped_session.get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) ? _O
Return exactly one instance based on the given primary key identifier, or raise an exception if not found.
Proxied for the Session class on behalf of the scoped_session class.
Raises NoResultFound if the query selects no rows.
For a detailed documentation of the arguments see the method Session.get().
Added in version 2.0.22.
Returns:
The object instance.
See also
Session.get() - equivalent method that instead
returns None if no row was found with the provided primary key
classmethod sqlalchemy.orm.scoped_session.identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) ? _IdentityKeyType[Any]
Return an identity key.
Proxied for the Session class on behalf of the scoped_session class.
This is an alias of identity_key().
attribute sqlalchemy.orm.scoped_session.identity_map
Proxy for the Session.identity_map attribute on behalf of the scoped_session class.
attribute sqlalchemy.orm.scoped_session.info
A user-modifiable dictionary.
Proxied for the Session class on behalf of the scoped_session class.
The initial value of this dictionary can be populated using the info argument to the Session constructor or sessionmaker constructor or factory methods. The dictionary here is always local to this Session and can be modified independently of all other Session objects.
attribute sqlalchemy.orm.scoped_session.is_active
True if this Session not in “partial rollback” state.
Proxied for the Session class on behalf of the scoped_session class.
Changed in version 1.4: The Session no longer begins a new transaction immediately, so this attribute will be False when the Session is first instantiated.
“partial rollback” state typically indicates that the flush process of the Session has failed, and that the Session.rollback() method must be emitted in order to fully roll back the transaction.
If this Session is not in a transaction at all, the Session will autobegin when it is first used, so in this case Session.is_active will return True.
Otherwise, if this Session is within a transaction, and that transaction has not been rolled back internally, the Session.is_active will also return True.
See also
“This Session’s transaction has been rolled back due to a previous exception during flush.” (or similar)
Session.in_transaction()
method sqlalchemy.orm.scoped_session.is_modified(instance: object, include_collections: bool = True) ? bool
Return True if the given instance has locally modified attributes.
Proxied for the Session class on behalf of the scoped_session class.
This method retrieves the history for each instrumented attribute on the instance and performs a comparison of the current value to its previously flushed or committed value, if any.
It is in effect a more expensive and accurate version of checking for the given instance in the Session.dirty collection; a full test for each attribute’s net “dirty” status is performed.
E.g.:
return session.is_modified(someobject)
A few caveats to this method apply:
* Instances present in the Session.dirty collection may report False when tested with this method. This is because the object may have received change events via attribute mutation, thus placing it in Session.dirty, but ultimately the state is the same as that loaded from the database, resulting in no net change here.
* Scalar attributes may not have recorded the previously set value when a new value was applied, if the attribute was not loaded, or was expired, at the time the new value was received - in these cases, the attribute is assumed to have a change, even if there is ultimately no net change against its database value. SQLAlchemy in most cases does not need the “old” value when a set event occurs, so it skips the expense of a SQL call if the old value isn’t present, based on the assumption that an UPDATE of the scalar value is usually needed, and in those few cases where it isn’t, is less expensive on average than issuing a defensive SELECT.
The “old” value is fetched unconditionally upon set only if the attribute container has the active_history flag set to True. This flag is set typically for primary key attributes and scalar object references that are not a simple many-to-one. To set this flag for any arbitrary mapped column, use the active_history argument with column_property().
Parameters:
* instance – mapped instance to be tested for pending changes.
* include_collections – Indicates if multivalued collections should be included in the operation. Setting this to False is a way to detect only local-column based properties (i.e. scalar columns or many-to-one foreign keys) that would result in an UPDATE for this instance upon flush.
method sqlalchemy.orm.scoped_session.merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) ? _O
Copy the state of a given instance into a corresponding instance within this Session.
Proxied for the Session class on behalf of the scoped_session class.
Session.merge() examines the primary key attributes of the source instance, and attempts to reconcile it with an instance of the same primary key in the session. If not found locally, it attempts to load the object from the database based on primary key, and if none can be located, creates a new instance. The state of each attribute on the source instance is then copied to the target instance. The resulting target instance is then returned by the method; the original source instance is left unmodified, and un-associated with the Session if not already.
This operation cascades to associated instances if the association is mapped with cascade="merge".
See Merging for a detailed discussion of merging.
Parameters:
* instance – Instance to be merged.
* load – 
Boolean, when False, merge() switches into a “high performance” mode which causes it to forego emitting history events as well as all database access. This flag is used for cases such as transferring graphs of objects into a Session from a second level cache, or to transfer just-loaded objects into the Session owned by a worker thread or process without re-querying the database.
The load=False use case adds the caveat that the given object has to be in a “clean” state, that is, has no pending changes to be flushed - even if the incoming object is detached from any Session. This is so that when the merge operation populates local attributes and cascades to related objects and collections, the values can be “stamped” onto the target object as is, without generating any history or attribute events, and without the need to reconcile the incoming data with any existing related objects or collections that might not be loaded. The resulting objects from load=False are always produced as “clean”, so it is only appropriate that the given objects should be “clean” as well, else this suggests a mis-use of the method.
* options – 
optional sequence of loader options which will be applied to the Session.get() method when the merge operation loads the existing version of the object from the database.
Added in version 1.4.24.
See also
make_transient_to_detached() - provides for an alternative means of “merging” a single object into the Session
attribute sqlalchemy.orm.scoped_session.new
The set of all instances marked as ‘new’ within this Session.
Proxied for the Session class on behalf of the scoped_session class.
attribute sqlalchemy.orm.scoped_session.no_autoflush
Return a context manager that disables autoflush.
Proxied for the Session class on behalf of the scoped_session class.
e.g.:
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
Operations that proceed within the with: block will not be subject to flushes occurring upon query access. This is useful when initializing a series of objects which involve existing database queries, where the uncompleted object should not yet be flushed.
classmethod sqlalchemy.orm.scoped_session.object_session(instance: object) ? Session | None
Return the Session to which an object belongs.
Proxied for the Session class on behalf of the scoped_session class.
This is an alias of object_session().
method sqlalchemy.orm.scoped_session.query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) ? Query[Any]
Return a new Query object corresponding to this Session.
Proxied for the Session class on behalf of the scoped_session class.
Note that the Query object is legacy as of SQLAlchemy 2.0; the select() construct is now used to construct ORM queries.
See also
SQLAlchemy Unified Tutorial
ORM Querying Guide
Legacy Query API - legacy API doc
method sqlalchemy.orm.scoped_session.query_property(query_cls: Type[Query[_T]] | None = None) ? QueryPropertyDescriptor
return a class property which produces a legacy Query object against the class and the current Session when called.
Legacy Feature
The scoped_session.query_property() accessor is specific to the legacy Query object and is not considered to be part of 2.0-style ORM use.
e.g.:
from sqlalchemy.orm import QueryPropertyDescriptor
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker

Session = scoped_session(sessionmaker())


class MyClass:
    query: QueryPropertyDescriptor = Session.query_property()


# after mappers are defined
result = MyClass.query.filter(MyClass.name == "foo").all()
Produces instances of the session’s configured query class by default. To override and use a custom implementation, provide a query_cls callable. The callable will be invoked with the class’s mapper as a positional argument and a session keyword argument.
There is no limit to the number of query properties placed on a class.
method sqlalchemy.orm.scoped_session.refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) ? None
Expire and refresh attributes on the given instance.
Proxied for the Session class on behalf of the scoped_session class.
The selected attributes will first be expired as they would when using Session.expire(); then a SELECT statement will be issued to the database to refresh column-oriented attributes with the current value available in the current transaction.
relationship() oriented attributes will also be immediately loaded if they were already eagerly loaded on the object, using the same eager loading strategy that they were loaded with originally.
Added in version 1.4: - the Session.refresh() method can also refresh eagerly loaded attributes.
relationship() oriented attributes that would normally load using the select (or “lazy”) loader strategy will also load if they are named explicitly in the attribute_names collection, emitting a SELECT statement for the attribute using the immediate loader strategy. If lazy-loaded relationships are not named in Session.refresh.attribute_names, then they remain as “lazy loaded” attributes and are not implicitly refreshed.
Changed in version 2.0.4: The Session.refresh() method will now refresh lazy-loaded relationship() oriented attributes for those which are named explicitly in the Session.refresh.attribute_names collection.
Tip
While the Session.refresh() method is capable of refreshing both column and relationship oriented attributes, its primary focus is on refreshing of local column-oriented attributes on a single instance. For more open ended “refresh” functionality, including the ability to refresh the attributes on many objects at once while having explicit control over relationship loader strategies, use the populate existing feature instead.
Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction. Refreshing attributes usually only makes sense at the start of a transaction where database rows have not yet been accessed.
Parameters:
* attribute_names – optional. An iterable collection of string attribute names indicating a subset of attributes to be refreshed.
* with_for_update – optional boolean True indicating FOR UPDATE should be used, or may be a dictionary containing flags to indicate a more specific set of FOR UPDATE flags for the SELECT; flags should match the parameters of Query.with_for_update(). Supersedes the Session.refresh.lockmode parameter.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.expire_all()
Populate Existing - allows any ORM query to refresh objects as they would be loaded normally.
method sqlalchemy.orm.scoped_session.remove() ? None
Dispose of the current Session, if present.
This will first call Session.close() method on the current Session, which releases any existing transactional/connection resources still being held; transactions specifically are rolled back. The Session is then discarded. Upon next usage within the same scope, the scoped_session will produce a new Session object.
method sqlalchemy.orm.scoped_session.reset() ? None
Close out the transactional resources and ORM objects used by this Session, resetting the session to its initial state.
Proxied for the Session class on behalf of the scoped_session class.
This method provides for same “reset-only” behavior that the Session.close() method has provided historically, where the state of the Session is reset as though the object were brand new, and ready to be used again. This method may then be useful for Session objects which set Session.close_resets_only to False, so that “reset only” behavior is still available.
Added in version 2.0.22.
See also
Closing - detail on the semantics of Session.close() and Session.reset().
Session.close() - a similar method will additionally prevent re-use of the Session when the parameter Session.close_resets_only is set to False.
method sqlalchemy.orm.scoped_session.rollback() ? None
Rollback the current transaction in progress.
Proxied for the Session class on behalf of the scoped_session class.
If no transaction is in progress, this method is a pass-through.
The method always rolls back the topmost database transaction, discarding any nested transactions that may be in progress.
See also
Rolling Back
Managing Transactions
method sqlalchemy.orm.scoped_session.scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) ? Any
Execute a statement and return a scalar result.
Proxied for the Session class on behalf of the scoped_session class.
Usage and parameters are the same as that of Session.execute(); the return result is a scalar Python value.
method sqlalchemy.orm.scoped_session.scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) ? ScalarResult[Any]
Execute a statement and return the results as scalars.
Proxied for the Session class on behalf of the scoped_session class.
Usage and parameters are the same as that of Session.execute(); the return result is a ScalarResult filtering object which will return single elements rather than Row objects.
Returns:
a ScalarResult object
Added in version 1.4.24: Added Session.scalars()
Added in version 1.4.26: Added scoped_session.scalars()
See also
Selecting ORM Entities - contrasts the behavior of Session.execute() to Session.scalars()
attribute sqlalchemy.orm.scoped_session.session_factory: sessionmaker[_S]
The session_factory provided to __init__ is stored in this attribute and may be accessed at a later time. This can be useful when a new non-scoped Session is needed.
class sqlalchemy.util.ScopedRegistry
A Registry that can store one or multiple instances of a single class on the basis of a “scope” function.
The object implements __call__ as the “getter”, so by calling myregistry() the contained object is returned for the current scope.
Parameters:
* createfunc – a callable that returns a new object to be placed in the registry
* scopefunc – a callable that will return a key to store/retrieve an object.
Members
__init__(), clear(), has(), set()
Class signature
class sqlalchemy.util.ScopedRegistry (typing.Generic)
method sqlalchemy.util.ScopedRegistry.__init__(createfunc: Callable[[], _T], scopefunc: Callable[[], Any])
Construct a new ScopedRegistry.
Parameters:
* createfunc – A creation function that will generate a new value for the current scope, if none is present.
* scopefunc – A function that returns a hashable token representing the current scope (such as, current thread identifier).
method sqlalchemy.util.ScopedRegistry.clear() ? None
Clear the current scope, if any.
method sqlalchemy.util.ScopedRegistry.has() ? bool
Return True if an object is present in the current scope.
method sqlalchemy.util.ScopedRegistry.set(obj: _T) ? None
Set the value for the current scope.
class sqlalchemy.util.ThreadLocalRegistry
A ScopedRegistry that uses a threading.local() variable for storage.
Class signature
class sqlalchemy.util.ThreadLocalRegistry (sqlalchemy.util.ScopedRegistry)
class sqlalchemy.orm.QueryPropertyDescriptor
Describes the type applied to a class-level scoped_session.query_property() attribute.
Added in version 2.0.5.
Class signature
class sqlalchemy.orm.QueryPropertyDescriptor (typing.Protocol)


Tracking queries, object and Session Changes with Events
SQLAlchemy features an extensive Event Listening system used throughout the Core and ORM. Within the ORM, there are a wide variety of event listener hooks, which are documented at an API level at ORM Events. This collection of events has grown over the years to include lots of very useful new events as well as some older events that aren’t as relevant as they once were. This section will attempt to introduce the major event hooks and when they might be used.
Execute Events
Added in version 1.4: The Session now features a single comprehensive hook designed to intercept all SELECT statements made on behalf of the ORM as well as bulk UPDATE and DELETE statements. This hook supersedes the previous QueryEvents.before_compile() event as well QueryEvents.before_compile_update() and QueryEvents.before_compile_delete().
Session features a comprehensive system by which all queries invoked via the Session.execute() method, which includes all SELECT statements emitted by Query as well as all SELECT statements emitted on behalf of column and relationship loaders, may be intercepted and modified. The system makes use of the SessionEvents.do_orm_execute() event hook as well as the ORMExecuteState object to represent the event state.
Basic Query Interception
SessionEvents.do_orm_execute() is firstly useful for any kind of interception of a query, which includes those emitted by Query with 1.x style as well as when an ORM-enabled 2.0 style select(), update() or delete() construct is delivered to Session.execute(). The ORMExecuteState construct provides accessors to allow modifications to statements, parameters, and options:
Session = sessionmaker(engine)


@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if orm_execute_state.is_select:
        # add populate_existing for all SELECT statements

        orm_execute_state.update_execution_options(populate_existing=True)

        # check if the SELECT is against a certain entity and add an
        # ORDER BY if so
        col_descriptions = orm_execute_state.statement.column_descriptions

        if col_descriptions[0]["entity"] is MyEntity:
            orm_execute_state.statement = statement.order_by(MyEntity.name)
The above example illustrates some simple modifications to SELECT statements. At this level, the SessionEvents.do_orm_execute() event hook intends to replace the previous use of the QueryEvents.before_compile() event, which was not fired off consistently for various kinds of loaders; additionally, the QueryEvents.before_compile() only applies to 1.x style use with Query and not with 2.0 style use of Session.execute().
Adding global WHERE / ON criteria
One of the most requested query-extension features is the ability to add WHERE criteria to all occurrences of an entity in all queries. This is achievable by making use of the with_loader_criteria() query option, which may be used on its own, or is ideally suited to be used within the SessionEvents.do_orm_execute() event:
from sqlalchemy.orm import with_loader_criteria

Session = sessionmaker(engine)


@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(MyEntity.public == True)
        )
Above, an option is added to all SELECT statements that will limit all queries against MyEntity to filter on public == True. The criteria will be applied to all loads of that class within the scope of the immediate query. The with_loader_criteria() option by default will automatically propagate to relationship loaders as well, which will apply to subsequent relationship loads, which includes lazy loads, selectinloads, etc.
For a series of classes that all feature some common column structure, if the classes are composed using a declarative mixin, the mixin class itself may be used in conjunction with the with_loader_criteria() option by making use of a Python lambda. The Python lambda will be invoked at query compilation time against the specific entities which match the criteria. Given a series of classes based on a mixin called HasTimestamp:
import datetime


class HasTimestamp:
    timestamp = mapped_column(DateTime, default=datetime.datetime.now)


class SomeEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)


class SomeOtherEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)
The above classes SomeEntity and SomeOtherEntity will each have a column timestamp that defaults to the current date and time. An event may be used to intercept all objects that extend from HasTimestamp and filter their timestamp column on a date that is no older than one month ago:
@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(
                HasTimestamp,
                lambda cls: cls.timestamp >= one_month_ago,
                include_aliases=True,
            )
        )
Warning
The use of a lambda inside of the call to with_loader_criteria() is only invoked once per unique class. Custom functions should not be invoked within this lambda. See Using Lambdas to add significant speed gains to statement production for an overview of the “lambda SQL” feature, which is for advanced use only.
See also
ORM Query Events - includes working examples of the above with_loader_criteria() recipes.
Re-Executing Statements
Deep Alchemy
the statement re-execution feature involves a slightly intricate recursive sequence, and is intended to solve the fairly hard problem of being able to re-route the execution of a SQL statement into various non-SQL contexts. The twin examples of “dogpile caching” and “horizontal sharding”, linked below, should be used as a guide for when this rather advanced feature is appropriate to be used.
The ORMExecuteState is capable of controlling the execution of the given statement; this includes the ability to either not invoke the statement at all, allowing a pre-constructed result set retrieved from a cache to be returned instead, as well as the ability to invoke the same statement repeatedly with different state, such as invoking it against multiple database connections and then merging the results together in memory. Both of these advanced patterns are demonstrated in SQLAlchemy’s example suite as detailed below.
When inside the SessionEvents.do_orm_execute() event hook, the ORMExecuteState.invoke_statement() method may be used to invoke the statement using a new nested invocation of Session.execute(), which will then preempt the subsequent handling of the current execution in progress and instead return the Result returned by the inner execution. The event handlers thus far invoked for the SessionEvents.do_orm_execute() hook within this process will be skipped within this nested call as well.
The ORMExecuteState.invoke_statement() method returns a Result object; this object then features the ability for it to be “frozen” into a cacheable format and “unfrozen” into a new Result object, as well as for its data to be merged with that of other Result objects.
E.g., using SessionEvents.do_orm_execute() to implement a cache:
from sqlalchemy.orm import loading

cache = {}


@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if "my_cache_key" in orm_execute_state.execution_options:
        cache_key = orm_execute_state.execution_options["my_cache_key"]

        if cache_key in cache:
            frozen_result = cache[cache_key]
        else:
            frozen_result = orm_execute_state.invoke_statement().freeze()
            cache[cache_key] = frozen_result

        return loading.merge_frozen_result(
            orm_execute_state.session,
            orm_execute_state.statement,
            frozen_result,
            load=False,
        )
With the above hook in place, an example of using the cache would look like:
stmt = (
    select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
)

result = session.execute(stmt)
Above, a custom execution option is passed to Select.execution_options() in order to establish a “cache key” that will then be intercepted by the SessionEvents.do_orm_execute() hook. This cache key is then matched to a FrozenResult object that may be present in the cache, and if present, the object is re-used. The recipe makes use of the Result.freeze() method to “freeze” a Result object, which above will contain ORM results, such that it can be stored in a cache and used multiple times. In order to return a live result from the “frozen” result, the merge_frozen_result() function is used to merge the “frozen” data from the result object into the current session.
The above example is implemented as a complete example in Dogpile Caching.
The ORMExecuteState.invoke_statement() method may also be called multiple times, passing along different information to the ORMExecuteState.invoke_statement.bind_arguments parameter such that the Session will make use of different Engine objects each time. This will return a different Result object each time; these results can be merged together using the Result.merge() method. This is the technique employed by the Horizontal Sharding extension; see the source code to familiarize.
See also
Dogpile Caching
Horizontal Sharding
Persistence Events
Probably the most widely used series of events are the “persistence” events, which correspond to the flush process. The flush is where all the decisions are made about pending changes to objects and are then emitted out to the database in the form of INSERT, UPDATE, and DELETE statements.
before_flush()
The SessionEvents.before_flush() hook is by far the most generally useful event to use when an application wants to ensure that additional persistence changes to the database are made when a flush proceeds. Use SessionEvents.before_flush() in order to operate upon objects to validate their state as well as to compose additional objects and references before they are persisted. Within this event, it is safe to manipulate the Session’s state, that is, new objects can be attached to it, objects can be deleted, and individual attributes on objects can be changed freely, and these changes will be pulled into the flush process when the event hook completes.
The typical SessionEvents.before_flush() hook will be tasked with scanning the collections Session.new, Session.dirty and Session.deleted in order to look for objects where something will be happening.
For illustrations of SessionEvents.before_flush(), see examples such as Versioning with a History Table and Versioning using Temporal Rows.
after_flush()
The SessionEvents.after_flush() hook is called after the SQL has been emitted for a flush process, but before the state of the objects that were flushed has been altered. That is, you can still inspect the Session.new, Session.dirty and Session.deleted collections to see what was just flushed, and you can also use history tracking features like the ones provided by AttributeState to see what changes were just persisted. In the SessionEvents.after_flush() event, additional SQL can be emitted to the database based on what’s observed to have changed.
after_flush_postexec()
SessionEvents.after_flush_postexec() is called soon after SessionEvents.after_flush(), but is invoked after the state of the objects has been modified to account for the flush that just took place. The Session.new, Session.dirty and Session.deleted collections are normally completely empty here. Use SessionEvents.after_flush_postexec() to inspect the identity map for finalized objects and possibly emit additional SQL. In this hook, there is the ability to make new changes on objects, which means the Session will again go into a “dirty” state; the mechanics of the Session here will cause it to flush again if new changes are detected in this hook if the flush were invoked in the context of Session.commit(); otherwise, the pending changes will be bundled as part of the next normal flush. When the hook detects new changes within a Session.commit(), a counter ensures that an endless loop in this regard is stopped after 100 iterations, in the case that an SessionEvents.after_flush_postexec() hook continually adds new state to be flushed each time it is called.
Mapper-level Flush Events
In addition to the flush-level hooks, there is also a suite of hooks that are more fine-grained, in that they are called on a per-object basis and are broken out based on INSERT, UPDATE or DELETE within the flush process. These are the mapper persistence hooks, and they too are very popular, however these events need to be approached more cautiously, as they proceed within the context of the flush process that is already ongoing; many operations are not safe to proceed here.
The events are:
* MapperEvents.before_insert()
* MapperEvents.after_insert()
* MapperEvents.before_update()
* MapperEvents.after_update()
* MapperEvents.before_delete()
* MapperEvents.after_delete()
Note
It is important to note that these events apply only to the session flush operation , and not to the ORM-level INSERT/UPDATE/DELETE functionality described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM-level DML, use the SessionEvents.do_orm_execute() event.
Each event is passed the Mapper, the mapped object itself, and the Connection which is being used to emit an INSERT, UPDATE or DELETE statement. The appeal of these events is clear, in that if an application wants to tie some activity to when a specific type of object is persisted with an INSERT, the hook is very specific; unlike the SessionEvents.before_flush() event, there’s no need to search through collections like Session.new in order to find targets. However, the flush plan which represents the full list of every single INSERT, UPDATE, DELETE statement to be emitted has already been decided when these events are called, and no changes may be made at this stage. Therefore the only changes that are even possible to the given objects are upon attributes local to the object’s row. Any other change to the object or other objects will impact the state of the Session, which will fail to function properly.
Operations that are not supported within these mapper-level persistence events include:
* Session.add()
* Session.delete()
* Mapped collection append, add, remove, delete, discard, etc.
* Mapped relationship attribute set/del events, i.e. someobject.related = someotherobject
The reason the Connection is passed is that it is encouraged that simple SQL operations take place here, directly on the Connection, such as incrementing counters or inserting extra rows within log tables.
There are also many per-object operations that don’t need to be handled within a flush event at all. The most common alternative is to simply establish additional state along with an object inside its __init__() method, such as creating additional objects that are to be associated with the new object. Using validators as described in Simple Validators is another approach; these functions can intercept changes to attributes and establish additional state changes on the target object in response to the attribute change. With both of these approaches, the object is in the correct state before it ever gets to the flush step.
Object Lifecycle Events
Another use case for events is to track the lifecycle of objects. This refers to the states first introduced at Quickie Intro to Object States.
All the states above can be tracked fully with events. Each event represents a distinct state transition, meaning, the starting state and the destination state are both part of what are tracked. With the exception of the initial transient event, all the events are in terms of the Session object or class, meaning they can be associated either with a specific Session object:
from sqlalchemy import event
from sqlalchemy.orm import Session

session = Session()


@event.listens_for(session, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
Or with the Session class itself, as well as with a specific sessionmaker, which is likely the most useful form:
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()


@event.listens_for(maker, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
The listeners can of course be stacked on top of one function, as is likely to be common. For example, to track all objects that are entering the persistent state:
@event.listens_for(maker, "pending_to_persistent")
@event.listens_for(maker, "deleted_to_persistent")
@event.listens_for(maker, "detached_to_persistent")
@event.listens_for(maker, "loaded_as_persistent")
def detect_all_persistent(session, instance):
    print("object is now persistent: %s" % instance)
Transient
All mapped objects when first constructed start out as transient. In this state, the object exists alone and doesn’t have an association with any Session. For this initial state, there’s no specific “transition” event since there is no Session, however if one wanted to intercept when any transient object is created, the InstanceEvents.init() method is probably the best event. This event is applied to a specific class or superclass. For example, to intercept all new objects for a particular declarative base:
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import event


class Base(DeclarativeBase):
    pass


@event.listens_for(Base, "init", propagate=True)
def intercept_init(instance, args, kwargs):
    print("new transient: %s" % instance)
Transient to Pending
The transient object becomes pending when it is first associated with a Session via the Session.add() or Session.add_all() method. An object may also become part of a Session as a result of a “cascade” from a referencing object that was explicitly added. The transient to pending transition is detectable using the SessionEvents.transient_to_pending() event:
@event.listens_for(sessionmaker, "transient_to_pending")
def intercept_transient_to_pending(session, object_):
    print("transient to pending: %s" % object_)
Pending to Persistent
The pending object becomes persistent when a flush proceeds and an INSERT statement takes place for the instance. The object now has an identity key. Track pending to persistent with the SessionEvents.pending_to_persistent() event:
@event.listens_for(sessionmaker, "pending_to_persistent")
def intercept_pending_to_persistent(session, object_):
    print("pending to persistent: %s" % object_)
Pending to Transient
The pending object can revert back to transient if the Session.rollback() method is called before the pending object has been flushed, or if the Session.expunge() method is called for the object before it is flushed. Track pending to transient with the SessionEvents.pending_to_transient() event:
@event.listens_for(sessionmaker, "pending_to_transient")
def intercept_pending_to_transient(session, object_):
    print("transient to pending: %s" % object_)
Loaded as Persistent
Objects can appear in the Session directly in the persistent state when they are loaded from the database. Tracking this state transition is synonymous with tracking objects as they are loaded, and is synonymous with using the InstanceEvents.load() instance-level event. However, the SessionEvents.loaded_as_persistent() event is provided as a session-centric hook for intercepting objects as they enter the persistent state via this particular avenue:
@event.listens_for(sessionmaker, "loaded_as_persistent")
def intercept_loaded_as_persistent(session, object_):
    print("object loaded into persistent state: %s" % object_)
Persistent to Transient
The persistent object can revert to the transient state if the Session.rollback() method is called for a transaction where the object was first added as pending. In the case of the ROLLBACK, the INSERT statement that made this object persistent is rolled back, and the object is evicted from the Session to again become transient. Track objects that were reverted to transient from persistent using the SessionEvents.persistent_to_transient() event hook:
@event.listens_for(sessionmaker, "persistent_to_transient")
def intercept_persistent_to_transient(session, object_):
    print("persistent to transient: %s" % object_)
Persistent to Deleted
The persistent object enters the deleted state when an object marked for deletion is deleted from the database within the flush process. Note that this is not the same as when the Session.delete() method is called for a target object. The Session.delete() method only marks the object for deletion; the actual DELETE statement is not emitted until the flush proceeds. It is subsequent to the flush that the “deleted” state is present for the target object.
Within the “deleted” state, the object is only marginally associated with the Session. It is not present in the identity map nor is it present in the Session.deleted collection that refers to when it was pending for deletion.
From the “deleted” state, the object can go either to the detached state when the transaction is committed, or back to the persistent state if the transaction is instead rolled back.
Track the persistent to deleted transition with SessionEvents.persistent_to_deleted():
@event.listens_for(sessionmaker, "persistent_to_deleted")
def intercept_persistent_to_deleted(session, object_):
    print("object was DELETEd, is now in deleted state: %s" % object_)
Deleted to Detached
The deleted object becomes detached when the session’s transaction is committed. After the Session.commit() method is called, the database transaction is final and the Session now fully discards the deleted object and removes all associations to it. Track the deleted to detached transition using SessionEvents.deleted_to_detached():
@event.listens_for(sessionmaker, "deleted_to_detached")
def intercept_deleted_to_detached(session, object_):
    print("deleted to detached: %s" % object_)
Note
While the object is in the deleted state, the InstanceState.deleted attribute, accessible using inspect(object).deleted, returns True. However when the object is detached, InstanceState.deleted will again return False. To detect that an object was deleted, regardless of whether or not it is detached, use the InstanceState.was_deleted accessor.
Persistent to Detached
The persistent object becomes detached when the object is de-associated with the Session, via the Session.expunge(), Session.expunge_all(), or Session.close() methods.
Note
An object may also become implicitly detached if its owning Session is dereferenced by the application and discarded due to garbage collection. In this case, no event is emitted.
Track objects as they move from persistent to detached using the SessionEvents.persistent_to_detached() event:
@event.listens_for(sessionmaker, "persistent_to_detached")
def intercept_persistent_to_detached(session, object_):
    print("object became detached: %s" % object_)
Detached to Persistent
The detached object becomes persistent when it is re-associated with a session using the Session.add() or equivalent method. Track objects moving back to persistent from detached using the SessionEvents.detached_to_persistent() event:
@event.listens_for(sessionmaker, "detached_to_persistent")
def intercept_detached_to_persistent(session, object_):
    print("object became persistent again: %s" % object_)
Deleted to Persistent
The deleted object can be reverted to the persistent state when the transaction in which it was DELETEd was rolled back using the Session.rollback() method. Track deleted objects moving back to the persistent state using the SessionEvents.deleted_to_persistent() event:
@event.listens_for(sessionmaker, "deleted_to_persistent")
def intercept_deleted_to_persistent(session, object_):
    print("deleted to persistent: %s" % object_)
Transaction Events
Transaction events allow an application to be notified when transaction boundaries occur at the Session level as well as when the Session changes the transactional state on Connection objects.
* SessionEvents.after_transaction_create(), SessionEvents.after_transaction_end() - these events track the logical transaction scopes of the Session in a way that is not specific to individual database connections. These events are intended to help with integration of transaction-tracking systems such as zope.sqlalchemy. Use these events when the application needs to align some external scope with the transactional scope of the Session. These hooks mirror the “nested” transactional behavior of the Session, in that they track logical “subtransactions” as well as “nested” (e.g. SAVEPOINT) transactions.
* SessionEvents.before_commit(), SessionEvents.after_commit(), SessionEvents.after_begin(), SessionEvents.after_rollback(), SessionEvents.after_soft_rollback() - These events allow tracking of transaction events from the perspective of database connections. SessionEvents.after_begin() in particular is a per-connection event; a Session that maintains more than one connection will emit this event for each connection individually as those connections become used within the current transaction. The rollback and commit events then refer to when the DBAPI connections themselves have received rollback or commit instructions directly.
Attribute Change Events
The attribute change events allow interception of when specific attributes on an object are modified. These events include AttributeEvents.set(), AttributeEvents.append(), and AttributeEvents.remove(). These events are extremely useful, particularly for per-object validation operations; however, it is often much more convenient to use a “validator” hook, which uses these hooks behind the scenes; see Simple Validators for background on this. The attribute events are also behind the mechanics of backreferences. An example illustrating use of attribute events is in Attribute Instrumentation.


Session API
Session and sessionmaker()
Object Name
Description
ORMExecuteState
Represents a call to the Session.execute() method, as passed to the SessionEvents.do_orm_execute() event hook.
Session
Manages persistence operations for ORM-mapped objects.
sessionmaker
A configurable Session factory.
SessionTransaction
A Session-level transaction.
SessionTransactionOrigin
indicates the origin of a SessionTransaction.
class sqlalchemy.orm.sessionmaker
A configurable Session factory.
The sessionmaker factory generates new Session objects when called, creating them given the configurational arguments established here.
e.g.:
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

Session = sessionmaker(engine)

with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
Context manager use is optional; otherwise, the returned Session object may be closed explicitly via the Session.close() method. Using a try:/finally: block is optional, however will ensure that the close takes place even if there are database errors:
session = Session()
try:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
finally:
    session.close()
sessionmaker acts as a factory for Session objects in the same way as an Engine acts as a factory for Connection objects. In this way it also includes a sessionmaker.begin() method, that provides a context manager which both begins and commits a transaction, as well as closes out the Session when complete, rolling back the transaction if any errors occur:
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits transaction, closes session
Added in version 1.4.
When calling upon sessionmaker to construct a Session, keyword arguments may also be passed to the method; these arguments will override that of the globally configured parameters. Below we use a sessionmaker bound to a certain Engine to produce a Session that is instead bound to a specific Connection procured from that engine:
Session = sessionmaker(engine)

# bind an individual session to a connection

with engine.connect() as connection:
    with Session(bind=connection) as session:
         # work with session
The class also includes a method sessionmaker.configure(), which can be used to specify additional keyword arguments to the factory, which will take effect for subsequent Session objects generated. This is usually used to associate one or more Engine objects with an existing sessionmaker factory before it is first used:
# application starts, sessionmaker does not have
# an engine bound yet
Session = sessionmaker()

# later, when an engine URL is read from a configuration
# file or other events allow the engine to be created
engine = create_engine("sqlite:///foo.db")
Session.configure(bind=engine)

sess = Session()
# work with session
See also
Opening and Closing a Session - introductory text on creating sessions using sessionmaker.
Members
__call__(), __init__(), begin(), close_all(), configure(), identity_key(), object_session()
Class signature
class sqlalchemy.orm.sessionmaker (sqlalchemy.orm.session._SessionClassMethods, typing.Generic)
method sqlalchemy.orm.sessionmaker.__call__(**local_kw: Any) ? _S
Produce a new Session object using the configuration established in this sessionmaker.
In Python, the __call__ method is invoked on an object when it is “called” in the same way as a function:
Session = sessionmaker(some_engine)
session = Session()  # invokes sessionmaker.__call__()
method sqlalchemy.orm.sessionmaker.__init__(bind: Optional[_SessionBind] = None, *, class_: Type[_S] = <class 'sqlalchemy.orm.session.Session'>, autoflush: bool = True, expire_on_commit: bool = True, info: Optional[_InfoType] = None, **kw: Any)
Construct a new sessionmaker.
All arguments here except for class_ correspond to arguments accepted by Session directly. See the Session.__init__() docstring for more details on parameters.
Parameters:
* bind – a Engine or other Connectable with which newly created Session objects will be associated.
* class_ – class to use in order to create new Session objects. Defaults to Session.
* autoflush – 
The autoflush setting to use with newly created Session objects.
See also
Flushing - additional background on autoflush
* expire_on_commit=True – the Session.expire_on_commit setting to use with newly created Session objects.
* info – optional dictionary of information that will be available via Session.info. Note this dictionary is updated, not replaced, when the info parameter is specified to the specific Session construction operation.
* **kw – all other keyword arguments are passed to the constructor of newly created Session objects.
method sqlalchemy.orm.sessionmaker.begin() ? AbstractContextManager[_S]
Produce a context manager that both provides a new Session as well as a transaction that commits.
e.g.:
Session = sessionmaker(some_engine)

with Session.begin() as session:
    session.add(some_object)

# commits transaction, closes session
Added in version 1.4.
classmethod sqlalchemy.orm.sessionmaker.close_all() ? None
inherited from the sqlalchemy.orm.session._SessionClassMethods.close_all method of sqlalchemy.orm.session._SessionClassMethods
Close all sessions in memory.
Deprecated since version 1.3: The Session.close_all() method is deprecated and will be removed in a future release. Please refer to close_all_sessions().
method sqlalchemy.orm.sessionmaker.configure(**new_kw: Any) ? None
(Re)configure the arguments for this sessionmaker.
e.g.:
Session = sessionmaker()

Session.configure(bind=create_engine("sqlite://"))
classmethod sqlalchemy.orm.sessionmaker.identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) ? _IdentityKeyType[Any]
inherited from the sqlalchemy.orm.session._SessionClassMethods.identity_key method of sqlalchemy.orm.session._SessionClassMethods
Return an identity key.
This is an alias of identity_key().
classmethod sqlalchemy.orm.sessionmaker.object_session(instance: object) ? Session | None
inherited from the sqlalchemy.orm.session._SessionClassMethods.object_session method of sqlalchemy.orm.session._SessionClassMethods
Return the Session to which an object belongs.
This is an alias of object_session().
class sqlalchemy.orm.ORMExecuteState
Represents a call to the Session.execute() method, as passed to the SessionEvents.do_orm_execute() event hook.
Added in version 1.4.
See also
Execute Events - top level documentation on how to use SessionEvents.do_orm_execute()
Members
__init__(), all_mappers, bind_arguments, bind_mapper, execution_options, invoke_statement(), is_column_load, is_delete, is_executemany, is_from_statement, is_insert, is_orm_statement, is_relationship_load, is_select, is_update, lazy_loaded_from, load_options, loader_strategy_path, local_execution_options, parameters, session, statement, update_delete_options, update_execution_options(), user_defined_options
Class signature
class sqlalchemy.orm.ORMExecuteState (sqlalchemy.util.langhelpers.MemoizedSlots)
method sqlalchemy.orm.ORMExecuteState.__init__(session: Session, statement: Executable, parameters: _CoreAnyExecuteParams | None, execution_options: _ExecuteOptions, bind_arguments: _BindArguments, compile_state_cls: Type[ORMCompileState] | None, events_todo: List[_InstanceLevelDispatch[Session]])
Construct a new ORMExecuteState.
this object is constructed internally.
attribute sqlalchemy.orm.ORMExecuteState.all_mappers
Return a sequence of all Mapper objects that are involved at the top level of this statement.
By “top level” we mean those Mapper objects that would be represented in the result set rows for a select() query, or for a update() or delete() query, the mapper that is the main subject of the UPDATE or DELETE.
Added in version 1.4.0b2.
See also
ORMExecuteState.bind_mapper
attribute sqlalchemy.orm.ORMExecuteState.bind_arguments: _BindArguments
The dictionary passed as the Session.execute.bind_arguments dictionary.
This dictionary may be used by extensions to Session to pass arguments that will assist in determining amongst a set of database connections which one should be used to invoke this statement.
attribute sqlalchemy.orm.ORMExecuteState.bind_mapper
Return the Mapper that is the primary “bind” mapper.
For an ORMExecuteState object invoking an ORM statement, that is, the ORMExecuteState.is_orm_statement attribute is True, this attribute will return the Mapper that is considered to be the “primary” mapper of the statement. The term “bind mapper” refers to the fact that a Session object may be “bound” to multiple Engine objects keyed to mapped classes, and the “bind mapper” determines which of those Engine objects would be selected.
For a statement that is invoked against a single mapped class, ORMExecuteState.bind_mapper is intended to be a reliable way of getting this mapper.
Added in version 1.4.0b2.
See also
ORMExecuteState.all_mappers
attribute sqlalchemy.orm.ORMExecuteState.execution_options: _ExecuteOptions
The complete dictionary of current execution options.
This is a merge of the statement level options with the locally passed execution options.
See also
ORMExecuteState.local_execution_options
Executable.execution_options()
ORM Execution Options
method sqlalchemy.orm.ORMExecuteState.invoke_statement(statement: Executable | None = None, params: _CoreAnyExecuteParams | None = None, execution_options: OrmExecuteOptionsParameter | None = None, bind_arguments: _BindArguments | None = None) ? Result[Any]
Execute the statement represented by this ORMExecuteState, without re-invoking events that have already proceeded.
This method essentially performs a re-entrant execution of the current statement for which the SessionEvents.do_orm_execute() event is being currently invoked. The use case for this is for event handlers that want to override how the ultimate Result object is returned, such as for schemes that retrieve results from an offline cache or which concatenate results from multiple executions.
When the Result object is returned by the actual handler function within SessionEvents.do_orm_execute() and is propagated to the calling Session.execute() method, the remainder of the Session.execute() method is preempted and the Result object is returned to the caller of Session.execute() immediately.
Parameters:
* statement – optional statement to be invoked, in place of the statement currently represented by ORMExecuteState.statement.
* params – 
optional dictionary of parameters or list of parameters which will be merged into the existing ORMExecuteState.parameters of this ORMExecuteState.
Changed in version 2.0: a list of parameter dictionaries is accepted for executemany executions.
* execution_options – optional dictionary of execution options will be merged into the existing ORMExecuteState.execution_options of this ORMExecuteState.
* bind_arguments – optional dictionary of bind_arguments which will be merged amongst the current ORMExecuteState.bind_arguments of this ORMExecuteState.
Returns:
a Result object with ORM-level results.
See also
Re-Executing Statements - background and examples on the appropriate usage of ORMExecuteState.invoke_statement().
attribute sqlalchemy.orm.ORMExecuteState.is_column_load
Return True if the operation is refreshing column-oriented attributes on an existing ORM object.
This occurs during operations such as Session.refresh(), as well as when an attribute deferred by defer() is being loaded, or an attribute that was expired either directly by Session.expire() or via a commit operation is being loaded.
Handlers will very likely not want to add any options to queries when such an operation is occurring as the query should be a straight primary key fetch which should not have any additional WHERE criteria, and loader options travelling with the instance will have already been added to the query.
Added in version 1.4.0b2.
See also
ORMExecuteState.is_relationship_load
attribute sqlalchemy.orm.ORMExecuteState.is_delete
return True if this is a DELETE operation.
Changed in version 2.0.30: - the attribute is also True for a Select.from_statement() construct that is itself against a Delete construct, such as select(Entity).from_statement(delete(..))
attribute sqlalchemy.orm.ORMExecuteState.is_executemany
return True if the parameters are a multi-element list of dictionaries with more than one dictionary.
Added in version 2.0.
attribute sqlalchemy.orm.ORMExecuteState.is_from_statement
return True if this operation is a Select.from_statement() operation.
This is independent from ORMExecuteState.is_select, as a select().from_statement() construct can be used with INSERT/UPDATE/DELETE RETURNING types of statements as well. ORMExecuteState.is_select will only be set if the Select.from_statement() is itself against a Select construct.
Added in version 2.0.30.
attribute sqlalchemy.orm.ORMExecuteState.is_insert
return True if this is an INSERT operation.
Changed in version 2.0.30: - the attribute is also True for a Select.from_statement() construct that is itself against a Insert construct, such as select(Entity).from_statement(insert(..))
attribute sqlalchemy.orm.ORMExecuteState.is_orm_statement
return True if the operation is an ORM statement.
This indicates that the select(), insert(), update(), or delete() being invoked contains ORM entities as subjects. For a statement that does not have ORM entities and instead refers only to Table metadata, it is invoked as a Core SQL statement and no ORM-level automation takes place.
attribute sqlalchemy.orm.ORMExecuteState.is_relationship_load
Return True if this load is loading objects on behalf of a relationship.
This means, the loader in effect is either a LazyLoader, SelectInLoader, SubqueryLoader, or similar, and the entire SELECT statement being emitted is on behalf of a relationship load.
Handlers will very likely not want to add any options to queries when such an operation is occurring, as loader options are already capable of being propagated to relationship loaders and should be already present.
See also
ORMExecuteState.is_column_load
attribute sqlalchemy.orm.ORMExecuteState.is_select
return True if this is a SELECT operation.
Changed in version 2.0.30: - the attribute is also True for a Select.from_statement() construct that is itself against a Select construct, such as select(Entity).from_statement(select(..))
attribute sqlalchemy.orm.ORMExecuteState.is_update
return True if this is an UPDATE operation.
Changed in version 2.0.30: - the attribute is also True for a Select.from_statement() construct that is itself against a Update construct, such as select(Entity).from_statement(update(..))
attribute sqlalchemy.orm.ORMExecuteState.lazy_loaded_from
An InstanceState that is using this statement execution for a lazy load operation.
The primary rationale for this attribute is to support the horizontal sharding extension, where it is available within specific query execution time hooks created by this extension. To that end, the attribute is only intended to be meaningful at query execution time, and importantly not any time prior to that, including query compilation time.
attribute sqlalchemy.orm.ORMExecuteState.load_options
Return the load_options that will be used for this execution.
attribute sqlalchemy.orm.ORMExecuteState.loader_strategy_path
Return the PathRegistry for the current load path.
This object represents the “path” in a query along relationships when a particular object or collection is being loaded.
attribute sqlalchemy.orm.ORMExecuteState.local_execution_options: _ExecuteOptions
Dictionary view of the execution options passed to the Session.execute() method.
This does not include options that may be associated with the statement being invoked.
See also
ORMExecuteState.execution_options
attribute sqlalchemy.orm.ORMExecuteState.parameters: _CoreAnyExecuteParams | None
Dictionary of parameters that was passed to Session.execute().
attribute sqlalchemy.orm.ORMExecuteState.session: Session
The Session in use.
attribute sqlalchemy.orm.ORMExecuteState.statement: Executable
The SQL statement being invoked.
For an ORM selection as would be retrieved from Query, this is an instance of select that was generated from the ORM query.
attribute sqlalchemy.orm.ORMExecuteState.update_delete_options
Return the update_delete_options that will be used for this execution.
method sqlalchemy.orm.ORMExecuteState.update_execution_options(**opts: Any) ? None
Update the local execution options with new values.
attribute sqlalchemy.orm.ORMExecuteState.user_defined_options
The sequence of UserDefinedOptions that have been associated with the statement being invoked.
class sqlalchemy.orm.Session
Manages persistence operations for ORM-mapped objects.
The Session is not safe for use in concurrent threads.. See Is the Session thread-safe? Is AsyncSession safe to share in concurrent tasks? for background.
The Session’s usage paradigm is described at Using the Session.
Members
__init__(), add(), add_all(), begin(), begin_nested(), bind_mapper(), bind_table(), bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), connection(), delete(), deleted, dirty, enable_relationship_loading(), execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_nested_transaction(), get_one(), get_transaction(), identity_key(), identity_map, in_nested_transaction(), in_transaction(), info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), prepare(), query(), refresh(), reset(), rollback(), scalar(), scalars()
Class signature
class sqlalchemy.orm.Session (sqlalchemy.orm.session._SessionClassMethods, sqlalchemy.event.registry.EventTarget)
method sqlalchemy.orm.Session.__init__(bind: _SessionBind | None = None, *, autoflush: bool = True, future: Literal[True] = True, expire_on_commit: bool = True, autobegin: bool = True, twophase: bool = False, binds: Dict[_SessionBindKey, _SessionBind] | None = None, enable_baked_queries: bool = True, info: _InfoType | None = None, query_cls: Type[Query[Any]] | None = None, autocommit: Literal[False] = False, join_transaction_mode: JoinTransactionMode = 'conditional_savepoint', close_resets_only: bool | _NoArg = _NoArg.NO_ARG)
Construct a new Session.
See also the sessionmaker function which is used to generate a Session-producing callable with a given set of arguments.
Parameters:
* autoflush – 
When True, all query operations will issue a Session.flush() call to this Session before proceeding. This is a convenience feature so that Session.flush() need not be called repeatedly in order for database queries to retrieve results.
See also
Flushing - additional background on autoflush
* autobegin – 
Automatically start transactions (i.e. equivalent to invoking Session.begin()) when database access is requested by an operation. Defaults to True. Set to False to prevent a Session from implicitly beginning transactions after construction, as well as after any of the Session.rollback(), Session.commit(), or Session.close() methods are called.
Added in version 2.0.
See also
Disabling Autobegin to Prevent Implicit Transactions
* bind – An optional Engine or Connection to which this Session should be bound. When specified, all SQL operations performed by this session will execute via this connectable.
* binds – 
A dictionary which may specify any number of Engine or Connection objects as the source of connectivity for SQL operations on a per-entity basis. The keys of the dictionary consist of any series of mapped classes, arbitrary Python classes that are bases for mapped classes, Table objects and Mapper objects. The values of the dictionary are then instances of Engine or less commonly Connection objects. Operations which proceed relative to a particular mapped class will consult this dictionary for the closest matching entity in order to determine which Engine should be used for a particular SQL operation. The complete heuristics for resolution are described at Session.get_bind(). Usage looks like:
Session = sessionmaker(
    binds={
        SomeMappedClass: create_engine("postgresql+psycopg2://engine1"),
        SomeDeclarativeBase: create_engine(
            "postgresql+psycopg2://engine2"
        ),
        some_mapper: create_engine("postgresql+psycopg2://engine3"),
        some_table: create_engine("postgresql+psycopg2://engine4"),
    }
)
* See also
Partitioning Strategies (e.g. multiple database backends per Session)
Session.bind_mapper()
Session.bind_table()
Session.get_bind()
* class_ – Specify an alternate class other than sqlalchemy.orm.session.Session which should be used by the returned class. This is the only argument that is local to the sessionmaker function, and is not sent directly to the constructor for Session.
* enable_baked_queries – 
legacy; defaults to True. A parameter consumed by the sqlalchemy.ext.baked extension to determine if “baked queries” should be cached, as is the normal operation of this extension. When set to False, caching as used by this particular extension is disabled.
Changed in version 1.4: The sqlalchemy.ext.baked extension is legacy and is not used by any of SQLAlchemy’s internals. This flag therefore only affects applications that are making explicit use of this extension within their own code.
* expire_on_commit – 
Defaults to True. When True, all instances will be fully expired after each commit(), so that all attribute/object access subsequent to a completed transaction will load from the most recent database state.
See also
Committing
* future – 
Deprecated; this flag is always True.
See also
SQLAlchemy 2.0 - Major Migration Guide
* info – optional dictionary of arbitrary data to be associated with this Session. Is available via the Session.info attribute. Note the dictionary is copied at construction time so that modifications to the per- Session dictionary will be local to that Session.
* query_cls – Class which should be used to create new Query objects, as returned by the Session.query() method. Defaults to Query.
* twophase – When True, all transactions will be started as a “two phase” transaction, i.e. using the “two phase” semantics of the database in use along with an XID. During a commit(), after flush() has been issued for all attached databases, the TwoPhaseTransaction.prepare() method on each database’s TwoPhaseTransaction will be called. This allows each database to roll back the entire transaction, before each transaction is committed.
* autocommit – the “autocommit” keyword is present for backwards compatibility but must remain at its default value of False.
* join_transaction_mode – 
Describes the transactional behavior to take when a given bind is a Connection that has already begun a transaction outside the scope of this Session; in other words the Connection.in_transaction() method returns True.
The following behaviors only take effect when the Session actually makes use of the connection given; that is, a method such as Session.execute(), Session.connection(), etc. are actually invoked:
o "conditional_savepoint" - this is the default. if the given Connection is begun within a transaction but does not have a SAVEPOINT, then "rollback_only" is used. If the Connection is additionally within a SAVEPOINT, in other words Connection.in_nested_transaction() method returns True, then "create_savepoint" is used.
"conditional_savepoint" behavior attempts to make use of savepoints in order to keep the state of the existing transaction unchanged, but only if there is already a savepoint in progress; otherwise, it is not assumed that the backend in use has adequate support for SAVEPOINT, as availability of this feature varies. "conditional_savepoint" also seeks to establish approximate backwards compatibility with previous Session behavior, for applications that are not setting a specific mode. It is recommended that one of the explicit settings be used.
o "create_savepoint" - the Session will use Connection.begin_nested() in all cases to create its own transaction. This transaction by its nature rides “on top” of any existing transaction that’s opened on the given Connection; if the underlying database and the driver in use has full, non-broken support for SAVEPOINT, the external transaction will remain unaffected throughout the lifespan of the Session.
The "create_savepoint" mode is the most useful for integrating a Session into a test suite where an externally initiated transaction should remain unaffected; however, it relies on proper SAVEPOINT support from the underlying driver and database.
Tip
When using SQLite, the SQLite driver included through Python 3.11 does not handle SAVEPOINTs correctly in all cases without workarounds. See the sections Serializable isolation / Savepoints / Transactional DDL and Serializable isolation / Savepoints / Transactional DDL (asyncio version) for details on current workarounds.
o "control_fully" - the Session will take control of the given transaction as its own; Session.commit() will call .commit() on the transaction, Session.rollback() will call .rollback() on the transaction, Session.close() will call .rollback on the transaction.
Tip
This mode of use is equivalent to how SQLAlchemy 1.4 would handle a Connection given with an existing SAVEPOINT (i.e. Connection.begin_nested()); the Session would take full control of the existing SAVEPOINT.
o "rollback_only" - the Session will take control of the given transaction for .rollback() calls only; .commit() calls will not be propagated to the given transaction. .close() calls will have no effect on the given transaction.
Tip
This mode of use is equivalent to how SQLAlchemy 1.4 would handle a Connection given with an existing regular database transaction (i.e. Connection.begin()); the Session would propagate Session.rollback() calls to the underlying transaction, but not Session.commit() or Session.close() calls.
Added in version 2.0.0rc1.
* close_resets_only – 
Defaults to True. Determines if the session should reset itself after calling .close() or should pass in a no longer usable state, disabling re-use.
Added in version 2.0.22: added flag close_resets_only. A future SQLAlchemy version may change the default value of this flag to False.
See also
Closing - Detail on the semantics of Session.close() and Session.reset().
method sqlalchemy.orm.Session.add(instance: object, _warn: bool = True) ? None
Place an object into this Session.
Objects that are in the transient state when passed to the Session.add() method will move to the pending state, until the next flush, at which point they will move to the persistent state.
Objects that are in the detached state when passed to the Session.add() method will move to the persistent state directly.
If the transaction used by the Session is rolled back, objects which were transient when they were passed to Session.add() will be moved back to the transient state, and will no longer be present within this Session.
See also
Session.add_all()
Adding New or Existing Items - at Basics of Using a Session
method sqlalchemy.orm.Session.add_all(instances: Iterable[object]) ? None
Add the given collection of instances to this Session.
See the documentation for Session.add() for a general behavioral description.
See also
Session.add()
Adding New or Existing Items - at Basics of Using a Session
method sqlalchemy.orm.Session.begin(nested: bool = False) ? SessionTransaction
Begin a transaction, or nested transaction, on this Session, if one is not already begun.
The Session object features autobegin behavior, so that normally it is not necessary to call the Session.begin() method explicitly. However, it may be used in order to control the scope of when the transactional state is begun.
When used to begin the outermost transaction, an error is raised if this Session is already inside of a transaction.
Parameters:
nested – if True, begins a SAVEPOINT transaction and is equivalent to calling Session.begin_nested(). For documentation on SAVEPOINT transactions, please see Using SAVEPOINT.
Returns:
the SessionTransaction object. Note that SessionTransaction acts as a Python context manager, allowing Session.begin() to be used in a “with” block. See Explicit Begin for an example.
See also
Auto Begin
Managing Transactions
Session.begin_nested()
method sqlalchemy.orm.Session.begin_nested() ? SessionTransaction
Begin a “nested” transaction on this Session, e.g. SAVEPOINT.
The target database(s) and associated drivers must support SQL SAVEPOINT for this method to function correctly.
For documentation on SAVEPOINT transactions, please see Using SAVEPOINT.
Returns:
the SessionTransaction object. Note that SessionTransaction acts as a context manager, allowing Session.begin_nested() to be used in a “with” block. See Using SAVEPOINT for a usage example.
See also
Using SAVEPOINT
Serializable isolation / Savepoints / Transactional DDL - special workarounds required with the SQLite driver in order for SAVEPOINT to work correctly. For asyncio use cases, see the section Serializable isolation / Savepoints / Transactional DDL (asyncio version).
method sqlalchemy.orm.Session.bind_mapper(mapper: _EntityBindKey[_O], bind: _SessionBind) ? None
Associate a Mapper or arbitrary Python class with a “bind”, e.g. an Engine or Connection.
The given entity is added to a lookup used by the Session.get_bind() method.
Parameters:
* mapper – a Mapper object, or an instance of a mapped class, or any Python class that is the base of a set of mapped classes.
* bind – an Engine or Connection object.
See also
Partitioning Strategies (e.g. multiple database backends per Session)
Session.binds
Session.bind_table()
method sqlalchemy.orm.Session.bind_table(table: TableClause, bind: Engine | Connection) ? None
Associate a Table with a “bind”, e.g. an Engine or Connection.
The given Table is added to a lookup used by the Session.get_bind() method.
Parameters:
* table – a Table object, which is typically the target of an ORM mapping, or is present within a selectable that is mapped.
* bind – an Engine or Connection object.
See also
Partitioning Strategies (e.g. multiple database backends per Session)
Session.binds
Session.bind_mapper()
method sqlalchemy.orm.Session.bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) ? None
Perform a bulk insert of the given list of mapping dictionaries.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key. The 2.0 API shares implementation details with this method and adds new features as well.
Parameters:
* mapper – a mapped class, or the actual Mapper object, representing the single kind of object represented within the mapping list.
* mappings – a sequence of dictionaries, each one containing the state of the mapped row to be inserted, in terms of the attribute names on the mapped class. If the mapping refers to multiple tables, such as a joined-inheritance mapping, each dictionary must contain all keys to be populated into all tables.
* return_defaults – 
when True, the INSERT process will be altered to ensure that newly generated primary key values will be fetched. The rationale for this parameter is typically to enable Joined Table Inheritance mappings to be bulk inserted.
Note
for backends that don’t support RETURNING, the Session.bulk_insert_mappings.return_defaults parameter can significantly decrease performance as INSERT statements can no longer be batched. See “Insert Many Values” Behavior for INSERT statements for background on which backends are affected.
* render_nulls – 
When True, a value of None will result in a NULL value being included in the INSERT statement, rather than the column being omitted from the INSERT. This allows all the rows being INSERTed to have the identical set of columns which allows the full set of rows to be batched to the DBAPI. Normally, each column-set that contains a different combination of NULL values than the previous row must omit a different series of columns from the rendered INSERT statement, which means it must be emitted as a separate statement. By passing this flag, the full set of rows are guaranteed to be batchable into one batch; the cost however is that server-side defaults which are invoked by an omitted column will be skipped, so care must be taken to ensure that these are not necessary.
Warning
When this flag is set, server side default SQL values will not be invoked for those columns that are inserted as NULL; the NULL value will be sent explicitly. Care must be taken to ensure that no server-side default functions need to be invoked for the operation as a whole.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_save_objects()
Session.bulk_update_mappings()
method sqlalchemy.orm.Session.bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) ? None
Perform a bulk save of the given list of objects.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key.
For general INSERT and UPDATE of existing ORM mapped objects, prefer standard unit of work data management patterns, introduced in the SQLAlchemy Unified Tutorial at Data Manipulation with the ORM. SQLAlchemy 2.0 now uses “Insert Many Values” Behavior for INSERT statements with modern dialects which solves previous issues of bulk INSERT slowness.
Parameters:
* objects – 
a sequence of mapped object instances. The mapped objects are persisted as is, and are not associated with the Session afterwards.
For each object, whether the object is sent as an INSERT or an UPDATE is dependent on the same rules used by the Session in traditional operation; if the object has the InstanceState.key attribute set, then the object is assumed to be “detached” and will result in an UPDATE. Otherwise, an INSERT is used.
In the case of an UPDATE, statements are grouped based on which attributes have changed, and are thus to be the subject of each SET clause. If update_changed_only is False, then all attributes present within each object are applied to the UPDATE statement, which may help in allowing the statements to be grouped together into a larger executemany(), and will also reduce the overhead of checking history on attributes.
* return_defaults – when True, rows that are missing values which generate defaults, namely integer primary key defaults and sequences, will be inserted one at a time, so that the primary key value is available. In particular this will allow joined-inheritance and other multi-table mappings to insert correctly without the need to provide primary key values ahead of time; however, Session.bulk_save_objects.return_defaults greatly reduces the performance gains of the method overall. It is strongly advised to please use the standard Session.add_all() approach.
* update_changed_only – when True, UPDATE statements are rendered based on those attributes in each state that have logged changes. When False, all attributes present are rendered into the SET clause with the exception of primary key attributes.
* preserve_order – when True, the order of inserts and updates matches exactly the order in which the objects are given. When False, common types of objects are grouped into inserts and updates, to allow for more batching opportunities.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_insert_mappings()
Session.bulk_update_mappings()
method sqlalchemy.orm.Session.bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) ? None
Perform a bulk update of the given list of mapping dictionaries.
Legacy Feature
This method is a legacy feature as of the 2.0 series of SQLAlchemy. For modern bulk INSERT and UPDATE, see the sections ORM Bulk INSERT Statements and ORM Bulk UPDATE by Primary Key. The 2.0 API shares implementation details with this method and adds new features as well.
Parameters:
* mapper – a mapped class, or the actual Mapper object, representing the single kind of object represented within the mapping list.
* mappings – a sequence of dictionaries, each one containing the state of the mapped row to be updated, in terms of the attribute names on the mapped class. If the mapping refers to multiple tables, such as a joined-inheritance mapping, each dictionary may contain keys corresponding to all tables. All those keys which are present and are not part of the primary key are applied to the SET clause of the UPDATE statement; the primary key values, which are required, are applied to the WHERE clause.
See also
ORM-Enabled INSERT, UPDATE, and DELETE statements
Session.bulk_insert_mappings()
Session.bulk_save_objects()
method sqlalchemy.orm.Session.close() ? None
Close out the transactional resources and ORM objects used by this Session.
This expunges all ORM objects associated with this Session, ends any transaction in progress and releases any Connection objects which this Session itself has checked out from associated Engine objects. The operation then leaves the Session in a state which it may be used again.
Tip
In the default running mode the Session.close() method does not prevent the Session from being used again. The Session itself does not actually have a distinct “closed” state; it merely means the Session will release all database connections and ORM objects.
Setting the parameter Session.close_resets_only to False will instead make the close final, meaning that any further action on the session will be forbidden.
Changed in version 1.4: The Session.close() method does not immediately create a new SessionTransaction object; instead, the new SessionTransaction is created only if the Session is used again for a database operation.
See also
Closing - detail on the semantics of Session.close() and Session.reset().
Session.reset() - a similar method that behaves like close() with the parameter Session.close_resets_only set to True.
classmethod sqlalchemy.orm.Session.close_all() ? None
inherited from the sqlalchemy.orm.session._SessionClassMethods.close_all method of sqlalchemy.orm.session._SessionClassMethods
Close all sessions in memory.
Deprecated since version 1.3: The Session.close_all() method is deprecated and will be removed in a future release. Please refer to close_all_sessions().
method sqlalchemy.orm.Session.commit() ? None
Flush pending changes and commit the current transaction.
When the COMMIT operation is complete, all objects are fully expired, erasing their internal contents, which will be automatically re-loaded when the objects are next accessed. In the interim, these objects are in an expired state and will not function if they are detached from the Session. Additionally, this re-load operation is not supported when using asyncio-oriented APIs. The Session.expire_on_commit parameter may be used to disable this behavior.
When there is no transaction in place for the Session, indicating that no operations were invoked on this Session since the previous call to Session.commit(), the method will begin and commit an internal-only “logical” transaction, that does not normally affect the database unless pending flush changes were detected, but will still invoke event handlers and object expiration rules.
The outermost database transaction is committed unconditionally, automatically releasing any SAVEPOINTs in effect.
See also
Committing
Managing Transactions
Preventing Implicit IO when Using AsyncSession
method sqlalchemy.orm.Session.connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) ? Connection
Return a Connection object corresponding to this Session object’s transactional state.
Either the Connection corresponding to the current transaction is returned, or if no transaction is in progress, a new one is begun and the Connection returned (note that no transactional state is established with the DBAPI until the first SQL statement is emitted).
Ambiguity in multi-bind or unbound Session objects can be resolved through any of the optional keyword arguments. This ultimately makes usage of the get_bind() method for resolution.
Parameters:
* bind_arguments – dictionary of bind arguments. May include “mapper”, “bind”, “clause”, other custom arguments that are passed to Session.get_bind().
* execution_options – 
a dictionary of execution options that will be passed to Connection.execution_options(), when the connection is first procured only. If the connection is already present within the Session, a warning is emitted and the arguments are ignored.
See also
Setting Transaction Isolation Levels / DBAPI AUTOCOMMIT
method sqlalchemy.orm.Session.delete(instance: object) ? None
Mark an instance as deleted.
The object is assumed to be either persistent or detached when passed; after the method is called, the object will remain in the persistent state until the next flush proceeds. During this time, the object will also be a member of the Session.deleted collection.
When the next flush proceeds, the object will move to the deleted state, indicating a DELETE statement was emitted for its row within the current transaction. When the transaction is successfully committed, the deleted object is moved to the detached state and is no longer present within this Session.
See also
Deleting - at Basics of Using a Session
attribute sqlalchemy.orm.Session.deleted
The set of all instances marked as ‘deleted’ within this Session
attribute sqlalchemy.orm.Session.dirty
The set of all persistent instances considered dirty.
E.g.:
some_mapped_object in session.dirty
Instances are considered dirty when they were modified but not deleted.
Note that this ‘dirty’ calculation is ‘optimistic’; most attribute-setting or collection modification operations will mark an instance as ‘dirty’ and place it in this set, even if there is no net change to the attribute’s value. At flush time, the value of each attribute is compared to its previously saved value, and if there’s no net change, no SQL operation will occur (this is a more expensive operation so it’s only done at flush time).
To check if an instance has actionable net changes to its attributes, use the Session.is_modified() method.
method sqlalchemy.orm.Session.enable_relationship_loading(obj: object) ? None
Associate an object with this Session for related object loading.
Warning
enable_relationship_loading() exists to serve special use cases and is not recommended for general use.
Accesses of attributes mapped with relationship() will attempt to load a value from the database using this Session as the source of connectivity. The values will be loaded based on foreign key and primary key values present on this object - if not present, then those relationships will be unavailable.
The object will be attached to this session, but will not participate in any persistence operations; its state for almost all purposes will remain either “transient” or “detached”, except for the case of relationship loading.
Also note that backrefs will often not work as expected. Altering a relationship-bound attribute on the target object may not fire off a backref event, if the effective value is what was already loaded from a foreign-key-holding value.
The Session.enable_relationship_loading() method is similar to the load_on_pending flag on relationship(). Unlike that flag, Session.enable_relationship_loading() allows an object to remain transient while still being able to load related items.
To make a transient object associated with a Session via Session.enable_relationship_loading() pending, add it to the Session using Session.add() normally. If the object instead represents an existing identity in the database, it should be merged using Session.merge().
Session.enable_relationship_loading() does not improve behavior when the ORM is used normally - object references should be constructed at the object level, not at the foreign key level, so that they are present in an ordinary way before flush() proceeds. This method is not intended for general use.
See also
relationship.load_on_pending - this flag allows per-relationship loading of many-to-ones on items that are pending.
make_transient_to_detached() - allows for an object to be added to a Session without SQL emitted, which then will unexpire attributes on access.
method sqlalchemy.orm.Session.execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) ? Result[Any]
Execute a SQL expression construct.
Returns a Result object representing results of the statement execution.
E.g.:
from sqlalchemy import select

result = session.execute(select(User).where(User.id == 5))
The API contract of Session.execute() is similar to that of Connection.execute(), the 2.0 style version of Connection.
Changed in version 1.4: the Session.execute() method is now the primary point of ORM statement execution when using 2.0 style ORM usage.
Parameters:
* statement – An executable statement (i.e. an Executable expression such as select()).
* params – Optional dictionary, or list of dictionaries, containing bound parameter values. If a single dictionary, single-row execution occurs; if a list of dictionaries, an “executemany” will be invoked. The keys in each dictionary must correspond to parameter names present in the statement.
* execution_options – 
optional dictionary of execution options, which will be associated with the statement execution. This dictionary can provide a subset of the options that are accepted by Connection.execution_options(), and may also provide additional options understood only in an ORM context.
See also
ORM Execution Options - ORM-specific execution options
* bind_arguments – dictionary of additional arguments to determine the bind. May include “mapper”, “bind”, or other custom arguments. Contents of this dictionary are passed to the Session.get_bind() method.
Returns:
a Result object.
method sqlalchemy.orm.Session.expire(instance: object, attribute_names: Iterable[str] | None = None) ? None
Expire the attributes on an instance.
Marks the attributes of an instance as out of date. When an expired attribute is next accessed, a query will be issued to the Session object’s current transactional context in order to load all expired attributes for the given instance. Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction.
To expire all objects in the Session simultaneously, use Session.expire_all().
The Session object’s default behavior is to expire all state whenever the Session.rollback() or Session.commit() methods are called, so that new state can be loaded for the new transaction. For this reason, calling Session.expire() only makes sense for the specific case that a non-ORM SQL statement was emitted in the current transaction.
Parameters:
* instance – The instance to be refreshed.
* attribute_names – optional list of string attribute names indicating a subset of attributes to be expired.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.refresh()
Query.populate_existing()
method sqlalchemy.orm.Session.expire_all() ? None
Expires all persistent instances within this Session.
When any attributes on a persistent instance is next accessed, a query will be issued using the Session object’s current transactional context in order to load all expired attributes for the given instance. Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction.
To expire individual objects and individual attributes on those objects, use Session.expire().
The Session object’s default behavior is to expire all state whenever the Session.rollback() or Session.commit() methods are called, so that new state can be loaded for the new transaction. For this reason, calling Session.expire_all() is not usually needed, assuming the transaction is isolated.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.refresh()
Query.populate_existing()
method sqlalchemy.orm.Session.expunge(instance: object) ? None
Remove the instance from this Session.
This will free all internal references to the instance. Cascading will be applied according to the expunge cascade rule.
method sqlalchemy.orm.Session.expunge_all() ? None
Remove all object instances from this Session.
This is equivalent to calling expunge(obj) on all objects in this Session.
method sqlalchemy.orm.Session.flush(objects: Sequence[Any] | None = None) ? None
Flush all the object changes to the database.
Writes out all pending object creations, deletions and modifications to the database as INSERTs, DELETEs, UPDATEs, etc. Operations are automatically ordered by the Session’s unit of work dependency solver.
Database operations will be issued in the current transactional context and do not affect the state of the transaction, unless an error occurs, in which case the entire transaction is rolled back. You may flush() as often as you like within a transaction to move changes from Python to the database’s transaction buffer.
Parameters:
objects – 
Optional; restricts the flush operation to operate only on elements that are in the given collection.
This feature is for an extremely narrow set of use cases where particular objects may need to be operated upon before the full flush() occurs. It is not intended for general use.
method sqlalchemy.orm.Session.get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) ? _O | None
Return an instance based on the given primary key identifier, or None if not found.
E.g.:
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(VersionedFoo, {"id": 5, "version_id": 10})
Added in version 1.4: Added Session.get(), which is moved from the now legacy Query.get() method.
Session.get() is special in that it provides direct access to the identity map of the Session. If the given primary key identifier is present in the local identity map, the object is returned directly from this collection and no SQL is emitted, unless the object has been marked fully expired. If not present, a SELECT is performed in order to locate the object.
Session.get() also will perform a check if the object is present in the identity map and marked as expired - a SELECT is emitted to refresh the object as well as to ensure that the row is still present. If not, ObjectDeletedError is raised.
Parameters:
* entity – a mapped class or Mapper indicating the type of entity to be loaded.
* ident – 
A scalar, tuple, or dictionary representing the primary key. For a composite (e.g. multiple column) primary key, a tuple or dictionary should be passed.
For a single-column primary key, the scalar calling form is typically the most expedient. If the primary key of a row is the value “5”, the call looks like:
my_object = session.get(SomeClass, 5)
The tuple form contains primary key values typically in the order in which they correspond to the mapped Table object’s primary key columns, or if the Mapper.primary_key configuration parameter were used, in the order used for that parameter. For example, if the primary key of a row is represented by the integer digits “5, 10” the call would look like:
my_object = session.get(SomeClass, (5, 10))
The dictionary form should include as keys the mapped attribute names corresponding to each element of the primary key. If the mapped class has the attributes id, version_id as the attributes which store the object’s primary key value, the call would look like:
my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
* 
* options – optional sequence of loader options which will be applied to the query, if one is emitted.
* populate_existing – causes the method to unconditionally emit a SQL query and refresh the object with the newly loaded data, regardless of whether or not the object is already present.
* with_for_update – optional boolean True indicating FOR UPDATE should be used, or may be a dictionary containing flags to indicate a more specific set of FOR UPDATE flags for the SELECT; flags should match the parameters of Query.with_for_update(). Supersedes the Session.refresh.lockmode parameter.
* execution_options – 
optional dictionary of execution options, which will be associated with the query execution if one is emitted. This dictionary can provide a subset of the options that are accepted by Connection.execution_options(), and may also provide additional options understood only in an ORM context.
Added in version 1.4.29.
See also
ORM Execution Options - ORM-specific execution options
* bind_arguments – 
dictionary of additional arguments to determine the bind. May include “mapper”, “bind”, or other custom arguments. Contents of this dictionary are passed to the Session.get_bind() method.
Returns:
The object instance, or None.
method sqlalchemy.orm.Session.get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) ? Engine | Connection
Return a “bind” to which this Session is bound.
The “bind” is usually an instance of Engine, except in the case where the Session has been explicitly bound directly to a Connection.
For a multiply-bound or unbound Session, the mapper or clause arguments are used to determine the appropriate bind to return.
Note that the “mapper” argument is usually present when Session.get_bind() is called via an ORM operation such as a Session.query(), each individual INSERT/UPDATE/DELETE operation within a Session.flush(), call, etc.
The order of resolution is:
1. if mapper given and Session.binds is present, locate a bind based first on the mapper in use, then on the mapped class in use, then on any base classes that are present in the __mro__ of the mapped class, from more specific superclasses to more general.
2. if clause given and Session.binds is present, locate a bind based on Table objects found in the given clause present in Session.binds.
3. if Session.binds is present, return that.
4. if clause given, attempt to return a bind linked to the MetaData ultimately associated with the clause.
5. if mapper given, attempt to return a bind linked to the MetaData ultimately associated with the Table or other selectable to which the mapper is mapped.
6. No bind can be found, UnboundExecutionError is raised.
Note that the Session.get_bind() method can be overridden on a user-defined subclass of Session to provide any kind of bind resolution scheme. See the example at Custom Vertical Partitioning.
Parameters:
* mapper – Optional mapped class or corresponding Mapper instance. The bind can be derived from a Mapper first by consulting the “binds” map associated with this Session, and secondly by consulting the MetaData associated with the Table to which the Mapper is mapped for a bind.
* clause – A ClauseElement (i.e. select(), text(), etc.). If the mapper argument is not present or could not produce a bind, the given expression construct will be searched for a bound element, typically a Table associated with bound MetaData.
See also
Partitioning Strategies (e.g. multiple database backends per Session)
Session.binds
Session.bind_mapper()
Session.bind_table()
method sqlalchemy.orm.Session.get_nested_transaction() ? SessionTransaction | None
Return the current nested transaction in progress, if any.
Added in version 1.4.
method sqlalchemy.orm.Session.get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) ? _O
Return exactly one instance based on the given primary key identifier, or raise an exception if not found.
Raises NoResultFound if the query selects no rows.
For a detailed documentation of the arguments see the method Session.get().
Added in version 2.0.22.
Returns:
The object instance.
See also
Session.get() - equivalent method that instead
returns None if no row was found with the provided primary key
method sqlalchemy.orm.Session.get_transaction() ? SessionTransaction | None
Return the current root transaction in progress, if any.
Added in version 1.4.
classmethod sqlalchemy.orm.Session.identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) ? _IdentityKeyType[Any]
inherited from the sqlalchemy.orm.session._SessionClassMethods.identity_key method of sqlalchemy.orm.session._SessionClassMethods
Return an identity key.
This is an alias of identity_key().
attribute sqlalchemy.orm.Session.identity_map: IdentityMap
A mapping of object identities to objects themselves.
Iterating through Session.identity_map.values() provides access to the full set of persistent objects (i.e., those that have row identity) currently in the session.
See also
identity_key() - helper function to produce the keys used in this dictionary.
method sqlalchemy.orm.Session.in_nested_transaction() ? bool
Return True if this Session has begun a nested transaction, e.g. SAVEPOINT.
Added in version 1.4.
method sqlalchemy.orm.Session.in_transaction() ? bool
Return True if this Session has begun a transaction.
Added in version 1.4.
See also
Session.is_active
attribute sqlalchemy.orm.Session.info
A user-modifiable dictionary.
The initial value of this dictionary can be populated using the info argument to the Session constructor or sessionmaker constructor or factory methods. The dictionary here is always local to this Session and can be modified independently of all other Session objects.
method sqlalchemy.orm.Session.invalidate() ? None
Close this Session, using connection invalidation.
This is a variant of Session.close() that will additionally ensure that the Connection.invalidate() method will be called on each Connection object that is currently in use for a transaction (typically there is only one connection unless the Session is used with multiple engines).
This can be called when the database is known to be in a state where the connections are no longer safe to be used.
Below illustrates a scenario when using gevent, which can produce Timeout exceptions that may mean the underlying connection should be discarded:
import gevent

try:
    sess = Session()
    sess.add(User())
    sess.commit()
except gevent.Timeout:
    sess.invalidate()
    raise
except:
    sess.rollback()
    raise
The method additionally does everything that Session.close() does, including that all ORM objects are expunged.
attribute sqlalchemy.orm.Session.is_active
True if this Session not in “partial rollback” state.
Changed in version 1.4: The Session no longer begins a new transaction immediately, so this attribute will be False when the Session is first instantiated.
“partial rollback” state typically indicates that the flush process of the Session has failed, and that the Session.rollback() method must be emitted in order to fully roll back the transaction.
If this Session is not in a transaction at all, the Session will autobegin when it is first used, so in this case Session.is_active will return True.
Otherwise, if this Session is within a transaction, and that transaction has not been rolled back internally, the Session.is_active will also return True.
See also
“This Session’s transaction has been rolled back due to a previous exception during flush.” (or similar)
Session.in_transaction()
method sqlalchemy.orm.Session.is_modified(instance: object, include_collections: bool = True) ? bool
Return True if the given instance has locally modified attributes.
This method retrieves the history for each instrumented attribute on the instance and performs a comparison of the current value to its previously flushed or committed value, if any.
It is in effect a more expensive and accurate version of checking for the given instance in the Session.dirty collection; a full test for each attribute’s net “dirty” status is performed.
E.g.:
return session.is_modified(someobject)
A few caveats to this method apply:
* Instances present in the Session.dirty collection may report False when tested with this method. This is because the object may have received change events via attribute mutation, thus placing it in Session.dirty, but ultimately the state is the same as that loaded from the database, resulting in no net change here.
* Scalar attributes may not have recorded the previously set value when a new value was applied, if the attribute was not loaded, or was expired, at the time the new value was received - in these cases, the attribute is assumed to have a change, even if there is ultimately no net change against its database value. SQLAlchemy in most cases does not need the “old” value when a set event occurs, so it skips the expense of a SQL call if the old value isn’t present, based on the assumption that an UPDATE of the scalar value is usually needed, and in those few cases where it isn’t, is less expensive on average than issuing a defensive SELECT.
The “old” value is fetched unconditionally upon set only if the attribute container has the active_history flag set to True. This flag is set typically for primary key attributes and scalar object references that are not a simple many-to-one. To set this flag for any arbitrary mapped column, use the active_history argument with column_property().
Parameters:
* instance – mapped instance to be tested for pending changes.
* include_collections – Indicates if multivalued collections should be included in the operation. Setting this to False is a way to detect only local-column based properties (i.e. scalar columns or many-to-one foreign keys) that would result in an UPDATE for this instance upon flush.
method sqlalchemy.orm.Session.merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) ? _O
Copy the state of a given instance into a corresponding instance within this Session.
Session.merge() examines the primary key attributes of the source instance, and attempts to reconcile it with an instance of the same primary key in the session. If not found locally, it attempts to load the object from the database based on primary key, and if none can be located, creates a new instance. The state of each attribute on the source instance is then copied to the target instance. The resulting target instance is then returned by the method; the original source instance is left unmodified, and un-associated with the Session if not already.
This operation cascades to associated instances if the association is mapped with cascade="merge".
See Merging for a detailed discussion of merging.
Parameters:
* instance – Instance to be merged.
* load – 
Boolean, when False, merge() switches into a “high performance” mode which causes it to forego emitting history events as well as all database access. This flag is used for cases such as transferring graphs of objects into a Session from a second level cache, or to transfer just-loaded objects into the Session owned by a worker thread or process without re-querying the database.
The load=False use case adds the caveat that the given object has to be in a “clean” state, that is, has no pending changes to be flushed - even if the incoming object is detached from any Session. This is so that when the merge operation populates local attributes and cascades to related objects and collections, the values can be “stamped” onto the target object as is, without generating any history or attribute events, and without the need to reconcile the incoming data with any existing related objects or collections that might not be loaded. The resulting objects from load=False are always produced as “clean”, so it is only appropriate that the given objects should be “clean” as well, else this suggests a mis-use of the method.
* options – 
optional sequence of loader options which will be applied to the Session.get() method when the merge operation loads the existing version of the object from the database.
Added in version 1.4.24.
See also
make_transient_to_detached() - provides for an alternative means of “merging” a single object into the Session
attribute sqlalchemy.orm.Session.new
The set of all instances marked as ‘new’ within this Session.
attribute sqlalchemy.orm.Session.no_autoflush
Return a context manager that disables autoflush.
e.g.:
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
Operations that proceed within the with: block will not be subject to flushes occurring upon query access. This is useful when initializing a series of objects which involve existing database queries, where the uncompleted object should not yet be flushed.
classmethod sqlalchemy.orm.Session.object_session(instance: object) ? Session | None
inherited from the sqlalchemy.orm.session._SessionClassMethods.object_session method of sqlalchemy.orm.session._SessionClassMethods
Return the Session to which an object belongs.
This is an alias of object_session().
method sqlalchemy.orm.Session.prepare() ? None
Prepare the current transaction in progress for two phase commit.
If no transaction is in progress, this method raises an InvalidRequestError.
Only root transactions of two phase sessions can be prepared. If the current transaction is not such, an InvalidRequestError is raised.
method sqlalchemy.orm.Session.query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) ? Query[Any]
Return a new Query object corresponding to this Session.
Note that the Query object is legacy as of SQLAlchemy 2.0; the select() construct is now used to construct ORM queries.
See also
SQLAlchemy Unified Tutorial
ORM Querying Guide
Legacy Query API - legacy API doc
method sqlalchemy.orm.Session.refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) ? None
Expire and refresh attributes on the given instance.
The selected attributes will first be expired as they would when using Session.expire(); then a SELECT statement will be issued to the database to refresh column-oriented attributes with the current value available in the current transaction.
relationship() oriented attributes will also be immediately loaded if they were already eagerly loaded on the object, using the same eager loading strategy that they were loaded with originally.
Added in version 1.4: - the Session.refresh() method can also refresh eagerly loaded attributes.
relationship() oriented attributes that would normally load using the select (or “lazy”) loader strategy will also load if they are named explicitly in the attribute_names collection, emitting a SELECT statement for the attribute using the immediate loader strategy. If lazy-loaded relationships are not named in Session.refresh.attribute_names, then they remain as “lazy loaded” attributes and are not implicitly refreshed.
Changed in version 2.0.4: The Session.refresh() method will now refresh lazy-loaded relationship() oriented attributes for those which are named explicitly in the Session.refresh.attribute_names collection.
Tip
While the Session.refresh() method is capable of refreshing both column and relationship oriented attributes, its primary focus is on refreshing of local column-oriented attributes on a single instance. For more open ended “refresh” functionality, including the ability to refresh the attributes on many objects at once while having explicit control over relationship loader strategies, use the populate existing feature instead.
Note that a highly isolated transaction will return the same values as were previously read in that same transaction, regardless of changes in database state outside of that transaction. Refreshing attributes usually only makes sense at the start of a transaction where database rows have not yet been accessed.
Parameters:
* attribute_names – optional. An iterable collection of string attribute names indicating a subset of attributes to be refreshed.
* with_for_update – optional boolean True indicating FOR UPDATE should be used, or may be a dictionary containing flags to indicate a more specific set of FOR UPDATE flags for the SELECT; flags should match the parameters of Query.with_for_update(). Supersedes the Session.refresh.lockmode parameter.
See also
Refreshing / Expiring - introductory material
Session.expire()
Session.expire_all()
Populate Existing - allows any ORM query to refresh objects as they would be loaded normally.
method sqlalchemy.orm.Session.reset() ? None
Close out the transactional resources and ORM objects used by this Session, resetting the session to its initial state.
This method provides for same “reset-only” behavior that the Session.close() method has provided historically, where the state of the Session is reset as though the object were brand new, and ready to be used again. This method may then be useful for Session objects which set Session.close_resets_only to False, so that “reset only” behavior is still available.
Added in version 2.0.22.
See also
Closing - detail on the semantics of Session.close() and Session.reset().
Session.close() - a similar method will additionally prevent re-use of the Session when the parameter Session.close_resets_only is set to False.
method sqlalchemy.orm.Session.rollback() ? None
Rollback the current transaction in progress.
If no transaction is in progress, this method is a pass-through.
The method always rolls back the topmost database transaction, discarding any nested transactions that may be in progress.
See also
Rolling Back
Managing Transactions
method sqlalchemy.orm.Session.scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) ? Any
Execute a statement and return a scalar result.
Usage and parameters are the same as that of Session.execute(); the return result is a scalar Python value.
method sqlalchemy.orm.Session.scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) ? ScalarResult[Any]
Execute a statement and return the results as scalars.
Usage and parameters are the same as that of Session.execute(); the return result is a ScalarResult filtering object which will return single elements rather than Row objects.
Returns:
a ScalarResult object
Added in version 1.4.24: Added Session.scalars()
Added in version 1.4.26: Added scoped_session.scalars()
See also
Selecting ORM Entities - contrasts the behavior of Session.execute() to Session.scalars()
class sqlalchemy.orm.SessionTransaction
A Session-level transaction.
SessionTransaction is produced from the Session.begin() and Session.begin_nested() methods. It’s largely an internal object that in modern use provides a context manager for session transactions.
Documentation on interacting with SessionTransaction is at: Managing Transactions.
Changed in version 1.4: The scoping and API methods to work with the SessionTransaction object directly have been simplified.
See also
Managing Transactions
Session.begin()
Session.begin_nested()
Session.rollback()
Session.commit()
Session.in_transaction()
Session.in_nested_transaction()
Session.get_transaction()
Session.get_nested_transaction()
Members
nested, origin, parent
Class signature
class sqlalchemy.orm.SessionTransaction (sqlalchemy.orm.state_changes._StateChange, sqlalchemy.engine.util.TransactionalContext)
attribute sqlalchemy.orm.SessionTransaction.nested: bool = False
Indicates if this is a nested, or SAVEPOINT, transaction.
When SessionTransaction.nested is True, it is expected that SessionTransaction.parent will be present as well, linking to the enclosing SessionTransaction.
See also
SessionTransaction.origin
attribute sqlalchemy.orm.SessionTransaction.origin: SessionTransactionOrigin
Origin of this SessionTransaction.
Refers to a SessionTransactionOrigin instance which is an enumeration indicating the source event that led to constructing this SessionTransaction.
Added in version 2.0.
attribute sqlalchemy.orm.SessionTransaction.parent
The parent SessionTransaction of this SessionTransaction.
If this attribute is None, indicates this SessionTransaction is at the top of the stack, and corresponds to a real “COMMIT”/”ROLLBACK” block. If non-None, then this is either a “subtransaction” (an internal marker object used by the flush process) or a “nested” / SAVEPOINT transaction. If the SessionTransaction.nested attribute is True, then this is a SAVEPOINT, and if False, indicates this a subtransaction.
class sqlalchemy.orm.SessionTransactionOrigin
indicates the origin of a SessionTransaction.
This enumeration is present on the SessionTransaction.origin attribute of any SessionTransaction object.
Added in version 2.0.
Members
AUTOBEGIN, BEGIN, BEGIN_NESTED, SUBTRANSACTION
Class signature
class sqlalchemy.orm.SessionTransactionOrigin (enum.Enum)
attribute sqlalchemy.orm.SessionTransactionOrigin.AUTOBEGIN = 0
transaction were started by autobegin
attribute sqlalchemy.orm.SessionTransactionOrigin.BEGIN = 1
transaction were started by calling Session.begin()
attribute sqlalchemy.orm.SessionTransactionOrigin.BEGIN_NESTED = 2
tranaction were started by Session.begin_nested()
attribute sqlalchemy.orm.SessionTransactionOrigin.SUBTRANSACTION = 3
transaction is an internal “subtransaction”
Session Utilities
Object Name
Description
close_all_sessions()
Close all sessions in memory.
make_transient(instance)
Alter the state of the given instance so that it is transient.
make_transient_to_detached(instance)
Make the given transient instance detached.
object_session(instance)
Return the Session to which the given instance belongs.
was_deleted(object_)
Return True if the given object was deleted within a session flush.
function sqlalchemy.orm.close_all_sessions() ? None
Close all sessions in memory.
This function consults a global registry of all Session objects and calls Session.close() on them, which resets them to a clean state.
This function is not for general use but may be useful for test suites within the teardown scheme.
Added in version 1.3.
function sqlalchemy.orm.make_transient(instance: object) ? None
Alter the state of the given instance so that it is transient.
Note
make_transient() is a special-case function for advanced use cases only.
The given mapped instance is assumed to be in the persistent or detached state. The function will remove its association with any Session as well as its InstanceState.identity. The effect is that the object will behave as though it were newly constructed, except retaining any attribute / collection values that were loaded at the time of the call. The InstanceState.deleted flag is also reset if this object had been deleted as a result of using Session.delete().
Warning
make_transient() does not “unexpire” or otherwise eagerly load ORM-mapped attributes that are not currently loaded at the time the function is called. This includes attributes which:
* were expired via Session.expire()
* were expired as the natural effect of committing a session transaction, e.g. Session.commit()
* are normally lazy loaded but are not currently loaded
* are “deferred” (see Limiting which Columns Load with Column Deferral) and are not yet loaded
* were not present in the query which loaded this object, such as that which is common in joined table inheritance and other scenarios.
After make_transient() is called, unloaded attributes such as those above will normally resolve to the value None when accessed, or an empty collection for a collection-oriented attribute. As the object is transient and un-associated with any database identity, it will no longer retrieve these values.
See also
make_transient_to_detached()
function sqlalchemy.orm.make_transient_to_detached(instance: object) ? None
Make the given transient instance detached.
Note
make_transient_to_detached() is a special-case function for advanced use cases only.
All attribute history on the given instance will be reset as though the instance were freshly loaded from a query. Missing attributes will be marked as expired. The primary key attributes of the object, which are required, will be made into the “key” of the instance.
The object can then be added to a session, or merged possibly with the load=False flag, at which point it will look as if it were loaded that way, without emitting SQL.
This is a special use case function that differs from a normal call to Session.merge() in that a given persistent state can be manufactured without any SQL calls.
See also
make_transient()
Session.enable_relationship_loading()
function sqlalchemy.orm.object_session(instance: object) ? Session | None
Return the Session to which the given instance belongs.
This is essentially the same as the InstanceState.session accessor. See that attribute for details.
function sqlalchemy.orm.util.was_deleted(object_: object) ? bool
Return True if the given object was deleted within a session flush.
This is regardless of whether or not the object is persistent or detached.
See also
InstanceState.was_deleted
Attribute and State Management Utilities
These functions are provided by the SQLAlchemy attribute instrumentation API to provide a detailed interface for dealing with instances, attribute values, and history. Some of them are useful when constructing event listener functions, such as those described in ORM Events.
Object Name
Description
del_attribute(instance, key)
Delete the value of an attribute, firing history events.
flag_dirty(instance)
Mark an instance as ‘dirty’ without any specific attribute mentioned.
flag_modified(instance, key)
Mark an attribute on an instance as ‘modified’.
get_attribute(instance, key)
Get the value of an attribute, firing any callables required.
get_history(obj, key[, passive])
Return a History record for the given object and attribute key.
History
A 3-tuple of added, unchanged and deleted values, representing the changes which have occurred on an instrumented attribute.
init_collection(obj, key)
Initialize a collection attribute and return the collection adapter.
instance_state
Return the InstanceState for a given mapped object.
is_instrumented(instance, key)
Return True if the given attribute on the given instance is instrumented by the attributes package.
object_state(instance)
Given an object, return the InstanceState associated with the object.
set_attribute(instance, key, value[, initiator])
Set the value of an attribute, firing history events.
set_committed_value(instance, key, value)
Set the value of an attribute with no history events.
function sqlalchemy.orm.util.object_state(instance: _T) ? InstanceState[_T]
Given an object, return the InstanceState associated with the object.
Raises sqlalchemy.orm.exc.UnmappedInstanceError if no mapping is configured.
Equivalent functionality is available via the inspect() function as:
inspect(instance)
Using the inspection system will raise sqlalchemy.exc.NoInspectionAvailable if the instance is not part of a mapping.
function sqlalchemy.orm.attributes.del_attribute(instance: object, key: str) ? None
Delete the value of an attribute, firing history events.
This function may be used regardless of instrumentation applied directly to the class, i.e. no descriptors are required. Custom attribute management schemes will need to make usage of this method to establish attribute state as understood by SQLAlchemy.
function sqlalchemy.orm.attributes.get_attribute(instance: object, key: str) ? Any
Get the value of an attribute, firing any callables required.
This function may be used regardless of instrumentation applied directly to the class, i.e. no descriptors are required. Custom attribute management schemes will need to make usage of this method to make usage of attribute state as understood by SQLAlchemy.
function sqlalchemy.orm.attributes.get_history(obj: object, key: str, passive: PassiveFlag = symbol('PASSIVE_OFF')) ? History
Return a History record for the given object and attribute key.
This is the pre-flush history for a given attribute, which is reset each time the Session flushes changes to the current database transaction.
Note
Prefer to use the AttributeState.history and AttributeState.load_history() accessors to retrieve the History for instance attributes.
Parameters:
* obj – an object whose class is instrumented by the attributes package.
* key – string attribute name.
* passive – indicates loading behavior for the attribute if the value is not already present. This is a bitflag attribute, which defaults to the symbol PASSIVE_OFF indicating all necessary SQL should be emitted.
See also
AttributeState.history
AttributeState.load_history() - retrieve history using loader callables if the value is not locally present.
function sqlalchemy.orm.attributes.init_collection(obj: object, key: str) ? CollectionAdapter
Initialize a collection attribute and return the collection adapter.
This function is used to provide direct access to collection internals for a previously unloaded attribute. e.g.:
collection_adapter = init_collection(someobject, "elements")
for elem in values:
    collection_adapter.append_without_event(elem)
For an easier way to do the above, see set_committed_value().
Parameters:
* obj – a mapped object
* key – string attribute name where the collection is located.
function sqlalchemy.orm.attributes.flag_modified(instance: object, key: str) ? None
Mark an attribute on an instance as ‘modified’.
This sets the ‘modified’ flag on the instance and establishes an unconditional change event for the given attribute. The attribute must have a value present, else an InvalidRequestError is raised.
To mark an object “dirty” without referring to any specific attribute so that it is considered within a flush, use the flag_dirty() call.
See also
flag_dirty()
function sqlalchemy.orm.attributes.flag_dirty(instance: object) ? None
Mark an instance as ‘dirty’ without any specific attribute mentioned.
This is a special operation that will allow the object to travel through the flush process for interception by events such as SessionEvents.before_flush(). Note that no SQL will be emitted in the flush process for an object that has no changes, even if marked dirty via this method. However, a SessionEvents.before_flush() handler will be able to see the object in the Session.dirty collection and may establish changes on it, which will then be included in the SQL emitted.
Added in version 1.2.
See also
flag_modified()
function sqlalchemy.orm.attributes.instance_state()
Return the InstanceState for a given mapped object.
This function is the internal version of object_state(). The object_state() and/or the inspect() function is preferred here as they each emit an informative exception if the given object is not mapped.
function sqlalchemy.orm.instrumentation.is_instrumented(instance, key)
Return True if the given attribute on the given instance is instrumented by the attributes package.
This function may be used regardless of instrumentation applied directly to the class, i.e. no descriptors are required.
function sqlalchemy.orm.attributes.set_attribute(instance: object, key: str, value: Any, initiator: AttributeEventToken | None = None) ? None
Set the value of an attribute, firing history events.
This function may be used regardless of instrumentation applied directly to the class, i.e. no descriptors are required. Custom attribute management schemes will need to make usage of this method to establish attribute state as understood by SQLAlchemy.
Parameters:
* instance – the object that will be modified
* key – string name of the attribute
* value – value to assign
* initiator – 
an instance of Event that would have been propagated from a previous event listener. This argument is used when the set_attribute() function is being used within an existing event listening function where an Event object is being supplied; the object may be used to track the origin of the chain of events.
Added in version 1.2.3.
function sqlalchemy.orm.attributes.set_committed_value(instance: object, key: str, value: Any) ? None
Set the value of an attribute with no history events.
Cancels any previous history present. The value should be a scalar value for scalar-holding attributes, or an iterable for any collection-holding attribute.
This is the same underlying method used when a lazy loader fires off and loads additional data from the database. In particular, this method can be used by application code which has loaded additional attributes or collections through separate queries, which can then be attached to an instance as though it were part of its original loaded state.
class sqlalchemy.orm.attributes.History
A 3-tuple of added, unchanged and deleted values, representing the changes which have occurred on an instrumented attribute.
The easiest way to get a History object for a particular attribute on an object is to use the inspect() function:
from sqlalchemy import inspect

hist = inspect(myobject).attrs.myattribute.history
Each tuple member is an iterable sequence:
* added - the collection of items added to the attribute (the first tuple element).
* unchanged - the collection of items that have not changed on the attribute (the second tuple element).
* deleted - the collection of items that have been removed from the attribute (the third tuple element).
Members
added, deleted, empty(), has_changes(), non_added(), non_deleted(), sum(), unchanged
Class signature
class sqlalchemy.orm.History (builtins.tuple)
attribute sqlalchemy.orm.attributes.History.added: Tuple | List[Any]
Alias for field number 0
attribute sqlalchemy.orm.attributes.History.deleted: Tuple | List[Any]
Alias for field number 2
method sqlalchemy.orm.attributes.History.empty() ? bool
Return True if this History has no changes and no existing, unchanged state.
method sqlalchemy.orm.attributes.History.has_changes() ? bool
Return True if this History has changes.
method sqlalchemy.orm.attributes.History.non_added() ? Sequence[Any]
Return a collection of unchanged + deleted.
method sqlalchemy.orm.attributes.History.non_deleted() ? Sequence[Any]
Return a collection of added + unchanged.
method sqlalchemy.orm.attributes.History.sum() ? Sequence[Any]
Return a collection of added + unchanged + deleted.
attribute sqlalchemy.orm.attributes.History.unchanged: Tuple | List[Any]
Alias for field number 1


Events and Internals
The SQLAlchemy ORM as well as Core are extended generally through the use of event hooks. Be sure to review the use of the event system in general at Events.
* ORM Events
o Session Events
o Mapper Events
o Instance Events
o Attribute Events
o Query Events
o Instrumentation Events
* ORM Internals
o AttributeState
o CascadeOptions
o ClassManager
o ColumnProperty
o Composite
o CompositeProperty
o AttributeEventToken
o IdentityMap
o InspectionAttr
o InspectionAttrInfo
o InstanceState
o InstrumentedAttribute
o LoaderCallableStatus
o Mapped
o MappedColumn
o MapperProperty
o MappedSQLExpression
o InspectionAttrExtensionType
o NotExtension
o merge_result()
o merge_frozen_result()
o PropComparator
o Relationship
o RelationshipDirection
o RelationshipProperty
o SQLORMExpression
o Synonym
o SynonymProperty
o QueryContext
o QueryableAttribute
o UOWTransaction
* ORM Exceptions
o ConcurrentModificationError
o DetachedInstanceError
o FlushError
o LoaderStrategyException
o MappedAnnotationError
o NO_STATE
o ObjectDeletedError
o ObjectDereferencedError
o StaleDataError
o UnmappedClassError
o UnmappedColumnError
o UnmappedError
o UnmappedInstanceError


ORM Events
The ORM includes a wide variety of hooks available for subscription.
For an introduction to the most commonly used ORM events, see the section Tracking queries, object and Session Changes with Events. The event system in general is discussed at Events. Non-ORM events such as those regarding connections and low-level statement execution are described in Core Events.
Session Events
The most basic event hooks are available at the level of the ORM Session object. The types of things that are intercepted here include:
* Persistence Operations - the ORM flush process that sends changes to the database can be extended using events that fire off at different parts of the flush, to augment or modify the data being sent to the database or to allow other things to happen when persistence occurs. Read more about persistence events at Persistence Events.
* Object lifecycle events - hooks when objects are added, persisted, deleted from sessions. Read more about these at Object Lifecycle Events.
* Execution Events - Part of the 2.0 style execution model, all SELECT statements against ORM entities emitted, as well as bulk UPDATE and DELETE statements outside of the flush process, are intercepted from the Session.execute() method using the SessionEvents.do_orm_execute() method. Read more about this event at Execute Events.
Be sure to read the Tracking queries, object and Session Changes with Events chapter for context on these events.
Object Name
Description
SessionEvents
Define events specific to Session lifecycle.
class sqlalchemy.orm.SessionEvents
Define events specific to Session lifecycle.
e.g.:
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker


def my_before_commit(session):
    print("before commit!")


Session = sessionmaker()

event.listen(Session, "before_commit", my_before_commit)
The listen() function will accept Session objects as well as the return result of sessionmaker() and scoped_session().
Additionally, it accepts the Session class which will apply listeners to all Session instances globally.
Parameters:
* raw=False – 
When True, the “target” argument passed to applicable event listener functions that work on individual objects will be the instance’s InstanceState management object, rather than the mapped instance itself.
Added in version 1.3.14.
* restore_load_context=False – 
Applies to the SessionEvents.loaded_as_persistent() event. Restores the loader context of the object when the event hook is complete, so that ongoing eager load operations continue to target the object appropriately. A warning is emitted if the object is moved to a new loader context from within this event if this flag is not set.
Added in version 1.3.14.
Members
after_attach(), after_begin(), after_bulk_delete(), after_bulk_update(), after_commit(), after_flush(), after_flush_postexec(), after_rollback(), after_soft_rollback(), after_transaction_create(), after_transaction_end(), before_attach(), before_commit(), before_flush(), deleted_to_detached(), deleted_to_persistent(), detached_to_persistent(), dispatch, do_orm_execute(), loaded_as_persistent(), pending_to_persistent(), pending_to_transient(), persistent_to_deleted(), persistent_to_detached(), persistent_to_transient(), transient_to_pending()
Class signature
class sqlalchemy.orm.SessionEvents (sqlalchemy.event.Events)
method sqlalchemy.orm.SessionEvents.after_attach(session: Session, instance: _O) ? None
Execute after an instance is attached to a session.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_attach')
def receive_after_attach(session, instance):
    "listen for the 'after_attach' event"

    # (event handling logic) 
This is called after an add, delete or merge.
Note
As of 0.8, this event fires off after the item has been fully associated with the session, which is different than previous releases. For event handlers that require the object not yet be part of session state (such as handlers which may autoflush while the target object is not yet complete) consider the new before_attach() event.
See also
SessionEvents.before_attach()
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.after_begin(session: Session, transaction: SessionTransaction, connection: Connection) ? None
Execute after a transaction is begun on a connection.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_begin')
def receive_after_begin(session, transaction, connection):
    "listen for the 'after_begin' event"

    # (event handling logic) 
Note
This event is called within the process of the Session modifying its own internal state. To invoke SQL operations within this hook, use the Connection provided to the event; do not run SQL operations using the Session directly.
Parameters:
* session – The target Session.
* transaction – The SessionTransaction.
* connection – The Connection object which will be used for SQL statements.
See also
SessionEvents.before_commit()
SessionEvents.after_commit()
SessionEvents.after_transaction_create()
SessionEvents.after_transaction_end()
method sqlalchemy.orm.SessionEvents.after_bulk_delete(delete_context: _O) ? None
Event for after the legacy Query.delete() method has been called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(delete_context):
    "listen for the 'after_bulk_delete' event"

    # (event handling logic) 

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(session, query, query_context, result):
    "listen for the 'after_bulk_delete' event"

    # (event handling logic) 
Changed in version 0.9: The SessionEvents.after_bulk_delete() event now accepts the arguments SessionEvents.after_bulk_delete.delete_context. Support for listener functions which accept the previous argument signature(s) listed above as “deprecated” will be removed in a future release.
Legacy Feature
The SessionEvents.after_bulk_delete() method is a legacy event hook as of SQLAlchemy 2.0. The event does not participate in 2.0 style invocations using delete() documented at ORM UPDATE and DELETE with Custom WHERE Criteria. For 2.0 style use, the SessionEvents.do_orm_execute() hook will intercept these calls.
Parameters:
delete_context – 
a “delete context” object which contains details about the update, including these attributes:
* session - the Session involved
* query -the Query object that this update operation was called upon.
* result the CursorResult returned as a result of the bulk DELETE operation.
Changed in version 1.4: the update_context no longer has a QueryContext object associated with it.
See also
QueryEvents.before_compile_delete()
SessionEvents.after_bulk_update()
method sqlalchemy.orm.SessionEvents.after_bulk_update(update_context: _O) ? None
Event for after the legacy Query.update() method has been called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(update_context):
    "listen for the 'after_bulk_update' event"

    # (event handling logic) 

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(session, query, query_context, result):
    "listen for the 'after_bulk_update' event"

    # (event handling logic) 
Changed in version 0.9: The SessionEvents.after_bulk_update() event now accepts the arguments SessionEvents.after_bulk_update.update_context. Support for listener functions which accept the previous argument signature(s) listed above as “deprecated” will be removed in a future release.
Legacy Feature
The SessionEvents.after_bulk_update() method is a legacy event hook as of SQLAlchemy 2.0. The event does not participate in 2.0 style invocations using update() documented at ORM UPDATE and DELETE with Custom WHERE Criteria. For 2.0 style use, the SessionEvents.do_orm_execute() hook will intercept these calls.
Parameters:
update_context – 
an “update context” object which contains details about the update, including these attributes:
* session - the Session involved
* query -the Query object that this update operation was called upon.
* values The “values” dictionary that was passed to Query.update().
* result the CursorResult returned as a result of the bulk UPDATE operation.
Changed in version 1.4: the update_context no longer has a QueryContext object associated with it.
See also
QueryEvents.before_compile_update()
SessionEvents.after_bulk_delete()
method sqlalchemy.orm.SessionEvents.after_commit(session: Session) ? None
Execute after a commit has occurred.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_commit')
def receive_after_commit(session):
    "listen for the 'after_commit' event"

    # (event handling logic) 
Note
The SessionEvents.after_commit() hook is not per-flush, that is, the Session can emit SQL to the database many times within the scope of a transaction. For interception of these events, use the SessionEvents.before_flush(), SessionEvents.after_flush(), or SessionEvents.after_flush_postexec() events.
Note
The Session is not in an active transaction when the SessionEvents.after_commit() event is invoked, and therefore can not emit SQL. To emit SQL corresponding to every transaction, use the SessionEvents.before_commit() event.
Parameters:
session – The target Session.
See also
SessionEvents.before_commit()
SessionEvents.after_begin()
SessionEvents.after_transaction_create()
SessionEvents.after_transaction_end()
method sqlalchemy.orm.SessionEvents.after_flush(session: Session, flush_context: UOWTransaction) ? None
Execute after flush has completed, but before commit has been called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_flush')
def receive_after_flush(session, flush_context):
    "listen for the 'after_flush' event"

    # (event handling logic) 
Note that the session’s state is still in pre-flush, i.e. ‘new’, ‘dirty’, and ‘deleted’ lists still show pre-flush state as well as the history settings on instance attributes.
Warning
This event runs after the Session has emitted SQL to modify the database, but before it has altered its internal state to reflect those changes, including that newly inserted objects are placed into the identity map. ORM operations emitted within this event such as loads of related items may produce new identity map entries that will immediately be replaced, sometimes causing confusing results. SQLAlchemy will emit a warning for this condition as of version 1.3.9.
Parameters:
* session – The target Session.
* flush_context – Internal UOWTransaction object which handles the details of the flush.
See also
SessionEvents.before_flush()
SessionEvents.after_flush_postexec()
Persistence Events
method sqlalchemy.orm.SessionEvents.after_flush_postexec(session: Session, flush_context: UOWTransaction) ? None
Execute after flush has completed, and after the post-exec state occurs.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_flush_postexec')
def receive_after_flush_postexec(session, flush_context):
    "listen for the 'after_flush_postexec' event"

    # (event handling logic) 
This will be when the ‘new’, ‘dirty’, and ‘deleted’ lists are in their final state. An actual commit() may or may not have occurred, depending on whether or not the flush started its own transaction or participated in a larger transaction.
Parameters:
* session – The target Session.
* flush_context – Internal UOWTransaction object which handles the details of the flush.
See also
SessionEvents.before_flush()
SessionEvents.after_flush()
Persistence Events
method sqlalchemy.orm.SessionEvents.after_rollback(session: Session) ? None
Execute after a real DBAPI rollback has occurred.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_rollback')
def receive_after_rollback(session):
    "listen for the 'after_rollback' event"

    # (event handling logic) 
Note that this event only fires when the actual rollback against the database occurs - it does not fire each time the Session.rollback() method is called, if the underlying DBAPI transaction has already been rolled back. In many cases, the Session will not be in an “active” state during this event, as the current transaction is not valid. To acquire a Session which is active after the outermost rollback has proceeded, use the SessionEvents.after_soft_rollback() event, checking the Session.is_active flag.
Parameters:
session – The target Session.
method sqlalchemy.orm.SessionEvents.after_soft_rollback(session: Session, previous_transaction: SessionTransaction) ? None
Execute after any rollback has occurred, including “soft” rollbacks that don’t actually emit at the DBAPI level.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_soft_rollback')
def receive_after_soft_rollback(session, previous_transaction):
    "listen for the 'after_soft_rollback' event"

    # (event handling logic) 
This corresponds to both nested and outer rollbacks, i.e. the innermost rollback that calls the DBAPI’s rollback() method, as well as the enclosing rollback calls that only pop themselves from the transaction stack.
The given Session can be used to invoke SQL and Session.query() operations after an outermost rollback by first checking the Session.is_active flag:
@event.listens_for(Session, "after_soft_rollback")
def do_something(session, previous_transaction):
    if session.is_active:
        session.execute(text("select * from some_table"))
Parameters:
* session – The target Session.
* previous_transaction – The SessionTransaction transactional marker object which was just closed. The current SessionTransaction for the given Session is available via the Session.transaction attribute.
method sqlalchemy.orm.SessionEvents.after_transaction_create(session: Session, transaction: SessionTransaction) ? None
Execute when a new SessionTransaction is created.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_transaction_create')
def receive_after_transaction_create(session, transaction):
    "listen for the 'after_transaction_create' event"

    # (event handling logic) 
This event differs from SessionEvents.after_begin() in that it occurs for each SessionTransaction overall, as opposed to when transactions are begun on individual database connections. It is also invoked for nested transactions and subtransactions, and is always matched by a corresponding SessionEvents.after_transaction_end() event (assuming normal operation of the Session).
Parameters:
* session – the target Session.
* transaction – 
the target SessionTransaction.
To detect if this is the outermost SessionTransaction, as opposed to a “subtransaction” or a SAVEPOINT, test that the SessionTransaction.parent attribute is None:
@event.listens_for(session, "after_transaction_create")
def after_transaction_create(session, transaction):
    if transaction.parent is None:
         # work with top-level transaction
To detect if the SessionTransaction is a SAVEPOINT, use the SessionTransaction.nested attribute:
@event.listens_for(session, "after_transaction_create")
def after_transaction_create(session, transaction):
    if transaction.nested:
         # work with SAVEPOINT transaction
* 
See also
SessionTransaction
SessionEvents.after_transaction_end()
method sqlalchemy.orm.SessionEvents.after_transaction_end(session: Session, transaction: SessionTransaction) ? None
Execute when the span of a SessionTransaction ends.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'after_transaction_end')
def receive_after_transaction_end(session, transaction):
    "listen for the 'after_transaction_end' event"

    # (event handling logic) 
This event differs from SessionEvents.after_commit() in that it corresponds to all SessionTransaction objects in use, including those for nested transactions and subtransactions, and is always matched by a corresponding SessionEvents.after_transaction_create() event.
Parameters:
* session – the target Session.
* transaction – 
the target SessionTransaction.
To detect if this is the outermost SessionTransaction, as opposed to a “subtransaction” or a SAVEPOINT, test that the SessionTransaction.parent attribute is None:
@event.listens_for(session, "after_transaction_create")
def after_transaction_end(session, transaction):
    if transaction.parent is None:
         # work with top-level transaction
To detect if the SessionTransaction is a SAVEPOINT, use the SessionTransaction.nested attribute:
@event.listens_for(session, "after_transaction_create")
def after_transaction_end(session, transaction):
    if transaction.nested:
         # work with SAVEPOINT transaction
* 
See also
SessionTransaction
SessionEvents.after_transaction_create()
method sqlalchemy.orm.SessionEvents.before_attach(session: Session, instance: _O) ? None
Execute before an instance is attached to a session.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'before_attach')
def receive_before_attach(session, instance):
    "listen for the 'before_attach' event"

    # (event handling logic) 
This is called before an add, delete or merge causes the object to be part of the session.
See also
SessionEvents.after_attach()
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.before_commit(session: Session) ? None
Execute before commit is called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'before_commit')
def receive_before_commit(session):
    "listen for the 'before_commit' event"

    # (event handling logic) 
Note
The SessionEvents.before_commit() hook is not per-flush, that is, the Session can emit SQL to the database many times within the scope of a transaction. For interception of these events, use the SessionEvents.before_flush(), SessionEvents.after_flush(), or SessionEvents.after_flush_postexec() events.
Parameters:
session – The target Session.
See also
SessionEvents.after_commit()
SessionEvents.after_begin()
SessionEvents.after_transaction_create()
SessionEvents.after_transaction_end()
method sqlalchemy.orm.SessionEvents.before_flush(session: Session, flush_context: UOWTransaction, instances: Sequence[_O] | None) ? None
Execute before flush process has started.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'before_flush')
def receive_before_flush(session, flush_context, instances):
    "listen for the 'before_flush' event"

    # (event handling logic) 
Parameters:
* session – The target Session.
* flush_context – Internal UOWTransaction object which handles the details of the flush.
* instances – Usually None, this is the collection of objects which can be passed to the Session.flush() method (note this usage is deprecated).
See also
SessionEvents.after_flush()
SessionEvents.after_flush_postexec()
Persistence Events
method sqlalchemy.orm.SessionEvents.deleted_to_detached(session: Session, instance: _O) ? None
Intercept the “deleted to detached” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'deleted_to_detached')
def receive_deleted_to_detached(session, instance):
    "listen for the 'deleted_to_detached' event"

    # (event handling logic) 
This event is invoked when a deleted object is evicted from the session. The typical case when this occurs is when the transaction for a Session in which the object was deleted is committed; the object moves from the deleted state to the detached state.
It is also invoked for objects that were deleted in a flush when the Session.expunge_all() or Session.close() events are called, as well as if the object is individually expunged from its deleted state via Session.expunge().
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.deleted_to_persistent(session: Session, instance: _O) ? None
Intercept the “deleted to persistent” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'deleted_to_persistent')
def receive_deleted_to_persistent(session, instance):
    "listen for the 'deleted_to_persistent' event"

    # (event handling logic) 
This transition occurs only when an object that’s been deleted successfully in a flush is restored due to a call to Session.rollback(). The event is not called under any other circumstances.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.detached_to_persistent(session: Session, instance: _O) ? None
Intercept the “detached to persistent” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'detached_to_persistent')
def receive_detached_to_persistent(session, instance):
    "listen for the 'detached_to_persistent' event"

    # (event handling logic) 
This event is a specialization of the SessionEvents.after_attach() event which is only invoked for this specific transition. It is invoked typically during the Session.add() call, as well as during the Session.delete() call if the object was not previously associated with the Session (note that an object marked as “deleted” remains in the “persistent” state until the flush proceeds).
Note
If the object becomes persistent as part of a call to Session.delete(), the object is not yet marked as deleted when this event is called. To detect deleted objects, check the deleted flag sent to the SessionEvents.persistent_to_detached() to event after the flush proceeds, or check the Session.deleted collection within the SessionEvents.before_flush() event if deleted objects need to be intercepted before the flush.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
attribute sqlalchemy.orm.SessionEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.SessionEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events
method sqlalchemy.orm.SessionEvents.do_orm_execute(orm_execute_state: ORMExecuteState) ? None
Intercept statement executions that occur on behalf of an ORM Session object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'do_orm_execute')
def receive_do_orm_execute(orm_execute_state):
    "listen for the 'do_orm_execute' event"

    # (event handling logic) 
This event is invoked for all top-level SQL statements invoked from the Session.execute() method, as well as related methods such as Session.scalars() and Session.scalar(). As of SQLAlchemy 1.4, all ORM queries that run through the Session.execute() method as well as related methods Session.scalars(), Session.scalar() etc. will participate in this event. This event hook does not apply to the queries that are emitted internally within the ORM flush process, i.e. the process described at Flushing.
Note
The SessionEvents.do_orm_execute() event hook is triggered for ORM statement executions only, meaning those invoked via the Session.execute() and similar methods on the Session object. It does not trigger for statements that are invoked by SQLAlchemy Core only, i.e. statements invoked directly using Connection.execute() or otherwise originating from an Engine object without any Session involved. To intercept all SQL executions regardless of whether the Core or ORM APIs are in use, see the event hooks at ConnectionEvents, such as ConnectionEvents.before_execute() and ConnectionEvents.before_cursor_execute().
Also, this event hook does not apply to queries that are emitted internally within the ORM flush process, i.e. the process described at Flushing; to intercept steps within the flush process, see the event hooks described at Persistence Events as well as Mapper-level Flush Events.
This event is a do_ event, meaning it has the capability to replace the operation that the Session.execute() method normally performs. The intended use for this includes sharding and result-caching schemes which may seek to invoke the same statement across multiple database connections, returning a result that is merged from each of them, or which don’t invoke the statement at all, instead returning data from a cache.
The hook intends to replace the use of the Query._execute_and_instances method that could be subclassed prior to SQLAlchemy 1.4.
Parameters:
orm_execute_state – an instance of ORMExecuteState which contains all information about the current execution, as well as helper functions used to derive other commonly required information. See that object for details.
See also
Execute Events - top level documentation on how to use SessionEvents.do_orm_execute()
ORMExecuteState - the object passed to the SessionEvents.do_orm_execute() event which contains all information about the statement to be invoked. It also provides an interface to extend the current statement, options, and parameters as well as an option that allows programmatic invocation of the statement at any point.
ORM Query Events - includes examples of using SessionEvents.do_orm_execute()
Dogpile Caching - an example of how to integrate Dogpile caching with the ORM Session making use of the SessionEvents.do_orm_execute() event hook.
Horizontal Sharding - the Horizontal Sharding example / extension relies upon the SessionEvents.do_orm_execute() event hook to invoke a SQL statement on multiple backends and return a merged result.
Added in version 1.4.
method sqlalchemy.orm.SessionEvents.loaded_as_persistent(session: Session, instance: _O) ? None
Intercept the “loaded as persistent” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'loaded_as_persistent')
def receive_loaded_as_persistent(session, instance):
    "listen for the 'loaded_as_persistent' event"

    # (event handling logic) 
This event is invoked within the ORM loading process, and is invoked very similarly to the InstanceEvents.load() event. However, the event here is linkable to a Session class or instance, rather than to a mapper or class hierarchy, and integrates with the other session lifecycle events smoothly. The object is guaranteed to be present in the session’s identity map when this event is called.
Note
This event is invoked within the loader process before eager loaders may have been completed, and the object’s state may not be complete. Additionally, invoking row-level refresh operations on the object will place the object into a new loader context, interfering with the existing load context. See the note on InstanceEvents.load() for background on making use of the SessionEvents.restore_load_context parameter, which works in the same manner as that of InstanceEvents.restore_load_context, in order to resolve this scenario.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.pending_to_persistent(session: Session, instance: _O) ? None
Intercept the “pending to persistent”” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'pending_to_persistent')
def receive_pending_to_persistent(session, instance):
    "listen for the 'pending_to_persistent' event"

    # (event handling logic) 
This event is invoked within the flush process, and is similar to scanning the Session.new collection within the SessionEvents.after_flush() event. However, in this case the object has already been moved to the persistent state when the event is called.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.pending_to_transient(session: Session, instance: _O) ? None
Intercept the “pending to transient” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'pending_to_transient')
def receive_pending_to_transient(session, instance):
    "listen for the 'pending_to_transient' event"

    # (event handling logic) 
This less common transition occurs when an pending object that has not been flushed is evicted from the session; this can occur when the Session.rollback() method rolls back the transaction, or when the Session.expunge() method is used.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.persistent_to_deleted(session: Session, instance: _O) ? None
Intercept the “persistent to deleted” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'persistent_to_deleted')
def receive_persistent_to_deleted(session, instance):
    "listen for the 'persistent_to_deleted' event"

    # (event handling logic) 
This event is invoked when a persistent object’s identity is deleted from the database within a flush, however the object still remains associated with the Session until the transaction completes.
If the transaction is rolled back, the object moves again to the persistent state, and the SessionEvents.deleted_to_persistent() event is called. If the transaction is committed, the object becomes detached, which will emit the SessionEvents.deleted_to_detached() event.
Note that while the Session.delete() method is the primary public interface to mark an object as deleted, many objects get deleted due to cascade rules, which are not always determined until flush time. Therefore, there’s no way to catch every object that will be deleted until the flush has proceeded. the SessionEvents.persistent_to_deleted() event is therefore invoked at the end of a flush.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.persistent_to_detached(session: Session, instance: _O) ? None
Intercept the “persistent to detached” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'persistent_to_detached')
def receive_persistent_to_detached(session, instance):
    "listen for the 'persistent_to_detached' event"

    # (event handling logic) 
This event is invoked when a persistent object is evicted from the session. There are many conditions that cause this to happen, including:
* using a method such as Session.expunge() or Session.close()
* Calling the Session.rollback() method, when the object was part of an INSERT statement for that session’s transaction
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
* deleted – boolean. If True, indicates this object moved to the detached state because it was marked as deleted and flushed.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.persistent_to_transient(session: Session, instance: _O) ? None
Intercept the “persistent to transient” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'persistent_to_transient')
def receive_persistent_to_transient(session, instance):
    "listen for the 'persistent_to_transient' event"

    # (event handling logic) 
This less common transition occurs when an pending object that has has been flushed is evicted from the session; this can occur when the Session.rollback() method rolls back the transaction.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
method sqlalchemy.orm.SessionEvents.transient_to_pending(session: Session, instance: _O) ? None
Intercept the “transient to pending” transition for a specific object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeSessionClassOrObject, 'transient_to_pending')
def receive_transient_to_pending(session, instance):
    "listen for the 'transient_to_pending' event"

    # (event handling logic) 
This event is a specialization of the SessionEvents.after_attach() event which is only invoked for this specific transition. It is invoked typically during the Session.add() call.
Parameters:
* session – target Session
* instance – the ORM-mapped instance being operated upon.
See also
Object Lifecycle Events
Mapper Events
Mapper event hooks encompass things that happen as related to individual or multiple Mapper objects, which are the central configurational object that maps a user-defined class to a Table object. Types of things which occur at the Mapper level include:
* Per-object persistence operations - the most popular mapper hooks are the unit-of-work hooks such as MapperEvents.before_insert(), MapperEvents.after_update(), etc. These events are contrasted to the more coarse grained session-level events such as SessionEvents.before_flush() in that they occur within the flush process on a per-object basis; while finer grained activity on an object is more straightforward, availability of Session features is limited.
* Mapper configuration events - the other major class of mapper hooks are those which occur as a class is mapped, as a mapper is finalized, and when sets of mappers are configured to refer to each other. These events include MapperEvents.instrument_class(), MapperEvents.before_mapper_configured() and MapperEvents.mapper_configured() at the individual Mapper level, and MapperEvents.before_configured() and MapperEvents.after_configured() at the level of collections of Mapper objects.
Object Name
Description
MapperEvents
Define events specific to mappings.
class sqlalchemy.orm.MapperEvents
Define events specific to mappings.
e.g.:
from sqlalchemy import event


def my_before_insert_listener(mapper, connection, target):
    # execute a stored procedure upon INSERT,
    # apply the value to the row to be inserted
    target.calculated_value = connection.execute(
        text("select my_special_function(%d)" % target.special_number)
    ).scalar()


# associate the listener function with SomeClass,
# to execute during the "before_insert" hook
event.listen(SomeClass, "before_insert", my_before_insert_listener)
Available targets include:
* mapped classes
* unmapped superclasses of mapped or to-be-mapped classes (using the propagate=True flag)
* Mapper objects
* the Mapper class itself indicates listening for all mappers.
Mapper events provide hooks into critical sections of the mapper, including those related to object instrumentation, object loading, and object persistence. In particular, the persistence methods MapperEvents.before_insert(), and MapperEvents.before_update() are popular places to augment the state being persisted - however, these methods operate with several significant restrictions. The user is encouraged to evaluate the SessionEvents.before_flush() and SessionEvents.after_flush() methods as more flexible and user-friendly hooks in which to apply additional database state during a flush.
When using MapperEvents, several modifiers are available to the listen() function.
Parameters:
* propagate=False – When True, the event listener should be applied to all inheriting mappers and/or the mappers of inheriting classes, as well as any mapper which is the target of this listener.
* raw=False – When True, the “target” argument passed to applicable event listener functions will be the instance’s InstanceState management object, rather than the mapped instance itself.
* retval=False – 
when True, the user-defined event function must have a return value, the purpose of which is either to control subsequent event propagation, or to otherwise alter the operation in progress by the mapper. Possible return values are:
o sqlalchemy.orm.interfaces.EXT_CONTINUE - continue event processing normally.
o sqlalchemy.orm.interfaces.EXT_STOP - cancel all subsequent event handlers in the chain.
o other values - the return value specified by specific listeners.
Members
after_configured(), after_delete(), after_insert(), after_mapper_constructed(), after_update(), before_configured(), before_delete(), before_insert(), before_mapper_configured(), before_update(), dispatch, instrument_class(), mapper_configured()
Class signature
class sqlalchemy.orm.MapperEvents (sqlalchemy.event.Events)
method sqlalchemy.orm.MapperEvents.after_configured() ? None
Called after a series of mappers have been configured.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'after_configured')
def receive_after_configured():
    "listen for the 'after_configured' event"

    # (event handling logic) 
The MapperEvents.after_configured() event is invoked each time the configure_mappers() function is invoked, after the function has completed its work. configure_mappers() is typically invoked automatically as mappings are first used, as well as each time new mappers have been made available and new mapper use is detected.
Contrast this event to the MapperEvents.mapper_configured() event, which is called on a per-mapper basis while the configuration operation proceeds; unlike that event, when this event is invoked, all cross-configurations (e.g. backrefs) will also have been made available for any mappers that were pending. Also contrast to MapperEvents.before_configured(), which is invoked before the series of mappers has been configured.
This event can only be applied to the Mapper class, and not to individual mappings or mapped classes. It is only invoked for all mappings as a whole:
from sqlalchemy.orm import Mapper


@event.listens_for(Mapper, "after_configured")
def go(): 
Theoretically this event is called once per application, but is actually called any time new mappers have been affected by a configure_mappers() call. If new mappings are constructed after existing ones have already been used, this event will likely be called again. To ensure that a particular event is only called once and no further, the once=True argument (new in 0.9.4) can be applied:
from sqlalchemy.orm import mapper


@event.listens_for(mapper, "after_configured", once=True)
def go(): 
See also
MapperEvents.before_mapper_configured()
MapperEvents.mapper_configured()
MapperEvents.before_configured()
method sqlalchemy.orm.MapperEvents.after_delete(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance after a DELETE statement has been emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'after_delete')
def receive_after_delete(mapper, connection, target):
    "listen for the 'after_delete' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to emit additional SQL statements on the given connection as well as to perform application specific bookkeeping related to a deletion event.
The event is often called for a batch of objects of the same class after their DELETE statements have been emitted at once in a previous step.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit DELETE statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being deleted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
method sqlalchemy.orm.MapperEvents.after_insert(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance after an INSERT statement is emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'after_insert')
def receive_after_insert(mapper, connection, target):
    "listen for the 'after_insert' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to modify in-Python-only state on the instance after an INSERT occurs, as well as to emit additional SQL statements on the given connection.
The event is often called for a batch of objects of the same class after their INSERT statements have been emitted at once in a previous step. In the extremely rare case that this is not desirable, the Mapper object can be configured with batch=False, which will cause batches of instances to be broken up into individual (and more poorly performing) event->persist->event steps.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit INSERT statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being persisted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
method sqlalchemy.orm.MapperEvents.after_mapper_constructed(mapper: Mapper[_O], class_: Type[_O]) ? None
Receive a class and mapper when the Mapper has been fully constructed.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'after_mapper_constructed')
def receive_after_mapper_constructed(mapper, class_):
    "listen for the 'after_mapper_constructed' event"

    # (event handling logic) 
This event is called after the initial constructor for Mapper completes. This occurs after the MapperEvents.instrument_class() event and after the Mapper has done an initial pass of its arguments to generate its collection of MapperProperty objects, which are accessible via the Mapper.get_property() method and the Mapper.iterate_properties attribute.
This event differs from the MapperEvents.before_mapper_configured() event in that it is invoked within the constructor for Mapper, rather than within the registry.configure() process. Currently, this event is the only one which is appropriate for handlers that wish to create additional mapped classes in response to the construction of this Mapper, which will be part of the same configure step when registry.configure() next runs.
Added in version 2.0.2.
See also
Versioning Objects - an example which illustrates the use of the MapperEvents.before_mapper_configured() event to create new mappers to record change-audit histories on objects.
method sqlalchemy.orm.MapperEvents.after_update(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance after an UPDATE statement is emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'after_update')
def receive_after_update(mapper, connection, target):
    "listen for the 'after_update' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to modify in-Python-only state on the instance after an UPDATE occurs, as well as to emit additional SQL statements on the given connection.
This method is called for all instances that are marked as “dirty”, even those which have no net changes to their column-based attributes, and for which no UPDATE statement has proceeded. An object is marked as dirty when any of its column-based attributes have a “set attribute” operation called or when any of its collections are modified. If, at update time, no column-based attributes have any net changes, no UPDATE statement will be issued. This means that an instance being sent to MapperEvents.after_update() is not a guarantee that an UPDATE statement has been issued.
To detect if the column-based attributes on the object have net changes, and therefore resulted in an UPDATE statement, use object_session(instance).is_modified(instance, include_collections=False).
The event is often called for a batch of objects of the same class after their UPDATE statements have been emitted at once in a previous step. In the extremely rare case that this is not desirable, the Mapper can be configured with batch=False, which will cause batches of instances to be broken up into individual (and more poorly performing) event->persist->event steps.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit UPDATE statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being persisted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
method sqlalchemy.orm.MapperEvents.before_configured() ? None
Called before a series of mappers have been configured.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'before_configured')
def receive_before_configured():
    "listen for the 'before_configured' event"

    # (event handling logic) 
The MapperEvents.before_configured() event is invoked each time the configure_mappers() function is invoked, before the function has done any of its work. configure_mappers() is typically invoked automatically as mappings are first used, as well as each time new mappers have been made available and new mapper use is detected.
This event can only be applied to the Mapper class, and not to individual mappings or mapped classes. It is only invoked for all mappings as a whole:
from sqlalchemy.orm import Mapper


@event.listens_for(Mapper, "before_configured")
def go(): 
Contrast this event to MapperEvents.after_configured(), which is invoked after the series of mappers has been configured, as well as MapperEvents.before_mapper_configured() and MapperEvents.mapper_configured(), which are both invoked on a per-mapper basis.
Theoretically this event is called once per application, but is actually called any time new mappers are to be affected by a configure_mappers() call. If new mappings are constructed after existing ones have already been used, this event will likely be called again. To ensure that a particular event is only called once and no further, the once=True argument (new in 0.9.4) can be applied:
from sqlalchemy.orm import mapper


@event.listens_for(mapper, "before_configured", once=True)
def go(): 
See also
MapperEvents.before_mapper_configured()
MapperEvents.mapper_configured()
MapperEvents.after_configured()
method sqlalchemy.orm.MapperEvents.before_delete(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance before a DELETE statement is emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'before_delete')
def receive_before_delete(mapper, connection, target):
    "listen for the 'before_delete' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to emit additional SQL statements on the given connection as well as to perform application specific bookkeeping related to a deletion event.
The event is often called for a batch of objects of the same class before their DELETE statements are emitted at once in a later step.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit DELETE statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being deleted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
method sqlalchemy.orm.MapperEvents.before_insert(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance before an INSERT statement is emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'before_insert')
def receive_before_insert(mapper, connection, target):
    "listen for the 'before_insert' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to modify local, non-object related attributes on the instance before an INSERT occurs, as well as to emit additional SQL statements on the given connection.
The event is often called for a batch of objects of the same class before their INSERT statements are emitted at once in a later step. In the extremely rare case that this is not desirable, the Mapper object can be configured with batch=False, which will cause batches of instances to be broken up into individual (and more poorly performing) event->persist->event steps.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit INSERT statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being persisted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
method sqlalchemy.orm.MapperEvents.before_mapper_configured(mapper: Mapper[_O], class_: Type[_O]) ? None
Called right before a specific mapper is to be configured.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'before_mapper_configured')
def receive_before_mapper_configured(mapper, class_):
    "listen for the 'before_mapper_configured' event"

    # (event handling logic) 
This event is intended to allow a specific mapper to be skipped during the configure step, by returning the interfaces.EXT_SKIP symbol which indicates to the configure_mappers() call that this particular mapper (or hierarchy of mappers, if propagate=True is used) should be skipped in the current configuration run. When one or more mappers are skipped, the “new mappers” flag will remain set, meaning the configure_mappers() function will continue to be called when mappers are used, to continue to try to configure all available mappers.
In comparison to the other configure-level events, MapperEvents.before_configured(), MapperEvents.after_configured(), and MapperEvents.mapper_configured(), the MapperEvents.before_mapper_configured() event provides for a meaningful return value when it is registered with the retval=True parameter.
Added in version 1.3.
e.g.:
from sqlalchemy.orm import EXT_SKIP

Base = declarative_base()

DontConfigureBase = declarative_base()


@event.listens_for(
    DontConfigureBase,
    "before_mapper_configured",
    retval=True,
    propagate=True,
)
def dont_configure(mapper, cls):
    return EXT_SKIP
See also
MapperEvents.before_configured()
MapperEvents.after_configured()
MapperEvents.mapper_configured()
method sqlalchemy.orm.MapperEvents.before_update(mapper: Mapper[_O], connection: Connection, target: _O) ? None
Receive an object instance before an UPDATE statement is emitted corresponding to that instance.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'before_update')
def receive_before_update(mapper, connection, target):
    "listen for the 'before_update' event"

    # (event handling logic) 
Note
this event only applies to the session flush operation and does not apply to the ORM DML operations described at ORM-Enabled INSERT, UPDATE, and DELETE statements. To intercept ORM DML events, use SessionEvents.do_orm_execute().
This event is used to modify local, non-object related attributes on the instance before an UPDATE occurs, as well as to emit additional SQL statements on the given connection.
This method is called for all instances that are marked as “dirty”, even those which have no net changes to their column-based attributes. An object is marked as dirty when any of its column-based attributes have a “set attribute” operation called or when any of its collections are modified. If, at update time, no column-based attributes have any net changes, no UPDATE statement will be issued. This means that an instance being sent to MapperEvents.before_update() is not a guarantee that an UPDATE statement will be issued, although you can affect the outcome here by modifying attributes so that a net change in value does exist.
To detect if the column-based attributes on the object have net changes, and will therefore generate an UPDATE statement, use object_session(instance).is_modified(instance, include_collections=False).
The event is often called for a batch of objects of the same class before their UPDATE statements are emitted at once in a later step. In the extremely rare case that this is not desirable, the Mapper can be configured with batch=False, which will cause batches of instances to be broken up into individual (and more poorly performing) event->persist->event steps.
Warning
Mapper-level flush events only allow very limited operations, on attributes local to the row being operated upon only, as well as allowing any SQL to be emitted on the given Connection. Please read fully the notes at Mapper-level Flush Events for guidelines on using these methods; generally, the SessionEvents.before_flush() method should be preferred for general on-flush changes.
Parameters:
* mapper – the Mapper which is the target of this event.
* connection – the Connection being used to emit UPDATE statements for this instance. This provides a handle into the current transaction on the target database specific to this instance.
* target – the mapped instance being persisted. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
Returns:
No return value is supported by this event.
See also
Persistence Events
attribute sqlalchemy.orm.MapperEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.MapperEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events
method sqlalchemy.orm.MapperEvents.instrument_class(mapper: Mapper[_O], class_: Type[_O]) ? None
Receive a class when the mapper is first constructed, before instrumentation is applied to the mapped class.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'instrument_class')
def receive_instrument_class(mapper, class_):
    "listen for the 'instrument_class' event"

    # (event handling logic) 
This event is the earliest phase of mapper construction. Most attributes of the mapper are not yet initialized. To receive an event within initial mapper construction where basic state is available such as the Mapper.attrs collection, the MapperEvents.after_mapper_constructed() event may be a better choice.
This listener can either be applied to the Mapper class overall, or to any un-mapped class which serves as a base for classes that will be mapped (using the propagate=True flag):
Base = declarative_base()


@event.listens_for(Base, "instrument_class", propagate=True)
def on_new_class(mapper, cls_):
    ""
Parameters:
* mapper – the Mapper which is the target of this event.
* class_ – the mapped class.
See also
MapperEvents.after_mapper_constructed()
method sqlalchemy.orm.MapperEvents.mapper_configured(mapper: Mapper[_O], class_: Type[_O]) ? None
Called when a specific mapper has completed its own configuration within the scope of the configure_mappers() call.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'mapper_configured')
def receive_mapper_configured(mapper, class_):
    "listen for the 'mapper_configured' event"

    # (event handling logic) 
The MapperEvents.mapper_configured() event is invoked for each mapper that is encountered when the configure_mappers() function proceeds through the current list of not-yet-configured mappers. configure_mappers() is typically invoked automatically as mappings are first used, as well as each time new mappers have been made available and new mapper use is detected.
When the event is called, the mapper should be in its final state, but not including backrefs that may be invoked from other mappers; they might still be pending within the configuration operation. Bidirectional relationships that are instead configured via the relationship.back_populates argument will be fully available, since this style of relationship does not rely upon other possibly-not-configured mappers to know that they exist.
For an event that is guaranteed to have all mappers ready to go including backrefs that are defined only on other mappings, use the MapperEvents.after_configured() event; this event invokes only after all known mappings have been fully configured.
The MapperEvents.mapper_configured() event, unlike MapperEvents.before_configured() or MapperEvents.after_configured(), is called for each mapper/class individually, and the mapper is passed to the event itself. It also is called exactly once for a particular mapper. The event is therefore useful for configurational steps that benefit from being invoked just once on a specific mapper basis, which don’t require that “backref” configurations are necessarily ready yet.
Parameters:
* mapper – the Mapper which is the target of this event.
* class_ – the mapped class.
See also
MapperEvents.before_configured()
MapperEvents.after_configured()
MapperEvents.before_mapper_configured()
Instance Events
Instance events are focused on the construction of ORM mapped instances, including when they are instantiated as transient objects, when they are loaded from the database and become persistent objects, as well as when database refresh or expiration operations occur on the object.
Object Name
Description
InstanceEvents
Define events specific to object lifecycle.
class sqlalchemy.orm.InstanceEvents
Define events specific to object lifecycle.
e.g.:
from sqlalchemy import event


def my_load_listener(target, context):
    print("on load!")


event.listen(SomeClass, "load", my_load_listener)
Available targets include:
* mapped classes
* unmapped superclasses of mapped or to-be-mapped classes (using the propagate=True flag)
* Mapper objects
* the Mapper class itself indicates listening for all mappers.
Instance events are closely related to mapper events, but are more specific to the instance and its instrumentation, rather than its system of persistence.
When using InstanceEvents, several modifiers are available to the listen() function.
Parameters:
* propagate=False – When True, the event listener should be applied to all inheriting classes as well as the class which is the target of this listener.
* raw=False – When True, the “target” argument passed to applicable event listener functions will be the instance’s InstanceState management object, rather than the mapped instance itself.
* restore_load_context=False – 
Applies to the InstanceEvents.load() and InstanceEvents.refresh() events. Restores the loader context of the object when the event hook is complete, so that ongoing eager load operations continue to target the object appropriately. A warning is emitted if the object is moved to a new loader context from within one of these events if this flag is not set.
Added in version 1.3.14.
Members
dispatch, expire(), first_init(), init(), init_failure(), load(), pickle(), refresh(), refresh_flush(), unpickle()
Class signature
class sqlalchemy.orm.InstanceEvents (sqlalchemy.event.Events)
attribute sqlalchemy.orm.InstanceEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstanceEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events
method sqlalchemy.orm.InstanceEvents.expire(target: _O, attrs: Iterable[str] | None) ? None
Receive an object instance after its attributes or some subset have been expired.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'expire')
def receive_expire(target, attrs):
    "listen for the 'expire' event"

    # (event handling logic) 
‘keys’ is a list of attribute names. If None, the entire state was expired.
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* attrs – sequence of attribute names which were expired, or None if all attributes were expired.
method sqlalchemy.orm.InstanceEvents.first_init(manager: ClassManager[_O], cls: Type[_O]) ? None
Called when the first instance of a particular mapping is called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'first_init')
def receive_first_init(manager, cls):
    "listen for the 'first_init' event"

    # (event handling logic) 
This event is called when the __init__ method of a class is called the first time for that particular class. The event invokes before __init__ actually proceeds as well as before the InstanceEvents.init() event is invoked.
method sqlalchemy.orm.InstanceEvents.init(target: _O, args: Any, kwargs: Any) ? None
Receive an instance when its constructor is called.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'init')
def receive_init(target, args, kwargs):
    "listen for the 'init' event"

    # (event handling logic) 
This method is only called during a userland construction of an object, in conjunction with the object’s constructor, e.g. its __init__ method. It is not called when an object is loaded from the database; see the InstanceEvents.load() event in order to intercept a database load.
The event is called before the actual __init__ constructor of the object is called. The kwargs dictionary may be modified in-place in order to affect what is passed to __init__.
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* args – positional arguments passed to the __init__ method. This is passed as a tuple and is currently immutable.
* kwargs – keyword arguments passed to the __init__ method. This structure can be altered in place.
See also
InstanceEvents.init_failure()
InstanceEvents.load()
method sqlalchemy.orm.InstanceEvents.init_failure(target: _O, args: Any, kwargs: Any) ? None
Receive an instance when its constructor has been called, and raised an exception.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'init_failure')
def receive_init_failure(target, args, kwargs):
    "listen for the 'init_failure' event"

    # (event handling logic) 
This method is only called during a userland construction of an object, in conjunction with the object’s constructor, e.g. its __init__ method. It is not called when an object is loaded from the database.
The event is invoked after an exception raised by the __init__ method is caught. After the event is invoked, the original exception is re-raised outwards, so that the construction of the object still raises an exception. The actual exception and stack trace raised should be present in sys.exc_info().
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* args – positional arguments that were passed to the __init__ method.
* kwargs – keyword arguments that were passed to the __init__ method.
See also
InstanceEvents.init()
InstanceEvents.load()
method sqlalchemy.orm.InstanceEvents.load(target: _O, context: QueryContext) ? None
Receive an object instance after it has been created via __new__, and after initial attribute population has occurred.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'load')
def receive_load(target, context):
    "listen for the 'load' event"

    # (event handling logic) 
This typically occurs when the instance is created based on incoming result rows, and is only called once for that instance’s lifetime.
Warning
During a result-row load, this event is invoked when the first row received for this instance is processed. When using eager loading with collection-oriented attributes, the additional rows that are to be loaded / processed in order to load subsequent collection items have not occurred yet. This has the effect both that collections will not be fully loaded, as well as that if an operation occurs within this event handler that emits another database load operation for the object, the “loading context” for the object can change and interfere with the existing eager loaders still in progress.
Examples of what can cause the “loading context” to change within the event handler include, but are not necessarily limited to:
* accessing deferred attributes that weren’t part of the row, will trigger an “undefer” operation and refresh the object
* accessing attributes on a joined-inheritance subclass that weren’t part of the row, will trigger a refresh operation.
As of SQLAlchemy 1.3.14, a warning is emitted when this occurs. The InstanceEvents.restore_load_context option may be used on the event to prevent this warning; this will ensure that the existing loading context is maintained for the object after the event is called:
@event.listens_for(SomeClass, "load", restore_load_context=True)
def on_load(instance, context):
    instance.some_unloaded_attribute
Changed in version 1.3.14: Added InstanceEvents.restore_load_context and SessionEvents.restore_load_context flags which apply to “on load” events, which will ensure that the loading context for an object is restored when the event hook is complete; a warning is emitted if the load context of the object changes without this flag being set.
The InstanceEvents.load() event is also available in a class-method decorator format called reconstructor().
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* context – the QueryContext corresponding to the current Query in progress. This argument may be None if the load does not correspond to a Query, such as during Session.merge().
See also
Maintaining Non-Mapped State Across Loads
InstanceEvents.init()
InstanceEvents.refresh()
SessionEvents.loaded_as_persistent()
method sqlalchemy.orm.InstanceEvents.pickle(target: _O, state_dict: _InstanceDict) ? None
Receive an object instance when its associated state is being pickled.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'pickle')
def receive_pickle(target, state_dict):
    "listen for the 'pickle' event"

    # (event handling logic) 
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* state_dict – the dictionary returned by __getstate__, containing the state to be pickled.
method sqlalchemy.orm.InstanceEvents.refresh(target: _O, context: QueryContext, attrs: Iterable[str] | None) ? None
Receive an object instance after one or more attributes have been refreshed from a query.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'refresh')
def receive_refresh(target, context, attrs):
    "listen for the 'refresh' event"

    # (event handling logic) 
Contrast this to the InstanceEvents.load() method, which is invoked when the object is first loaded from a query.
Note
This event is invoked within the loader process before eager loaders may have been completed, and the object’s state may not be complete. Additionally, invoking row-level refresh operations on the object will place the object into a new loader context, interfering with the existing load context. See the note on InstanceEvents.load() for background on making use of the InstanceEvents.restore_load_context parameter, in order to resolve this scenario.
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* context – the QueryContext corresponding to the current Query in progress.
* attrs – sequence of attribute names which were populated, or None if all column-mapped, non-deferred attributes were populated.
See also
Maintaining Non-Mapped State Across Loads
InstanceEvents.load()
method sqlalchemy.orm.InstanceEvents.refresh_flush(target: _O, flush_context: UOWTransaction, attrs: Iterable[str] | None) ? None
Receive an object instance after one or more attributes that contain a column-level default or onupdate handler have been refreshed during persistence of the object’s state.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'refresh_flush')
def receive_refresh_flush(target, flush_context, attrs):
    "listen for the 'refresh_flush' event"

    # (event handling logic) 
This event is the same as InstanceEvents.refresh() except it is invoked within the unit of work flush process, and includes only non-primary-key columns that have column level default or onupdate handlers, including Python callables as well as server side defaults and triggers which may be fetched via the RETURNING clause.
Note
While the InstanceEvents.refresh_flush() event is triggered for an object that was INSERTed as well as for an object that was UPDATEd, the event is geared primarily towards the UPDATE process; it is mostly an internal artifact that INSERT actions can also trigger this event, and note that primary key columns for an INSERTed row are explicitly omitted from this event. In order to intercept the newly INSERTed state of an object, the SessionEvents.pending_to_persistent() and MapperEvents.after_insert() are better choices.
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* flush_context – Internal UOWTransaction object which handles the details of the flush.
* attrs – sequence of attribute names which were populated.
See also
Maintaining Non-Mapped State Across Loads
Fetching Server-Generated Defaults
Column INSERT/UPDATE Defaults
method sqlalchemy.orm.InstanceEvents.unpickle(target: _O, state_dict: _InstanceDict) ? None
Receive an object instance after its associated state has been unpickled.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass, 'unpickle')
def receive_unpickle(target, state_dict):
    "listen for the 'unpickle' event"

    # (event handling logic) 
Parameters:
* target – the mapped instance. If the event is configured with raw=True, this will instead be the InstanceState state-management object associated with the instance.
* state_dict – the dictionary sent to __setstate__, containing the state dictionary which was pickled.
Attribute Events
Attribute events are triggered as things occur on individual attributes of ORM mapped objects. These events form the basis for things like custom validation functions as well as backref handlers.
See also
Changing Attribute Behavior
Object Name
Description
AttributeEvents
Define events for object attributes.
class sqlalchemy.orm.AttributeEvents
Define events for object attributes.
These are typically defined on the class-bound descriptor for the target class.
For example, to register a listener that will receive the AttributeEvents.append() event:
from sqlalchemy import event


@event.listens_for(MyClass.collection, "append", propagate=True)
def my_append_listener(target, value, initiator):
    print("received append event for target: %s" % target)
Listeners have the option to return a possibly modified version of the value, when the AttributeEvents.retval flag is passed to listen() or listens_for(), such as below, illustrated using the AttributeEvents.set() event:
def validate_phone(target, value, oldvalue, initiator):
    "Strip non-numeric characters from a phone number"

    return re.sub(r"\D", "", value)


# setup listener on UserContact.phone attribute, instructing
# it to use the return value
listen(UserContact.phone, "set", validate_phone, retval=True)
A validation function like the above can also raise an exception such as ValueError to halt the operation.
The AttributeEvents.propagate flag is also important when applying listeners to mapped classes that also have mapped subclasses, as when using mapper inheritance patterns:
@event.listens_for(MySuperClass.attr, "set", propagate=True)
def receive_set(target, value, initiator):
    print("value set: %s" % target)
The full list of modifiers available to the listen() and listens_for() functions are below.
Parameters:
* active_history=False – When True, indicates that the “set” event would like to receive the “old” value being replaced unconditionally, even if this requires firing off database loads. Note that active_history can also be set directly via column_property() and relationship().
* propagate=False – When True, the listener function will be established not just for the class attribute given, but for attributes of the same name on all current subclasses of that class, as well as all future subclasses of that class, using an additional listener that listens for instrumentation events.
* raw=False – When True, the “target” argument to the event will be the InstanceState management object, rather than the mapped instance itself.
* retval=False – when True, the user-defined event listening must return the “value” argument from the function. This gives the listening function the opportunity to change the value that is ultimately used for a “set” or “append” event.
Members
append(), append_wo_mutation(), bulk_replace(), dispatch, dispose_collection(), init_collection(), init_scalar(), modified(), remove(), set()
Class signature
class sqlalchemy.orm.AttributeEvents (sqlalchemy.event.Events)
method sqlalchemy.orm.AttributeEvents.append(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) ? _T | None
Receive a collection append event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'append')
def receive_append(target, value, initiator):
    "listen for the 'append' event"

    # (event handling logic) 
The append event is invoked for each element as it is appended to the collection. This occurs for single-item appends as well as for a “bulk replace” operation.
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – the value being appended. If this listener is registered with retval=True, the listener function must return this value, or a new value which replaces it.
* initiator – An instance of Event representing the initiation of the event. May be modified from its original value by backref handlers in order to control chained event propagation, as well as be inspected for information about the source of the event.
* key – 
When the event is established using the AttributeEvents.include_key parameter set to True, this will be the key used in the operation, such as collection[some_key_or_index] = value. The parameter is not passed to the event at all if the the AttributeEvents.include_key was not used to set up the event; this is to allow backwards compatibility with existing event handlers that don’t include the key parameter.
Added in version 2.0.
Returns:
if the event was registered with retval=True, the given value, or a new effective value, should be returned.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
AttributeEvents.bulk_replace()
method sqlalchemy.orm.AttributeEvents.append_wo_mutation(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) ? None
Receive a collection append event where the collection was not actually mutated.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'append_wo_mutation')
def receive_append_wo_mutation(target, value, initiator):
    "listen for the 'append_wo_mutation' event"

    # (event handling logic) 
This event differs from AttributeEvents.append() in that it is fired off for de-duplicating collections such as sets and dictionaries, when the object already exists in the target collection. The event does not have a return value and the identity of the given object cannot be changed.
The event is used for cascading objects into a Session when the collection has already been mutated via a backref event.
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – the value that would be appended if the object did not already exist in the collection.
* initiator – An instance of Event representing the initiation of the event. May be modified from its original value by backref handlers in order to control chained event propagation, as well as be inspected for information about the source of the event.
* key – 
When the event is established using the AttributeEvents.include_key parameter set to True, this will be the key used in the operation, such as collection[some_key_or_index] = value. The parameter is not passed to the event at all if the the AttributeEvents.include_key was not used to set up the event; this is to allow backwards compatibility with existing event handlers that don’t include the key parameter.
Added in version 2.0.
Returns:
No return value is defined for this event.
Added in version 1.4.15.
method sqlalchemy.orm.AttributeEvents.bulk_replace(target: _O, values: Iterable[_T], initiator: Event, *, keys: Iterable[EventConstants] | None = None) ? None
Receive a collection ‘bulk replace’ event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'bulk_replace')
def receive_bulk_replace(target, values, initiator):
    "listen for the 'bulk_replace' event"

    # (event handling logic) 
This event is invoked for a sequence of values as they are incoming to a bulk collection set operation, which can be modified in place before the values are treated as ORM objects. This is an “early hook” that runs before the bulk replace routine attempts to reconcile which objects are already present in the collection and which are being removed by the net replace operation.
It is typical that this method be combined with use of the AttributeEvents.append() event. When using both of these events, note that a bulk replace operation will invoke the AttributeEvents.append() event for all new items, even after AttributeEvents.bulk_replace() has been invoked for the collection as a whole. In order to determine if an AttributeEvents.append() event is part of a bulk replace, use the symbol attributes.OP_BULK_REPLACE to test the incoming initiator:
from sqlalchemy.orm.attributes import OP_BULK_REPLACE


@event.listens_for(SomeObject.collection, "bulk_replace")
def process_collection(target, values, initiator):
    values[:] = [_make_value(value) for value in values]


@event.listens_for(SomeObject.collection, "append", retval=True)
def process_collection(target, value, initiator):
    # make sure bulk_replace didn't already do it
    if initiator is None or initiator.op is not OP_BULK_REPLACE:
        return _make_value(value)
    else:
        return value
Added in version 1.2.
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – a sequence (e.g. a list) of the values being set. The handler can modify this list in place.
* initiator – An instance of Event representing the initiation of the event.
* keys – 
When the event is established using the AttributeEvents.include_key parameter set to True, this will be the sequence of keys used in the operation, typically only for a dictionary update. The parameter is not passed to the event at all if the the AttributeEvents.include_key was not used to set up the event; this is to allow backwards compatibility with existing event handlers that don’t include the key parameter.
Added in version 2.0.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
attribute sqlalchemy.orm.AttributeEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.AttributeEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events
method sqlalchemy.orm.AttributeEvents.dispose_collection(target: _O, collection: Collection[Any], collection_adapter: CollectionAdapter) ? None
Receive a ‘collection dispose’ event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'dispose_collection')
def receive_dispose_collection(target, collection, collection_adapter):
    "listen for the 'dispose_collection' event"

    # (event handling logic) 
This event is triggered for a collection-based attribute when a collection is replaced, that is:
u1.addresses.append(a1)

u1.addresses = [a2, a3]  # <- old collection is disposed
The old collection received will contain its previous contents.
Changed in version 1.2: The collection passed to AttributeEvents.dispose_collection() will now have its contents before the dispose intact; previously, the collection would be empty.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
method sqlalchemy.orm.AttributeEvents.init_collection(target: _O, collection: Type[Collection[Any]], collection_adapter: CollectionAdapter) ? None
Receive a ‘collection init’ event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'init_collection')
def receive_init_collection(target, collection, collection_adapter):
    "listen for the 'init_collection' event"

    # (event handling logic) 
This event is triggered for a collection-based attribute, when the initial “empty collection” is first generated for a blank attribute, as well as for when the collection is replaced with a new one, such as via a set event.
E.g., given that User.addresses is a relationship-based collection, the event is triggered here:
u1 = User()
u1.addresses.append(a1)  #  <- new collection
and also during replace operations:
u1.addresses = [a2, a3]  #  <- new collection
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* collection – the new collection. This will always be generated from what was specified as relationship.collection_class, and will always be empty.
* collection_adapter – the CollectionAdapter that will mediate internal access to the collection.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
AttributeEvents.init_scalar() - “scalar” version of this event.
method sqlalchemy.orm.AttributeEvents.init_scalar(target: _O, value: _T, dict_: Dict[Any, Any]) ? None
Receive a scalar “init” event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'init_scalar')
def receive_init_scalar(target, value, dict_):
    "listen for the 'init_scalar' event"

    # (event handling logic) 
This event is invoked when an uninitialized, unpersisted scalar attribute is accessed, e.g. read:
x = my_object.some_attribute
The ORM’s default behavior when this occurs for an un-initialized attribute is to return the value None; note this differs from Python’s usual behavior of raising AttributeError. The event here can be used to customize what value is actually returned, with the assumption that the event listener would be mirroring a default generator that is configured on the Core Column object as well.
Since a default generator on a Column might also produce a changing value such as a timestamp, the AttributeEvents.init_scalar() event handler can also be used to set the newly returned value, so that a Core-level default generation function effectively fires off only once, but at the moment the attribute is accessed on the non-persisted object. Normally, no change to the object’s state is made when an uninitialized attribute is accessed (much older SQLAlchemy versions did in fact change the object’s state).
If a default generator on a column returned a particular constant, a handler might be used as follows:
SOME_CONSTANT = 3.1415926


class MyClass(Base):
    # 

    some_attribute = Column(Numeric, default=SOME_CONSTANT)


@event.listens_for(
    MyClass.some_attribute, "init_scalar", retval=True, propagate=True
)
def _init_some_attribute(target, dict_, value):
    dict_["some_attribute"] = SOME_CONSTANT
    return SOME_CONSTANT
Above, we initialize the attribute MyClass.some_attribute to the value of SOME_CONSTANT. The above code includes the following features:
* By setting the value SOME_CONSTANT in the given dict_, we indicate that this value is to be persisted to the database. This supersedes the use of SOME_CONSTANT in the default generator for the Column. The active_column_defaults.py example given at Attribute Instrumentation illustrates using the same approach for a changing default, e.g. a timestamp generator. In this particular example, it is not strictly necessary to do this since SOME_CONSTANT would be part of the INSERT statement in either case.
* By establishing the retval=True flag, the value we return from the function will be returned by the attribute getter. Without this flag, the event is assumed to be a passive observer and the return value of our function is ignored.
* The propagate=True flag is significant if the mapped class includes inheriting subclasses, which would also make use of this event listener. Without this flag, an inheriting subclass will not use our event handler.
In the above example, the attribute set event AttributeEvents.set() as well as the related validation feature provided by validates is not invoked when we apply our value to the given dict_. To have these events to invoke in response to our newly generated value, apply the value to the given object as a normal attribute set operation:
SOME_CONSTANT = 3.1415926


@event.listens_for(
    MyClass.some_attribute, "init_scalar", retval=True, propagate=True
)
def _init_some_attribute(target, dict_, value):
    # will also fire off attribute set events
    target.some_attribute = SOME_CONSTANT
    return SOME_CONSTANT
When multiple listeners are set up, the generation of the value is “chained” from one listener to the next by passing the value returned by the previous listener that specifies retval=True as the value argument of the next listener.
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – the value that is to be returned before this event listener were invoked. This value begins as the value None, however will be the return value of the previous event handler function if multiple listeners are present.
* dict_ – the attribute dictionary of this mapped object. This is normally the __dict__ of the object, but in all cases represents the destination that the attribute system uses to get at the actual value of this attribute. Placing the value in this dictionary has the effect that the value will be used in the INSERT statement generated by the unit of work.
See also
AttributeEvents.init_collection() - collection version of this event
AttributeEvents - background on listener options such as propagation to subclasses.
Attribute Instrumentation - see the active_column_defaults.py example.
method sqlalchemy.orm.AttributeEvents.modified(target: _O, initiator: Event) ? None
Receive a ‘modified’ event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'modified')
def receive_modified(target, initiator):
    "listen for the 'modified' event"

    # (event handling logic) 
This event is triggered when the flag_modified() function is used to trigger a modify event on an attribute without any specific value being set.
Added in version 1.2.
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* initiator – An instance of Event representing the initiation of the event.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
method sqlalchemy.orm.AttributeEvents.remove(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) ? None
Receive a collection remove event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'remove')
def receive_remove(target, value, initiator):
    "listen for the 'remove' event"

    # (event handling logic) 
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – the value being removed.
* initiator – An instance of Event representing the initiation of the event. May be modified from its original value by backref handlers in order to control chained event propagation.
* key – 
When the event is established using the AttributeEvents.include_key parameter set to True, this will be the key used in the operation, such as del collection[some_key_or_index]. The parameter is not passed to the event at all if the the AttributeEvents.include_key was not used to set up the event; this is to allow backwards compatibility with existing event handlers that don’t include the key parameter.
Added in version 2.0.
Returns:
No return value is defined for this event.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
method sqlalchemy.orm.AttributeEvents.set(target: _O, value: _T, oldvalue: _T, initiator: Event) ? None
Receive a scalar set event.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeClass.some_attribute, 'set')
def receive_set(target, value, oldvalue, initiator):
    "listen for the 'set' event"

    # (event handling logic) 
Parameters:
* target – the object instance receiving the event. If the listener is registered with raw=True, this will be the InstanceState object.
* value – the value being set. If this listener is registered with retval=True, the listener function must return this value, or a new value which replaces it.
* oldvalue – the previous value being replaced. This may also be the symbol NEVER_SET or NO_VALUE. If the listener is registered with active_history=True, the previous value of the attribute will be loaded from the database if the existing value is currently unloaded or expired.
* initiator – An instance of Event representing the initiation of the event. May be modified from its original value by backref handlers in order to control chained event propagation.
Returns:
if the event was registered with retval=True, the given value, or a new effective value, should be returned.
See also
AttributeEvents - background on listener options such as propagation to subclasses.
Query Events
Object Name
Description
QueryEvents
Represent events within the construction of a Query object.
class sqlalchemy.orm.QueryEvents
Represent events within the construction of a Query object.
Legacy Feature
The QueryEvents event methods are legacy as of SQLAlchemy 2.0, and only apply to direct use of the Query object. They are not used for 2.0 style statements. For events to intercept and modify 2.0 style ORM use, use the SessionEvents.do_orm_execute() hook.
The QueryEvents hooks are now superseded by the SessionEvents.do_orm_execute() event hook.
Members
before_compile(), before_compile_delete(), before_compile_update(), dispatch
Class signature
class sqlalchemy.orm.QueryEvents (sqlalchemy.event.Events)
method sqlalchemy.orm.QueryEvents.before_compile(query: Query) ? None
Receive the Query object before it is composed into a core Select object.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeQuery, 'before_compile')
def receive_before_compile(query):
    "listen for the 'before_compile' event"

    # (event handling logic) 
Deprecated since version 1.4: The QueryEvents.before_compile() event is superseded by the much more capable SessionEvents.do_orm_execute() hook. In version 1.4, the QueryEvents.before_compile() event is no longer used for ORM-level attribute loads, such as loads of deferred or expired attributes as well as relationship loaders. See the new examples in ORM Query Events which illustrate new ways of intercepting and modifying ORM queries for the most common purpose of adding arbitrary filter criteria.
This event is intended to allow changes to the query given:
@event.listens_for(Query, "before_compile", retval=True)
def no_deleted(query):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
The event should normally be listened with the retval=True parameter set, so that the modified query may be returned.
The QueryEvents.before_compile() event by default will disallow “baked” queries from caching a query, if the event hook returns a new Query object. This affects both direct use of the baked query extension as well as its operation within lazy loaders and eager loaders for relationships. In order to re-establish the query being cached, apply the event adding the bake_ok flag:
@event.listens_for(Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
When bake_ok is set to True, the event hook will only be invoked once, and not called for subsequent invocations of a particular query that is being cached.
Added in version 1.3.11: - added the “bake_ok” flag to the QueryEvents.before_compile() event and disallowed caching via the “baked” extension from occurring for event handlers that return a new Query object if this flag is not set.
See also
QueryEvents.before_compile_update()
QueryEvents.before_compile_delete()
Using the before_compile event
method sqlalchemy.orm.QueryEvents.before_compile_delete(query: Query, delete_context: BulkDelete) ? None
Allow modifications to the Query object within Query.delete().
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeQuery, 'before_compile_delete')
def receive_before_compile_delete(query, delete_context):
    "listen for the 'before_compile_delete' event"

    # (event handling logic) 
Deprecated since version 1.4: The QueryEvents.before_compile_delete() event is superseded by the much more capable SessionEvents.do_orm_execute() hook.
Like the QueryEvents.before_compile() event, this event should be configured with retval=True, and the modified Query object returned, as in
@event.listens_for(Query, "before_compile_delete", retval=True)
def no_deleted(query, delete_context):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
Parameters:
* query – a Query instance; this is also the .query attribute of the given “delete context” object.
* delete_context – a “delete context” object which is the same kind of object as described in QueryEvents.after_bulk_delete.delete_context.
Added in version 1.2.17.
See also
QueryEvents.before_compile()
QueryEvents.before_compile_update()
method sqlalchemy.orm.QueryEvents.before_compile_update(query: Query, update_context: BulkUpdate) ? None
Allow modifications to the Query object within Query.update().
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeQuery, 'before_compile_update')
def receive_before_compile_update(query, update_context):
    "listen for the 'before_compile_update' event"

    # (event handling logic) 
Deprecated since version 1.4: The QueryEvents.before_compile_update() event is superseded by the much more capable SessionEvents.do_orm_execute() hook.
Like the QueryEvents.before_compile() event, if the event is to be used to alter the Query object, it should be configured with retval=True, and the modified Query object returned, as in
@event.listens_for(Query, "before_compile_update", retval=True)
def no_deleted(query, update_context):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)

            update_context.values["timestamp"] = datetime.datetime.now(
                datetime.UTC
            )
    return query
The .values dictionary of the “update context” object can also be modified in place as illustrated above.
Parameters:
* query – a Query instance; this is also the .query attribute of the given “update context” object.
* update_context – an “update context” object which is the same kind of object as described in QueryEvents.after_bulk_update.update_context. The object has a .values attribute in an UPDATE context which is the dictionary of parameters passed to Query.update(). This dictionary can be modified to alter the VALUES clause of the resulting UPDATE statement.
Added in version 1.2.17.
See also
QueryEvents.before_compile()
QueryEvents.before_compile_delete()
attribute sqlalchemy.orm.QueryEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.QueryEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events
Instrumentation Events
Defines SQLAlchemy’s system of class instrumentation.
This module is usually not directly visible to user applications, but defines a large part of the ORM’s interactivity.
instrumentation.py deals with registration of end-user classes for state tracking. It interacts closely with state.py and attributes.py which establish per-instance and per-class-attribute instrumentation, respectively.
The class instrumentation system can be customized on a per-class or global basis using the sqlalchemy.ext.instrumentation module, which provides the means to build and specify alternate instrumentation forms.
Object Name
Description
InstrumentationEvents
Events related to class instrumentation events.
class sqlalchemy.orm.InstrumentationEvents
Events related to class instrumentation events.
The listeners here support being established against any new style class, that is any object that is a subclass of ‘type’. Events will then be fired off for events against that class. If the “propagate=True” flag is passed to event.listen(), the event will fire off for subclasses of that class as well.
The Python type builtin is also accepted as a target, which when used has the effect of events being emitted for all classes.
Note the “propagate” flag here is defaulted to True, unlike the other class level events where it defaults to False. This means that new subclasses will also be the subject of these events, when a listener is established on a superclass.
Members
attribute_instrument(), class_instrument(), class_uninstrument(), dispatch
Class signature
class sqlalchemy.orm.InstrumentationEvents (sqlalchemy.event.Events)
method sqlalchemy.orm.InstrumentationEvents.attribute_instrument(cls: ClassManager[_O], key: _KT, inst: _O) ? None
Called when an attribute is instrumented.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeBaseClass, 'attribute_instrument')
def receive_attribute_instrument(cls, key, inst):
    "listen for the 'attribute_instrument' event"

    # (event handling logic) 
method sqlalchemy.orm.InstrumentationEvents.class_instrument(cls: ClassManager[_O]) ? None
Called after the given class is instrumented.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeBaseClass, 'class_instrument')
def receive_class_instrument(cls):
    "listen for the 'class_instrument' event"

    # (event handling logic) 
To get at the ClassManager, use manager_of_class().
method sqlalchemy.orm.InstrumentationEvents.class_uninstrument(cls: ClassManager[_O]) ? None
Called before the given class is uninstrumented.
Example argument forms:
from sqlalchemy import event


@event.listens_for(SomeBaseClass, 'class_uninstrument')
def receive_class_uninstrument(cls):
    "listen for the 'class_uninstrument' event"

    # (event handling logic) 
To get at the ClassManager, use manager_of_class().
attribute sqlalchemy.orm.InstrumentationEvents.dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstrumentationEventsDispatch object>
reference back to the _Dispatch class.
Bidirectional against _Dispatch._events


ORM Internals
Key ORM constructs, not otherwise covered in other sections, are listed here.
Object Name
Description
AttributeEventToken
A token propagated throughout the course of a chain of attribute events.
AttributeState
Provide an inspection interface corresponding to a particular attribute on a particular mapped object.
CascadeOptions
Keeps track of the options sent to relationship.cascade
ClassManager
Tracks state information at the class level.
ColumnProperty
Describes an object attribute that corresponds to a table column or other column expression.
Composite
Declarative-compatible front-end for the CompositeProperty class.
CompositeProperty
Defines a “composite” mapped attribute, representing a collection of columns as one attribute.
IdentityMap

InspectionAttr
A base class applied to all ORM objects and attributes that are related to things that can be returned by the inspect() function.
InspectionAttrExtensionType
Symbols indicating the type of extension that a InspectionAttr is part of.
InspectionAttrInfo
Adds the .info attribute to InspectionAttr.
InstanceState
Tracks state information at the instance level.
InstrumentedAttribute
Base class for descriptor objects that intercept attribute events on behalf of a MapperProperty object. The actual MapperProperty is accessible via the QueryableAttribute.property attribute.
LoaderCallableStatus

Mapped
Represent an ORM mapped attribute on a mapped class.
MappedColumn
Maps a single Column on a class.
MappedSQLExpression
Declarative front-end for the ColumnProperty class.
MapperProperty
Represent a particular class attribute mapped by Mapper.
merge_frozen_result(session, statement, frozen_result[, load])
Merge a FrozenResult back into a Session, returning a new Result object with persistent objects.
merge_result(query, iterator[, load])
Merge a result into the given Query object’s Session.
NotExtension

PropComparator
Defines SQL operations for ORM mapped attributes.
QueryableAttribute
Base class for descriptor objects that intercept attribute events on behalf of a MapperProperty object. The actual MapperProperty is accessible via the QueryableAttribute.property attribute.
QueryContext

Relationship
Describes an object property that holds a single item or list of items that correspond to a related database table.
RelationshipDirection
enumeration which indicates the ‘direction’ of a RelationshipProperty.
RelationshipProperty
Describes an object property that holds a single item or list of items that correspond to a related database table.
SQLORMExpression
A type that may be used to indicate any ORM-level attribute or object that acts in place of one, in the context of SQL expression construction.
Synonym
Declarative front-end for the SynonymProperty class.
SynonymProperty
Denote an attribute name as a synonym to a mapped property, in that the attribute will mirror the value and expression behavior of another attribute.
UOWTransaction

class sqlalchemy.orm.AttributeState
Provide an inspection interface corresponding to a particular attribute on a particular mapped object.
The AttributeState object is accessed via the InstanceState.attrs collection of a particular InstanceState:
Members
history, load_history(), loaded_value, value
from sqlalchemy import inspect

insp = inspect(some_mapped_object)
attr_state = insp.attrs.some_attribute
attribute sqlalchemy.orm.AttributeState.history
Return the current pre-flush change history for this attribute, via the History interface.
This method will not emit loader callables if the value of the attribute is unloaded.
Note
The attribute history system tracks changes on a per flush basis. Each time the Session is flushed, the history of each attribute is reset to empty. The Session by default autoflushes each time a Query is invoked. For options on how to control this, see Flushing.
See also
AttributeState.load_history() - retrieve history using loader callables if the value is not locally present.
get_history() - underlying function
method sqlalchemy.orm.AttributeState.load_history() ? History
Return the current pre-flush change history for this attribute, via the History interface.
This method will emit loader callables if the value of the attribute is unloaded.
Note
The attribute history system tracks changes on a per flush basis. Each time the Session is flushed, the history of each attribute is reset to empty. The Session by default autoflushes each time a Query is invoked. For options on how to control this, see Flushing.
See also
AttributeState.history
get_history() - underlying function
attribute sqlalchemy.orm.AttributeState.loaded_value
The current value of this attribute as loaded from the database.
If the value has not been loaded, or is otherwise not present in the object’s dictionary, returns NO_VALUE.
attribute sqlalchemy.orm.AttributeState.value
Return the value of this attribute.
This operation is equivalent to accessing the object’s attribute directly or via getattr(), and will fire off any pending loader callables if needed.
class sqlalchemy.orm.CascadeOptions
Keeps track of the options sent to relationship.cascade
Class signature
class sqlalchemy.orm.CascadeOptions (builtins.frozenset, typing.Generic)
class sqlalchemy.orm.ClassManager
Tracks state information at the class level.
Members
deferred_scalar_loader, expired_attribute_loader, has_parent(), manage(), state_getter(), unregister()
Class signature
class sqlalchemy.orm.ClassManager (sqlalchemy.util.langhelpers.HasMemoized, builtins.dict, typing.Generic, sqlalchemy.event.registry.EventTarget)
attribute sqlalchemy.orm.ClassManager.deferred_scalar_loader
Deprecated since version 1.4: The ClassManager.deferred_scalar_loader attribute is now named expired_attribute_loader
attribute sqlalchemy.orm.ClassManager.expired_attribute_loader: _ExpiredAttributeLoaderProto
previously known as deferred_scalar_loader
method sqlalchemy.orm.ClassManager.has_parent(state: InstanceState[_O], key: str, optimistic: bool = False) ? bool
TODO
method sqlalchemy.orm.ClassManager.manage()
Mark this instance as the manager for its class.
method sqlalchemy.orm.ClassManager.state_getter()
Return a (instance) -> InstanceState callable.
“state getter” callables should raise either KeyError or AttributeError if no InstanceState could be found for the instance.
method sqlalchemy.orm.ClassManager.unregister() ? None
remove all instrumentation established by this ClassManager.
class sqlalchemy.orm.ColumnProperty
Describes an object attribute that corresponds to a table column or other column expression.
Public constructor is the column_property() function.
Members
expressions, operate(), reverse_operate(), columns_to_assign, declarative_scan(), do_init(), expression, instrument_class(), mapper_property_to_assign, merge()
Class signature
class sqlalchemy.orm.ColumnProperty (sqlalchemy.orm._MapsColumns, sqlalchemy.orm.StrategizedProperty, sqlalchemy.orm._IntrospectsAnnotations, sqlalchemy.log.Identified)
class Comparator
Produce boolean, comparison, and other operators for ColumnProperty attributes.
See the documentation for PropComparator for a brief overview.
See also
PropComparator
ColumnOperators
Redefining and Creating New Operators
TypeEngine.comparator_factory
Class signature
class sqlalchemy.orm.ColumnProperty.Comparator (sqlalchemy.util.langhelpers.MemoizedSlots, sqlalchemy.orm.PropComparator)
attribute sqlalchemy.orm.ColumnProperty.Comparator.expressions: Sequence[NamedColumn[Any]]
The full sequence of columns referenced by this
attribute, adjusted for any aliasing in progress.
Added in version 1.3.17.
See also
Mapping a Class against Multiple Tables - usage example
method sqlalchemy.orm.ColumnProperty.Comparator.operate(op: OperatorType, *other: Any, **kwargs: Any) ? ColumnElement[Any]
Operate on an argument.
This is the lowest level of operation, raises NotImplementedError by default.
Overriding this on a subclass can allow common behavior to be applied to all operations. For example, overriding ColumnOperators to apply func.lower() to the left and right side:
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
Parameters:
* op – Operator callable.
* *other – the ‘other’ side of the operation. Will be a single scalar for most operations.
* **kwargs – modifiers. These may be passed by special operators such as ColumnOperators.contains().
method sqlalchemy.orm.ColumnProperty.Comparator.reverse_operate(op: OperatorType, other: Any, **kwargs: Any) ? ColumnElement[Any]
Reverse operate on an argument.
Usage is the same as operate().
attribute sqlalchemy.orm.ColumnProperty.columns_to_assign
method sqlalchemy.orm.ColumnProperty.declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) ? None
Perform class-specific initializaton at early declarative scanning time.
Added in version 2.0.
method sqlalchemy.orm.ColumnProperty.do_init() ? None
Perform subclass-specific initialization post-mapper-creation steps.
This is a template method called by the MapperProperty object’s init() method.
attribute sqlalchemy.orm.ColumnProperty.expression
Return the primary column or expression for this ColumnProperty.
E.g.:
class File(Base):
    # 

    name = Column(String(64))
    extension = Column(String(8))
    filename = column_property(name + "." + extension)
    path = column_property("C:/" + filename.expression)
See also
Composing from Column Properties at Mapping Time
method sqlalchemy.orm.ColumnProperty.instrument_class(mapper: Mapper[Any]) ? None
Hook called by the Mapper to the property to initiate instrumentation of the class attribute managed by this MapperProperty.
The MapperProperty here will typically call out to the attributes module to set up an InstrumentedAttribute.
This step is the first of two steps to set up an InstrumentedAttribute, and is called early in the mapper setup process.
The second step is typically the init_class_attribute step, called from StrategizedProperty via the post_instrument_class() hook. This step assigns additional state to the InstrumentedAttribute (specifically the “impl”) which has been determined after the MapperProperty has determined what kind of persistence management it needs to do (e.g. scalar, object, collection, etc).
attribute sqlalchemy.orm.ColumnProperty.mapper_property_to_assign
method sqlalchemy.orm.ColumnProperty.merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) ? None
Merge the attribute represented by this MapperProperty from source to destination object.
class sqlalchemy.orm.Composite
Declarative-compatible front-end for the CompositeProperty class.
Public constructor is the composite() function.
Changed in version 2.0: Added Composite as a Declarative compatible subclass of CompositeProperty.
See also
Composite Column Types
Class signature
class sqlalchemy.orm.Composite (sqlalchemy.orm.descriptor_props.CompositeProperty, sqlalchemy.orm.base._DeclarativeMapped)
class sqlalchemy.orm.CompositeProperty
Defines a “composite” mapped attribute, representing a collection of columns as one attribute.
CompositeProperty is constructed using the composite() function.
See also
Composite Column Types
Members
create_row_processor(), columns_to_assign, declarative_scan(), do_init(), get_history(), instrument_class(), mapper_property_to_assign
Class signature
class sqlalchemy.orm.CompositeProperty (sqlalchemy.orm._MapsColumns, sqlalchemy.orm._IntrospectsAnnotations, sqlalchemy.orm.descriptor_props.DescriptorProperty)
class Comparator
Produce boolean, comparison, and other operators for Composite attributes.
See the example in Redefining Comparison Operations for Composites for an overview of usage , as well as the documentation for PropComparator.
See also
PropComparator
ColumnOperators
Redefining and Creating New Operators
TypeEngine.comparator_factory
Class signature
class sqlalchemy.orm.CompositeProperty.Comparator (sqlalchemy.orm.PropComparator)
class CompositeBundle
Class signature
class sqlalchemy.orm.CompositeProperty.CompositeBundle (sqlalchemy.orm.Bundle)
method sqlalchemy.orm.CompositeProperty.CompositeBundle.create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) ? Callable[[Row[Any]], Any]
Produce the “row processing” function for this Bundle.
May be overridden by subclasses to provide custom behaviors when results are fetched. The method is passed the statement object and a set of “row processor” functions at query execution time; these processor functions when given a result row will return the individual attribute value, which can then be adapted into any kind of return data structure.
The example below illustrates replacing the usual Row return structure with a straight Python dictionary:
from sqlalchemy.orm import Bundle


class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
        "Override create_row_processor to return values as dictionaries"

        def proc(row):
            return dict(zip(labels, (proc(row) for proc in procs)))

        return proc
A result from the above Bundle will return dictionary values:
bn = DictBundle("mybundle", MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == "d1"):
    print(row.mybundle["data1"], row.mybundle["data2"])
attribute sqlalchemy.orm.CompositeProperty.columns_to_assign
method sqlalchemy.orm.CompositeProperty.declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) ? None
Perform class-specific initializaton at early declarative scanning time.
Added in version 2.0.
method sqlalchemy.orm.CompositeProperty.do_init() ? None
Initialization which occurs after the Composite has been associated with its parent mapper.
method sqlalchemy.orm.CompositeProperty.get_history(state: InstanceState[Any], dict_: _InstanceDict, passive: PassiveFlag = symbol('PASSIVE_OFF')) ? History
Provided for userland code that uses attributes.get_history().
method sqlalchemy.orm.CompositeProperty.instrument_class(mapper: Mapper[Any]) ? None
Hook called by the Mapper to the property to initiate instrumentation of the class attribute managed by this MapperProperty.
The MapperProperty here will typically call out to the attributes module to set up an InstrumentedAttribute.
This step is the first of two steps to set up an InstrumentedAttribute, and is called early in the mapper setup process.
The second step is typically the init_class_attribute step, called from StrategizedProperty via the post_instrument_class() hook. This step assigns additional state to the InstrumentedAttribute (specifically the “impl”) which has been determined after the MapperProperty has determined what kind of persistence management it needs to do (e.g. scalar, object, collection, etc).
attribute sqlalchemy.orm.CompositeProperty.mapper_property_to_assign
class sqlalchemy.orm.AttributeEventToken
A token propagated throughout the course of a chain of attribute events.
Serves as an indicator of the source of the event and also provides a means of controlling propagation across a chain of attribute operations.
The Event object is sent as the initiator argument when dealing with events such as AttributeEvents.append(), AttributeEvents.set(), and AttributeEvents.remove().
The Event object is currently interpreted by the backref event handlers, and is used to control the propagation of operations across two mutually-dependent attributes.
Changed in version 2.0: Changed the name from AttributeEvent to AttributeEventToken.
Attribute impl:
The AttributeImpl which is the current event initiator.
Attribute op:
The symbol OP_APPEND, OP_REMOVE, OP_REPLACE, or OP_BULK_REPLACE, indicating the source operation.
class sqlalchemy.orm.IdentityMap
Members
check_modified()
method sqlalchemy.orm.IdentityMap.check_modified() ? bool
return True if any InstanceStates present have been marked as ‘modified’.
class sqlalchemy.orm.InspectionAttr
A base class applied to all ORM objects and attributes that are related to things that can be returned by the inspect() function.
The attributes defined here allow the usage of simple boolean checks to test basic facts about the object returned.
Members
extension_type, is_aliased_class, is_attribute, is_bundle, is_clause_element, is_instance, is_mapper, is_property, is_selectable
While the boolean checks here are basically the same as using the Python isinstance() function, the flags here can be used without the need to import all of these classes, and also such that the SQLAlchemy class system can change while leaving the flags here intact for forwards-compatibility.
attribute sqlalchemy.orm.InspectionAttr.extension_type: InspectionAttrExtensionType = 'not_extension'
The extension type, if any. Defaults to NotExtension.NOT_EXTENSION
See also
HybridExtensionType
AssociationProxyExtensionType
attribute sqlalchemy.orm.InspectionAttr.is_aliased_class = False
True if this object is an instance of AliasedClass.
attribute sqlalchemy.orm.InspectionAttr.is_attribute = False
True if this object is a Python descriptor.
This can refer to one of many types. Usually a QueryableAttribute which handles attributes events on behalf of a MapperProperty. But can also be an extension type such as AssociationProxy or hybrid_property. The InspectionAttr.extension_type will refer to a constant identifying the specific subtype.
See also
Mapper.all_orm_descriptors
attribute sqlalchemy.orm.InspectionAttr.is_bundle = False
True if this object is an instance of Bundle.
attribute sqlalchemy.orm.InspectionAttr.is_clause_element = False
True if this object is an instance of ClauseElement.
attribute sqlalchemy.orm.InspectionAttr.is_instance = False
True if this object is an instance of InstanceState.
attribute sqlalchemy.orm.InspectionAttr.is_mapper = False
True if this object is an instance of Mapper.
attribute sqlalchemy.orm.InspectionAttr.is_property = False
True if this object is an instance of MapperProperty.
attribute sqlalchemy.orm.InspectionAttr.is_selectable = False
Return True if this object is an instance of Selectable.
class sqlalchemy.orm.InspectionAttrInfo
Adds the .info attribute to InspectionAttr.
The rationale for InspectionAttr vs. InspectionAttrInfo is that the former is compatible as a mixin for classes that specify __slots__; this is essentially an implementation artifact.
Members
info
Class signature
class sqlalchemy.orm.InspectionAttrInfo (sqlalchemy.orm.base.InspectionAttr)
attribute sqlalchemy.orm.InspectionAttrInfo.info
Info dictionary associated with the object, allowing user-defined data to be associated with this InspectionAttr.
The dictionary is generated when first accessed. Alternatively, it can be specified as a constructor argument to the column_property(), relationship(), or composite() functions.
See also
QueryableAttribute.info
SchemaItem.info
class sqlalchemy.orm.InstanceState
Tracks state information at the instance level.
The InstanceState is a key object used by the SQLAlchemy ORM in order to track the state of an object; it is created the moment an object is instantiated, typically as a result of instrumentation which SQLAlchemy applies to the __init__() method of the class.
InstanceState is also a semi-public object, available for runtime inspection as to the state of a mapped instance, including information such as its current status within a particular Session and details about data on individual attributes. The public API in order to acquire a InstanceState object is to use the inspect() system:
from sqlalchemy import inspect
insp = inspect(some_mapped_object)
insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
See also
Inspection of Mapped Instances
Members
async_session, attrs, callables, deleted, detached, dict, expired, expired_attributes, has_identity, identity, identity_key, is_instance, mapper, modified, object, pending, persistent, session, transient, unloaded, unloaded_expirable, unmodified, unmodified_intersection(), was_deleted
Class signature
class sqlalchemy.orm.InstanceState (sqlalchemy.orm.base.InspectionAttrInfo, typing.Generic)
attribute sqlalchemy.orm.InstanceState.async_session
Return the owning AsyncSession for this instance, or None if none available.
This attribute is only non-None when the sqlalchemy.ext.asyncio API is in use for this ORM object. The returned AsyncSession object will be a proxy for the Session object that would be returned from the InstanceState.session attribute for this InstanceState.
Added in version 1.4.18.
See also
Asynchronous I/O (asyncio)
attribute sqlalchemy.orm.InstanceState.attrs
Return a namespace representing each attribute on the mapped object, including its current value and history.
The returned object is an instance of AttributeState. This object allows inspection of the current data within an attribute as well as attribute history since the last flush.
attribute sqlalchemy.orm.InstanceState.callables: Dict[str, Callable[[InstanceState[_O], PassiveFlag], Any]] = {}
A namespace where a per-state loader callable can be associated.
In SQLAlchemy 1.0, this is only used for lazy loaders / deferred loaders that were set up via query option.
Previously, callables was used also to indicate expired attributes by storing a link to the InstanceState itself in this dictionary. This role is now handled by the expired_attributes set.
attribute sqlalchemy.orm.InstanceState.deleted
Return True if the object is deleted.
An object that is in the deleted state is guaranteed to not be within the Session.identity_map of its parent Session; however if the session’s transaction is rolled back, the object will be restored to the persistent state and the identity map.
Note
The InstanceState.deleted attribute refers to a specific state of the object that occurs between the “persistent” and “detached” states; once the object is detached, the InstanceState.deleted attribute no longer returns True; in order to detect that a state was deleted, regardless of whether or not the object is associated with a Session, use the InstanceState.was_deleted accessor.
See also
Quickie Intro to Object States
attribute sqlalchemy.orm.InstanceState.detached
Return True if the object is detached.
See also
Quickie Intro to Object States
attribute sqlalchemy.orm.InstanceState.dict
Return the instance dict used by the object.
Under normal circumstances, this is always synonymous with the __dict__ attribute of the mapped object, unless an alternative instrumentation system has been configured.
In the case that the actual object has been garbage collected, this accessor returns a blank dictionary.
attribute sqlalchemy.orm.InstanceState.expired: bool = False
When True the object is expired.
See also
Refreshing / Expiring
attribute sqlalchemy.orm.InstanceState.expired_attributes: Set[str]
The set of keys which are ‘expired’ to be loaded by the manager’s deferred scalar loader, assuming no pending changes.
See also the unmodified collection which is intersected against this set when a refresh operation occurs.
attribute sqlalchemy.orm.InstanceState.has_identity
Return True if this object has an identity key.
This should always have the same value as the expression state.persistent or state.detached.
attribute sqlalchemy.orm.InstanceState.identity
Return the mapped identity of the mapped object. This is the primary key identity as persisted by the ORM which can always be passed directly to Query.get().
Returns None if the object has no primary key identity.
Note
An object which is transient or pending does not have a mapped identity until it is flushed, even if its attributes include primary key values.
attribute sqlalchemy.orm.InstanceState.identity_key
Return the identity key for the mapped object.
This is the key used to locate the object within the Session.identity_map mapping. It contains the identity as returned by identity within it.
attribute sqlalchemy.orm.InstanceState.is_instance: bool = True
True if this object is an instance of InstanceState.
attribute sqlalchemy.orm.InstanceState.mapper
Return the Mapper used for this mapped object.
attribute sqlalchemy.orm.InstanceState.modified: bool = False
When True the object was modified.
attribute sqlalchemy.orm.InstanceState.object
Return the mapped object represented by this InstanceState.
Returns None if the object has been garbage collected
attribute sqlalchemy.orm.InstanceState.pending
Return True if the object is pending.
See also
Quickie Intro to Object States
attribute sqlalchemy.orm.InstanceState.persistent
Return True if the object is persistent.
An object that is in the persistent state is guaranteed to be within the Session.identity_map of its parent Session.
See also
Quickie Intro to Object States
attribute sqlalchemy.orm.InstanceState.session
Return the owning Session for this instance, or None if none available.
Note that the result here can in some cases be different from that of obj in session; an object that’s been deleted will report as not in session, however if the transaction is still in progress, this attribute will still refer to that session. Only when the transaction is completed does the object become fully detached under normal circumstances.
See also
InstanceState.async_session
attribute sqlalchemy.orm.InstanceState.transient
Return True if the object is transient.
See also
Quickie Intro to Object States
attribute sqlalchemy.orm.InstanceState.unloaded
Return the set of keys which do not have a loaded value.
This includes expired attributes and any other attribute that was never populated or modified.
attribute sqlalchemy.orm.InstanceState.unloaded_expirable
Synonymous with InstanceState.unloaded.
Deprecated since version 2.0: The InstanceState.unloaded_expirable attribute is deprecated. Please use InstanceState.unloaded.
This attribute was added as an implementation-specific detail at some point and should be considered to be private.
attribute sqlalchemy.orm.InstanceState.unmodified
Return the set of keys which have no uncommitted changes
method sqlalchemy.orm.InstanceState.unmodified_intersection(keys: Iterable[str]) ? Set[str]
Return self.unmodified.intersection(keys).
attribute sqlalchemy.orm.InstanceState.was_deleted
Return True if this object is or was previously in the “deleted” state and has not been reverted to persistent.
This flag returns True once the object was deleted in flush. When the object is expunged from the session either explicitly or via transaction commit and enters the “detached” state, this flag will continue to report True.
See also
InstanceState.deleted - refers to the “deleted” state
was_deleted() - standalone function
Quickie Intro to Object States
class sqlalchemy.orm.InstrumentedAttribute
Base class for descriptor objects that intercept attribute events on behalf of a MapperProperty object. The actual MapperProperty is accessible via the QueryableAttribute.property attribute.
See also
InstrumentedAttribute
MapperProperty
Mapper.all_orm_descriptors
Mapper.attrs
Class signature
class sqlalchemy.orm.InstrumentedAttribute (sqlalchemy.orm.QueryableAttribute)
class sqlalchemy.orm.LoaderCallableStatus
Members
ATTR_EMPTY, ATTR_WAS_SET, NEVER_SET, NO_VALUE, PASSIVE_CLASS_MISMATCH, PASSIVE_NO_RESULT
Class signature
class sqlalchemy.orm.LoaderCallableStatus (enum.Enum)
attribute sqlalchemy.orm.LoaderCallableStatus.ATTR_EMPTY = 3
Symbol used internally to indicate an attribute had no callable.
attribute sqlalchemy.orm.LoaderCallableStatus.ATTR_WAS_SET = 2
Symbol returned by a loader callable to indicate the retrieved value, or values, were assigned to their attributes on the target object.
attribute sqlalchemy.orm.LoaderCallableStatus.NEVER_SET = 4
Synonymous with NO_VALUE
Changed in version 1.4: NEVER_SET was merged with NO_VALUE
attribute sqlalchemy.orm.LoaderCallableStatus.NO_VALUE = 4
Symbol which may be placed as the ‘previous’ value of an attribute, indicating no value was loaded for an attribute when it was modified, and flags indicated we were not to load it.
attribute sqlalchemy.orm.LoaderCallableStatus.PASSIVE_CLASS_MISMATCH = 1
Symbol indicating that an object is locally present for a given primary key identity but it is not of the requested class. The return value is therefore None and no SQL should be emitted.
attribute sqlalchemy.orm.LoaderCallableStatus.PASSIVE_NO_RESULT = 0
Symbol returned by a loader callable or other attribute/history retrieval operation when a value could not be determined, based on loader callable flags.
class sqlalchemy.orm.Mapped
Represent an ORM mapped attribute on a mapped class.
This class represents the complete descriptor interface for any class attribute that will have been instrumented by the ORM Mapper class. Provides appropriate information to type checkers such as pylance and mypy so that ORM-mapped attributes are correctly typed.
The most prominent use of Mapped is in the Declarative Mapping form of Mapper configuration, where used explicitly it drives the configuration of ORM attributes such as mapped_class() and relationship().
See also
Using a Declarative Base Class
Declarative Table with mapped_column()
Tip
The Mapped class represents attributes that are handled directly by the Mapper class. It does not include other Python descriptor classes that are provided as extensions, including Hybrid Attributes and the Association Proxy. While these systems still make use of ORM-specific superclasses and structures, they are not instrumented by the Mapper and instead provide their own functionality when they are accessed on a class.
Added in version 1.4.
Class signature
class sqlalchemy.orm.Mapped (sqlalchemy.orm.base.SQLORMExpression, sqlalchemy.orm.base.ORMDescriptor, sqlalchemy.orm.base._MappedAnnotationBase, sqlalchemy.sql.roles.DDLConstraintColumnRole)
class sqlalchemy.orm.MappedColumn
Maps a single Column on a class.
MappedColumn is a specialization of the ColumnProperty class and is oriented towards declarative configuration.
To construct MappedColumn objects, use the mapped_column() constructor function.
Added in version 2.0.
Class signature
class sqlalchemy.orm.MappedColumn (sqlalchemy.orm._IntrospectsAnnotations, sqlalchemy.orm._MapsColumns, sqlalchemy.orm.base._DeclarativeMapped)
class sqlalchemy.orm.MapperProperty
Represent a particular class attribute mapped by Mapper.
The most common occurrences of MapperProperty are the mapped Column, which is represented in a mapping as an instance of ColumnProperty, and a reference to another class produced by relationship(), represented in the mapping as an instance of Relationship.
Members
cascade_iterator(), class_attribute, comparator, create_row_processor(), do_init(), doc, info, init(), instrument_class(), is_property, key, merge(), parent, post_instrument_class(), set_parent(), setup()
Class signature
class sqlalchemy.orm.MapperProperty (sqlalchemy.sql.cache_key.HasCacheKey, sqlalchemy.orm._DCAttributeOptions, sqlalchemy.orm.base._MappedAttribute, sqlalchemy.orm.base.InspectionAttrInfo, sqlalchemy.util.langhelpers.MemoizedSlots)
method sqlalchemy.orm.MapperProperty.cascade_iterator(type_: str, state: InstanceState[Any], dict_: _InstanceDict, visited_states: Set[InstanceState[Any]], halt_on: Callable[[InstanceState[Any]], bool] | None = None) ? Iterator[Tuple[object, Mapper[Any], InstanceState[Any], _InstanceDict]]
Iterate through instances related to the given instance for a particular ‘cascade’, starting with this MapperProperty.
Return an iterator3-tuples (instance, mapper, state).
Note that the ‘cascade’ collection on this MapperProperty is checked first for the given type before cascade_iterator is called.
This method typically only applies to Relationship.
attribute sqlalchemy.orm.MapperProperty.class_attribute
Return the class-bound descriptor corresponding to this MapperProperty.
This is basically a getattr() call:
return getattr(self.parent.class_, self.key)
I.e. if this MapperProperty were named addresses, and the class to which it is mapped is User, this sequence is possible:
from sqlalchemy import inspect
mapper = inspect(User)
addresses_property = mapper.attrs.addresses
addresses_property.class_attribute is User.addresses
True
User.addresses.property is addresses_property
True
attribute sqlalchemy.orm.MapperProperty.comparator: PropComparator[_T]
The PropComparator instance that implements SQL expression construction on behalf of this mapped attribute.
method sqlalchemy.orm.MapperProperty.create_row_processor(context: ORMCompileState, query_entity: _MapperEntity, path: AbstractEntityRegistry, mapper: Mapper[Any], result: Result[Any], adapter: ORMAdapter | None, populators: _PopulatorDict) ? None
Produce row processing functions and append to the given set of populators lists.
method sqlalchemy.orm.MapperProperty.do_init() ? None
Perform subclass-specific initialization post-mapper-creation steps.
This is a template method called by the MapperProperty object’s init() method.
attribute sqlalchemy.orm.MapperProperty.doc: str | None
optional documentation string
attribute sqlalchemy.orm.MapperProperty.info: _InfoType
Info dictionary associated with the object, allowing user-defined data to be associated with this InspectionAttr.
The dictionary is generated when first accessed. Alternatively, it can be specified as a constructor argument to the column_property(), relationship(), or composite() functions.
See also
QueryableAttribute.info
SchemaItem.info
method sqlalchemy.orm.MapperProperty.init() ? None
Called after all mappers are created to assemble relationships between mappers and perform other post-mapper-creation initialization steps.
method sqlalchemy.orm.MapperProperty.instrument_class(mapper: Mapper[Any]) ? None
Hook called by the Mapper to the property to initiate instrumentation of the class attribute managed by this MapperProperty.
The MapperProperty here will typically call out to the attributes module to set up an InstrumentedAttribute.
This step is the first of two steps to set up an InstrumentedAttribute, and is called early in the mapper setup process.
The second step is typically the init_class_attribute step, called from StrategizedProperty via the post_instrument_class() hook. This step assigns additional state to the InstrumentedAttribute (specifically the “impl”) which has been determined after the MapperProperty has determined what kind of persistence management it needs to do (e.g. scalar, object, collection, etc).
attribute sqlalchemy.orm.MapperProperty.is_property = True
Part of the InspectionAttr interface; states this object is a mapper property.
attribute sqlalchemy.orm.MapperProperty.key: str
name of class attribute
method sqlalchemy.orm.MapperProperty.merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) ? None
Merge the attribute represented by this MapperProperty from source to destination object.
attribute sqlalchemy.orm.MapperProperty.parent: Mapper[Any]
the Mapper managing this property.
method sqlalchemy.orm.MapperProperty.post_instrument_class(mapper: Mapper[Any]) ? None
Perform instrumentation adjustments that need to occur after init() has completed.
The given Mapper is the Mapper invoking the operation, which may not be the same Mapper as self.parent in an inheritance scenario; however, Mapper will always at least be a sub-mapper of self.parent.
This method is typically used by StrategizedProperty, which delegates it to LoaderStrategy.init_class_attribute() to perform final setup on the class-bound InstrumentedAttribute.
method sqlalchemy.orm.MapperProperty.set_parent(parent: Mapper[Any], init: bool) ? None
Set the parent mapper that references this MapperProperty.
This method is overridden by some subclasses to perform extra setup when the mapper is first known.
method sqlalchemy.orm.MapperProperty.setup(context: ORMCompileState, query_entity: _MapperEntity, path: AbstractEntityRegistry, adapter: ORMAdapter | None, **kwargs: Any) ? None
Called by Query for the purposes of constructing a SQL statement.
Each MapperProperty associated with the target mapper processes the statement referenced by the query context, adding columns and/or criterion as appropriate.
class sqlalchemy.orm.MappedSQLExpression
Declarative front-end for the ColumnProperty class.
Public constructor is the column_property() function.
Changed in version 2.0: Added MappedSQLExpression as a Declarative compatible subclass for ColumnProperty.
See also
MappedColumn
Class signature
class sqlalchemy.orm.MappedSQLExpression (sqlalchemy.orm.properties.ColumnProperty, sqlalchemy.orm.base._DeclarativeMapped)
class sqlalchemy.orm.InspectionAttrExtensionType
Symbols indicating the type of extension that a InspectionAttr is part of.
Class signature
class sqlalchemy.orm.InspectionAttrExtensionType (enum.Enum)
class sqlalchemy.orm.NotExtension
Members
NOT_EXTENSION
Class signature
class sqlalchemy.orm.NotExtension (sqlalchemy.orm.base.InspectionAttrExtensionType)
attribute sqlalchemy.orm.NotExtension.NOT_EXTENSION = 'not_extension'
Symbol indicating an InspectionAttr that’s not part of sqlalchemy.ext.
Is assigned to the InspectionAttr.extension_type attribute.
function sqlalchemy.orm.merge_result(query: Query[Any], iterator: FrozenResult | Iterable[Sequence[Any]] | Iterable[object], load: bool = True) ? FrozenResult | Iterable[Any]
Merge a result into the given Query object’s Session.
Deprecated since version 2.0: The merge_result() function is considered legacy as of the 1.x series of SQLAlchemy and becomes a legacy construct in 2.0. The function as well as the method on Query is superseded by the merge_frozen_result() function. (Background on SQLAlchemy 2.0 at: SQLAlchemy 2.0 - Major Migration Guide)
See Query.merge_result() for top-level documentation on this function.
function sqlalchemy.orm.merge_frozen_result(session, statement, frozen_result, load=True)
Merge a FrozenResult back into a Session, returning a new Result object with persistent objects.
See the section Re-Executing Statements for an example.
See also
Re-Executing Statements
Result.freeze()
FrozenResult
class sqlalchemy.orm.PropComparator
Defines SQL operations for ORM mapped attributes.
SQLAlchemy allows for operators to be redefined at both the Core and ORM level. PropComparator is the base class of operator redefinition for ORM-level operations, including those of ColumnProperty, Relationship, and Composite.
User-defined subclasses of PropComparator may be created. The built-in Python comparison and math operator methods, such as ColumnOperators.__eq__(), ColumnOperators.__lt__(), and ColumnOperators.__add__(), can be overridden to provide new operator behavior. The custom PropComparator is passed to the MapperProperty instance via the comparator_factory argument. In each case, the appropriate subclass of PropComparator should be used:
# definition of custom PropComparator subclasses

from sqlalchemy.orm.properties import (
    ColumnProperty,
    Composite,
    Relationship,
)


class MyColumnComparator(ColumnProperty.Comparator):
    def __eq__(self, other):
        return self.__clause_element__() == other


class MyRelationshipComparator(Relationship.Comparator):
    def any(self, expression):
        "define the 'any' operation"
        # 


class MyCompositeComparator(Composite.Comparator):
    def __gt__(self, other):
        "redefine the 'greater than' operation"

        return sql.and_(
            *[
                a > b
                for a, b in zip(
                    self.__clause_element__().clauses,
                    other.__composite_values__(),
                )
            ]
        )


# application of custom PropComparator subclasses

from sqlalchemy.orm import column_property, relationship, composite
from sqlalchemy import Column, String


class SomeMappedClass(Base):
    some_column = column_property(
        Column("some_column", String),
        comparator_factory=MyColumnComparator,
    )

    some_relationship = relationship(
        SomeOtherClass, comparator_factory=MyRelationshipComparator
    )

    some_composite = composite(
        Column("a", String),
        Column("b", String),
        comparator_factory=MyCompositeComparator,
    )
Note that for column-level operator redefinition, it’s usually simpler to define the operators at the Core level, using the TypeEngine.comparator_factory attribute. See Redefining and Creating New Operators for more detail.
See also
Comparator
Comparator
Comparator
ColumnOperators
Redefining and Creating New Operators
TypeEngine.comparator_factory
Members
__eq__(), __le__(), __lt__(), __ne__(), adapt_to_entity(), adapter, all_(), and_(), any(), any_(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), of_type(), op(), operate(), property, regexp_match(), regexp_replace(), reverse_operate(), startswith(), timetuple
Class signature
class sqlalchemy.orm.PropComparator (sqlalchemy.orm.base.SQLORMOperations, typing.Generic, sqlalchemy.sql.expression.ColumnOperators)
method sqlalchemy.orm.PropComparator.__eq__(other: Any) ? ColumnOperators
inherited from the sqlalchemy.sql.expression.ColumnOperators.__eq__ method of ColumnOperators
Implement the == operator.
In a column context, produces the clause a = b. If the target is None, produces a IS NULL.
method sqlalchemy.orm.PropComparator.__le__(other: Any) ? ColumnOperators
inherited from the sqlalchemy.sql.expression.ColumnOperators.__le__ method of ColumnOperators
Implement the <= operator.
In a column context, produces the clause a <= b.
method sqlalchemy.orm.PropComparator.__lt__(other: Any) ? ColumnOperators
inherited from the sqlalchemy.sql.expression.ColumnOperators.__lt__ method of ColumnOperators
Implement the < operator.
In a column context, produces the clause a < b.
method sqlalchemy.orm.PropComparator.__ne__(other: Any) ? ColumnOperators
inherited from the sqlalchemy.sql.expression.ColumnOperators.__ne__ method of ColumnOperators
Implement the != operator.
In a column context, produces the clause a != b. If the target is None, produces a IS NOT NULL.
method sqlalchemy.orm.PropComparator.adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) ? PropComparator[_T_co]
Return a copy of this PropComparator which will use the given AliasedInsp to produce corresponding expressions.
attribute sqlalchemy.orm.PropComparator.adapter
Produce a callable that adapts column expressions to suit an aliased version of this comparator.
method sqlalchemy.orm.PropComparator.all_() ? ColumnOperators
inherited from the ColumnOperators.all_() method of ColumnOperators
Produce an all_() clause against the parent object.
See the documentation for all_() for examples.
Note
be sure to not confuse the newer ColumnOperators.all_() method with the legacy version of this method, the Comparator.all() method that’s specific to ARRAY, which uses a different calling style.
method sqlalchemy.orm.PropComparator.and_(*criteria: _ColumnExpressionArgument[bool]) ? PropComparator[bool]
Add additional criteria to the ON clause that’s represented by this relationship attribute.
E.g.:
stmt = select(User).join(
    User.addresses.and_(Address.email_address != "foo")
)

stmt = select(User).options(
    joinedload(User.addresses.and_(Address.email_address != "foo"))
)
Added in version 1.4.
See also
Combining Relationship with Custom ON Criteria
Adding Criteria to loader options
with_loader_criteria()
method sqlalchemy.orm.PropComparator.any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) ? ColumnElement[bool]
Return a SQL expression representing true if this element references a member which meets the given criterion.
The usual implementation of any() is Comparator.any().
Parameters:
* criterion – an optional ClauseElement formulated against the member class’ table or attributes.
* **kwargs – key/value pairs corresponding to member class attribute names which will be compared via equality to the corresponding values.
method sqlalchemy.orm.PropComparator.any_() ? ColumnOperators
inherited from the ColumnOperators.any_() method of ColumnOperators
Produce an any_() clause against the parent object.
See the documentation for any_() for examples.
Note
be sure to not confuse the newer ColumnOperators.any_() method with the legacy version of this method, the Comparator.any() method that’s specific to ARRAY, which uses a different calling style.
method sqlalchemy.orm.PropComparator.asc() ? ColumnOperators
inherited from the ColumnOperators.asc() method of ColumnOperators
Produce a asc() clause against the parent object.
method sqlalchemy.orm.PropComparator.between(cleft: Any, cright: Any, symmetric: bool = False) ? ColumnOperators
inherited from the ColumnOperators.between() method of ColumnOperators
Produce a between() clause against the parent object, given the lower and upper range.
method sqlalchemy.orm.PropComparator.bitwise_and(other: Any) ? ColumnOperators
inherited from the ColumnOperators.bitwise_and() method of ColumnOperators
Produce a bitwise AND operation, typically via the & operator.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bitwise_lshift(other: Any) ? ColumnOperators
inherited from the ColumnOperators.bitwise_lshift() method of ColumnOperators
Produce a bitwise LSHIFT operation, typically via the << operator.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bitwise_not() ? ColumnOperators
inherited from the ColumnOperators.bitwise_not() method of ColumnOperators
Produce a bitwise NOT operation, typically via the ~ operator.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bitwise_or(other: Any) ? ColumnOperators
inherited from the ColumnOperators.bitwise_or() method of ColumnOperators
Produce a bitwise OR operation, typically via the | operator.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bitwise_rshift(other: Any) ? ColumnOperators
inherited from the ColumnOperators.bitwise_rshift() method of ColumnOperators
Produce a bitwise RSHIFT operation, typically via the >> operator.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bitwise_xor(other: Any) ? ColumnOperators
inherited from the ColumnOperators.bitwise_xor() method of ColumnOperators
Produce a bitwise XOR operation, typically via the ^ operator, or # for PostgreSQL.
Added in version 2.0.2.
See also
Bitwise Operators
method sqlalchemy.orm.PropComparator.bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[], Any] | None = None) ? Callable[[Any], Operators]
inherited from the Operators.bool_op() method of Operators
Return a custom boolean operator.
This method is shorthand for calling Operators.op() and passing the Operators.op.is_comparison flag with True. A key advantage to using Operators.bool_op() is that when using column constructs, the “boolean” nature of the returned expression will be present for PEP 484 purposes.
See also
Operators.op()
method sqlalchemy.orm.PropComparator.collate(collation: str) ? ColumnOperators
inherited from the ColumnOperators.collate() method of ColumnOperators
Produce a collate() clause against the parent object, given the collation string.
See also
collate()
method sqlalchemy.orm.PropComparator.concat(other: Any) ? ColumnOperators
inherited from the ColumnOperators.concat() method of ColumnOperators
Implement the ‘concat’ operator.
In a column context, produces the clause a || b, or uses the concat() operator on MySQL.
method sqlalchemy.orm.PropComparator.contains(other: Any, **kw: Any) ? ColumnOperators
inherited from the ColumnOperators.contains() method of ColumnOperators
Implement the ‘contains’ operator.
Produces a LIKE expression that tests against a match for the middle of a string value:
column LIKE '%' || <other> || '%'
E.g.:
stmt = select(sometable).where(sometable.c.column.contains("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.contains.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.contains.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.contains.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.contains("foo%bar", autoescape=True)
Will render as:
somecolumn LIKE '%' || :param || '%' ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.contains("foo/%bar", escape="^")
Will render as:
somecolumn LIKE '%' || :param || '%' ESCAPE '^'
The parameter may also be combined with ColumnOperators.contains.autoescape:
somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.startswith()
ColumnOperators.endswith()
ColumnOperators.like()
method sqlalchemy.orm.PropComparator.desc() ? ColumnOperators
inherited from the ColumnOperators.desc() method of ColumnOperators
Produce a desc() clause against the parent object.
method sqlalchemy.orm.PropComparator.distinct() ? ColumnOperators
inherited from the ColumnOperators.distinct() method of ColumnOperators
Produce a distinct() clause against the parent object.
method sqlalchemy.orm.PropComparator.endswith(other: Any, escape: str | None = None, autoescape: bool = False) ? ColumnOperators
inherited from the ColumnOperators.endswith() method of ColumnOperators
Implement the ‘endswith’ operator.
Produces a LIKE expression that tests against a match for the end of a string value:
column LIKE '%' || <other>
E.g.:
stmt = select(sometable).where(sometable.c.column.endswith("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.endswith.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.endswith.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.endswith.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.endswith("foo%bar", autoescape=True)
Will render as:
somecolumn LIKE '%' || :param ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.endswith("foo/%bar", escape="^")
Will render as:
somecolumn LIKE '%' || :param ESCAPE '^'
The parameter may also be combined with ColumnOperators.endswith.autoescape:
somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.startswith()
ColumnOperators.contains()
ColumnOperators.like()
method sqlalchemy.orm.PropComparator.has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) ? ColumnElement[bool]
Return a SQL expression representing true if this element references a member which meets the given criterion.
The usual implementation of has() is Comparator.has().
Parameters:
* criterion – an optional ClauseElement formulated against the member class’ table or attributes.
* **kwargs – key/value pairs corresponding to member class attribute names which will be compared via equality to the corresponding values.
method sqlalchemy.orm.PropComparator.icontains(other: Any, **kw: Any) ? ColumnOperators
inherited from the ColumnOperators.icontains() method of ColumnOperators
Implement the icontains operator, e.g. case insensitive version of ColumnOperators.contains().
Produces a LIKE expression that tests against an insensitive match for the middle of a string value:
lower(column) LIKE '%' || lower(<other>) || '%'
E.g.:
stmt = select(sometable).where(sometable.c.column.icontains("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.icontains.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.icontains.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.icontains.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.icontains("foo%bar", autoescape=True)
Will render as:
lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.icontains("foo/%bar", escape="^")
Will render as:
lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
The parameter may also be combined with ColumnOperators.contains.autoescape:
somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.contains()
method sqlalchemy.orm.PropComparator.iendswith(other: Any, escape: str | None = None, autoescape: bool = False) ? ColumnOperators
inherited from the ColumnOperators.iendswith() method of ColumnOperators
Implement the iendswith operator, e.g. case insensitive version of ColumnOperators.endswith().
Produces a LIKE expression that tests against an insensitive match for the end of a string value:
lower(column) LIKE '%' || lower(<other>)
E.g.:
stmt = select(sometable).where(sometable.c.column.iendswith("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.iendswith.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.iendswith.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.iendswith.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.iendswith("foo%bar", autoescape=True)
Will render as:
lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.iendswith("foo/%bar", escape="^")
Will render as:
lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
The parameter may also be combined with ColumnOperators.iendswith.autoescape:
somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.endswith()
method sqlalchemy.orm.PropComparator.ilike(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.ilike() method of ColumnOperators
Implement the ilike operator, e.g. case insensitive LIKE.
In a column context, produces an expression either of the form:
lower(a) LIKE lower(other)
Or on backends that support the ILIKE operator:
a ILIKE other
E.g.:
stmt = select(sometable).where(sometable.c.column.ilike("%foobar%"))
Parameters:
* other – expression to be compared
* escape – 
optional escape character, renders the ESCAPE keyword, e.g.:
somecolumn.ilike("foo/%bar", escape="/")
* 
See also
ColumnOperators.like()
method sqlalchemy.orm.PropComparator.in_(other: Any) ? ColumnOperators
inherited from the ColumnOperators.in_() method of ColumnOperators
Implement the in operator.
In a column context, produces the clause column IN <other>.
The given parameter other may be:
* A list of literal values, e.g.:
stmt.where(column.in_([1, 2, 3]))
In this calling form, the list of items is converted to a set of bound parameters the same length as the list given:
WHERE COL IN (?, ?, ?)
A list of tuples may be provided if the comparison is against a tuple_() containing multiple expressions:
from sqlalchemy import tuple_

stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
An empty list, e.g.:
stmt.where(column.in_([]))
In this calling form, the expression renders an “empty set” expression. These expressions are tailored to individual backends and are generally trying to get an empty SELECT statement as a subquery. Such as on SQLite, the expression is:
WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
*  Changed in version 1.4: empty IN expressions now use an execution-time generated SELECT subquery in all cases.
*  A bound parameter, e.g. bindparam(), may be used if it includes the bindparam.expanding flag:
stmt.where(column.in_(bindparam("value", expanding=True)))
In this calling form, the expression renders a special non-SQL placeholder expression that looks like:
WHERE COL IN ([EXPANDING_value])
This placeholder expression is intercepted at statement execution time to be converted into the variable number of bound parameter form illustrated earlier. If the statement were executed as:
connection.execute(stmt, {"value": [1, 2, 3]})
The database would be passed a bound parameter for each value:
WHERE COL IN (?, ?, ?)
Added in version 1.2: added “expanding” bound parameters
If an empty list is passed, a special “empty list” expression, which is specific to the database in use, is rendered. On SQLite this would be:
WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
*  Added in version 1.3: “expanding” bound parameters now support empty lists
*  a select() construct, which is usually a correlated scalar select:
stmt.where(
    column.in_(select(othertable.c.y).where(table.c.x == othertable.c.x))
)
In this calling form, ColumnOperators.in_() renders as given:
WHERE COL IN (SELECT othertable.y
FROM othertable WHERE othertable.x = table.x)
Parameters:
other – a list of literals, a select() construct, or a bindparam() construct that includes the bindparam.expanding flag set to True.
method sqlalchemy.orm.PropComparator.is_(other: Any) ? ColumnOperators
inherited from the ColumnOperators.is_() method of ColumnOperators
Implement the IS operator.
Normally, IS is generated automatically when comparing to a value of None, which resolves to NULL. However, explicit usage of IS may be desirable if comparing to boolean values on certain platforms.
See also
ColumnOperators.is_not()
method sqlalchemy.orm.PropComparator.is_distinct_from(other: Any) ? ColumnOperators
inherited from the ColumnOperators.is_distinct_from() method of ColumnOperators
Implement the IS DISTINCT FROM operator.
Renders “a IS DISTINCT FROM b” on most platforms; on some such as SQLite may render “a IS NOT b”.
method sqlalchemy.orm.PropComparator.is_not(other: Any) ? ColumnOperators
inherited from the ColumnOperators.is_not() method of ColumnOperators
Implement the IS NOT operator.
Normally, IS NOT is generated automatically when comparing to a value of None, which resolves to NULL. However, explicit usage of IS NOT may be desirable if comparing to boolean values on certain platforms.
Changed in version 1.4: The is_not() operator is renamed from isnot() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.is_()
method sqlalchemy.orm.PropComparator.is_not_distinct_from(other: Any) ? ColumnOperators
inherited from the ColumnOperators.is_not_distinct_from() method of ColumnOperators
Implement the IS NOT DISTINCT FROM operator.
Renders “a IS NOT DISTINCT FROM b” on most platforms; on some such as SQLite may render “a IS b”.
Changed in version 1.4: The is_not_distinct_from() operator is renamed from isnot_distinct_from() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.isnot(other: Any) ? ColumnOperators
inherited from the ColumnOperators.isnot() method of ColumnOperators
Implement the IS NOT operator.
Normally, IS NOT is generated automatically when comparing to a value of None, which resolves to NULL. However, explicit usage of IS NOT may be desirable if comparing to boolean values on certain platforms.
Changed in version 1.4: The is_not() operator is renamed from isnot() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.is_()
method sqlalchemy.orm.PropComparator.isnot_distinct_from(other: Any) ? ColumnOperators
inherited from the ColumnOperators.isnot_distinct_from() method of ColumnOperators
Implement the IS NOT DISTINCT FROM operator.
Renders “a IS NOT DISTINCT FROM b” on most platforms; on some such as SQLite may render “a IS b”.
Changed in version 1.4: The is_not_distinct_from() operator is renamed from isnot_distinct_from() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.istartswith(other: Any, escape: str | None = None, autoescape: bool = False) ? ColumnOperators
inherited from the ColumnOperators.istartswith() method of ColumnOperators
Implement the istartswith operator, e.g. case insensitive version of ColumnOperators.startswith().
Produces a LIKE expression that tests against an insensitive match for the start of a string value:
lower(column) LIKE lower(<other>) || '%'
E.g.:
stmt = select(sometable).where(sometable.c.column.istartswith("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.istartswith.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.istartswith.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.istartswith.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.istartswith("foo%bar", autoescape=True)
Will render as:
lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.istartswith("foo/%bar", escape="^")
Will render as:
lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
The parameter may also be combined with ColumnOperators.istartswith.autoescape:
somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.startswith()
method sqlalchemy.orm.PropComparator.like(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.like() method of ColumnOperators
Implement the like operator.
In a column context, produces the expression:
a LIKE other
E.g.:
stmt = select(sometable).where(sometable.c.column.like("%foobar%"))
Parameters:
* other – expression to be compared
* escape – 
optional escape character, renders the ESCAPE keyword, e.g.:
somecolumn.like("foo/%bar", escape="/")
* 
See also
ColumnOperators.ilike()
method sqlalchemy.orm.PropComparator.match(other: Any, **kwargs: Any) ? ColumnOperators
inherited from the ColumnOperators.match() method of ColumnOperators
Implements a database-specific ‘match’ operator.
ColumnOperators.match() attempts to resolve to a MATCH-like function or operator provided by the backend. Examples include:
* PostgreSQL - renders x @@ plainto_tsquery(y)
Changed in version 2.0: plainto_tsquery() is used instead of to_tsquery() for PostgreSQL now; for compatibility with other forms, see Full Text Search.
* MySQL - renders MATCH (x) AGAINST (y IN BOOLEAN MODE)
See also
match - MySQL specific construct with additional features.
* Oracle Database - renders CONTAINS(x, y)
* other backends may provide special implementations.
* Backends without any special implementation will emit the operator as “MATCH”. This is compatible with SQLite, for example.
method sqlalchemy.orm.PropComparator.not_ilike(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.not_ilike() method of ColumnOperators
implement the NOT ILIKE operator.
This is equivalent to using negation with ColumnOperators.ilike(), i.e. ~x.ilike(y).
Changed in version 1.4: The not_ilike() operator is renamed from notilike() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.ilike()
method sqlalchemy.orm.PropComparator.not_in(other: Any) ? ColumnOperators
inherited from the ColumnOperators.not_in() method of ColumnOperators
implement the NOT IN operator.
This is equivalent to using negation with ColumnOperators.in_(), i.e. ~x.in_(y).
In the case that other is an empty sequence, the compiler produces an “empty not in” expression. This defaults to the expression “1 = 1” to produce true in all cases. The create_engine.empty_in_strategy may be used to alter this behavior.
Changed in version 1.4: The not_in() operator is renamed from notin_() in previous releases. The previous name remains available for backwards compatibility.
Changed in version 1.2: The ColumnOperators.in_() and ColumnOperators.not_in() operators now produce a “static” expression for an empty IN sequence by default.
See also
ColumnOperators.in_()
method sqlalchemy.orm.PropComparator.not_like(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.not_like() method of ColumnOperators
implement the NOT LIKE operator.
This is equivalent to using negation with ColumnOperators.like(), i.e. ~x.like(y).
Changed in version 1.4: The not_like() operator is renamed from notlike() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.like()
method sqlalchemy.orm.PropComparator.notilike(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.notilike() method of ColumnOperators
implement the NOT ILIKE operator.
This is equivalent to using negation with ColumnOperators.ilike(), i.e. ~x.ilike(y).
Changed in version 1.4: The not_ilike() operator is renamed from notilike() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.ilike()
method sqlalchemy.orm.PropComparator.notin_(other: Any) ? ColumnOperators
inherited from the ColumnOperators.notin_() method of ColumnOperators
implement the NOT IN operator.
This is equivalent to using negation with ColumnOperators.in_(), i.e. ~x.in_(y).
In the case that other is an empty sequence, the compiler produces an “empty not in” expression. This defaults to the expression “1 = 1” to produce true in all cases. The create_engine.empty_in_strategy may be used to alter this behavior.
Changed in version 1.4: The not_in() operator is renamed from notin_() in previous releases. The previous name remains available for backwards compatibility.
Changed in version 1.2: The ColumnOperators.in_() and ColumnOperators.not_in() operators now produce a “static” expression for an empty IN sequence by default.
See also
ColumnOperators.in_()
method sqlalchemy.orm.PropComparator.notlike(other: Any, escape: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.notlike() method of ColumnOperators
implement the NOT LIKE operator.
This is equivalent to using negation with ColumnOperators.like(), i.e. ~x.like(y).
Changed in version 1.4: The not_like() operator is renamed from notlike() in previous releases. The previous name remains available for backwards compatibility.
See also
ColumnOperators.like()
method sqlalchemy.orm.PropComparator.nulls_first() ? ColumnOperators
inherited from the ColumnOperators.nulls_first() method of ColumnOperators
Produce a nulls_first() clause against the parent object.
Changed in version 1.4: The nulls_first() operator is renamed from nullsfirst() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.nulls_last() ? ColumnOperators
inherited from the ColumnOperators.nulls_last() method of ColumnOperators
Produce a nulls_last() clause against the parent object.
Changed in version 1.4: The nulls_last() operator is renamed from nullslast() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.nullsfirst() ? ColumnOperators
inherited from the ColumnOperators.nullsfirst() method of ColumnOperators
Produce a nulls_first() clause against the parent object.
Changed in version 1.4: The nulls_first() operator is renamed from nullsfirst() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.nullslast() ? ColumnOperators
inherited from the ColumnOperators.nullslast() method of ColumnOperators
Produce a nulls_last() clause against the parent object.
Changed in version 1.4: The nulls_last() operator is renamed from nullslast() in previous releases. The previous name remains available for backwards compatibility.
method sqlalchemy.orm.PropComparator.of_type(class_: _EntityType[Any]) ? PropComparator[_T_co]
Redefine this object in terms of a polymorphic subclass, with_polymorphic() construct, or aliased() construct.
Returns a new PropComparator from which further criterion can be evaluated.
e.g.:
query.join(Company.employees.of_type(Engineer)).filter(
    Engineer.name == "foo"
)
Parameters:
class_ – a class or mapper indicating that criterion will be against this specific subclass.
See also
Using Relationship to join between aliased targets - in the ORM Querying Guide
Joining to specific sub-types or with_polymorphic() entities
method sqlalchemy.orm.PropComparator.op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[, Any] | None = None) ? Callable[[Any], Operators]
inherited from the Operators.op() method of Operators
Produce a generic operator function.
e.g.:
somecolumn.op("*")(5)
produces:
somecolumn * 5
This function can also be used to make bitwise operators explicit. For example:
somecolumn.op("&")(0xFF)
is a bitwise AND of the value in somecolumn.
Parameters:
* opstring – a string which will be output as the infix operator between this element and the expression passed to the generated function.
* precedence – 
precedence which the database is expected to apply to the operator in SQL expressions. This integer value acts as a hint for the SQL compiler to know when explicit parenthesis should be rendered around a particular operation. A lower number will cause the expression to be parenthesized when applied against another operator with higher precedence. The default value of 0 is lower than all operators except for the comma (,) and AS operators. A value of 100 will be higher or equal to all operators, and -100 will be lower than or equal to all operators.
See also
I’m using op() to generate a custom operator and my parenthesis are not coming out correctly - detailed description of how the SQLAlchemy SQL compiler renders parenthesis
* is_comparison – 
legacy; if True, the operator will be considered as a “comparison” operator, that is which evaluates to a boolean true/false value, like ==, >, etc. This flag is provided so that ORM relationships can establish that the operator is a comparison operator when used in a custom join condition.
Using the is_comparison parameter is superseded by using the Operators.bool_op() method instead; this more succinct operator sets this parameter automatically, but also provides correct PEP 484 typing support as the returned object will express a “boolean” datatype, i.e. BinaryExpression[bool].
* return_type – a TypeEngine class or object that will force the return type of an expression produced by this operator to be of that type. By default, operators that specify Operators.op.is_comparison will resolve to Boolean, and those that do not will be of the same type as the left-hand operand.
* python_impl – 
an optional Python function that can evaluate two Python values in the same way as this operator works when run on the database server. Useful for in-Python SQL expression evaluation functions, such as for ORM hybrid attributes, and the ORM “evaluator” used to match objects in a session after a multi-row update or delete.
e.g.:
expr = column("x").op("+", python_impl=lambda a, b: a + b)("y")
The operator for the above expression will also work for non-SQL left and right objects:
expr.operator(5, 10)
15
* Added in version 2.0.
See also
Operators.bool_op()
Redefining and Creating New Operators
Using custom operators in join conditions
method sqlalchemy.orm.PropComparator.operate(op: OperatorType, *other: Any, **kwargs: Any) ? Operators
inherited from the Operators.operate() method of Operators
Operate on an argument.
This is the lowest level of operation, raises NotImplementedError by default.
Overriding this on a subclass can allow common behavior to be applied to all operations. For example, overriding ColumnOperators to apply func.lower() to the left and right side:
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
Parameters:
* op – Operator callable.
* *other – the ‘other’ side of the operation. Will be a single scalar for most operations.
* **kwargs – modifiers. These may be passed by special operators such as ColumnOperators.contains().
attribute sqlalchemy.orm.PropComparator.property
Return the MapperProperty associated with this PropComparator.
Return values here will commonly be instances of ColumnProperty or Relationship.
method sqlalchemy.orm.PropComparator.regexp_match(pattern: Any, flags: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.regexp_match() method of ColumnOperators
Implements a database-specific ‘regexp match’ operator.
E.g.:
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match("^(b|c)")
)
ColumnOperators.regexp_match() attempts to resolve to a REGEXP-like function or operator provided by the backend, however the specific regular expression syntax and flags available are not backend agnostic.
Examples include:
* PostgreSQL - renders x ~ y or x !~ y when negated.
* Oracle Database - renders REGEXP_LIKE(x, y)
* SQLite - uses SQLite’s REGEXP placeholder operator and calls into the Python re.match() builtin.
* other backends may provide special implementations.
* Backends without any special implementation will emit the operator as “REGEXP” or “NOT REGEXP”. This is compatible with SQLite and MySQL, for example.
Regular expression support is currently implemented for Oracle Database, PostgreSQL, MySQL and MariaDB. Partial support is available for SQLite. Support among third-party dialects may vary.
Parameters:
* pattern – The regular expression pattern string or column clause.
* flags – Any regular expression string flags to apply, passed as plain Python string only. These flags are backend specific. Some backends, like PostgreSQL and MariaDB, may alternatively specify the flags as part of the pattern. When using the ignore case flag ‘i’ in PostgreSQL, the ignore case regexp match operator ~* or !~* will be used.
Added in version 1.4.
Changed in version 1.4.48,: 2.0.18 Note that due to an implementation error, the “flags” parameter previously accepted SQL expression objects such as column expressions in addition to plain Python strings. This implementation did not work correctly with caching and was removed; strings only should be passed for the “flags” parameter, as these flags are rendered as literal inline values within SQL expressions.
See also
ColumnOperators.regexp_replace()
method sqlalchemy.orm.PropComparator.regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) ? ColumnOperators
inherited from the ColumnOperators.regexp_replace() method of ColumnOperators
Implements a database-specific ‘regexp replace’ operator.
E.g.:
stmt = select(
    table.c.some_column.regexp_replace("b(..)", "XY", flags="g")
)
ColumnOperators.regexp_replace() attempts to resolve to a REGEXP_REPLACE-like function provided by the backend, that usually emit the function REGEXP_REPLACE(). However, the specific regular expression syntax and flags available are not backend agnostic.
Regular expression replacement support is currently implemented for Oracle Database, PostgreSQL, MySQL 8 or greater and MariaDB. Support among third-party dialects may vary.
Parameters:
* pattern – The regular expression pattern string or column clause.
* pattern – The replacement string or column clause.
* flags – Any regular expression string flags to apply, passed as plain Python string only. These flags are backend specific. Some backends, like PostgreSQL and MariaDB, may alternatively specify the flags as part of the pattern.
Added in version 1.4.
Changed in version 1.4.48,: 2.0.18 Note that due to an implementation error, the “flags” parameter previously accepted SQL expression objects such as column expressions in addition to plain Python strings. This implementation did not work correctly with caching and was removed; strings only should be passed for the “flags” parameter, as these flags are rendered as literal inline values within SQL expressions.
See also
ColumnOperators.regexp_match()
method sqlalchemy.orm.PropComparator.reverse_operate(op: OperatorType, other: Any, **kwargs: Any) ? Operators
inherited from the Operators.reverse_operate() method of Operators
Reverse operate on an argument.
Usage is the same as operate().
method sqlalchemy.orm.PropComparator.startswith(other: Any, escape: str | None = None, autoescape: bool = False) ? ColumnOperators
inherited from the ColumnOperators.startswith() method of ColumnOperators
Implement the startswith operator.
Produces a LIKE expression that tests against a match for the start of a string value:
column LIKE <other> || '%'
E.g.:
stmt = select(sometable).where(sometable.c.column.startswith("foobar"))
Since the operator uses LIKE, wildcard characters "%" and "_" that are present inside the <other> expression will behave like wildcards as well. For literal string values, the ColumnOperators.startswith.autoescape flag may be set to True to apply escaping to occurrences of these characters within the string value so that they match as themselves and not as wildcard characters. Alternatively, the ColumnOperators.startswith.escape parameter will establish a given character as an escape character which can be of use when the target expression is not a literal string.
Parameters:
* other – expression to be compared. This is usually a plain string value, but can also be an arbitrary SQL expression. LIKE wildcard characters % and _ are not escaped by default unless the ColumnOperators.startswith.autoescape flag is set to True.
* autoescape – 
boolean; when True, establishes an escape character within the LIKE expression, then applies it to all occurrences of "%", "_" and the escape character itself within the comparison value, which is assumed to be a literal string and not a SQL expression.
An expression such as:
somecolumn.startswith("foo%bar", autoescape=True)
Will render as:
somecolumn LIKE :param || '%' ESCAPE '/'
*  With the value of :param as "foo/%bar".
*  escape – 
a character which when given will render with the ESCAPE keyword to establish that character as the escape character. This character can then be placed preceding occurrences of % and _ to allow them to act as themselves and not wildcard characters.
An expression such as:
somecolumn.startswith("foo/%bar", escape="^")
Will render as:
somecolumn LIKE :param || '%' ESCAPE '^'
The parameter may also be combined with ColumnOperators.startswith.autoescape:
somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
* Where above, the given literal parameter will be converted to "foo^%bar^^bat" before being passed to the database.
See also
ColumnOperators.endswith()
ColumnOperators.contains()
ColumnOperators.like()
attribute sqlalchemy.orm.PropComparator.timetuple: Literal[None] = None
inherited from the ColumnOperators.timetuple attribute of ColumnOperators
Hack, allows datetime objects to be compared on the LHS.
class sqlalchemy.orm.Relationship
Describes an object property that holds a single item or list of items that correspond to a related database table.
Public constructor is the relationship() function.
See also
Relationship Configuration
Changed in version 2.0: Added Relationship as a Declarative compatible subclass for RelationshipProperty.
Class signature
class sqlalchemy.orm.Relationship (sqlalchemy.orm.RelationshipProperty, sqlalchemy.orm.base._DeclarativeMapped)
class sqlalchemy.orm.RelationshipDirection
enumeration which indicates the ‘direction’ of a RelationshipProperty.
RelationshipDirection is accessible from the Relationship.direction attribute of RelationshipProperty.
Members
MANYTOMANY, MANYTOONE, ONETOMANY
Class signature
class sqlalchemy.orm.RelationshipDirection (enum.Enum)
attribute sqlalchemy.orm.RelationshipDirection.MANYTOMANY = 3
Indicates the many-to-many direction for a relationship().
This symbol is typically used by the internals but may be exposed within certain API features.
attribute sqlalchemy.orm.RelationshipDirection.MANYTOONE = 2
Indicates the many-to-one direction for a relationship().
This symbol is typically used by the internals but may be exposed within certain API features.
attribute sqlalchemy.orm.RelationshipDirection.ONETOMANY = 1
Indicates the one-to-many direction for a relationship().
This symbol is typically used by the internals but may be exposed within certain API features.
class sqlalchemy.orm.RelationshipProperty
Describes an object property that holds a single item or list of items that correspond to a related database table.
Public constructor is the relationship() function.
See also
Relationship Configuration
Members
__eq__(), __init__(), __ne__(), adapt_to_entity(), and_(), any(), contains(), entity, has(), in_(), mapper, of_type(), cascade, cascade_iterator(), declarative_scan(), do_init(), entity, instrument_class(), mapper, merge()
Class signature
class sqlalchemy.orm.RelationshipProperty (sqlalchemy.orm._IntrospectsAnnotations, sqlalchemy.orm.StrategizedProperty, sqlalchemy.log.Identified)
class Comparator
Produce boolean, comparison, and other operators for RelationshipProperty attributes.
See the documentation for PropComparator for a brief overview of ORM level operator definition.
See also
PropComparator
Comparator
ColumnOperators
Redefining and Creating New Operators
TypeEngine.comparator_factory
Class signature
class sqlalchemy.orm.RelationshipProperty.Comparator (sqlalchemy.util.langhelpers.MemoizedSlots, sqlalchemy.orm.PropComparator)
method sqlalchemy.orm.RelationshipProperty.Comparator.__eq__(other: Any) ? ColumnElement[bool]
Implement the == operator.
In a many-to-one context, such as:
MyClass.some_prop == <some object>
this will typically produce a clause such as:
mytable.related_id == <some id>
Where <some id> is the primary key of the given object.
The == operator provides partial functionality for non- many-to-one comparisons:
* Comparisons against collections are not supported. Use Comparator.contains().
* Compared to a scalar one-to-many, will produce a clause that compares the target columns in the parent to the given target.
* Compared to a scalar many-to-many, an alias of the association table will be rendered as well, forming a natural join that is part of the main body of the query. This will not work for queries that go beyond simple AND conjunctions of comparisons, such as those which use OR. Use explicit joins, outerjoins, or Comparator.has() for more comprehensive non-many-to-one scalar membership tests.
* Comparisons against None given in a one-to-many or many-to-many context produce a NOT EXISTS clause.
method sqlalchemy.orm.RelationshipProperty.Comparator.__init__(prop: RelationshipProperty[_PT], parentmapper: _InternalEntityType[Any], adapt_to_entity: AliasedInsp[Any] | None = None, of_type: _EntityType[_PT] | None = None, extra_criteria: Tuple[ColumnElement[bool], ] = ())
Construction of Comparator is internal to the ORM’s attribute mechanics.
method sqlalchemy.orm.RelationshipProperty.Comparator.__ne__(other: Any) ? ColumnElement[bool]
Implement the != operator.
In a many-to-one context, such as:
MyClass.some_prop != <some object>
This will typically produce a clause such as:
mytable.related_id != <some id>
Where <some id> is the primary key of the given object.
The != operator provides partial functionality for non- many-to-one comparisons:
* Comparisons against collections are not supported. Use Comparator.contains() in conjunction with not_().
* Compared to a scalar one-to-many, will produce a clause that compares the target columns in the parent to the given target.
* Compared to a scalar many-to-many, an alias of the association table will be rendered as well, forming a natural join that is part of the main body of the query. This will not work for queries that go beyond simple AND conjunctions of comparisons, such as those which use OR. Use explicit joins, outerjoins, or Comparator.has() in conjunction with not_() for more comprehensive non-many-to-one scalar membership tests.
* Comparisons against None given in a one-to-many or many-to-many context produce an EXISTS clause.
method sqlalchemy.orm.RelationshipProperty.Comparator.adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) ? RelationshipProperty.Comparator[Any]
Return a copy of this PropComparator which will use the given AliasedInsp to produce corresponding expressions.
method sqlalchemy.orm.RelationshipProperty.Comparator.and_(*criteria: _ColumnExpressionArgument[bool]) ? PropComparator[Any]
Add AND criteria.
See PropComparator.and_() for an example.
Added in version 1.4.
method sqlalchemy.orm.RelationshipProperty.Comparator.any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) ? ColumnElement[bool]
Produce an expression that tests a collection against particular criterion, using EXISTS.
An expression like:
session.query(MyClass).filter(
    MyClass.somereference.any(SomeRelated.x == 2)
)
Will produce a query like:
SELECT * FROM my_table WHERE
EXISTS (SELECT 1 FROM related WHERE related.my_id=my_table.id
AND related.x=2)
Because Comparator.any() uses a correlated subquery, its performance is not nearly as good when compared against large target tables as that of using a join.
Comparator.any() is particularly useful for testing for empty collections:
session.query(MyClass).filter(~MyClass.somereference.any())
will produce:
SELECT * FROM my_table WHERE
NOT (EXISTS (SELECT 1 FROM related WHERE
related.my_id=my_table.id))
Comparator.any() is only valid for collections, i.e. a relationship() that has uselist=True. For scalar references, use Comparator.has().
method sqlalchemy.orm.RelationshipProperty.Comparator.contains(other: _ColumnExpressionArgument[Any], **kwargs: Any) ? ColumnElement[bool]
Return a simple expression that tests a collection for containment of a particular item.
Comparator.contains() is only valid for a collection, i.e. a relationship() that implements one-to-many or many-to-many with uselist=True.
When used in a simple one-to-many context, an expression like:
MyClass.contains(other)
Produces a clause like:
mytable.id == <some id>
Where <some id> is the value of the foreign key attribute on other which refers to the primary key of its parent object. From this it follows that Comparator.contains() is very useful when used with simple one-to-many operations.
For many-to-many operations, the behavior of Comparator.contains() has more caveats. The association table will be rendered in the statement, producing an “implicit” join, that is, includes multiple tables in the FROM clause which are equated in the WHERE clause:
query(MyClass).filter(MyClass.contains(other))
Produces a query like:
SELECT * FROM my_table, my_association_table AS
my_association_table_1 WHERE
my_table.id = my_association_table_1.parent_id
AND my_association_table_1.child_id = <some id>
Where <some id> would be the primary key of other. From the above, it is clear that Comparator.contains() will not work with many-to-many collections when used in queries that move beyond simple AND conjunctions, such as multiple Comparator.contains() expressions joined by OR. In such cases subqueries or explicit “outer joins” will need to be used instead. See Comparator.any() for a less-performant alternative using EXISTS, or refer to Query.outerjoin() as well as Joins for more details on constructing outer joins.
kwargs may be ignored by this operator but are required for API conformance.
attribute sqlalchemy.orm.RelationshipProperty.Comparator.entity: _InternalEntityType[_PT]
The target entity referred to by this Comparator.
This is either a Mapper or AliasedInsp object.
This is the “target” or “remote” side of the relationship().
method sqlalchemy.orm.RelationshipProperty.Comparator.has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) ? ColumnElement[bool]
Produce an expression that tests a scalar reference against particular criterion, using EXISTS.
An expression like:
session.query(MyClass).filter(
    MyClass.somereference.has(SomeRelated.x == 2)
)
Will produce a query like:
SELECT * FROM my_table WHERE
EXISTS (SELECT 1 FROM related WHERE
related.id==my_table.related_id AND related.x=2)
Because Comparator.has() uses a correlated subquery, its performance is not nearly as good when compared against large target tables as that of using a join.
Comparator.has() is only valid for scalar references, i.e. a relationship() that has uselist=False. For collection references, use Comparator.any().
method sqlalchemy.orm.RelationshipProperty.Comparator.in_(other: Any) ? NoReturn
Produce an IN clause - this is not implemented for relationship()-based attributes at this time.
attribute sqlalchemy.orm.RelationshipProperty.Comparator.mapper: Mapper[_PT]
The target Mapper referred to by this Comparator.
This is the “target” or “remote” side of the relationship().
method sqlalchemy.orm.RelationshipProperty.Comparator.of_type(class_: _EntityType[Any]) ? PropComparator[_PT]
Redefine this object in terms of a polymorphic subclass.
See PropComparator.of_type() for an example.
attribute sqlalchemy.orm.RelationshipProperty.cascade
Return the current cascade setting for this RelationshipProperty.
method sqlalchemy.orm.RelationshipProperty.cascade_iterator(type_: str, state: InstanceState[Any], dict_: _InstanceDict, visited_states: Set[InstanceState[Any]], halt_on: Callable[[InstanceState[Any]], bool] | None = None) ? Iterator[Tuple[Any, Mapper[Any], InstanceState[Any], _InstanceDict]]
Iterate through instances related to the given instance for a particular ‘cascade’, starting with this MapperProperty.
Return an iterator3-tuples (instance, mapper, state).
Note that the ‘cascade’ collection on this MapperProperty is checked first for the given type before cascade_iterator is called.
This method typically only applies to Relationship.
method sqlalchemy.orm.RelationshipProperty.declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) ? None
Perform class-specific initializaton at early declarative scanning time.
Added in version 2.0.
method sqlalchemy.orm.RelationshipProperty.do_init() ? None
Perform subclass-specific initialization post-mapper-creation steps.
This is a template method called by the MapperProperty object’s init() method.
attribute sqlalchemy.orm.RelationshipProperty.entity
Return the target mapped entity, which is an inspect() of the class or aliased class that is referenced by this RelationshipProperty.
method sqlalchemy.orm.RelationshipProperty.instrument_class(mapper: Mapper[Any]) ? None
Hook called by the Mapper to the property to initiate instrumentation of the class attribute managed by this MapperProperty.
The MapperProperty here will typically call out to the attributes module to set up an InstrumentedAttribute.
This step is the first of two steps to set up an InstrumentedAttribute, and is called early in the mapper setup process.
The second step is typically the init_class_attribute step, called from StrategizedProperty via the post_instrument_class() hook. This step assigns additional state to the InstrumentedAttribute (specifically the “impl”) which has been determined after the MapperProperty has determined what kind of persistence management it needs to do (e.g. scalar, object, collection, etc).
attribute sqlalchemy.orm.RelationshipProperty.mapper
Return the targeted Mapper for this RelationshipProperty.
method sqlalchemy.orm.RelationshipProperty.merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) ? None
Merge the attribute represented by this MapperProperty from source to destination object.
class sqlalchemy.orm.SQLORMExpression
A type that may be used to indicate any ORM-level attribute or object that acts in place of one, in the context of SQL expression construction.
SQLORMExpression extends from the Core SQLColumnExpression to add additional SQL methods that are ORM specific, such as PropComparator.of_type(), and is part of the bases for InstrumentedAttribute. It may be used in PEP 484 typing to indicate arguments or return values that should behave as ORM-level attribute expressions.
Added in version 2.0.0b4.
Class signature
class sqlalchemy.orm.SQLORMExpression (sqlalchemy.orm.base.SQLORMOperations, sqlalchemy.sql.expression.SQLColumnExpression, sqlalchemy.util.langhelpers.TypingOnly)
class sqlalchemy.orm.Synonym
Declarative front-end for the SynonymProperty class.
Public constructor is the synonym() function.
Changed in version 2.0: Added Synonym as a Declarative compatible subclass for SynonymProperty
See also
Synonyms - Overview of synonyms
Class signature
class sqlalchemy.orm.Synonym (sqlalchemy.orm.descriptor_props.SynonymProperty, sqlalchemy.orm.base._DeclarativeMapped)
class sqlalchemy.orm.SynonymProperty
Denote an attribute name as a synonym to a mapped property, in that the attribute will mirror the value and expression behavior of another attribute.
Synonym is constructed using the synonym() function.
See also
Synonyms - Overview of synonyms
Members
doc, info, key, parent, set_parent(), uses_objects
Class signature
class sqlalchemy.orm.SynonymProperty (sqlalchemy.orm.descriptor_props.DescriptorProperty)
attribute sqlalchemy.orm.SynonymProperty.doc: str | None
inherited from the DescriptorProperty.doc attribute of DescriptorProperty
optional documentation string
attribute sqlalchemy.orm.SynonymProperty.info: _InfoType
inherited from the MapperProperty.info attribute of MapperProperty
Info dictionary associated with the object, allowing user-defined data to be associated with this InspectionAttr.
The dictionary is generated when first accessed. Alternatively, it can be specified as a constructor argument to the column_property(), relationship(), or composite() functions.
See also
QueryableAttribute.info
SchemaItem.info
attribute sqlalchemy.orm.SynonymProperty.key: str
inherited from the MapperProperty.key attribute of MapperProperty
name of class attribute
attribute sqlalchemy.orm.SynonymProperty.parent: Mapper[Any]
inherited from the MapperProperty.parent attribute of MapperProperty
the Mapper managing this property.
method sqlalchemy.orm.SynonymProperty.set_parent(parent: Mapper[Any], init: bool) ? None
Set the parent mapper that references this MapperProperty.
This method is overridden by some subclasses to perform extra setup when the mapper is first known.
attribute sqlalchemy.orm.SynonymProperty.uses_objects
class sqlalchemy.orm.QueryContext
class default_load_options
Class signature
class sqlalchemy.orm.QueryContext.default_load_options (sqlalchemy.sql.expression.Options)
class sqlalchemy.orm.QueryableAttribute
Base class for descriptor objects that intercept attribute events on behalf of a MapperProperty object. The actual MapperProperty is accessible via the QueryableAttribute.property attribute.
See also
InstrumentedAttribute
MapperProperty
Mapper.all_orm_descriptors
Mapper.attrs
Members
adapt_to_entity(), and_(), expression, info, is_attribute, of_type(), operate(), parent, reverse_operate()
Class signature
class sqlalchemy.orm.QueryableAttribute (sqlalchemy.orm.base._DeclarativeMapped, sqlalchemy.orm.base.SQLORMExpression, sqlalchemy.orm.base.InspectionAttr, sqlalchemy.orm.PropComparator, sqlalchemy.sql.roles.JoinTargetRole, sqlalchemy.sql.roles.OnClauseRole, sqlalchemy.sql.expression.Immutable, sqlalchemy.sql.cache_key.SlotsMemoizedHasCacheKey, sqlalchemy.util.langhelpers.MemoizedSlots, sqlalchemy.event.registry.EventTarget)
method sqlalchemy.orm.QueryableAttribute.adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) ? Self
Return a copy of this PropComparator which will use the given AliasedInsp to produce corresponding expressions.
method sqlalchemy.orm.QueryableAttribute.and_(*clauses: _ColumnExpressionArgument[bool]) ? QueryableAttribute[bool]
Add additional criteria to the ON clause that’s represented by this relationship attribute.
E.g.:
stmt = select(User).join(
    User.addresses.and_(Address.email_address != "foo")
)

stmt = select(User).options(
    joinedload(User.addresses.and_(Address.email_address != "foo"))
)
Added in version 1.4.
See also
Combining Relationship with Custom ON Criteria
Adding Criteria to loader options
with_loader_criteria()
attribute sqlalchemy.orm.QueryableAttribute.expression: ColumnElement[_T_co]
The SQL expression object represented by this QueryableAttribute.
This will typically be an instance of a ColumnElement subclass representing a column expression.
attribute sqlalchemy.orm.QueryableAttribute.info
Return the ‘info’ dictionary for the underlying SQL element.
The behavior here is as follows:
* If the attribute is a column-mapped property, i.e. ColumnProperty, which is mapped directly to a schema-level Column object, this attribute will return the SchemaItem.info dictionary associated with the core-level Column object.
* If the attribute is a ColumnProperty but is mapped to any other kind of SQL expression other than a Column, the attribute will refer to the MapperProperty.info dictionary associated directly with the ColumnProperty, assuming the SQL expression itself does not have its own .info attribute (which should be the case, unless a user-defined SQL construct has defined one).
* If the attribute refers to any other kind of MapperProperty, including Relationship, the attribute will refer to the MapperProperty.info dictionary associated with that MapperProperty.
* To access the MapperProperty.info dictionary of the MapperProperty unconditionally, including for a ColumnProperty that’s associated directly with a Column, the attribute can be referred to using QueryableAttribute.property attribute, as MyClass.someattribute.property.info.
See also
SchemaItem.info
MapperProperty.info
attribute sqlalchemy.orm.QueryableAttribute.is_attribute = True
True if this object is a Python descriptor.
This can refer to one of many types. Usually a QueryableAttribute which handles attributes events on behalf of a MapperProperty. But can also be an extension type such as AssociationProxy or hybrid_property. The InspectionAttr.extension_type will refer to a constant identifying the specific subtype.
See also
Mapper.all_orm_descriptors
method sqlalchemy.orm.QueryableAttribute.of_type(entity: _EntityType[_T]) ? QueryableAttribute[_T]
Redefine this object in terms of a polymorphic subclass, with_polymorphic() construct, or aliased() construct.
Returns a new PropComparator from which further criterion can be evaluated.
e.g.:
query.join(Company.employees.of_type(Engineer)).filter(
    Engineer.name == "foo"
)
Parameters:
class_ – a class or mapper indicating that criterion will be against this specific subclass.
See also
Using Relationship to join between aliased targets - in the ORM Querying Guide
Joining to specific sub-types or with_polymorphic() entities
method sqlalchemy.orm.QueryableAttribute.operate(op: OperatorType, *other: Any, **kwargs: Any) ? ColumnElement[Any]
Operate on an argument.
This is the lowest level of operation, raises NotImplementedError by default.
Overriding this on a subclass can allow common behavior to be applied to all operations. For example, overriding ColumnOperators to apply func.lower() to the left and right side:
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
Parameters:
* op – Operator callable.
* *other – the ‘other’ side of the operation. Will be a single scalar for most operations.
* **kwargs – modifiers. These may be passed by special operators such as ColumnOperators.contains().
attribute sqlalchemy.orm.QueryableAttribute.parent: _InternalEntityType[Any]
Return an inspection instance representing the parent.
This will be either an instance of Mapper or AliasedInsp, depending upon the nature of the parent entity which this attribute is associated with.
method sqlalchemy.orm.QueryableAttribute.reverse_operate(op: OperatorType, other: Any, **kwargs: Any) ? ColumnElement[Any]
Reverse operate on an argument.
Usage is the same as operate().
class sqlalchemy.orm.UOWTransaction
method sqlalchemy.orm.UOWTransaction.filter_states_for_dep(dep, states)
Filter the given list of InstanceStates to those relevant to the given DependencyProcessor.
method sqlalchemy.orm.UOWTransaction.finalize_flush_changes() ? None
Mark processed objects as clean / deleted after a successful flush().
This method is called within the flush() method after the execute() method has succeeded and the transaction has been committed.
method sqlalchemy.orm.UOWTransaction.get_attribute_history(state, key, passive=symbol('PASSIVE_NO_INITIALIZE'))
Facade to attributes.get_state_history(), including caching of results.
method sqlalchemy.orm.UOWTransaction.is_deleted(state)
Return True if the given state is marked as deleted within this uowtransaction.
method sqlalchemy.orm.UOWTransaction.remove_state_actions(state)
Remove pending actions for a state from the uowtransaction.
Members
filter_states_for_dep(), finalize_flush_changes(), get_attribute_history(), is_deleted(), remove_state_actions(), was_already_deleted()
method sqlalchemy.orm.UOWTransaction.was_already_deleted(state)
Return True if the given state is expired and was deleted previously.


ORM Exceptions
SQLAlchemy ORM exceptions.
Object Name
Description
ConcurrentModificationError
alias of StaleDataError
NO_STATE
Exception types that may be raised by instrumentation implementations.
attribute sqlalchemy.orm.exc..sqlalchemy.orm.exc.ConcurrentModificationError
alias of StaleDataError
exception sqlalchemy.orm.exc.DetachedInstanceError
An attempt to access unloaded attributes on a mapped instance that is detached.
Class signature
class sqlalchemy.orm.exc.DetachedInstanceError (sqlalchemy.exc.SQLAlchemyError)
exception sqlalchemy.orm.exc.FlushError
A invalid condition was detected during flush().
Class signature
class sqlalchemy.orm.exc.FlushError (sqlalchemy.exc.SQLAlchemyError)
exception sqlalchemy.orm.exc.LoaderStrategyException
A loader strategy for an attribute does not exist.
Class signature
class sqlalchemy.orm.exc.LoaderStrategyException (sqlalchemy.exc.InvalidRequestError)
method sqlalchemy.orm.exc.LoaderStrategyException.__init__(applied_to_property_type: Type[Any], requesting_property: MapperProperty[Any], applies_to: Type[MapperProperty[Any]] | None, actual_strategy_type: Type[LoaderStrategy] | None, strategy_key: Tuple[Any, ])
exception sqlalchemy.orm.exc.MappedAnnotationError
Raised when ORM annotated declarative cannot interpret the expression present inside of the Mapped construct.
Added in version 2.0.40.
Class signature
class sqlalchemy.orm.exc.MappedAnnotationError (sqlalchemy.exc.ArgumentError)
sqlalchemy.orm.exc.NO_STATE = (<class 'AttributeError'>, <class 'KeyError'>)
Exception types that may be raised by instrumentation implementations.
exception sqlalchemy.orm.exc.ObjectDeletedError
A refresh operation failed to retrieve the database row corresponding to an object’s known primary key identity.
A refresh operation proceeds when an expired attribute is accessed on an object, or when Query.get() is used to retrieve an object which is, upon retrieval, detected as expired. A SELECT is emitted for the target row based on primary key; if no row is returned, this exception is raised.
The true meaning of this exception is simply that no row exists for the primary key identifier associated with a persistent object. The row may have been deleted, or in some cases the primary key updated to a new value, outside of the ORM’s management of the target object.
Class signature
class sqlalchemy.orm.exc.ObjectDeletedError (sqlalchemy.exc.InvalidRequestError)
method sqlalchemy.orm.exc.ObjectDeletedError.__init__(state: InstanceState[Any], msg: str | None = None)
exception sqlalchemy.orm.exc.ObjectDereferencedError
An operation cannot complete due to an object being garbage collected.
Class signature
class sqlalchemy.orm.exc.ObjectDereferencedError (sqlalchemy.exc.SQLAlchemyError)
exception sqlalchemy.orm.exc.StaleDataError
An operation encountered database state that is unaccounted for.
Conditions which cause this to happen include:
* A flush may have attempted to update or delete rows and an unexpected number of rows were matched during the UPDATE or DELETE statement. Note that when version_id_col is used, rows in UPDATE or DELETE statements are also matched against the current known version identifier.
* A mapped object with version_id_col was refreshed, and the version number coming back from the database does not match that of the object itself.
* A object is detached from its parent object, however the object was previously attached to a different parent identity which was garbage collected, and a decision cannot be made if the new parent was really the most recent “parent”.
Class signature
class sqlalchemy.orm.exc.StaleDataError (sqlalchemy.exc.SQLAlchemyError)
exception sqlalchemy.orm.exc.UnmappedClassError
An mapping operation was requested for an unknown class.
Class signature
class sqlalchemy.orm.exc.UnmappedClassError (sqlalchemy.orm.exc.UnmappedError)
method sqlalchemy.orm.exc.UnmappedClassError.__init__(cls: Type[_T], msg: str | None = None)
exception sqlalchemy.orm.exc.UnmappedColumnError
Mapping operation was requested on an unknown column.
Class signature
class sqlalchemy.orm.exc.UnmappedColumnError (sqlalchemy.exc.InvalidRequestError)
exception sqlalchemy.orm.exc.UnmappedError
Base for exceptions that involve expected mappings not present.
Class signature
class sqlalchemy.orm.exc.UnmappedError (sqlalchemy.exc.InvalidRequestError)
exception sqlalchemy.orm.exc.UnmappedInstanceError
An mapping operation was requested for an unknown instance.
Class signature
class sqlalchemy.orm.exc.UnmappedInstanceError (sqlalchemy.orm.exc.UnmappedError)
method sqlalchemy.orm.exc.UnmappedInstanceError.__init__(obj: object, msg: str | None = None)


