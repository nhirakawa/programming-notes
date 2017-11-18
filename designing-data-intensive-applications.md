# Notes on Designing Data-Intensive Applications

## 2. Data Models and Query Languages

### Relational Model versus Document Model
- Best known datamodel is SQL
    - Data is organized into relations
    - Each relation is an unordered colection of tuples
- Theoretical proposal, doubtful whether or not it could be implemented efficiently
- Other models (network model, hierarchical model) came about, but relational still dominated

### The Birth of NoSQL
- Several driving forces:
    - Greater scalability (large datasets or higher throughput)
    - FOSS over commercial products
    - Specialized query operations
    - Relational schemas too restrictive
- Different requirements dictate different technologies

### The Object-Relational Mismatch
- Most programming is object-oriented, but data is relational
    - impedance mismatch
- Object-relational mapping reduces boilerplate, but not 100% effective
- Example - persisting a resume
    - Traditional SQL model dictates normalized representation of positions, education, and contact information in separate tables
    - Later SQL standards allowed structured datatypes and XML, or even JSON
    - Could store information as JSON or XML and store in text column
        - Lose ability to query inside the column
- JSON representation has better locality
    - No multi table joins

### Many-to-one and Many-to-many Relationships
- Storing standardized lists and letting users choose (rather than free-text) has advantages
    - Consistent style and spelling
    - Avoid ambiguitiy
    - Easy to update
    - Localization
    - Better search
- Storing an ID allows de-duplication
    - User strings must be duplicated everywhere they are referenced
- Because an ID has no meaning to humans, it never needs to change
    - Updating human-relevant information means updating it everywhere - expensive and inconsistent
- Normalizing means many-to-one relationships - not well supported by document databases
    - Support for joins is weak
- Data has a tendency to become more interconnected as features are added

### Are Docuemnt Databases Repeating History?
- Document databases have roots in IBM's Infomation Management System (1970's)
    - Hierarchical model
    - Worked well for one-to-many relationships, but made many-to-many relationships hard
    - No joins
    - Duplicate data or manually resolve references between records

### The Network Model
- CODASYL model was generalization of hierarchical model
- Each record could have multiple parents
- Links were not foreign keys, but like pointers
    - Follow a path from root record along chain of links
- Simplest cases resembles linked list
- In more complicated case, several paths to same record can exists
    - Need to keep track of different access paths in mental model
- Application code had to keep track of various relationships as cursor moved through database
- Difficult to make changes to data model

### The Relational Model
- Lay out the data in the open
    - Simply a collection of tuples
- Read all rows in table, selecting those that match a condition
- Query optimizer decides which parts of the query to execute in order and which indexes to use
    - Effectively the access path in the network model, but no human intervention
- To make a new query, only need to declare a new index
- Only need to build one query optimizer
    - Generalizes well

### Comparison to Document Databases
- Document databases similar to hierarchical
    - Nested records are stored on the parent record
- Representing many-to-many and many-to-one relationships not fundamentally different
    - Maintaining a unique identifier to some other entity

### Relational versus Document Databases Today
- Differences include fault-tolerance properties and handling concurrency
- Main arguments for document model are schema flexibility, locality, sometimes closer to actual structures used by application
- Relational provides better support for joins and relationships

### Which Data Model Leads to Simpler Application Code?
- If application has document-like structure, use a document model
- Document model can resemble access path
    - As long as doucments are not too deeply nested, not a problem
- If application uses many-to-many relationships, joins mean cleaner application code
- tl;dr - it depends

### Schema Flexibility in the Document Model
- Most document databases do not enforce schema
    - Arbitrary keys and values
    - Clients have no guarantees which fields a document may contain
- Sometimes called schemaless, but not quite true
    - Clients assume *some* structure
    - More accurate to call it schema-on-read, vs. schema-on-write for relational databases
- To change schema, can simply start writing with new schema
    - Need to handle the changes in the client
- Relational databases would need a migration (`ALTER TABLE`)
- Schema-on-read is good if items don't all have same structure
    - Many different types of objects
    - Structure of data is determined by external systems

### Data Locality for Queries
- Documents usually stored continuously
- Locality advantage if entire document is needed
    - Wasteful if only portion of document is needed (including updates)
- Google's Spanner offers same locality in a relational model
    - Allow schema to declare that table rows should be nested within a parent table
- Oracle allows the same in multi-table index cluster tables
- Column-family concept from Bigtable used in Cassandra and HBase

### Convergence of Document and Relational Databases
- Relational databases have allowed XML for years
- Similar support for JSON
- Document databases starting to support joins in query language

### Query Languages for Data
- SQL is a declarative language, while IMS and CODASYL are imperative
    - Imperative -  perform operations in certain order
    - Declarative - Specify the result, not how to achieve
- Declarative hides implementation details
- Declarative can also parallelize
    - No particular order, so operations can be done and sorted just before return

### MapReduce Querying
- Query is specified in snippets
    - Called repeatedly by framework
- Restricted functions
    - Must be pure (no side effects)
- Low-level model for distributed execution
    - SQL can be implemented on top of MapReduce
- map and reduce functions are carefully coordinated
    - More complex than just one query

### Graph-Like Data Models
- Many-to-many relationships resemble a graph
- Not limited to homogenous data

### Graph Queries in SQL
- Can put graph data in relational structure
    - Can be difficult to query
    - May need to traverse a variable number of edges to find relevant vertex

### Triple-Stores and SPARQL
- 