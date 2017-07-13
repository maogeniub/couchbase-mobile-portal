---
permalink: guides/couchbase-lite/native-api/document/index.html
---

<block class="all" />

## Documents

In Couchbase Lite, a document's body takes the form of a JSON object — a collection of key/value pairs where the values can be different types of data such as numbers, strings, arrays or even nested objects. Every document is identified by a document ID, which can be automatically generated (as a UUID) or determined by the application; the only constraints are that it must be unique within the database, and it can't be changed. The following methods/initializers can be used:

- The {% st Document()|document|new Document()|getDocument() %} initializer can be used to create a new document (check the API references for alternative initializers that accept an ID and properties as parameters).
- The {% st database.getDocument(id: String)|documentWithID:|GetDocument(string id)|getDocument(String docID) %} method can be used to  get a document. If it doesn't exist in the database, it will return {% st nil|nil|null|null %}.

[//]: # (TODO: Since this identifier must be unique, you may want to check if a document with this ID already exists in the database using the {% st a|b|c|d %} method.)

The following code example creates a document and persists it to the database.

<block class="swift" />

```swift
let dict: [String: Any] = ["type": "task",
                           "owner": "todo",
                           "createdAt": Date()]
let newTask = Document(dictionary: dict)
try database.save(newTask)
```

<block class="objc" />

```objectivec
NSError* error;
CBLDocument* newTask = [[CBLDocument alloc] init];
[newTask setObject:@"task-list" forKey:@"type"];
[newTask setObject:@"todo" forKey:@"owner"];
[newTask setObject:[NSDate date] forKey:@"createAt"];
[database saveDocument: newTask error: &error];
```

<block class="csharp" />

```csharp
var newTask = new Document();
newTask.Set("type", "task");
newTask.Set("owner", "todo");
newTask.Set("createdAt", DateTimeOffset.UtcNow);
database.Save(newTask);
```

<block class="java" />

```java
Document newTask = new Document();
newTask.set("type", "task");
newTask.set("owner", "todo");
newTask.set("createdAt", new Date());
database.save(newTask);
```

<block class="all" />

### Mutability

The biggest change is that {% st Document|CBLDocument|Document|Document %} properties are now mutable. Instead of having to make a mutable copy of the properties dictionary, update it, and then save it back to the document, you can now modify individual properties in place and then save.

<block class="swift" />

```swift
newTask.set("Apples", forKey:"name")
try database.save(newTask)
```

<block class="objc" />

```objectivec
[newTask setObject:@"Apples" forKey:@"name"];
[database saveDocument:newTask error:&error];
```

<block class="csharp" />

```csharp
newTask.Set("name", "Apples")
database.Save(newTask)
```

<block class="java" />

```java
newTask.set("name", "Apples");
database.save(newTask);
```

<block class="all" />

This does create the possibility of confusion, since the document's in-memory state may not match what's in the database. Unsaved changes are not visible to other {% st Database|CBLDatabase|Database|Database %} instances (i.e. other threads that may have other instances), or to queries.

### Typed Accessors

The {% st Document|CBLDocument|Document|Document %} class now offers a set of property accessors for various scalar types, including boolean, integers, floating-point and strings. These accessors take care of converting to/from JSON encoding, and make sure you get the type you're expecting: for example, {% st document.string(forKey: String)|stringForKey:|GetString(string key)|getString(String key) %} returns either a {% st String|NSString|string|String %} or {% st nil|nil|null|null %}, so you can't get an unexpected object class and crash trying to use it as a string. (Even if the property in the document has an incompatible type, the accessor returns {% st nil|nil|null|null %}.)

In addition, as a convenience we offer {% st Date|NSDate|DateTimeOffset|Date %} accessors. Dates are a common data type, but JSON doesn't natively support them, so the convention is to store them as strings in ISO-8601 format. The following example sets the date on the `createdAt` property and reads it back using the {% st document.date(forKey: String)|dateForKey:|GetDate(string key)|getDate(String key) %} accessor method.

<block class="swift" />

```swift
newTask.set(Date(), forKey: "createdAt")
let date = newTask.date(forKey: "createdAt")
```

<block class="objc" />

```objectivec
[newTask setObject:[NSDate date] forKey:@"createdAt"];
NSDate* date = [newTask dateForKey:@"createdAt"];
```

<block class="csharp" />

```csharp
newTask.Set("createdAt", DateTimeOffset.UtcNow);
var date = newTask.GetDate("createdAt");
```

<block class="java" />

```java
newTask.set("createdAt", new Date());
Date date = newTask.getDate("createdAt");
```

<block class="all" />

### Transactions / batch operations

As before, if you're making multiple changes to a database at once, it's *much* faster to group them together, otherwise each individual change incurs overhead, from flushing writes to the filesystem to ensure durability. In 2.0 we've renamed the method to {% st inBatch()|inBatch:do:|InBatch()|inBatch(Runnable action) %} to emphasize that Couchbase Lite does not offer transactional guarantees, and that the purpose of the method is to optimize batch operations rather than to enable ACID transactions. The following example persists a few documents in batch.

<block class="swift" />

```swift
do {
	try database.inBatch {
		for i in 0...10 {
			let doc = Document()
			doc.set("user", forKey: "type")
			doc.set("user \(i)", forKey: "name")
			try database.save(doc)
			print("saved user document \(doc.string(forKey: "name"))")
		}
	}
} catch let error {
	print(error.localizedDescription)
}
```

<block class="objc" />

```objectivec
[database inBatch:&error do:^{
	for (int i = 1; i <= 10; i++)
	{
		NSError* error;
		CBLDocument *doc = [[CBLDocument alloc] init];
		[doc setObject:@"user" forKey:@"type"];
		[doc setObject:[NSString stringWithFormat:@"user %d", i] forKey:@"name"];
		[database saveDocument:doc error:&error];
		NSLog(@"saved user document %@", [doc stringForKey:@"name"]);
	}
}];
```

<block class="csharp" />

```csharp
database.InBatch(() =>
{
	for (int i = 0; i < 10; i++)
	{
		var doc = new Document()
		doc.Set(""type", user")
		doc.Set("name", ""user {i}")
		database.Save(doc)
		Console.WriteLine($"saved user document {doc.GetString("name")}")
	}
	
	return true;
});
```

<block class="java" />

```java
database.inBatch(new TimerTask() {
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			Document doc = new Document()
			doc.set("type", "user");
			doc.set("name", String.format("user %s", i));
			database.save(doc);
			Log.d("app", String.format("saved user document %s", doc.getString("name")));
		}
	}
});
```

<block class="all" />

At the *local* level this operation is still transactional: no other {% st Database|CBLDatabase|Database|Database %} instances, including ones managed by the replicator or HTTP listener, can make changes during the execution of the block, and other instances will not see partial changes. But Couchbase Mobile is a *distributed* system, and due to the way replication works, there's no guarantee that Sync Gateway or other devices will receive your changes all at once.

Again, the behavior of the method hasn't changed, just its name.