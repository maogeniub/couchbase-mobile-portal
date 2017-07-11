---
permalink: guides/couchbase-lite/native-api/full-text-search/index.html
---

<block class="all" />

## Full-Text Search

To run a full-text search (FTS) query, you must have created a full-text index on the expression being matched. Unlike regular queries, the index is not optional. The index's (single) expression should be the property name you wish to search on. The index type must also be {% st fullTextIndex|kCBLFullTextIndex|FullTextIndex|IndexType.FullText %}. The following code example inserts three documents of type `task` and creates an FTS index on the `name` property.

<block class="swift" />

```swift
// Insert documents
let tasks = ["buy groceries", "play chess", "book travels", "buy museum tickets"]
for task in tasks {
	let doc = Document()
	doc.set("task", forKey: "type")
	doc.set(task, forKey: "name")
	try database.save(doc)
}

// Create index
do {
	try database.createIndex(["name"], options: .fullTextIndex(language: nil, ignoreDiacritics: false))
} catch let error {
	print(error.localizedDescription)
}
```

<block class="objc" />

```objectivec
// insert documents
NSArray *tasks = @[@"buy groceries", @"play chess", @"book travels", @"buy museum tickets"];
for (NSString* task in tasks) {
	CBLDocument* doc = [[CBLDocument alloc] init];
	[doc setObject: @"task" forKey: @"type"];
	[doc setObject: task forKey: @"name"];
	
	NSError* error;
	[database saveDocument: newTask error:&error];
	if (error) {
		NSLog(@"Cannot save document %@", error);
	}
}

// create index
[database createIndexOn:@[@"name"] type:kCBLFullTextIndex options:NULL error:&error];
if (error) {
	NSLog(@"Cannot create index %@", error);
}
```

<block class="csharp" />

```csharp
// insert documents
var tasks = new string[] { "buy groceries", "play chess", "book travels", "buy museum tickets" };
foreach (string task in tasks)
{
	var doc = new Docment();
	doc.Set("type", "task").Set("name", task); // Chaining is possible
	database.Save(doc);
}

// create Index
database.CreateIndex(new[] { "name" }, IndexType.FullTextIndex, null);
```

<block class="java" />

```java
// insert documents
List<String> tasks = new ArrayList<>(Arrays.asList("buy groceries", "play chess", "book travels", "buy museum tickets"));
for (String task : tasks) {
	Document doc = new Docment();
	doc.set("type", "task");
	doc.set("name", task);
	database.save(doc);
}

// create index
List<Expression> expressions = Arrays.<Expression>asList(Expression.property("name"));
database.createIndex(expressions, IndexType.FullText, new IndexOptions(null, false));
```

<block class="all" />

With the index created, an FTS query on the property that is being indexed can be constructed and ran. The full-text search criteria is defined as a {% st Expression|CBLQueryExpression|IExpression|Expression %}. The left-hand side is usually a document property, but can be any expression producing a string. The right-hand side is the pattern to match: usually a word or a space-separated list of words, but it can be a more powerful [FTS4 search expression](https://www.sqlite.org/fts3.html#full_text_index_queries). The following code example matches all documents that contain the word 'buy' in the value of the `name` property.

<block class="swift" />

```swift
let whereClause = Expression.property("name").match("'buy'")
let ftsQuery = Query.select().from(DataSource.database(database)).where(whereClause)

do {
	let ftsQueryResult = try ftsQuery.run()
	for row in ftsQueryResult {
		print("document properties \(row.document.toDictionary())")
	}
} catch let error {
	print(error.localizedDescription)
}
```

<block class="objc" />

```objectivec
CBLQueryExpression* where = [[CBLQueryExpression property:@"name"] match:@"'buy'"];
CBLQuery *ftsQuery = [CBLQuery select:@[]
                                 from:[CBLQueryDataSource database:database]
                                where:where];

NSEnumerator* ftsQueryResult = [ftsQuery run:&error];
for (CBLFullTextQueryRow *row in ftsQueryResult) {
	NSLog(@"document properties :: %@", [row.document toDictionary]);
}
```

<block class="csharp" />

```csharp
var query = Query.Select()
		.From(DataSource.Database(database))
		.Where(Expression.Property("name").Match("'buy'"));

var rows = query.Run();
foreach (var row in rows)
{
	Console.WriteLine($"document properties ${row.Document.Properties}");
}
```

<block class="java" />

```java
Query ftsQuery = Query.select()
		.from(DataSource.database(database))
		.where(Expression.property("name").match("'buy'"));
		
ResultSet ftsQueryResult = ftsQuery.run();
FullTextQueryRow ftsRow;
while ((ftsRow = (FullTextQueryRow) ftsQueryResult.next()) != null) {
	Log.d("app", String.format("document properties :: %s", ftsRow.getDocument().toMap()));
}
```

<block class="all" />

When you run a full-text query, the resulting rows are instances of {% st FullTextQueryRow|CBLFullTextQueryRow|FullTextQueryRow|FullTextQueryRow %}. This class has additional methods that let you access the full string that was matched, and the character range(s) in that string where the match(es) occur.

<block class="objc" />

It's very common to sort full-text results in descending order of relevance. This can be a very difficult heuristic to define, but Couchbase Lite comes with a fairly simple ranking function you can use. In the `orderBy:` array, use a string of the form `rank(X)`, where `X` is the property or expression being searched, to represent the ranking of the result. Since higher rankings are better, you'll probably want to reverse the order by prefixing the string with a `-`.

### Under The Hood

For the time being, the Objective-C NSPredicate query API also allows you to compose queries using the underlying [JSON-based query syntax](https://github.com/couchbase/couchbase-lite-core/wiki/JSON-Query-Schema) recognized by LiteCore. This can be useful as a workaround if you run into limitations or bugs in the NSPredicate/NSExpression-based API. (But if so, please report the issue to us so we can fix it.)

> Disclaimer: This low-level query syntax is not part of Couchbase Lite's public API. We will probably remove this
 access to it before the final release of Couchbase Lite 2.0. By then the public API should be robust enough to handle your needs.

Using the JSON query syntax is very simple: just construct a JSON object tree out of Foundation objects, in accordance with the [spec](https://github.com/couchbase/couchbase-lite-core/wiki/JSON-Query-Schema), then pass the top level NSArray or NSDictionary to the CBLDatabase method that creates a query or creates/deletes an index:

* `-createQueryWhere:` — The `query` parameter can be a JSON NSArray (interpreted as a WHERE clause), or NSDictionary (interpreted as an entire SELECT query.)
* `-createIndexOn:error:` — Any item of the `expressions` array can be a JSON NSArray (interpreted as an expression to index.)
* `-createIndexOn:type:options:error:` — Same as above.
* `-deleteIndexOn:type:error:` — Same as above.

**Troubleshooting:** If LiteCore doesn't like your JSON, the call will return with an error. More usefully, LiteCore will log an error message to the console, so check that. (For internal reasons these messages don't propagate all the way up to the NSError yet.) If you're still stuck, it may help to set an Xcode breakpoint on all C++ exceptions; this will get hit when the parser gives up, and the stack backtrace _might_ give a clue. A common mistake is to pass an expression where an _array of_ expressions is expected; this is easy to do since expressions themselves are arrays. For example, `returning: @[@".", @"x"]` won't work; instead use `returning: @[@[@".", @"x"]]`.