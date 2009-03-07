<h1>News</h1>

I've merged the changes from the beta branch which brings many improvements. Notably;

<ol>
<li>Indexing is a separate process to searching and is triggered by update notifications.
<li>Rhino integration has landed, user customization of indexing is now possible.
</ol>

You are advised to delete indexes created prior to this update.

<h1>Build couchdb-lucene</h1>

<ol>
<li>Install Maven 2.
<li>checkout repository
<li>type 'mvn'
<li>configure couchdb (see below)
</ol>

<h1>Configure CouchDB</h1>

<pre>
[couchdb]
os_process_timeout=60000 ; increase the timeout from 5 seconds.

[external]
searcher=/usr/bin/java -jar /path/to/couchdb-lucene*-jar-with-dependencies.jar -search

[update_notification]
indexer=/usr/bin/java -jar /path/to/couchdb-lucene*-jar-with-dependencies.jar -index

[httpd_db_handlers]
_fti = {couch_httpd_external, handle_external_req, <<"fti">>}
</pre>

<h1>Indexing Strategy</h1>

<h2>Document Indexing</h2>

By default all attributes are indexed. You can customize this process by adding a design document at _design/lucene. You must supply an attribute called "transform" which takes and returns a document. For example;

<pre>
{
  "transform":"function(doc) { return doc; }"
}
</pre>

The function is evaluated by <a href="http://www.mozilla.org/rhino/">Rhino</a>. You may add, modify and remove any attributes. Additionally, returning null will exclude the document from indexing entirely.

<h2>Attachment Indexing</h2>

CouchDB uses <a href="http://lucene.apache.org/tika/">Apache Tika</a> to index attachments of the following types, assuming the correct content_type is set in couchdb;

<h3>Supported Formats</h3>

<ul>
<li>Excel spreadsheets (application/vnd.ms-excel)
<li>Word documents (application/msword)
<li>Powerpoint presentations (application/vnd.ms-powerpoint)
<li>Visio (application/vnd.visio)
<li>Outlook (application/vnd.ms-outlook)
<li>XML (application/xml)
<li>HTML (text/html)
<li>Images (image/*)
<li>Java class files
<li>Java jar archives
<li>MP3 (audio/mp3)
<li>OpenDocument (application/vnd.oasis.opendocument.*)
<li>Plain text (text/plain)
<li>PDF (application/pdf)
<li>RTF (application/rtf)
</ul>

<h1>Searching with couchdb-lucene</h1>

You can perform all types of queries using Lucene's default <a href="http://lucene.apache.org/java/2_4_0/queryparsersyntax.html">query syntax</a>. The following parameters can be passed for more sophisticated searches;

<dl>
<dt>q<dd>the query to run (e.g, subject:hello)
<dt>sort<dd>the comma-separated fields to sort on.
<dt>asc<dd>sort ascending (true) or descending (false), only when sorting on a single field.
<dt>limit<dd>the maximum number of results to return
<dt>skip<dd>the number of results to skip
<dt>include_docs<dd>whether to include the source docs
<dt>debug<dd>if false, a normal application/json response with results appears. if true, an pretty-printed HTML blob is returned instead.
</dl>

<i>All parameters except 'q' are optional.</i>

<h2>Special Fields</h2>

<dl>
<dt>_id<dd>The _id of the document.
<dt>_rev<dd>The _rev of the document.
<dt>_db<dd>The source database of the document.
<dt>_body<dd>Any text extracted from any attachment (name may change).
<dt>_author<dd>The author of any attachment (name may change).
<dt>_title<dd>The title of any attachment (name may change).
</dl>

<h2>Examples</h2>

<pre>
http://localhost:5984/dbname/_fti?q=field_name:value
http://localhost:5984/dbname/_fti?q=field_name:value&sort=other_field
http://localhost:5984/dbname/_fti?debug=true&sort=billing_size&q=body:document AND customer:[A TO C]
</pre>

<h2>Search Results Format</h2>

Here's an example of a JSON response without sorting;

<pre>
{
  "q": "+_db:enron +content:enron",
  "skip": 0,
  "limit": 2,
  "total_rows": 176852,
  "search_duration": 518,
  "fetch_duration": 4,
  "rows":   [
        {
      "_id": "hain-m-all_documents-257.",
      "_rev": "3750319208",
      "score": 1.601625680923462
    },
        {
      "_id": "hain-m-notes_inbox-257.",
      "_rev": "2603032545",
      "score": 1.601625680923462
    }
  ]
}
</pre>

And the same with sorting;

<pre>
{
  "q": "+_db:enron +content:enron",
  "skip": 0,
  "limit": 3,
  "total_rows": 176852,
  "search_duration": 660,
  "fetch_duration": 4,
  "sort_order":   [
        {
      "field": "source",
      "reverse": false,
      "type": "string"
    },
        {
      "reverse": false,
      "type": "doc"
    }
  ],
  "rows":   [
        {
      "_id": "shankman-j-inbox-105.",
      "_rev": "4289412378",
      "score": 0.6131107211112976,
      "sort_order":       [
        "enron",
        6
      ]
    },
        {
      "_id": "shankman-j-inbox-8.",
      "_rev": "1417542355",
      "score": 0.7492915391921997,
      "sort_order":       [
        "enron",
        7
      ]
    },
        {
      "_id": "shankman-j-inbox-30.",
      "_rev": "951793815",
      "score": 0.507369875907898,
      "sort_order":       [
        "enron",
        8
      ]
    }
  ]
}
</pre>

<h1>Working With The Source</h1>

To develop "live", type "mvn dependency:unpack-dependencies" and change the external line to something like this;

<pre>
fti=/usr/bin/java -cp /path/to/couchdb-lucene/target/classes:\
/path/to/couchdb-lucene/target/dependency org.apache.couchdb.lucene.Main
</pre>

You will need to restart CouchDB if you change couchdb-lucene source code but this is very fast.

<h1>Configuration</h1>

couchdb-lucene respects several system properties;

<dl>
<dt>couchdb.url<dd>the url to contact CouchDB with (default is "http://localhost:5984")
<dt>couchdb.lucene.dir<dd>specify the path to the lucene indexes (the default is to make a directory called 'lucene' relative to couchdb's current working directory.
</dl>

You can override these properties like this;

<pre>
fti=/usr/bin/java -D couchdb.lucene.dir=/tmp \
-cp /home/rnewson/Source/couchdb-lucene/target/classes:\
/home/rnewson/Source/couchdb-lucene/target/dependency\
org.apache.couchdb.lucene.Main
</pre>