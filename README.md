## Overview 
PyRavenDB is a python client api for [RavenDB](https://ravendb.net/), a NoSQL document database.
The API can handle most CRUD scenarios, including full support for replication, failover, dynamic queries, etc.


```python
with document_store.DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB") as store:
    store.initialize()
    with store.open_session() as session:
        foo = session.load("foos/1")
```

## Installation
There are three ways to install pyravendb.

1. Install from [PyPi](https://pypi.python.org/pypi), as [pyravendb](https://pypi.python.org/pypi/pyravendb).
	```bash
	pip install pyravendb
	```

2. Install from source, via PyPi. From pyravendb, download, open the source (pyravendb-x.x.x.zip) and run setup.py.
	```bash
    python setup.py install
	```
	
3. Install from source via [GitHub](https://github.com/ravendb/RavenDB-Python-Client).
 
	```bash
    git clone https://github.com/IdanHaim/RavenDB-Python-Client.git
    cd RavenDB-Python-Client
    git fetch <remote_name> v4.0
    git checkout v4.0
    git pull <remote_name> v4.0
    python setup.py install
	```

## Working with secured servers

You can use either PEM files (cert / key) or PFX/P12 files to authenticate against a remote RavenDB cluster.
Here is how to setup the Python RavenDB client to support it. 

 ```python
from pyravendb.store import document_store

urls = ["https://a.prod.roll.ravendb.cloud",
        "https://b.prod.roll.ravendb.cloud",
        "https://c.prod.roll.ravendb.cloud"]

# use PFX file
cert = {"pfx": "/path/to/cert.pfx", "password": "optional password"}

# use PEM file
# cert = ("/path/to/cert.pem")

# use cert / key files
# cert = ("/path/to/cert.crt", "/path/to/cert.key")

store =  document_store.DocumentStore(urls=urls, database="PyRavenDB", certificate=cert)
store.initialize() 
with store.open_session() as session:
    foo = session.load("foos/1")
```


## Usage
##### Load a single or several document\s from the store:
 ```python
from pyravendb.store import document_store
    
store =  document_store.DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB")
store.initialize() 
with store.open_session() as session:
    foo = session.load("foos/1")
```

load method have the option to track entity for you the only thing you need to do is add ```object_type```  when you call to load 
(load method will return a dynamic_structure object by default) for class with nested object you can call load with ```nested_object_types``` dictionary for the other types. just put in the key the name of the object and in the value his class (without this option you will get the document as it is) .

```python
foo = session.load("foos/1", object_type=Foo)
```

```python
class FooBar(object):
    def __init__(self,name,foo):
            self.name = name
            self.foo = foo
	
	foo_bar = session.load("FooBars/1", object_type=FooBar, nested_object_types={"foo":Foo})
			
```
To load several documents at once, supply a list of ids to session.load.

```python
foo = session.load(["foos/1", "foos/2", "foos/3"], object_type=Foo)
```

##### Delete a document
To delete a document from the store,  use ```session.delete()``` with the id or entity you would like to delete.

```python
with store.open_session() as session:
    foo = session.delete("foos/1")
```

##### Store a new document
to store a new document, use ```session.store(entity)``` with the entity you would like to store (entity must be an object)
For storing with dict we will use database_commands the put command (see the source code for that).

```python
class Foo(object):
   def __init__(self, name, key = None):
        self.name = name
        self.key = key
      
class FooBar(object):
    def __init__(self, name, foo):
        self.name = name
        self.foo = foo
	    
with store.open_session() as session:
    foo = Foo("PyRavenDB")
    session.store(foo)
    session.save_changes()
```

###### To save all the changes we made we need to call ```session.save_changes()```.

##### Query

* ```object_type``` - Give the object type you want to get from query.
* ```index_name``` -  The name of index you want the query to work on (If empty the index will be dynamic).
* ```collection_name``` -  Name of the collection (mutually exclusive with indexName).
* ```is_map_reduce``` - Whether we are querying a map/reduce index(modify how we treat identifier properties).
* ```wait_for_non_stale_results``` - False by default if True the query will wait until the index will be non stale.
* ```default_operator``` - The default query operator (OR or AND).
* ``` with_statistics``` - when True the qury will return stats about the query.
* ```nested_object_types``` - A dict of classes for nested object the key will be the name of the class and the value will be 
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&nbsp;the object we want to get for that attribute
	
```python
with store.open_session() as session:
    query_result = list(session.query().where_equals("name", "test101")
    query_result = list(session.query(object_type=Foo).where_starts_with("name", "n"))
    query_result = list(session.query(object_type=Foo).where_ends_with("name", "7"))
    query_result = list(session.query(object_type=FooBar, nested_object_types={"foo":Foo}).where(name="foo_bar"))
	
```

You can also build the query with several options using the builder pattern.

```python
with store.open_session() as session:
    list(session.query(wait_for_non_stale_results=True).where_not_none("name").order_by_descending("name"))
``` 

For the query you can also use the `where` feature which can get a variable number of arguments (**kwargs)
```python
with store.open_session() as session:
    query_results = list(session.query().where(name="test101", key=[4, 6, 90]))

```
`name` and `key` are the field names for which we query

##### Includes
A list of the properties we like to include in the query or in load.
The include property wont show in our result but when we load or query for it we wont requests it from the server.
The includes will save on the session cache.
```python
class FooBar(object):
    def __init__(name, foo_key)
        self.name = name
        self.foo = foo_key
        
store =  document_store.DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB")
store.initialize() 
with store.open_session() as session:
    query_result = list(session.query().where_equals("name", "testing_includes").include("foo")
    foo_bar = session.load("FooBars/1", object_type=FooBar, includes=foo)
    
```

### Changes Api
The RavenDB client offers a push notification feature that allows you to receive messages from a server about events that occurred there. 
You are able to subscribe to events for all documents, indexes and operations as well as to indicate a particular 
one that you are interested in. This mechanism lets you notify users if something has changed without 
the need to do any expensive polling.

##### Example:
```python
with document_store.DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB") as store:
    store.initialize() 
    documents = []
    indexes = []
    
    observer = store.changes().for_all_documents()
    observer.subscribe(documents.append)
    observer.ensure_subscribe_now()
    
    observer = store.changes().for_index('Users')
    observer.subscribe(ActionObserver(on_next=indexes.append))
    observer.ensure_subscribe_now()
    
    index_definition_users = IndexDefinition("Users", "from doc in docs.Users select new {doc.Name}")
    index_definition_dogs = IndexDefinition("Dogs", "from doc in docs.Dogs select new {doc.brand}")
    
    store.maintenance.send(PutIndexesOperation(index_definition_dogs, index_definition_users))
    
    with store.open_session() as session:
        session.store(User("Idan"), key="users/1")
    session.save_changes()
```

In this example we want to observe to changes from all documents and for index with the name of Users.
After we register the observable we can subscribe for the notification and decide what to do with them (like append).

If you did not create an Observer for the subscription we will create one for you with the method you put.
(To create Observer you can make any class that you want that will inherit from the class Observer
can be found in pyravendb.changes.observers or to use the ActionObserver).

```python
class Observer(metaclass=ABCMeta):
    @abstractmethod
    def on_completed(self):
        pass

    @abstractmethod
    def on_error(self, exception):
        pass

    @abstractmethod
    def on_next(self, value):
        pass
```

## What`s new
### Mappers

mappers have been added to `DocumentConvention` to be able to parse custom objects.
For using the mappers, only update `conventions.mappers` from the `DocumentStore`
with your own dict.
The key of the mappers will be the type of the object you want to initialize and the value will be a key, value method:
the key will be the property name inside the object and the value will be the value of the property inside the document.
like before you must specify the `object_type` property to be able to fetch the mapper method that you supplied
if `nested_object_types` is initialized the mappers won't work.

##### Example:

```python
class Address:
    def __init__(self, street, city, state):
        self.street = street
        self.city = city
        self.state = state


class Owner:
    def __init__(self, name, address):
        self.name = name
        self.address = address


class Dog:
    def __init__(self, name, owner):
        self.name = name
        self.owner = owner

# This method will be called for each of the object properties
def get_dog(key, value):
    if not value:
        return None
    if key == "address":
        return Address(**value)
    elif key == "owner":
        return Owner(**value)


address = Address("Rub", "Unknown", "Israel")
owner = Owner("Idan", address)
dog = Dog(name="Donald", owner=owner)

with DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB") as store:
    store.conventions.mappers.update({Dog: get_dog})
    store.initialize()
    with store.open_session() as session:
        result = session.load("dogs/1-A", object_type=Dog)
```

### Custom Class Name

Now you can customize the collection name. This is necessary when the language is not English and the plural generated is not correct. It is also important for environments where a Python program will access a RavenDB database that is also powered by a C# program where it is already possible to customize the name.

As with mappers, you can change the name of collections via `DocumentConvention`, as shown below.

If a custom name is not specified, RavenDB will automatically generate the collection name as it did until now.

##### Example:

```python
class Address:
    def __init__(self, street, city, state):
        self.street = street
        self.city = city
        self.state = state
        self.Id: str = None


class Customer:
    def __init__(self, name):
        self.name = name
        self.Id: str = None

class SomeClass:
    def __init__(self, name):
        self.name = name
        self.Id: str = None

with DocumentStore(urls=["http://localhost:8080"], database="PyRavenDB") as store:
    store.conventions.default_collection_names = {Address: 'Address',SomeClass:'SomeNewCoolName'}
    store.initialize()
    

```

##### Bug Tracker
http://issues.hibernatingrhinos.com/issues/RDBC