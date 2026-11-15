---
title: "Capped Collections in MongoDB"
description: "MongoDB capped collections for fast, fixed-size storage and real-time data streaming"
date: "2025-03-01"
tags:
  - MongoDB
  - Capped-Collections
  - Database
---

Imagine you’re managing a real-time logging system for a web server. Every second, thousands of log entries are generated, but you only need the most recent records.

Storing all logs would waste storage, and manually deleting old logs would be a nightmare.

### What if your database could automatically manage this for you?

That is exactly what **capped collections** in MongoDB do!

They provide a high-performance, fixed-size storage solution that automatically removes old data while keeping the latest ones accessible.

We will dive deep into capped collections.

### What Are Capped Collections?

Capped collections are fixed-size collections in MongoDB that maintain a **first-in, first-out (FIFO)** order for documents.

Once the collection reaches its size limit, MongoDB automatically removes the oldest documents (records in SQL databases) to make space for new ones.

Let’s create a capped collection.

### Example: Create a 1MB Capped Collection

```javascript
db.createCollection("logs", { capped: true, size: 1048576 });
```

1. `"logs"` is the collection name.
2. `"capped: true"` marks it as a capped collection.
3. `"size: 1048576"` sets a maximum size of 1MB. MongoDB will delete old documents when this size is reached.

You can also limit the number of documents in addition to the size:

```javascript
db.createCollection("logs", {
  capped: true,
  size: 1048576,
  max: 1000
});
```

This ensures that a maximum of **1000 documents** are stored in the collection.

To check if a collection is capped:

```javascript
db.logs.isCapped(); // Returns true if capped, else false
```

### Inserting Data in a Capped Collection

Capped collections behave like normal collections, so you can use usual insertion methods like `insertOne()` and `insertMany()`.

```javascript
db.logs.insertOne({
  level: "info",
  message: "Application started",
  timestamp: new Date()
});

db.logs.insertMany([
  {
    level: "info",
    message: "User 1ab3c logged in",
    timestamp: new Date()
  },
  {
    level: "info",
    message: "User 3m9s3 logged out",
    timestamp: new Date()
  }
]);
```

### Querying a Capped Collection

Capped collections are just like normal collections in MongoDB, so you can query them using `find()`:

```javascript
db.logs.find();
```

### Real-Time Streaming with Tailable Cursors

MongoDB allows you to create a **tailable cursor** to continuously read new documents.

Let’s say we want to continuously stream new log entries from our logs collection:

```javascript
const cursor = db.logs.find(
  {},
  { tailable: true, awaitData: true }
);

while (cursor.hasNext()) {
  printjson(cursor.next());
}
```

1. `"tailable: true"` keeps the cursor open after reaching the end of the results.
2. `"awaitData: true"` allows the cursor to wait for new data instead of immediately returning when there are no new documents.
3. `"cursor.hasNext()"` checks for and iterates over new log entries as they appear.

However, this approach can terminate unexpectedly if the cursor reaches the end and remains without new data.

To continuously monitor logs without breaking, you can wrap it inside an infinite loop:

```javascript
const cursor = db.logs.find(
  {},
  { tailable: true, awaitData: true }
);

while (true) {
  if (cursor.hasNext()) {
    printjson(cursor.next());
  }
}
```

### What Have We Learned About Capped Collections?

1. **Fixed size:** The collection has a predefined size limit in bytes.
2. **FIFO deletion:** When the collection reaches its limit, the oldest documents are automatically removed to make room for new ones.
3. **High performance for inserts and retrievals:** Capped collections are optimized for sequential writes and reads.
4. **Preserves insertion order:** Documents are stored in insertion order.
5. **Document size restrictions:** Updates cannot increase a document's size beyond the space originally allocated for that document.

### Wait, Are Capped Collections That Good?

Nope! They have some drawbacks.

1. **Cannot increase document size:** Updating a document so that it becomes larger than its original size is not allowed.
2. **Cannot delete specific documents:** You generally cannot remove individual documents from a capped collection; old documents are automatically removed according to the collection's FIFO behavior.
3. **No multi-document transactions:** Capped collections cannot be used in multi-document transactions.

### So, When Should You Use Capped Collections?

Use capped collections when you need to store a limited amount of recent data without worrying about manual cleanup.

Some common use cases include:

- **Recent application logs**
- **Server logs**
- **Real-time data streams**
- **Chat or messaging data**
- **Frequently accessed temporary data**
- **Recent activity/history**

You can also use them for storing history, such as search histories.

But remember: in that case, a user cannot delete a specific history entry from the capped collection.

Building your own capped data structure is fun too!

You can use a **circular queue** or a **linear queue with array-shifting techniques** to implement similar behavior yourself.

So, this is all about **capped collections in MongoDB**.