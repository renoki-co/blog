## What are SQL indexes and why do they matter?

Building strong, dominant applications means you should take advantage of every single tool that you can use. Starting from infrastructure — using IOPS SSDs over General-Purpose SSDs or choosing to connect to the nearest location to application structure and how it stores, processes, and reads the data on the hardware. This time, we’ll be looking at the application structure, and to be more specific — why is it important to use indexes when storing data in our database?

### Short Answer

Either it’s NoSQL or SQL, most database engines do have support for indexes. Either they’re hash keys in DynamoDB or auto-increment, unsigned integer primary keys in MySQL. Indexing is hard to explain, but here’s a good analogy between NOT using indexes and using indexes.

> You have a bookshelf. All books within this bookshelf contain the same sentence, but one word is different: “[word] is power.” You pick one of the books from the shelf. You are seaching for the word “knowledge”. You start to iterate through all the pages, page-by-page, and try to find the word given. You finally found it. “Knowledge is power.”

There’s no other way than iterating through all the pages and stop when finding the word given. This is an analogy to querying the database engine: the database is receiving your query, and based on that query, iterates through the database to find records that match your criteria.

Now here’s a simpler way to tell what indexes are.

> You pick another book from the same bookshelf. This book looks awkward: has many, many bookmarks between pages and one of the bookmark has a word “knowledge” on it, and you saw it before even opening the book. You open to the page where the bookmark is, and find the word, along with the sentence: “Knowledge is power”.

So, in other words, indexing is like storing additional data and putting it “somewhere”, where you can access it blazing-fast, and in case you need to query the database with that specific field ([word] in our case), it will point to the right result without having to hit all the pages, one-by-one.

Yes! This is a performance improvement — it’s better to already know where to search before even opening the first page and the next ones, searching our needed data in a better way than starting kilometers away from our target location.

## Real use cases

It’s a good practice to create indexes for fields that will be involved in queries. If you have a `posts` table, a good index can be put on `slug`, because when querying with `WHERE slug = 'my-slug'`, you’ll get the post faster, since it knows where to search and retrieve the data you need.

Another good example is using indexes with column names that need to be ordered. In case you have a table that contains an `order_number` column and you sort your data by that column, indexing will help you order it faster — indexes are sorted structure, your query will execute faster.

## Drawbacks (wait, what?)

Indexing is fun, right? One thing I have learned during elementary school and that stood with me during years was on a biology class:

> “Medicine in high amounts is poison. Poison in tiny amounts is medicine.”

This is the case where this masterpiece-sentence can be applied. Indexing can hurt a lot if used heavily. We can pretty much find a good analogy on how indexing is stored:

> If you have a column called `slug` and you index it, basically, on each record created, there’s an “imaginary” table that contains the primary key and the column “slug” with data stored in it. When trying to search for “slug”, it will first search in the “imaginary” table and if it finds something, it will add the entire referenced row through the primary key in your search result.

Although indexing is a must when working with data and wanting to query it more efficiently, it has some drawbacks — one of them is storage. Indexes are also stored on the disk, and sometimes, indexes can be huge. If you aren’t cheap on storage, it should not be a huge problem for you — but still, avoid unuseful index declarations.

Indexing also affects data change in your database. Inserting, updating or deleting data will recreate all indexes — not structurally, but the data itself. That “imaginary” table will have to be thrown away and be recreated, with the new indexed values.

## What did we learn?

Indexing has a great role in databases — it structures the data so it can be queried faster. In the end, you either end up having more data and slowdown on writing or deleting over the speed of reading or keep the write/delete speed even with the reading speed by not using indexes. It’s your choice.