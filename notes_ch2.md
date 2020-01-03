# Data Models and Languages

Since most programming language we use today is OOD, in SQL data is stored in tables. The disconnect between the models is called impedance mismatch. Object relational mapping (ORM) framework like Hibernate reduce amount the boilerplate code for this translation layer.

##Many-to-One and Many-to-Many Relationships
Stores ID instead of plain text in database:

- consistent style and spelling across profiles
- avoiding ambiguity
- ease of updating - the name is stored in only one place
- localization support: easy to translate
- better search

Stores ID instead of plain text in database removes duplication that is the key idea behind normalization in databases

##Are document database repeating history ?
Although document database works well for one to many relationship, it struggles to many to many relationships.

##Relational vs Document database today

- document database:

  - Pro: schema flexibility, better performance due to locality, and for some app, it's closer to the data structures used by app
  - Con:

    - you can't refer directly to a nested item within a document.
    - poor support of joins that could be emulated by application code by making multiple request but adds complexity

* relational database: better support for joins, and many to one and many to many relationships.

### Schema flexibility in the document model

Most document database do not enforce any schema on the data in documents. No schema means that arbitrary keys and values can be added to a document, and when reading clients have no guarantees as what fields the documents may contain.

Although document database are called schema-less, it's misleading because the code calls assume some kind of implicit schema. More accurate name might be schema on read in contrast with schema on write

- schema on read: easy to alter and add a field such as name becomes first and last name (two separate fields)
- schema on write: takes time to alter schema and update every row
