# Enhancing Attribute Autocomplete Functionality -  Auto Correction and Type Suggestion

In this blog, we discuss the two new enhancement added to the attribute name autocomplete feature in Texera, auto attribute correction and attribute type suggestion, making the feature more user-friendly.

## Background

### What are operator properties

Texera workflows consist of sequences of operators. Every operator has properties as the parameters of the data process function. For example, in Fig. 1, the "Filter" operator has three properties, `Attribute`, `Condition`, and `Value`. The property values in Fig. 1 indicate that the operator filters the records with amount greater than 1.

**![](https://lh6.googleusercontent.com/lxGfMeDfnw14f4NskeiSVBlz3_CFUQYrQJ8l97GnDo5L5LXoZ4w8mxJUa9AnnOWn7FLlxO-kwmValePube-Hw21BlwuhzFLco_pPhdrJu7SHObcQXJEtug4h-oBDIqnXdGXfwYlerpGvkZyaZ84bK4s)**

Fig. 1, Filter operator properties

### What is operator input schema

Various formats of data can be processed by Texera, and the format of data is represented by its schema, which is a set of attributes. In order to let every operator know what attributes of the data are available, each operator deduces its output schema using its input schema and propagates it to the next operator. This mechanism is called "schema propagation." Schema propagation is an important part of the system, as it is the base of many useful features such as operator validation and attribute autocomplete.

### What is attribute autocomplete

Many operators have properties that take an attribute name of the input data as value. For example, the `Attribute` property in the "Sentiment Analysis" operator takes the name of the attribute on which it will perform analysis. Autocomplete feature allows users to select an input attribute from a dropdown menu instead of manually typing in the name.

**![](https://lh3.googleusercontent.com/vqgcIM-ae7mRCIbah-ZyZRJkM4T4MMNdo3HGQtVT0RFfeh9J69z7SIUspMfY1IUjJ0V30U84PLdBMd3QouXKLEbfjUBc3Ac5q3_CtKk5XAP55I3QY0oG2U6Q11TIWcXUrVfjhbGpUUgV7LPp1R4ACeM)**

Fig. 2, Dropdown menu for attribute autocomplete

## Auto Attribute Correction

### Motivation

In the previous autocomplete implementation, when an attribute name in the input schema of an operator changes, the field of this operator that selects the attribute will not update its value according to the change in the attribute name, and the operator will therefore become invalid. Intuitively, it is expected that the field will be updated accordingly. 

Therefore, we introduce the auto attribute correction feature that automatically corrects the attribute selection in operators whenever an attribute is deleted or renamed.

### Demonstration

The auto attribute correction feature updates the attribute selection in an operator under the following circumstances

1. when the name of the attribute selected in the operator changes,
2. when the attribute selected in the operator is deleted

#### When attribute is renamed

When user renames an attribute through an operator, the attribute name will be updated in all succeeding operators.

Fig. 3 shows how the attribute selection is automatically updated in the "Filter" operator when the result attribute name is changed in its preceding "Sentiment Analysis" operator.

When the user changes the name of the output attribute of the "Sentiment Analysis" operator from `"A"` to `"B"`, the `Attribute` field whose value was `"A"` in the succeeding “Filter” operator is automatically updated to `"B"`.

**![](https://lh6.googleusercontent.com/Zf6wNg1uX9m_zJLkqtcKAap8kNLQttz1whOKPG1xGFolZ6it0hYMIMw2hzCLukabM0P2hoJ43M4QIjIcULYP_mAKt0N3V82L-0yuK0FOLD9xjrXSx0DSSzlqZD9Qp8QUL1KvABBpbbw5pXXrGgVpqQ)**

Fig. 3, Auto correction process when attribute is renamed

#### When attribute is deleted

In case of an attribute selected in an operator is deleted, e.g., deselected in a projection operator, all succeeding operators containing the attribute will delete the attribute from themselves and become invalid.

### Implementation

To automatically correct the attribute selection in operators' properties, we need to do the following steps:

1. Trigger auto correction the input attributes, i.e. input schema, changes
2. Determine what input attribute changes (rename or delete)
3. Correct the attribute selection

The first step is done by listening to the input schema change event. We will focus on the implementation of the second and third steps.

#### Determine attribute change

An operator knows an attribute changes when the input schema changes. However, it can be challenging to determine what attribute is changed and what it changes to because all an operator knows about its input is the input schema, which is a list of attribute names and types.

In our auto attribute correction implementation, we choose to compare the input schema before change and after change. That is, we keep a copy of the input schemas for all operators, and when an input schema changes, we compare the new input schema with the last input schema in our copy and determine the change. 

When comparing the old and new input schemas, we can easily find out what attributes are unchanged, but there maybe attributes in the old and new input schemas that do not match, which means that these attributes may be 1) newly created in the new schema, 2) renamed, or 3) deleted in the new schema.

Table 1 and 2 are examples of the old and new input schemas. Let's try to determine how it changes. 

Here is a possible situation happened to the input schema. `Column A` is unchanged since it exists in both schema. `Column B` is deleted since it exists in the old schema but disappeared in the new schema. `Column C` is renamed to `Column D` since `Column C` is no longer there in the new schema but there's a new `Column D` with the same integer type in the new schema. Lastly, `Column E` is a new attribute since there is no attribute similar to it in the old schema. 

However, this determination could be wrong. It is possible that `Column C` is not renamed but deleted and `Column D` is a new attribute with the same type string. Moreover, it is also possible, despite unlikely, that `Column B` is not deleted but renamed to `Column E` and converted from integer type to boolean type.

|Attribute Name|Attribute Type|
|--|--|
|Column A|integer|
|Column B|integer|
|Column C|string|

Table 1, Example 1 - old input schema

|Attribute Name|Attribute Type|
|--|--|
|Column A|integer|
|Column D|string|
|Column E|boolean|

Table 2, Example 1 - new input schema

Given that the determination is not unique, we need an algorithm to determine what are the most likely attribute changes happened to the input schema update. With the algorithm, we can know for the unmatched attribute, what attribute in the old schema corresponds to the attribute in the new schema, or whether an attribute is created or deleted. Inspired by how we compute the similarity between two strings in computer science using the edit distance as the metric, we adapt our own edit distance algorithm to compute the similarity between two attributes (name and type). 

In the Levenshtein edit distance algorithm used for strings, the distance is measured by the minimum number of single-character edits (insertions, deletions or substitutions) required to change one word into the other. For example, the distance between "**k**itten" and  "**s**itten" is 1 because of the substitution of "s" for "k". ([Levenshtein distance - Wikipedia](https://en.wikipedia.org/wiki/Levenshtein_distance))

In our attribute edit distance algorithm, we compute the distance based on the name and the type. Unlike the Levenshtein distance where single-character insertions, deletions, and substitutions all have distance 1, we consider insertion and deletion to have distance 1, and update to have various distance depending on the name and type change, reflected in Table 3.

The distance is based on the likelyhood of the change. For the edit distance for update, if the attribute name and type are unchanged, then it is most likely that the attribute itself is unchanged, thus no distance. If either attribute name or type is changed, then it has distance value 1. If both the attribute name and type are changed, it is considered unlikely, thus having distance value 2.

| Attribute Name | Attribute Type | Distance |
|--|--|--|
| Unchanged | Unchanged | 0 |
| Changed | Unchanged | 1 |
| Unchanged | Changed | 1 |
| Changed | Changed | 2 |

Table 3, Attribute edit distance for update

With this algorithm, for the old and new input schema example, we can say that `Column C` is believed to be renamed to `Column D`, instead of that `Column C` is deleted and `Column D` is a new attribute, because the former case has distance value 1 while the later case has distance value 2 (1 deletion+ 1 insertion). Also, `Column B` is deleted instead of renamed and type casted to `Column E`, because deletion only has distance 1, but the rename and type casted is considered an update of distance 2.

#### Correct attribute selection

In every operator involving input schema changes, after the determination, we update the attribute selection in the operators recursively based on the determination.

The property update is conducted in a recursive way because the object of opeartor properties can be nested with deep property. All property values are recursively traversed to find the property with value of old attribute selection and update with new attribute name.

### Limitations

The implementation of the auto attribute correction cannot always find out the correct attribute changes because of the attribute edit distance algorithm. We should notice due to the limited information that an operator knows about its input schema, it can only "guess" what changes happened, but cannot confidently determine the changes. While we find out the most likely changes using our edit distance algorithm, the method shall not be considered 100% correct. Moreover, in some cases, two possible changes may be the same edit distance. In the Table 4 and 5 example, it is possible that `Column A` is renamed to `Column C` and `Column B` is renamed to `Column D`, meanwhile it is also possible that `Column A` is renamed to `Column D` and `Column B` is renamed to `Column C`. Both possibilities have distance 2. In this case, our edit distance algorithm may take either result.

|Attribute Name|Attribute Type|
|--|--|
|Column A|integer|
|Column B|integer|

Table 4, Example 2 - old input schema

|Attribute Name|Attribute Type|
|--|--|
|Column C|integer|
|Column D|integer|

Table 5, Example 2 - new input schema

## Attribute Type Suggestion

### Motivation

Many operators have type constraints on the attribute which they process. For example, the "Sentiment Analysis" operator is designed for text analysis, so it should only accept string-typed attribute as input. However, the attribute autocomplete feature allows users to select attributes with any type from the dropdown list. As a result, users may select a wrong attribute that the operator cannot handle or produce not meaningful results.

Therefore, we introduced the attribute type suggestion feature to warn users when they select a attribute with a type that is not suitable in the context.

### Demonstration

Operators have different constrains on the attribute type depending on their functionalities. We will take "Sentiment Analysis", "Hash Join", and "Aggregate" operators as example.

#### Sentiment Analysis Operator

"Sentiment Analysis" operator takes one attribute of string type. When users select an attribute with integer type, we prompt a warning message.

**![](https://lh6.googleusercontent.com/GO96SesVN8fAdWrrR59a6Dq_8ocI5ioaEOkCQ20oo-RNqHzuFUheMXf9-bMziEJ-RZ33Xaw8MUqqri0L1JfXK-hjaHOAH-AaBuRR7ArMmn4Ao3k0MT3rA6J8h_2DoNIpw1Sb4HyfN2Rb7jFJrPeJIek)**
Fig. 4, Sentimen Analysis type suggestion example

#### Hash Join Operator

"Hash Join" operator requires the two key attribute to have the same type.

**![](https://lh4.googleusercontent.com/sDvAMcvoZlZ9IhEL4Vt3FxtqdgmszacOGzoS7HzhKGhGUS0tM_b_alb_eSJushXsqWAaazlBQ_gpvSsw1-ejhvSqFJf_-TIRNd5l6NQHpNU7MwyYdQ8FMRlHXb9zo2YpcZRohpKF3tVZ7cdScZxymA8)**

Fig. 5, Hash Join type suggestion example

#### Aggregate Opeartor

"Aggregate" operator supports multiple aggregation functions, like sum, count, concat, etc. Each aggregation function has different requirement regarding its input attribute. For example, sum function must be computed on a numeric type attribute (integer, long, double, or timestamp), and concat function must be computed on a string type attribute, while count function has no type requirement.

**![](https://lh6.googleusercontent.com/2TSCNDgPyQz-1XAQneiRzDkXg4Ig1ZGHpcukHgXqD9YRUX_HPCOi0l9chI-zM2BsM4F9zABxIUt06UO3Abgy-at6RrzPf2Z71wERAWpS-HpxefwtBMcin0Fu6afTy86GsSyE-nAEhkBJlGlim6W34qU)**

Fig. 6, Aggregate type suggestion example

### Implementation

Given that each operator has different type constraints, we write attribute type rules for each operator that has type constraint and provide warning when the attribute selection violates the rules. 

We will focus how the type rules are written and checked.

The attribute type rules are considered operator metadata and thus are a part of the operator schema which is supplied from the backend. As the operator schema is in JSON Schema format, we designed our attribute type rules JSON to match the JSON Schema as much as possible. The attribute type rules are injected to in operator schemas as an attribute called `attributeTypeRules`. 

The attribute type rules may contain three constraints:  `enum`  that defines the possible values of the type,  `const`  (`const.$data`) that is used to check equlity of two attribute types, and  `allOf`  (for  `if`-`then`  sets) for more complicated cases. The rules may have multiple constraints but must satisfy all of them.

In general, our schema has the following format.

```
{
  "attributeTypeRules": {
    "<attribute name>": {
      <constraints>
    },
    ...
  }
}
```

Here are the attribute type schemas for the three operators. Those schemas cover the most common situations.

### Check if the attribute type is in a set

In "Sentiment Analysis" operator, the attribute type of `attribute` must be string.

```JSON
{  
  "attributeTypeRules": {  
    "attribute": {  
      "enum": ["string"]  
    }  
  }  
}
```

### Check the equality of two attribute types

In "Hash Join" operator, the attribute type of `buildAttributeName` (Left Input Attribute) must equal to that of `probeAttributeName` (Right Input Attribute).

```JSON
{
  "attributeTypeRules":{
    "buildAttributeName":{
      "const":{
        "$data":"probeAttributeName"
      }
    }
  }
}
```

*Note: the  `$data`  keyword is used to implement value comparison. Currently, it is the only way to check equality between values in JSON Schema. However, the  `$data`  keyword proposal is included not in the formal JSON Schema standard. See  [json-schema-org/json-schema-spec#51](https://github.com/json-schema-org/json-schema-spec/issues/51), and  [https://ajv.js.org/guide/combining-schemas.html#data-reference](https://ajv.js.org/guide/combining-schemas.html#data-reference).*

### Attribute type dependent on other property's value

In "Aggregate" operator, if  `aggFunction`'s value is "sum", "average", "min", or "max", then the attribute type of "attribute" can only be numeric (integer, long, double, or timestamp). Similarly, if  `aggFunction`'s value is "concat", then the attribute type of "attribute must be "string". If  `aggFunction`  has other values, like "count", then it has no constraint.

All  `if-then`  in  the `allOf`  array must be satisfied. 

```JSON
{
  "attributeTypeRules": {
    "attribute": {
      "allOf": [
        {
          "if": {
            "aggFunction": {
              "valEnum": ["sum", "average", "min", "max"]
            }
          },
          "then": {
            "enum": ["integer", "long", "double", "timestamp"]
          }
        },
        {
          "if": {
            "aggFunction": {
              "valEnum": ["concat"]
            }
          },
          "then": {
            "enum": ["string"]
          }
        }
      ]
    }
  }
}
```

*Note: the `valEnum` in the `if` clause is comparing the actual literal value of the property, not the attribute type, as opposed to elsewhere in the attribute type schema, so we used a custom keyword `valEnum` instead of `enum` here.*

### Operators with Type Rules

The following operators will have type suggestion feature. 

- Regular Expression
- Dictionary Matcher
- Keyword Search
- Sentiment Analysis
- Interval Join
- Hash Join
- Linear Regression
- Sort Partitions
- Aggregate
- Unnest String
- Type Casting
- Word Cloud
- ScatterPlot

There may be more operators to be added in the future that will support type suggestion.

## Acknowledgements

Special thanks to Dr. Zuozhi Wang, Yicong Huang, Prof. Chen Li, and the Texera team for their help in the task and the blog.