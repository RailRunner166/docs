---
layout: api-command
language: Ruby
permalink: api/ruby/insert/
command: insert
related_commands:
    update: update/
    replace: replace/
    delete: delete/
---

# Command syntax #

{% apibody %}
table.insert(object | [object1, object2, ...][, :durability => "hard", :return_changes => false, :conflict => "error"])
    &rarr; object
{% endapibody %}

# Description #

Insert documents into a table. Accepts a single document or an array of
documents.

The optional arguments are:

- `durability`: possible values are `hard` and `soft`. This option will override the table or query's durability setting (set in [run](/api/ruby/run/)). In soft durability mode RethinkDB will acknowledge the write immediately after receiving and caching it, but before the write has been committed to disk.
- `return_changes`:
    - `true`: return a `changes` array consisting of `old_val`/`new_val` objects describing the changes made, only including the documents actually updated.
    - `false`: do not return a `changes` array (the default).
    - `"always"`: behave as `true`, but include all documents the command tried to update whether or not the update was successful. (This was the behavior of `true` pre-2.0.)
- `conflict`: Determine handling of inserting documents with the same primary key as existing entries. There are three built-in methods: `"error"`, `"replace"` or `"update"`; alternatively, you may provide a conflict resolution function.
    - `"error"`: Do not insert the new document and record the conflict as an error. This is the default.
    - `"replace"`: [Replace](/api/ruby/replace/) the old document in its entirety with the new one.
    - `"update"`: [Update](/api/ruby/update/) fields of the old document with fields from the new one.
    - `lambda { |id, old_doc, new_doc| resolved_doc }`: a function that receives the id, old and new documents as arguments and returns a document which will be inserted in place of the conflicted one.

If `return_changes` is set to `true` or `"always"`, the `changes` array will follow the same order as the inserted documents. Documents in `changes` for which an error occurs (such as a key conflict) will have a third field, `error`, with an explanation of the error.

Insert returns an object that contains the following attributes:

- `inserted`: the number of documents successfully inserted.
- `replaced`: the number of documents updated when `conflict` is set to `"replace"` or `"update"`.
- `unchanged`: the number of documents whose fields are identical to existing documents with the same primary key when `conflict` is set to `"replace"` or `"update"`.
- `errors`: the number of errors encountered while performing the insert.
- `first_error`: If errors were encountered, contains the text of the first error.
- `deleted` and `skipped`: 0 for an insert operation.
- `generated_keys`: a list of generated primary keys for inserted documents whose primary keys were not specified (capped to 100,000).
- `warnings`: if the field `generated_keys` is truncated, you will get the warning _"Too many generated keys (&lt;X&gt;), array truncated to 100000."_.
- `changes`: if `return_changes` is set to `true`, this will be an array of objects, one for each objected affected by the `insert` operation. Each object will have two keys: `{:new_val => <new value>, :old_val => nil}`.

{% infobox alert %}
RethinkDB write operations will only throw exceptions if errors occur before any writes. Other errors will be listed in `first_error`, and `errors` will be set to a non-zero count. To properly handle errors with this term, code must both handle exceptions and check the `errors` return value!
{% endinfobox %}

__Example:__ Insert a document into the table `posts`.

```rb
r.table("posts").insert({
    :id => 1,
    :title => "Lorem ipsum",
    :content => "Dolor sit amet"
}).run(conn)
```

<!-- stop -->

The result will be:

```rb
{
    :deleted => 0,
    :errors => 0,
    :inserted => 1,
    :replaced => 0,
    :skipped => 0,
    :unchanged => 0
}
```


__Example:__ Insert a document without a defined primary key into the table `posts` where the
primary key is `id`.

```rb
r.table("posts").insert({
    :title => "Lorem ipsum",
    :content => "Dolor sit amet"
}).run(conn)
```

RethinkDB will generate a primary key and return it in `generated_keys`.

```rb
{
    :deleted => 0,
    :errors => 0,
    :generated_keys => [
        "dd782b64-70a7-43e4-b65e-dd14ae61d947"
    ],
    :inserted => 1,
    :replaced => 0,
    :skipped => 0,
    :unchanged => 0
}
```

Retrieve the document you just inserted with:

```rb
r.table("posts").get("dd782b64-70a7-43e4-b65e-dd14ae61d947").run(conn)
```

And you will get back:

```rb
{
    :id => "dd782b64-70a7-43e4-b65e-dd14ae61d947",
    :title => "Lorem ipsum",
    :content => "Dolor sit amet"
}
```


__Example:__ Insert multiple documents into the table `users`.

```rb
r.table("users").insert([
    {:id => "william", :email =>"william@rethinkdb.com"},
    {:id => "lara", :email => "lara@rethinkdb.com"}
]).run(conn)
```


__Example:__ Insert a document into the table `users`, replacing the document if the document
already exists.  

```rb
r.table("users").insert(
    {:id => "william", :email => "william@rethinkdb.com"},
    :conflict => "replace"
).run(conn)
```


__Example:__ Copy the documents from `posts` to `posts_backup`.

```rb
r.table("posts_backup").insert( r.table("posts") ).run(conn)
```


__Example:__ Get back a copy of the inserted document (with its generated primary key).

```rb
r.table("posts").insert(
    {:title => "Lorem ipsum", :content => "Dolor sit amet"},
    :return_changes => true
).run(conn)
```

The result will be

```rb
{
    :deleted => 0,
    :errors => 0,
    :generated_keys => [
        "dd782b64-70a7-43e4-b65e-dd14ae61d947"
    ],
    :inserted => 1,
    :replaced => 0,
    :skipped => 0,
    :unchanged => 0,
    :changes => [
        {
            :old_val => nil,
            :new_val => {
                :id => "dd782b64-70a7-43e4-b65e-dd14ae61d947",
                :title => "Lorem ipsum",
                :content => "Dolor sit amet"
            }
        }
    ]
}
```

__Example:__ Provide a resolution function that concatenates memo content in case of conflict.

```rb
# assume new_memos is a list of memo documents to insert
r.table("memos").insert(new_memos, :conflict => lambda { |id, old_doc, new_doc|
    new_doc.merge({:content => old_doc['content'] + "\n" + new_doc['content']})
}).run(conn)
```
