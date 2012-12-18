# Hiberlite ORM

C++ object-relational mapping with API inspired by the awesome Boost.Serialization - that means _almost no API to learn_.

## Usage

Just compile and link all files under src/ to your project.
Include the main header:

```c++
#include "hiberlite.h" 
```

## Features

Key features of Hiberlite are:
- Boost.Serialization like API
- no code generators / preprocessors
- support for one-to-many and many-to-many relations
- automatic code generation
- lazy loading
- smart pointers
- no need to inherit from a single base class

In contrast to most serialization libraries with SQL serializers, C++ objects mapped with
hiberlite behave similar to active record pattern - you are not forced to follow the
"read all your data/modify some small part/write everything back" path.

For people who need reliable data storage, ACID transactions, simple random-access to their data files,
and don't like coding in SQL.

-----------------------

# Turorial

While building a typical RDBMS application developer has to deal with the following tasks:

1. Open a database
2. Create database schema
3. Populate the tables with data
4. Read and/or modify that data

Hiberlite greatly reduces the implementation of each of these tasks.

## Creating a database

Opening a database is as simple as
       
``` c++
hiberlite::Database db;
db.open("sample.db");
```

Where

``` c++
Database::open(std::string filename)
```

opens a sqlite3 file. If it doesn't exist it will be created.
Another option is to have constructor automatically open/create the DB file:

``` c++
hiberlite::Database db("sample.db");
```


## Creating database schema

In C++ You deal with classes and objects. You know what objects You want to store in the database.
And hiberlite will figure out how to store that data.


### Defining data model

First You must prepare the data classes for use with hiberlite.
Suppose we develop a social-network application. We define a person class as:

``` c++
class Person{
public:
        string name;
        int age;
        vector<string> bio;
};
```

Now, to let hiberlite know about the internal structure of this class,
we add an access method and an export declaration:

``` c++
class Person{
    friend class hiberlite::access;
    template<class Archive>
    void hibernate(Archive & ar)
    {
        ar & HIBERLITE_NVP(name);
        ar & HIBERLITE_NVP(age);
        ar & HIBERLITE_NVP(bio);
    }
public:
    string name;
    double age;
    vector<string> bio;
};
HIBERLITE_EXPORT_CLASS(Person)
```

Inside the **hibernate(Archive & ar)** method we list all the data fields we want to persist in the database.
Macro **HIBERLITE_EXPORT_CLASS(Person)** is invoked to register the class name with Hiberlite.
For now just remember to invoke it once per class in your project. (Placing it in the Person.cpp is a good idea)

At this point hiberlite is able to map the Person to a database.

### Creating tables

Database schema in hiberlite is simply a set of classes, that are stored in the database.
A programmer defines that set of classes, and hiberlite determines the needed tables and their columns.

Each instance of Database maintains its own copy of the database schema (set of data classes). To insert a new class to that set, use registerBeanClass template method:

``` c++
db.registerBeanClass<Person>();
```

In most applications several classes are stored in one database - so usually You will call **registerBeanClass** several times:

``` c++
Database db("gmail2.0.db");
db.registerBeanClass<UserAccount>();
db.registerBeanClass<Letter>();
db.registerBeanClass<Attachment>();
```

When the classes are registered, Database can create tables to map them:

``` c++
db.dropModel();
db.createModel();
```
    
**createModel()** executes several **CREATE TABLE** queries.
**dropModel()** cleans the database - executes **DROP TABLE IF EXISTS** query for each table in the schema.
Besides uninstallers, this method may be used to destroy the old tables (from a probably obsolete scheme)
before calling createModel().

## Saving data

When the database with proper chema is opened, we can put some Person objects in it:

``` c++
Person x;
x.name="Amanda";
x.age=21;
x.bio.push_back("born 1987");
hiberlite::bean_ptr<Person> p=db.copyBean(x);
x.age=-1; //does NOT change any database record
p->age=22; //does change the database
```

**copyBean(x)** template method creates a copy of its argument and saves it to the database.
It returns a smart pointer - **bean_ptr**.

**bean_ptr** is inspired by the **boost::shared_ptr**. The main difference is that in addition
to deletion, **bean_ptr** guarantees that the referenced object will be also saved to the database,
when no longer needed.

An internal primary key is autogenerated for each object in the database. It can be read
with **sqlid_t bean_ptr::get_id()**.

Another way to create a bean is to use **createBean()** template method:

``` c++
hiberlite::bean_ptr<Person> p=db.createBean<Person>();
p->name="Amanda";
p->age=21;
p->bio.push_back("born 1987");
```


## Loading data

There are several methods to query the database:

**bean_ptr<C> Database::loadBean(sqlid_t id)** loads the bean with the given id.

``` c++
bean_ptr<Person> p=db.loadBean<Person>(1);
cout << "first person is " << p->name << endl;
```

In this case object itself is not loaded, when bean_ptr is returned.
bean_ptr is a lazy pointer, so the wrapped object will be loaded when first needed.
In this example Person will be loaded later, when we try to access the name field.

**std::vector< bean_ptr<C> > Database::getAllBeans()**
returns a vector with all the beans of a given class C.

``` c++
vector< bean_ptr<Person> > v=db.getAllBeans<Person>();
cout << "found " << v.size() << " persons in the database\n";
```

In this example objects are not loaded at all.

You can load the same object more than once - all the returned bean_ptr's will point the same bean.

## Deleting beans

To remove an object from the database, call bean_ptr::destroy():

``` c++
bean_ptr<Person> p==db.loadBean<Person>(1);
p.destroy();
```

## Examples

All the above code is put together in sample.cpp. For more demonstration see the poor-mans unit tests : tests.cpp.

User-defined class
First we define the class we plan to store in the database:

``` c++
class MyClass{
    friend class hiberlite::access;
    template<class Archive>
    void hibernate(Archive & ar)
    {
        ar & HIBERLITE_NVP(a);
        ar & HIBERLITE_NVP(b);
        ar & HIBERLITE_NVP(vs);
    }
public:
    int a;
    double b;
    vector<string> vs;
};
HIBERLITE_EXPORT_CLASS(MyClass)
```

Note the friend declaration and the hibernate(...) template method - these two pieces of code are necessary
for hiberlite to access the internal structure of the user-defined class.

**HIBERLITE_NVP** is a macro that creates a name-value pair with reference to its argument.

**HIBERLITE_EXPORT_CLASS()** defines the root table name for the class. More on this later.

## How it is stored

hiberlite will use 2 tables to represent MyClass instances in the database:

``` c++
CREATE TABLE MyClass 
(
    a INTEGER,
    b REAL,
    hiberlite_id INTEGER PRIMARY KEY AUTOINCREMENT
)
CREATE TABLE MyClass_vs_items 
(
    hiberlite_entry_indx INTEGER,
    hiberlite_id INTEGER PRIMARY KEY AUTOINCREMENT,
    hiberlite_parent_id INTEGER,
    item TEXT
)
```

Table MyClass is the root table for MyClass. It will contain one row per object. Columns a and b store values of the corresponding int and double fields.

HIBERLITE_EXPORT_CLASS(MyClass) macro simply declares that instances of MyClass must be stored int the MyClass root table and its subtables.

MyClass_vs_items table is the "subtable" of MyClass. It is used to store elements of the vs vector: hiberlite_parent_id references the hiberlite_id of the root table, hiberlite_entry_indx - is the zero-index of the string element of the vector, and item is the value of that element.

Objects in the database are called beans. To add a new bean to the database we call

``` c++
hiberlite::bean_ptr<MyClass> p=db.copyBean(x);
```

**copyBean()** creates an internal copy of its argument (by invoking copy-constructor **new MyClass(x)**),
saves it in the database and returns a **bean_ptr**, pointing to that copy.
**bean_ptr** is a smart-pointer, it will perform reference-counting and care about saving and destroying your
bean when it is no longer in use.
But the object will not be lost - it is stored in the database.
The only way to remove it is to call **bean_ptr<c>::destroy()**.

There are two other ways to create a bean in the database:
**bean_ptr<c> Database::createBean<c>()** creates a bean using default constructor and returns a
**bean_ptr<c> Database::manageBean<c>(C& ptr)** takes control over its argument and wraps it in the bean_ptr.
You must not call **delete ptr;** after calling **manageBean(ptr)** - **bean_ptr** will do it when necessary.

## Loading data

Call to 

``` c++
std::vector< hiberlite::bean_ptr<MyClass> > v=db.getAllBeans<MyClass>();
```

will return a vector with all objects in the database.

bean_ptr<c> is a smart pointer, but I forgot to mention that it is also a lazy pointer.
Beans itself are not loaded when with **getAllBeans<myclass>()** - only their primary keys.
To give user the access to the underlying object, C& bean_ptr<c>::operator->() is overloaded.
The object will be created and fetched from the database with the first use of operator->.
