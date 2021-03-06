# Execution

GraphQL generates a response from a request via execution.

A request for execution consists of a few pieces of information:

* The schema to use, typically solely provided by the GraphQL service.
* A Document containing GraphQL Operations and Fragments to execute.
* Optionally: The name of the Operation in the Document to execute.
* Optionally: Values for any Variables defined by the Operation.
* An initial value corresponding to the root type being executed.
  Conceptually, an initial value represents the "universe" of data available via
  a GraphQL Service. It is common for a GraphQL Service to always use the same
  initial value for every request.

Given this information, the result of {ExecuteRequest()} produces the response,
to be formatted according to the Response section below.


## Executing Requests

To execute a request, the executor must have a parsed `Document` (as defined
in the “Query Language” part of this spec) and a selected operation name to
run if the document defines multiple operations, otherwise the document is
expected to only contain a single operation. How this information is passed to the executor is determined by the implementation. The result of the request is
determined by the result of executing this operation according to the "Executing
Operations” section below.

ExecuteRequest(schema, document, operationName, variableValues, initialValue):

  * Let {operation} be the result of {GetOperation(document, operationName)}.
  * Let {coercedVariableValues} be the result of {CoerceVariableValues(schema, operation, variableValues)}.
  * If {operation} is a query operation:
    * Return {ExecuteQuery(operation, schema, coercedVariableValues, initialValue)}.
  * Otherwise if {operation} is a mutation operation:
    * Return {ExecuteMutation(operation, schema, coercedVariableValues, initialValue)}.

GetOperation(document, operationName):

  * If {operationName} is {null}:
    * If {document} contains exactly one operation.
      * Return the Operation contained in the {document}.
    * Otherwise produce a query error requiring {operationName}.
  * Otherwise:
    * Let {operation} be the Operation named {operationName} in {document}.
    * If {operation} was not found, produce a query error.
    * Return {operation}.


### Validating Requests

As explained in the Validation section, only requests which pass all validation
rules should be executed. If validation errors are known, they should be
reported in the list of "errors" in the response and the request must fail
without execution.

Typically validation is performed in the context of a request immediately
before execution, however a GraphQL service may execute a request without
immediately validating it if that exact same request is known to have been
validated before. A GraphQL service should only execute requests which *at some
point* were known to be free of any validation errors, and have since
not changed.

For example: the request may be validated during development, provided it does
not later change, or a service may validate a request once and memoize the
result to avoid validating the same request again in the future.


### Coercing Variable Values

If the operation has defined any variables, then the values for
those variables need to be coerced using the input coercion rules
of variable's declared type. If a query error is encountered during
input coercion of variable values, then the operation fails without
execution.

CoerceVariableValues(schema, operation, variableValues):

  * Let {coercedValues} be an empty unordered Map.
  * Let {variableDefinitions} be the variables defined by {operation}.
  * For each {variableDefinition} in {variableDefinitions}:
    * Let {variableName} be the name of {variableDefinition}.
    * Let {variableType} be the expected type of {variableDefinition}.
    * Let {defaultValue} be the default value for {variableDefinition}.
    * Let {value} be the value provided in {variableValues} for the name {variableName}.
    * If {value} does not exist (was not provided in {variableValues}):
      * If {defaultValue} exists (including {null}):
        * Add an entry to {coercedValues} named {variableName} with the
          value {defaultValue}.
      * Otherwise if {variableType} is a Non-Nullable type, throw a query error.
      * Otherwise, continue to the next variable definition.
    * Otherwise, if {value} cannot be coerced according to the input coercion
      rules of {variableType}, throw a query error.
    * Let {coercedValue} be the result of coercing {value} according to the
      input coercion rules of {variableType}.
    * Add an entry to {coercedValues} named {variableName} with the
      value {coercedValue}.
  * Return {coercedValues}.

Note: This algorithm is very similar to {CoerceArgumentValues()}.


## Executing Operations

The type system, as described in the “Type System” section of the spec, must
provide a query root object type. If mutations are supported, it must also
provide a mutation root object type.

If the operation is a query, the result of the operation is the result of
executing the query’s top level selection set with the query root object type.

An initial value may be provided when executing a query.

ExecuteQuery(query, schema, variableValues, initialValue):

  * Let {queryType} be the root Query type in {schema}.
  * Assert: {queryType} is an Object type.
  * Let {selectionSet} be the top level Selection Set in {query}.
  * Let {data} be the result of running
    {ExecuteSelectionSet(selectionSet, queryType, initialValue, variableValues)}
    *normally* (allowing parallelization).
  * Let {errors} be any *field errors* produced while executing the
    selection set.
  * Return an unordered map containing {data} and {errors}.

If the operation is a mutation, the result of the operation is the result of
executing the mutation’s top level selection set on the mutation root
object type. This selection set should be executed serially.

It is expected that the top level fields in a mutation operation perform
side-effects on the underlying data system. Serial execution of the provided
mutations ensures against race conditions during these side-effects.

ExecuteMutation(mutation, schema, variableValues, initialValue):

  * Let {mutationType} be the root Mutation type in {schema}.
  * Assert: {mutationType} is an Object type.
  * Let {selectionSet} be the top level Selection Set in {mutation}.
  * Let {data} be the result of running
    {ExecuteSelectionSet(selectionSet, mutationType, initialValue, variableValues)}
    *serially*.
  * Let {errors} be any *field errors* produced while executing the
    selection set.
  * Return an unordered map containing {data} and {errors}.


## Executing Selection Sets

To execute a selection set, the object value being evaluated and the object type
need to be known, as well as whether it must be executed serially, or may be
executed in parallel.

First, the selection set is turned into a grouped field set; then, each
represented field in the grouped field set produces an entry into a
response map.

ExecuteSelectionSet(selectionSet, objectType, objectValue, variableValues):

  * Let {groupedFieldSet} be the result of
    {CollectFields(objectType, selectionSet, variableValues)}.
  * Initialize {resultMap} to an empty ordered map.
  * For each {groupedFieldSet} as {responseKey} and {fields}:
    * Let {fieldName} be the name of the first entry in {fields}.
      Note: This value is unaffected if an alias is used.
    * Let {fieldType} be the return type defined for the field {fieldName} of {objectType}.
    * If {fieldType} is {null}:
      * Continue to the next iteration of {groupedFieldSet}.
    * Let {responseValue} be {ExecuteField(objectType, objectValue, fields, fieldType, variableValues)}.
    * Set {responseValue} as the value for {responseKey} in {resultMap}.
  * Return {resultMap}.

Note: {responseMap} is ordered by which fields appear first in the query. This
is explained in greater detail in the Field Collection section below.


### Normal and Serial Execution

Normally the executor can execute the entries in a grouped field set in whatever
order it chooses (often in parallel). Because the resolution of fields other
than top-level mutation fields must always be side effect-free and idempotent,
the execution order must not affect the result, and hence the server has the
freedom to execute the field entries in whatever order it deems optimal.

For example, given the following grouped field set to be executed normally:

```graphql
{
  birthday {
    month
  }
  address {
    street
  }
}
```

A valid GraphQL executor can resolve the four fields in whatever order it
chose (however of course `birthday` must be resolved before `month`, and
`address` before `street`).

When executing a mutation, the selections in the top most selection set will be
executed in serial order.

When executing a grouped field set serially, the executor must consider each entry
from the grouped field set in the order provided in the grouped field set. It must
determine the corresponding entry in the result map for each item to completion
before it continues on to the next item in the grouped field set:

For example, given the following selection set to be executed serially:

```graphql
{
  changeBirthday(birthday: $newBirthday) {
    month
  }
  changeAddress(address: $newAddress) {
    street
  }
}
```

The executor must, in serial:

 - Run {ExecuteField()} for `changeBirthday`, which during {CompleteValue()}
   will execute the `{ month }` sub-selection set normally.
 - Run {ExecuteField()} for `changeAddress`, which during {CompleteValue()}
   will execute the `{ street }` sub-selection set normally.

As an illustrative example, let's assume we have a mutation field
`changeTheNumber` that returns an object containing one field,
`theNumber`. If we execute the following selection set serially:

```graphql
{
  first: changeTheNumber(newNumber: 1) {
    theNumber
  }
  second: changeTheNumber(newNumber: 3) {
    theNumber
  }
  third: changeTheNumber(newNumber: 2) {
    theNumber
  }
}
```

The executor will execute the following serially:

 - Resolve the `changeTheNumber(newNumber: 1)` field
 - Execute the `{ theNumber }` sub-selection set of `first` normally
 - Resolve the `changeTheNumber(newNumber: 3)` field
 - Execute the `{ theNumber }` sub-selection set of `second` normally
 - Resolve the `changeTheNumber(newNumber: 2)` field
 - Execute the `{ theNumber }` sub-selection set of `third` normally

A correct executor must generate the following result for that selection set:

```js
{
  "first": {
    "theNumber": 1
  },
  "second": {
    "theNumber": 3
  },
  "third": {
    "theNumber": 2
  }
}
```


### Field Collection

Before execution, the selection set is converted to a grouped field set by
calling {CollectFields()}. Each entry in the grouped field set is a list of
fields that share a response key. This ensures all fields with the same response
key (alias or field name) included via referenced fragments are executed at the
same time.

As an example, collecting the fields of this selection set would collect two
instances of the field `a` and one of field `b`:

```graphql
{
  a {
    subfield1
  }
  ...ExampleFragment
}

fragment ExampleFragment on Query {
  a {
    subfield2
  }
  b
}
```

The depth-first-search order of the field groups produced by {CollectFields()}
is maintained through execution, ensuring that fields appear in the executed
response in a stable and predictable order.

CollectFields(objectType, selectionSet, variableValues, visitedFragments):

  * If {visitedFragments} if not provided, initialize it to the empty set.
  * Initialize {groupedFields} to an empty ordered map of lists.
  * For each {selection} in {selectionSet}:
    * If {selection} provides the directive `@skip`, let {skipDirective} be that directive.
      * If {skipDirective}'s {if} argument is {true} or is a variable in {variableValues} with the value {true}, continue with the next
      {selection} in {selectionSet}.
    * If {selection} provides the directive `@include`, let {includeDirective} be that directive.
      * If {includeDirective}'s {if} argument is not {true} and is not a variable in {variableValues} with the value {true}, continue with the next
      {selection} in {selectionSet}.
    * If {selection} is a {Field}:
      * Let {responseKey} be the response key of {selection}.
      * Let {groupForResponseKey} be the list in {groupedFields} for
        {responseKey}; if no such list exists, create it as an empty list.
      * Append {selection} to the {groupForResponseKey}.
    * If {selection} is a {FragmentSpread}:
      * Let {fragmentSpreadName} be the name of {selection}.
      * If {fragmentSpreadName} is in {visitedFragments}, continue with the
        next {selection} in {selectionSet}.
      * Add {fragmentSpreadName} to {visitedFragments}.
      * Let {fragment} be the Fragment in the current Document whose name is
        {fragmentSpreadName}.
      * If no such {fragment} exists, continue with the next {selection} in
        {selectionSet}.
      * Let {fragmentType} be the type condition on {fragment}.
      * If {DoesFragmentTypeApply(objectType, fragmentType)} is false, continue
        with the next {selection} in {selectionSet}.
      * Let {fragmentSelectionSet} be the top-level selection set of {fragment}.
      * Let {fragmentGroupedFieldSet} be the result of calling
        {CollectFields(objectType, fragmentSelectionSet, visitedFragments)}.
      * For each {fragmentGroup} in {fragmentGroupedFieldSet}:
        * Let {responseKey} be the response key shared by all fields in {fragmentGroup}
        * Let {groupForResponseKey} be the list in {groupedFields} for
          {responseKey}; if no such list exists, create it as an empty list.
        * Append all items in {fragmentGroup} to {groupForResponseKey}.
    * If {selection} is an {InlineFragment}:
      * Let {fragmentType} be the type condition on {selection}.
      * If {fragmentType} is not {null} and {DoesFragmentTypeApply(objectType, fragmentType)} is false, continue
        with the next {selection} in {selectionSet}.
      * Let {fragmentSelectionSet} be the top-level selection set of {selection}.
      * Let {fragmentGroupedFieldSet} be the result of calling {CollectFields(objectType, fragmentSelectionSet, variableValues, visitedFragments)}.
      * For each {fragmentGroup} in {fragmentGroupedFieldSet}:
        * Let {responseKey} be the response key shared by all fields in {fragmentGroup}
        * Let {groupForResponseKey} be the list in {groupedFields} for
          {responseKey}; if no such list exists, create it as an empty list.
        * Append all items in {fragmentGroup} to {groupForResponseKey}.
  * Return {groupedFields}.

DoesFragmentTypeApply(objectType, fragmentType):

  * If {fragmentType} is an Object Type:
    * if {objectType} and {fragmentType} are the same type, return {true}, otherwise return {false}.
  * If {fragmentType} is an Interface Type:
    * if {objectType} is an implementation of {fragmentType}, return {true} otherwise return {false}.
  * If {fragmentType} is a Union:
    * if {objectType} is a possible type of {fragmentType}, return {true} otherwise return {false}.


## Executing Fields

Each field requested in the grouped field set that is defined on the selected
objectType will result in an entry in the response map. Field execution first
coerces any provided argument values, then resolves a value for the field, and
finally completes that value either by recursively executing another selection
set or coercing a scalar value.

ExecuteField(objectType, objectValue, fieldType, fields, variableValues):
  * Let {field} be the first entry in {fields}.
  * Let {argumentValues} be the result of {CoerceArgumentValues(objectType, field, variableValues)}
  * Let {resolvedValue} be {ResolveFieldValue(objectType, objectValue, fieldName, argumentValues)}.
  * Return the result of {CompleteValue(fieldType, fields, resolvedValue, variableValues)}.


### Coercing Field Arguments

Fields may include arguments which are provided to the underlying runtime in
order to correctly produce a value. These arguments are defined by the field in
the type system to have a specific input type: Scalars, Enum, Input Object, or
List or Non-Null wrapped variations of these three.

At each argument position in a query may be a literal value or a variable to be
provided at runtime.

CoerceArgumentValues(objectType, field, variableValues):
  * Let {coercedValues} be an empty unordered Map.
  * Let {argumentValues} be the argument values provided in {field}.
  * Let {fieldName} be the name of {field}.
  * Let {argumentDefinitions} be the arguments defined by {objectType} for the
    field named {fieldName}.
  * For each {argumentDefinition} in {argumentDefinitions}:
    * Let {argumentName} be the name of {argumentDefinition}.
    * Let {argumentType} be the expected type of {argumentDefinition}.
    * Let {defaultValue} be the default value for {argumentDefinition}.
    * Let {value} be the value provided in {argumentValues} for the name {argumentName}.
    * If {value} is a Variable:
      * Let {variableName} be the name of Variable {value}.
      * Let {variableValue} be the value provided in {variableValues} for the name {variableName}.
      * If {variableValue} exists (including {null}):
        * Add an entry to {coercedValues} named {argName} with the
          value {variableValue}.
      * Otherwise, if {defaultValue} exists (including {null}):
        * Add an entry to {coercedValues} named {argName} with the
          value {defaultValue}.
      * Otherwise, if {argumentType} is a Non-Nullable type, throw a field error.
      * Otherwise, continue to the next argument definition.
    * Otherwise, if {value} does not exist (was not provided in {argumentValues}:
      * If {defaultValue} exists (including {null}):
        * Add an entry to {coercedValues} named {argName} with the
          value {defaultValue}.
      * Otherwise, if {argumentType} is a Non-Nullable type, throw a field error.
      * Otherwise, continue to the next argument definition.
    * Otherwise, if {value} cannot be coerced according to the input coercion
      rules of {argType}, throw a field error.
    * Let {coercedValue} be the result of coercing {value} according to the
      input coercion rules of {argType}.
    * Add an entry to {coercedValues} named {argName} with the
      value {coercedValue}.
  * Return {coercedValues}.

Note: Variable values are not coerced because they are expected to be coerced
before executing the operation in {CoerceVariableValues()}, and valid queries
must only allow usage of variables of appropriate types.


### Value Resolution

While nearly all of GraphQL execution can be described generically, ultimately
the internal system exposing the GraphQL interface must provide values.
This is exposed via {ResolveFieldValue}, which produces a value for a given
field on a type for a real value.

As an example, this might accept the {objectType} `Person`, the {field}
{"soulMate"}, and the {objectValue} representing John Lennon. It would be
expected to yield the value representing Yoko Ono.

ResolveFieldValue(objectType, objectValue, fieldName, argumentValues):
  * Let {resolver} be the internal function provided by {objectType} for
    determining the resolved value of a field named {fieldName}.
  * Return the result of calling {resolver}, providing {objectValue} and {argumentValues}.

Note: It is common for {resolver} to be asynchronous due to relying on reading
an underlying database or networked service to produce a value. This
necessitates the rest of a GraphQL executor to handle an asynchronous
execution flow.


### Value Completion

After resolving the value for a field, it is completed by ensuring it adheres
to the expected return type. If the return type is another Object type, then
the field execution process continues recursively.

CompleteValue(fieldType, fields, result, variableValues):
  * If the {fieldType} is a Non-Null type:
    * Let {innerType} be the inner type of {fieldType}.
    * Let {completedResult} be the result of calling
      {CompleteValue(innerType, fields, result, variableValues)}.
    * If {completedResult} is {null}, throw a field error.
    * Return {completedResult}.
  * If {result} is {null} (or another internal value similar to {null} such as
    {undefined} or {NaN}), return {null}.
  * If {fieldType} is a List type:
    * If {result} is not a collection of values, throw a field error.
    * Let {innerType} be the inner type of {fieldType}.
    * Return a list where each list item is the result of calling
      {CompleteValue(innerType, fields, resultItem, variableValues)}, where
      {resultItem} is each item in {result}.
  * If {fieldType} is a Scalar or Enum type:
    * Return the result of "coercing" {result}, ensuring it is a legal value of
      {fieldType}, otherwise {null}.
  * If {fieldType} is an Object, Interface, or Union type:
    * If {fieldType} is an Object type.
      * Let {objectType} be {fieldType}.
    * Otherwise if {fieldType} is an Interface or Union type.
      * Let {objectType} be ResolveAbstractType({fieldType}, {result}).
    * Let {subSelectionSet} be the result of calling {MergeSelectionSets(fields)}.
    * Return the result of evaluating ExecuteSelectionSet(subSelectionSet, objectType, result, variableValues) *normally* (allowing for parallelization).

**Resolving Abstract Types**

When completing a field with an abstract return type, that is an Interface or
Union return type, first the abstract type must be resolved to a relevant Object
type. This determination is made by the internal system using whatever
means appropriate.

Note: A common method of determining the Object type for an {objectValue} in
object-oriented environments, such as Java or C#, is to use the class name of
the {objectValue}.

ResolveAbstractType(abstractType, objectValue):
  * Return the result of calling the internal method provided by the type
    system for determining the Object type of {abstractType} given the
    value {objectValue}.

**Merging Selection Sets**

When more than one fields of the same name are executed in parallel, their
selection sets are merged together when completing the value in order to
continue execution of the sub-selection sets.

An example query illustrating parallel fields with the same name with
sub-selections.

```graphql
{
  me {
    firstName
  }
  me {
    lastName
  }
}
```

After resolving the value for `me`, the selection sets are merged together so
`firstName` and `lastName` can be resolved for one value.

MergeSelectionSets(fields):
  * Let {selectionSet} be an empty list.
  * For each {field} in {fields}:
    * Let {fieldSelectionSet} be the selection set of {field}.
    * If {fieldSelectionSet} is null or empty, continue to the next field.
    * Append all selections in {fieldSelectionSet} to {selectionSet}.
  * Return {selectionSet}.


### Errors and Non-Nullability

If an error is thrown while resolving a field, it should be treated as though
the field returned {null}, and an error must be added to the {"errors"} list
in the response.

If the result of resolving a field is {null} (either because the function to
resolve the field returned {null} or because an error occurred), and that
field is of a `Non-Null` type, then a field error is thrown. The
error must be added to the {"errors"} list in the response.

If the field returns {null} because of an error which has already been added to
the {"errors"} list in the response, the {"errors"} list must not be
further affected. That is, only one error should be added to the errors list per
field.

Since `Non-Null` type fields cannot be {null}, field errors are propagated to be
handled by the parent field. If the parent field may be {null} then it resolves
to {null}, otherwise if it is a `Non-Null` type, the field error is further
propagated to it's parent field.

If all fields from the root of the request to the source of the error return
`Non-Null` types, then the {"data"} entry in the response should be {null}.
