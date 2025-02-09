# FHIR Analytics
This document proposes an approach to make large-scale analysis of FHIR data
accessible to a larger audience and portable between systems.

The central goal is to make FHIR data work well with the best available analytic
tools, regardless of the technology stack.

*Note*: this is a work in progress and will have breaking changes without notice.

## Non-Goals
This is not a full analytic toolchain, but an attempt to adapt FHIR to such
toolchains. Therefore we scope this to create flat views of resources, and
explicitly scope out higher-level analytic capabilities since many tools do
this well today. Examples of capabilities we scope out include:

* Join operation between resources. This effort creates tabular views, but users
leverage the database engine or other tool of their choice to join them and
analyze at scale.
* Any form of data aggregation or statistical analysis.

# Requirements
The proposed system attempts to meet the following requirements:

## A portable, unambiguous specification
Any good standard is unambiguous and portable between technology stacks,
and this is no exception.

## Leverage existing standards whenever possible
Whenever practical we should avoid creating new standards and use existing approaches to these problems.

## Ability to select from repeated structures based on field values
Flattened repeated structures in FHIR requires checking the content of those fields.
For example, creating a table of patient home addresses requires checking that
the address.use field is ‘home’. Similarly, a table with columns for systolic
and diastolic blood pressures needs to check the Observation.component.code fields to select them properly.

## Ability to filter based on code values or other criteria
Many useful FHIR queries rely heavily on value sets to identify needed resources.
For instance, users may be interested in a table of statin meds for analysis,
requiring a value set of statin medication codes to allow such a flattened view
of statins. Therefore some form of value set-based filter should be used to create
the needed views.

## Support running on a wide variety of databases and tools
There are many excellent options for large-scale data analysis and new ones
continue to be created. Our efforts here should be generalizable across tools
as much as possible.

## Support direct exports from data sources
Some users have limited analytic needs and only need views over a small subset of
FHIR data that could be produced by a given system. Ideally a flattened FHIR definition
could be interpreted by a FHIR service so only the needed subset of data is
produced – whether directly in a tabular form or limited to the FHIR resources
needed for the views.

# System layers
The system consists of three logical layers: "Annotation", "View" and "Analytics". This
specification focuses primarily upon the "View" layer. The "Annotation" and "Analytics"
layers are optional, and are provided as general patterns to assist with implementation.

## 1. The Annotation Layer
The Annotation Layer is a set of lossless representations that collectively enable FHIR
to be used with a wide variety of different query technologies. It may
optionally be persisted and annotated to make it or implementations of the view
layer more efficient, but no specific annotation layer structure will be required by
this specification.

Implementers may choose from several options from this layer, including but
not limited to:

* FHIR in the NDJSON bulk format on disk
* FHIR resources stored directly as JSON in a database
* FHIR resources translated to a schematized structure within a database,
such as each FHIR field expanded into separate database columns for query
efficiency.

Implementations are encouraged but not required to further annotate the FHIR
resources to help "view" layer implementations run efficient queries. This
primarily applies when the underlying FHIR resources are stored in databases
that the "view" layer will query. Examples may include but are not limited to:

* Hashing resource IDs so they are evenly distributed, which can help some
database query engines.
* Adding native reference row IDs to FHIR References, so database engines can
efficiently join between resources.
* Expanding imprecise FHIR dates (e.g., those that have only a year) to
effectively be periods to simplify honoring date comparison semantics in the view layer.

## 2. The View Layer
The *View Layer* defines portable, tabular views of FHIR data that can more easily
be consumed by a wide variety of analytic tools. The use of these tools is
described in *Analytics Layer* section below. Our goal here is simply to get
the needed FHIR data in a form that matches user needs and common analytic
patterns.

The "view" layer itself has two key pieces:

* *View Definitions*, allowing users to define flattened views of FHIR data that
are portable between systems.
* *View Runners* are system-specific tools or libraries that apply view definitions to
the underlying annotation layer.

We will fully define View Definitions in a section below, as that is the central
aspect of this specification. View runners will be specific to the annotation
layer they use. Each annotation layer may have one or more corresponding view
runners, but a given View Definition can be run by many runners over many
annotation layers.

Example view runners may include:

* A runner that creates a virtual, tabular view in an analytic database
* A runner that queries FHIR JSON directly and creates a table in a web application
* A runner that loads data directly into a notebook or other data analysis tool

## 3. The Analytics Layer
Finally, users must be able to easily leverage the above views with the analytic
tools of their choice. This spec purposefully does not define what these are,
but common use cases may be SQL queries by consuming applications or integration
with business intelligence or data science tools.

# FHIR View Definitions
View definitions are the heart of this proposal. In a nutshell, view is tabular
projection of a FHIR resource, where the columns and criteria are defined by
FHIRPath expressions. This is defined in a simple JSON document for ease of use
and to iterate quickly, but future iterations may shift to defining views as
true FHIR resources.

## View Definition Structure
Here is the JSON structure that defines a view. The list of FHIRPath functions
and examples follow in sections below.

```js
{
  // Name of the FHIR view to be created. View runners can use this in whatever
  // way is appropriate for their use case, such as a database view or table
  // name.
  //
  // The name is limited to letters, numbers, or underscores and cannot
  // start with an underscore -- i.e. with a regular expression of
  // ^[^_][A-Za-z0-9_]+$. This makes it usable as table names in a wide variety
  // of databases.
  "name": "",

  // An optional human-readable description of the view.
  "desc": "",

  // The FHIR resource the view is based on, e.g. 'Patient' or 'Observation'.
  "resource": "",

  // An optional list of constants that can be used in any FHIRPath expression in
  // the view definition.  These are effectively strings or numbers that can be 
  // injected into FHIRPath expressions below by having `%constantName` in  the
  // expression.
  "constants": [{
    // The name of the variable that can be referenced by following variables
    // or columns, where users would use the "%variable_name" syntax.
    "name": "",

    // The value of the constant name to be used in expressions below.
    "value": ""
  }],

  // The select stanza defines the actual content of the view itself. This stanza is a list where each
  // item in is one of:
  // 
  // * a structure with the column name, expression, and optional description, 
  // * a 'from' structure indicating a relative path to pull fields from in a nested select, or
  // * a 'forEach' structure, unrolling the specified path and creating a new row for each item.
  //
  // See the comments below for details on the semantics.
  "select": [
    // Structures with a 'name' and 'expression' indicate a single colunn to select
    {
      // The name of the column produced in the output.
      //
      // The name is limited to letters, numbers, or underscores and cannot
      // start with an underscore -- i.e. with a regular expression of
      // ^[^_][A-Za-z0-9_]+$. This makes it usable as table names in a wide
      // variety of databases.
      name: "",

      // The FHIRPath expression for the column's content. This may be from
      // the resource root or from a variable defined above, using a
     // %var_name.rest.of.path structure.
      expr: "",

      // An optional human-readable description of the column.
      desc: ""},

      // A 'from' expression is a convenience to select values relative to some parent FHIRPath. 
      // This does not unnest or unroll multiple values. If the 'from' results in a FHIRPath collection, 
      // that full collection is used in the nested select, so the resulting view would have repeated
      // fields rather than a separate row per value.
      {
        // A FHIRPath expression for the parent path to select values from. 
        "from": "",

        // A nested select expression, using the same structure as defined at the root.
        "select": []
      },

      // A 'forEach' expression unnests a new row for each item in the specified FHIRPath expression, 
      // and users will select columns in the nested select. This differs from the 'from' expression above
      // because it creates a new row for each item in the matched collection, unrolling that part of the resource.
      {
        // A FHIRPath expression
        "forEach": "",

        // A nested select expression, using the same structure as defined at the root.
        "select": []
      }

  ],

  // 'where' filters are FHIRPath expressions joined with an implicit "and". This 
  // enables users to select a subset of rows that match a specific need. 
  // For example, a user may be interested only in a subset of observations 
  // based on code value and can filter them here.
  "where": [
    {
      // The FHIRPath expression for the filter.
      "expr": "",

      // An optional human-readable description of the filter.
      "desc": ""
    }
  ]
}
```

### Unnesting semantics
Some flattened FHIR views need to unnest repeated fields into a row for each item. For instance, patient addresses are repeated fields on the Patient resource, so that may be extracted to 'patient_address' table, with a row for each.

This is accomplished with the `forEach` expression seen in the view definition structure above. Here is a simple example creating rows for the city and postal code per patient:

```js
{
  "name": "patient_address",
  "resource": "Patient",
  "select": [{
    "name": "patient_id",
    "expr": "Patient.id"
  },{
    "foreach": "address"
    "select": [{
      "name": "city",
      "expr": "city",
    },{
      "name": "zip",
      "expr": "postalCode",
    }]
  }]
}
```

A more complete patient address example is below.



### Supported FHIRPath functions
Views are defined by a subset of FHIRPath, with simple nested field paths and
the following functions and operators:

#### Core required functions
All view runners must implement these functions:

##### FHIRPath subset:
* `where` function to select items in arrays (like the home address)
* `exists` function to support filtering items
* `empty` function to support filtering items
* `extension(url)` shortcut to retrieve extensions
* `join` function
* `equals` operator, primarily for use in the where function above
* `ofType` function to select the desired value type
* `first` function
* boolean operators (_and_, _or_, _not_)
* basic arithmetic (+, -, *, /)
* comparisons: =, !=, >, <=
* Literals for strings, numbers

##### Additional functions:
* `getId` function to return the resource id from a [reference](https://hl7.org/fhir/references.html#Reference) element. This function has no parameters. Note that implementations of getId may differ based on the structure of the underlying data being queried. For example, potential approaches could include dynamically extracting the last segment from the reference URL, looking up the id from an annotation stored in the annotation layer, or calculating a hash of the full reference URL.

#### Optional functions
View runners are encouraged to implement these functions as well to support a
broader set of use cases:

* Simple unit conversion operators. This is optional in FHIRPath as well, but
support would simplify data analysis patterns.
* `memberOf` function to allow checking for value sets. This can help filter
resource selection for the specific use case, like finding all instances of
a specific condition.

#### Additional functions to consider for inclusion:
Some additional functions are under discussion for inclusion but have not yet
reached consensus:

* string `split`, `substring`, `matches`, `startsWith`
* terminology `.hasConcept(system,code)`
* combining collections with `|`
* set membership checks

## Date/time conversion behavior
TODO -- Many datasets reliably have full dates in fields, so we should look
for ways to use first-class date types in the resulting view to improve
user experience on top of them.

## Examples

### Simple patient demographics

```js
{
  "name": "patient_demographics",
  "resource": "Patient",
  "desc": "A view of simple patient demographics",
  "select" [{
      "name": "id", 
      "expr": "id"
    },{
      "name": "gender", 
      "expr": "gender"
    },{
      // Use the first 'official' patient name for our demographics table.
      // Selects within this stanza are relative to the `from` result.
      "from": "name.where(use = 'official').first()",
      "select" [{
        "name": "given_name",
        // Use the FHIRPath join function to create a single given name field.
        "expr": "given.join(' ')",
      },{
        "name": "family_name",
        "expr": "family",
      }]
    }
  ]
}
```

### Unnesting repeated fields
Here is a more complete example of unnesting patient addresses into multiple rows:


```js
{
  "name": "patient_address",
  "resource": "Patient",
  "select": [{
    "name": "patient_id", 
    "expr": "Patient.id"
  },{
    // "foreach" rather than "from" to indicate we are unrolling these into separate rows
    "foreach": "address"
    "select": [{
      "name": "street",
      // Join all address lines together for this simple table.
      "expr": "line.join('\n')",
      "desc": "The full street address, including newlines if present."
    },{
      "name": "use",
      "expr": "use",
    },{
      "name": "city",
      "expr": "city",
    },{
      "name": "zip",
      "expr": "postalCode",
    }]
  }]
}
```

### Flattened Blood Pressure
An example demonstrating how to flatten a blood pressure Observation resource 
that complies with the [US Core](https://build.fhir.org/ig/HL7/US-Core/StructureDefinition-us-core-blood-pressure.html) 
profile. This definition will result in one row per blood pressure where 
both systolic and diastolic values are present.

```js
{
  "name": "us_core_blood_pressure",
  "resource": "Observation",
  // Thie example uses constants since these strings are repeated in multiple places.
  "constants": [
    {"name": "sbp_component", "value": "component.where(code.coding.exists(system='http://loinc.org' and code='8480-6')).first()"},
    {"name": "dbp_component", "value": "component.where(code.coding.exists(system='http://loinc.org' and code='8462-4')).first()"}],
  "select": [
    // Selects the columns relative to the resource root, since there is no "from" expression"
    {"name": "id", "expr": "id"},
    {"name": "patient_id", "expr": "subject.getId()"},
    {"name": "effective_date_time", "expr": "effective.ofType(dateTime)"},
    // Nested selects to retrieve items from specific locations within the resource. Since this is "select" 
    // rather than "forEach", it will not unroll into multiple rows.
    {
      // Select columns relative to the "from" expression. We reuse the constant above to reduce duplication.
      "from": "%sbp_component",
      "select": [
        {"name": "sbp_quantity_system",  "expr": "value.ofType(Quantity).system"},
        {"name": "sbp_quantity_code",  "expr": "value.ofType(Quantity).code"},
        {"name": "sbp_quantity_display",  "expr": "value.ofType(Quantity).unit"},
        {"name": "sbp_quantity_value",  "expr": "value.ofType(Quantity).value"}]
    },{
      "from": "%dbp_component",
      "select": [
        {"name": "dbp_quantity_system",  "expr": "value.ofType(Quantity).system"},
        {"name": "dbp_quantity_code",  "expr": "value.ofType(Quantity).code"},
        {"name": "dbp_quantity_display",  "expr": "value.ofType(Quantity).unit"},
        {"name": "dbp_quantity_value",  "expr": "value.ofType(Quantity).value"}]
      }]
  // filter to blood pressure observations without a data absent reason
  // for the systolic or diastolic component
  "where": [
    {"expr": "code.coding.exists(system='http://loinc.org' and code='85354-9')"},
    {"expr": "%sbp_component.dataAbsentReason.empty()"},
    {"expr": "%dbp_component.dataAbsentReason.empty()"}]
}


```

# Open questions and needs
The following open questions and needs remain:

* Define a pattern for working with imprecise date types.
  * Consider adding metadata or another approach to convert to full date types in the
  view if the underlying dataset supports it?
* Define a pattern and create examples for joining resources via references
* Additional examples to illustrate usage patterns
* Pressure test this with known use cases


