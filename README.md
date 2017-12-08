## Graph.cool error: Nested permissions not validated when using queries and listing relations as permission's fields

This repository reports two graphcool errors that occurs when using permissions and nested mutations. 


### Types

`Folder` and `File` types. A `Folder` may contain files. A `Folder` may be secret (`secret` field).
```graphql schema
type Folder @model {
  createdAt: DateTime!
  id: ID! @isUnique
  name: String!
  files: [File!]! @relation(name: "FolderOnFile")
  secret: Boolean! @defaultValue(value: false)
  updatedAt: DateTime!
}

type File @model {
  createdAt: DateTime!
  id: ID! @isUnique
  name: String!
  folder: Folder! @relation(name: "FolderOnFile")
  updatedAt: DateTime!
}
```

### Working Permissions (this works as expected)

```yaml
permissions:
  # Folder permissions
  - operation: Folder.create
    fields:
    - name
  - operation: Folder.read

  # File permissions
  - operation: File.create
    fields:
    - name
  - operation: File.read
  - operation: FolderOnFile.*

```

Create a `Folder` when creating a new `File` (using a nested creation):

```graphql
mutation{
  createFile(
    name: "File 1"
    folder: {
      name: "Folder 1"
    }
  ){
    id
  }
}
```
Response: 
```javascript
{
  "data": {
    "createFile": {
      "id": "cjay9w7jta6390152c5n7sjcq"
    }
  }
}
```

Cannot create a secret folder (permissions don't allow setting the field `secret` when creating a new `Folder`)

```graphql
mutation{
  createFile(
    name: "File 1"
    folder: {
      name: "Folder 1"
      secret: true # added this
    }
  ){
    id
  }
}
```
Response: 
```javascript
{
  ...
  "errors": [
    ...
      "message": "Insufficient permissions for this mutation",
    ...
  ]
}
```

Ok! So far so good!

### Error 1: Permissions with queries


```yaml
permissions:
  # Folder permissions
  - operation: Folder.create
    query: './src/query.graphql' # Added query
    fields:
    - name
  - operation: Folder.read

  # File permissions
  - operation: File.create
    query: './src/query.graphql' # Added query
    fields:
    - name
  - operation: File.read
  - operation: FolderOnFile.*

```

This doesn't work anymore:

```graphql
mutation{
  createFile(
    name: "File 1"
    folder: {
      name: "Folder 1"
    }
  ){
    id
  }
}
```
Response: 
```javascript
{
  ...
  "errors": [
    ...
      "message": "Insufficient permissions for this mutation",
    ...
  ]
}
```

To make it work I must add the relation `folder` as one of the File's file permission:
```yaml
  # File permissions
  - operation: File.create
    query: './src/query.graphql' # Added query
    fields:
    - name
    - folder # Added folder field
  - operation: File.read
  - operation: FolderOnFile.*

```

If I run the same query again, the nodes are created: 

```graphql
mutation{
  createFile(
    name: "File 1"
    folder: {
      name: "Folder 1"
    }
  ){
    id
  }
}
```
Response: 
```javascript
{
  "data": {
    "createFile": {
      "id": "cjayaukp42szx0101vopwcujd"
    }
  }
}
```

But that causes a new problem"nested permissions stop to work.

### Error 2: Nested permissions without validation

```yaml
permissions:
  # Folder permissions
  - operation: Folder.create
    query: './src/query.graphql' 
    fields:
    - name
  - operation: Folder.read

  # File permissions
  - operation: File.create
    query: './src/query.graphql'
    fields:
    - name
    - folder
  - operation: File.read
  - operation: FolderOnFile.*

```

Trying to create a `File`, creating also a secret `Folder` (OBS: the permissions don't allow the creation of a secret folder)

```graphql
mutation{
  createFile(
    name: "File 1"
    folder: {
      name: "Folder 1"
      secret: true
    }
  ){
    id
  }
}
```
Response: 
```javascript
{
  "data": {
    "createFile": {
      "id": "cjayaukp42szx0101vopwcujd"
    }
  }
}
```

It works! The File and Folder are created! It seems that the validations are totally skipped for the nested inserts!