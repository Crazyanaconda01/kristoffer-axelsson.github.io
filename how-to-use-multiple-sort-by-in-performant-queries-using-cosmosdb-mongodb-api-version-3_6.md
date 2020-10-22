## How to use multiple sort by in performant queries using CosmosDb / MongoDB API version 3.6

If you are developing a solution using CosmosDb, you may eventually come to a time when you need to sort by more than one property in a document. 

Let’s say you have an entity for a delivery of some sort that contains both a planned delivery date and then also the actual delivery, when the shipment was actually delivered. Let’s call these HomeArrivalPlanned and HomeArrivalActual. 

In this case it could be assumed that the second of those properties, HomeArrivalActual, might be null before the shipment has actually been delivered.

Let’s say you want to have a limit in place in order to limit the size of the response from a REST API. Then in order to present the most relevant data to user, you would probably want to a descending sort on both HomeArrivalActual, but then also HomeArrivalPlanned.

If doing this in i.e. a .Net Core 3.1 application using the MongoDB C#/.NET Driver and a strongly typed LINQ like lambda expression, you would probably write something like this:

var results = _context.Deliveries.Find(filter).SortByDescending(x => x.HomeArrivalActual).ThenByDescending(x => x.HomeArrivalPlanned).Limit(500).ToList();

This query will fail in CosmosDb however, and the CosmosDb server will throw an error.

### Composite indexes and multiple sort order properties in CosmosDb
![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/cosmosdb_composite_query1.png)

In the mentioned query, you would most likely get the above exception and error message.

In CosmosDb, multiple sort orders must be indexed, and composite indexes like this is something you need to create yourself. 

(One could wonder why not all properties are indexed perhaps, the default index policy do index all properties, but not in all available compose index variations. And that makes a lot of sense, as indexes consumes space, and the more properties a document has, the amount of possible composite indexes would grow exponentially, so this is something we need to create manually for now).
A great tool I often use for management and maintenance of CosmosDb collections is Studio 3T (formerly RoboMongo).

In order to make the above query successful, you would need to crate an index on the two properties in question, HomeArrivalActual and HomeArrivalPlanned. 

In Studio 3T, open an IntelliShell window and enter this (depending on collection name of course):

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/studio3t_create_cosmos_index.png)

db.getCollection("deliveries").createIndex( { "HomeArrivalActual" : -1, "HomeArrivalPlanned": -1 })

There are a few things to note here. First: as mentioned in the Microsoft docs, don’t forget that the order matters here. The order needs to be the same as in your where/find clause.

Also, note the -1. This means it is a descending index. If you have an expression that mixes a ascending and descending sorting, that require another index.

Having executed the above query, you will see in an output window in Studio 3T if the initial index creation succeeded or not, if the amount of indexes changed and increased the change was successful.

###Index rebuild
The above command will trigger a new index to be built. Depending on your collection size etc., it might take some time. If you try to run the query during this time, you will see an other error message:

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/cosmosdb_composite_query2.png)

The error message is quite self-explanatory, you need to wait a bit for the index to be built. 

After a while the query will work, and will have a great, performant way to do multi property sorts, perfect to use in conjunction with i.e. limits and server side pagination to return the most relevant results to the user.
