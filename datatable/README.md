# DataTable

* goal
  * way datatables -- can be -- converted 

* DataTable
  * == simple data structure
  * allows
    * using and transforming Gherkin data tables | Cucumber
  * goal
    * manual conversion | step definitions
    * automatic conversion -- by -- Cucumber

* how to register converters -> see [cucumber-java/README.md](../cucumber-java)

## Introduction

* goal
  * how data tables -- are mapped to -- certain data structures

* conversion -- can be done by --
  * Cucumber or
  * manually

* _Example:_ let's have a simple data table

    ```gherkin
    | firstName   | lastName | birthDate  |
    | Annie M. G. | Schmidt  | 1911-03-20 |
    | Roald       | Dahl     | 1916-09-13 |
    | Astrid      | Lindgren | 1907-11-14 |
    ```

  * natural representation -- would be a -- `java type: List<List<String>>`
    * Next one: ❌NOT useful ❌
      * Reason: 🧠NO labeled 🧠

        ```json
        [ 
          [ "firstName", "lastName", "birthDate" ],
          [ "Annie M.G", "Schmidt", "1911-03-20" ], 
          [ "Roald", "Dahl", "1916-09-13" ], 
          [ "Astrid", "Lindgren", "1907-11-14" ] 
        ]
        ```  

  * -- convert to -- `java type: List<Map<String, String>>`

    ```json
    [
      { "firstName": "Annie M.G", "lastName": "Schmidt",  "birthDate": "1911-03-20" }, 
      { "firstName": "Roald",     "lastName": "Dahl",     "birthDate": "1916-09-13" }, 
      { "firstName": "Astrid",    "lastName": "Lindgren", "birthDate": "1907-11-14" } 
    ]
    ```  

  * table's keys | first column -- via -- `java type: Map<String, String>`

    ```gherkin
    | KMSY | Louis Armstrong New Orleans International Airport |
    | KSFO | San Francisco International Airport               |
    | KSEA | Seattle–Tacoma International Airport              |
    | KJFK | John F. Kennedy International Airport             |
    ```

    ```json
    {
      "KMSY": "Louis Armstrong New Orleans International Airport",
      "KSFO": "San Francisco International Airport",
      "KSEA": "Seattle–Tacoma International Airport",
      "KJFK": "John F. Kennedy International Airport"
    }
    ```

* _Example:_ table / have MULTIPLE column values / key
  * table of airport codes + their coordinates expressed | latitude and longitude

    ```gherkin
    | KMSY | 29.993333 |  -90.258056 |
    | KSFO | 37.618889 | -122.375000 |
    | KSEA | 47.448889 | -122.309444 |
    | KJFK | 40.639722 |  -73.778889 |
    ```

    * -- via mapping to -- `java type: Map<String, List<String>>`


        ```json
        {
          "KMSY": ["29.993333", "-90.258056"],
          "KSFO": ["37.618889", "-122.375000"],
          "KSEA": ["47.448889", "-122.309444"],
          "KJFK": ["40.639722", "-73.778889"]
        }
        ```

  * adding a table's header / FIRST cell left blank

    ```gherkin
    |      |       lat |         lon |  
    | KMSY | 29.993333 |  -90.258056 |
    | KSFO | 37.618889 | -122.375000 |
    | KSEA | 47.448889 | -122.309444 |
    | KJFK | 40.639722 |  -73.778889 |
    ```
    * -- via mapping to -- `java type: Map<String, Map<String, String>>`

        ```json
        {
          "KMSY": { "lat": "29.993333", "lon": "-90.258056" },
          "KSFO": { "lat": "37.618889", "lon": "-122.375000" },
          "KSEA": { "lat": "47.448889", "lon": "-122.309444" },
          "KJFK": { "lat": "40.639722", "lon": "-73.778889" }
        }
        ```

## Table Types

* data table's individual cells -- can be transformed to -- 
  * Integers
  * Floats
  * Strings
  * & | JVM, ALSO
    * `BigInteger`,
    * `BigDecimal`,
    * `Byte`,
    * `Short`,
    * `Long`
    * `Double`
    * `Optional<T>`

* _Example:_ -- to -- `java type: Map<String, Map<String, Double>>`

  ```json
  {
    "KMSY": { "lat": 29.993333, "lon": -90.258056 },
    "KSFO": { "lat": 37.618889, "lon": -122.375 },
    "KSEA": { "lat": 47.448889, "lon": -122.309444 },
    "KJFK": { "lat": 40.639722, "lon": -73.778889 }
  }
  ```

### Custom Table Types

* TODO:
You can define custom data table types to represent tables from your own
domain. Doing this has the following benefits:

1. Automatic conversion to custom types
2. Document and evolve your ubiquitous domain language
3. Enforce certain patterns

There are two helpers for defining custom table types:

```java
// Defines a DataTableType that converts an entry (map of header name to row value) 
// to an object, using reflection.
registry.defineDataTableType(DataTableType#entry(Class))

// Defines a DataTableType that converts a single cell
// to an object, by calling its `String` constructor (if it exists).
registry.defineDataTableType(DataTableType#cell(Class))
```

In cases where these two reflection-based helpers are insufficient,
a custom table type can be registered as follows:

```java
registry.defineDataTableType(
  new DataTableType(
    LocalDate.class,                            // type
    new TableCellTransformer<LocalDate>() {     // transformer
  
      @Override
      public LocalDate transform(String cell) {
          return new LocalDate.parse(cell);
      }
    }, 
  )
```

The parameters are as follows:

* `type`
* `transformer` - a function that transforms either a cell, table entry, table
  row or table.

There are four ways to transform a table:

1. Transform the cells. Each cell represents an object.
2. Transform the rows. Each row represents an object.
3. Transform the entries. The entries of row paired with its corresponding
   header represent an object.
4. Transform the table. The table as a whole is transformed into a single
   object.

When combined, these four transforms are sufficient to convert a table to any
other reasonable type.

#### Example

Previously, we transformed the geolocation of airports to a map of Doubles. The
domain however uses a `Geolocation(latitude:Double, longitude:Double)` object to
represent geolocations. Airports are represented by `Airport(code:String)`.

```gherkin
|      |       lat |         lon |
| KMSY | 29.993333 |  -90.258056 |
| KSFO | 37.618889 | -122.375000 |
| KSEA | 47.448889 | -122.309444 |
| KJFK | 40.639722 |  -73.778889 |
```

By registering two table types:

```java
registry.defineDataTableType(DataTableType.cell(Airport.class));
registry.defineDataTableType(DataTableType.entry(Geolocation.class));
```

Alternatively, you can implement your own types if you need more control:

```java
registry.defineDataTableType(
    new DataTableType(
        "airport",
        Airport.class,
        new TableCellTransformer<Airport>() {
            @Override
            public Airport transform(String cell) {
                return new Airport(cell);
            }
        }
    )
);

registry.defineDataTableType(
    new DataTableType(
        Geolocation.class,
        new TableEntryTransformer<Geolocation>() {
            @Override
            public Geolocation transform(Map<String, String> entry) {
                return new Geolocation(
                    parseDouble(entry.get("lat")),
                    parseDouble(entry.get("lon"))
                );
            }
        }
    )
);
```

The table can be transformed to a map of airports to geolocations.

`java type: Map<Airport, Geolocation>`

```js
{
  Airport(code = "KMSY"): Geolocation(lat = 29.993333, lon = -90.258056 ),
  Airport(code = "KSFO"): Geolocation(lat = 37.618889, lon = -122.375 ),
  Airport(code = "KSEA"): Geolocation(lat = 47.448889, lon = -122.309444 ),
  Airport(code = "KJFK"): Geolocation(lat = 40.639722, lon = -73.778889 )
}
```

If the table does not include a header row, then a `TableRowTransformer` must be used.
As both the table row and entry transformer create a list of `Geolocation`,
it is recommended that you pick one representation only.

```gherkin
| KMSY | 29.993333 | -90.258056  |
| KSFO | 37.618889 | -122.375    |
| KSEA | 47.448889 | -122.309444 |
| KJFK | 40.639722 | -73.778889  |
```

```java
registry.defineDataTableType(
    new DataTableType(
        Geolocation.class,
        new TableRowTransformer<Geolocation>() {
            @Override
            public Geolocation transform(List<String> tableRow) {
                return new Geolocation(
                    Double.parseDouble(tableRow.get(0)),
                    Double.parseDouble(tableRow.get(1))
                );
            }
        }
    )
);
```

Custom transformation can also transform a table into a single object.

```gherkin
|   | A | B | C | 
| 3 | ♘ |   | ♝ | 
| 2 |   |   |   | 
| 1 |   | ♝ |   | 
```

```java
registry.defineDataTableType(new DataTableType(
  ChessBoard.class,
  new TableTransformer<ChessBoard>() {
    @Override
    public ChessBoard transform(DataTable table) {
        return new ChessBoard(table.subTable(1, 1).asList());
    }
  })
);

```

`java type: ChessBoard`

```
[A chess board with one black knight and two white bishops]
```

### Default Table Types

So far, all examples required transforms to be written manually. This is quite burdensome. By defining and registering
a `TableEntryByTypeTransformer` and `TableCellByTypeTransformer` it is possible to transform all table entries and cells
with a custom object mapper (e.g., Jackson Databind).


```java
private class JacksonDataTableTransformer implements TableEntryByTypeTransformer, TableCellByTypeTransformer {

    ObjectMapper objectMapper = new com.fasterxml.jackson.databind.ObjectMapper();
    
    @Override
    public <T> T transform(String value, Class<T> cellType) {
        return objectMapper.convertValue(value, cellType);
    }

    @Override
    public <T> T transform(Map<String, String> entry, Class<T> type, TableCellByTypeTransformer cellTransformer) {
        return objectMapper.convertValue(entry, type);
    }
}
```

The`TableEntryByTypeTransformer` and `TableCellByTypeTransformer` are used when there is no table entry or table cell
defined for a given type. Note that when installing both `TableEntryByTypeTransformer` and `TableCellByTypeTransformer`
it becomes impossible to disambiguate between table entries and table cells. By default, table entries are assumed over
table cells. This ambiguity can be resolved by adding a header.

## Diffing

Two tables can be compared using the `diff` or `unorderedDiff` methods.
This is useful for comparing a table with data from another system,
such as a UI or a database:

```java
DataTable actualTable = DataTable.create(listOfListOfString) // From DOM, DB or other source
expectedTable.diff(actualTable) // Throws an exception if they are not equal
```

You can also use [Hamcrest](http://hamcrest.org/) matchers from the `io.cucumber:datatable-matchers` module:

```java
assertThat(actualTable, hasTheSameRowsAs(expectedTable).inOrder());
assertThat(actualTable, hasTheSameRowsAs(expectedTable));
```

## DataTable object

* == table
  * m x n immutable -- of -- string values
  * == >=0 cells
    * if a table has 0 height or 0 width -> has 0 width or 0 height 
  * table header
    * == first row of the table
  * table body
    * == ALL table's elements / != table header

* built-in methods
  * `diff`
    * if 2 tables are DIFFERENT -> throws an exception 
  * `unorderedDiff`
    * if the 2 tables are different / ordering NOT taken in account -> throws an exception 
  * `isEmpty`
    * if the table has no cells -> returns true 
  * `transpose`
    * returns a transposed table
  * `height`
    * returns the table's height
  * `width`
    * returns the table's width
  * `cells`
    * returns the table's cells -- as a -- List<List<string>>
  * `row(index)`
    * returns 1! row
  * `rows(fromRow, toRow)`` returns table containing the rows between `fromRow`
    (inclusive) to `toRow` (exclusive).
  * `column(index)`
    * returns 1! column
  * `columns(fromColumn, toColumn)`` returns table containing the columns
    between `fromColumn` (inclusive) to `toColumn` (exclusive).
  * `subTable(fromRow, fromColumn, toRow, toColumn)` returns a tablw containing the
    cells between `fromRow` and `fromColumn` (inclusive) to `toRow` and `toColumn` (exclusive)
  * transformers / table -- is converted into -- OTHER data structures
    * `asList|Lists(type)`
      * table -- is converted to a -- List<someType> or List<List<someType>> 
    * `asMap|Maps(keyType, valueType)`
      * table -- is converted to a -- map<someType>
    * `convert(type)`
      * table -- is converted to a --  object<someType>

## For contributors

### Transformation in detail.

As described earlier, there are four primitive table types. These can be used to transform a table into a
list of lists, a list of maps, a map of string to lists, or a single object. These transformations follow a number of
simple algorithms.

**TableCellTransformer => list of lists of objects**

1. Determine the type of the object.
2. If no type could be determined, assume it to be string.
3. Lookup the TableCellTransformer for that type.
4. Apply the transformer to each cell.

**TableEntryTransformer => a list of maps of keys to values**

1. Split the header from the body of the table. Both are still tables.

```gherkin
Header: | firstName   | lastName | birthDate  |

  Body: | Annie M. G. | Schmidt  | 1911-03-20 |
        | Roald       | Dahl     | 1916-09-13 |
        | Astrid      | Lindgren | 1907-11-14 |
```

2. Transform the header to a list of lists take the first element
3. Transform the body to a list of lists.
4. For each row pair the elements of header with the elements of that row.

**TableRowTransformer => a list of objects**

1. Determine the type of the object.
2. If no type could be determined, assume it to be string.
3. Lookup the TableRowTransformer for that type.
4. Apply the transformer to each row.

**TableTransformer => an object**

1. Determine the type of the object.
2. Lookup the TableTransformer for that type.
4. Apply the transformer to the table.

**Combined => a map of keys and values**

Maps can be created combining the previous transformers.

1. Split the keys from the values in the table. Both are still tables.

```gherkin
         Keys:              Values:
Header: | firstName   |    | lastName | birthDate  |  

  Body: | Annie M. G. |    | Schmidt  | 1911-03-20 |
        | Roald       | => | Dahl     | 1916-09-13 |
        | Astrid      |    | Lindgren | 1907-11-14 
```


2a. If the first table cell is blank, use the TableCellTransformer to convert the other cells in the column.  
2b. Otherwise, use the TableEntryTransformer.

3a. If the first table cell is blank, use the TableEntryTransformer to convert the body values.  
3b. Otherwise, use the TableRowTransformer on all values.

4. Pair up the keys and values from steps 2 and 3.
