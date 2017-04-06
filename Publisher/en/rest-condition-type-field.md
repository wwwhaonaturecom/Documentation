# REST API: Condition type field

Conditions have different types of properties. Some concern the timeframe in 
which something happened (date properties), others concern mailing information 
(mailing properties) and others concern just the specific type of condition 
(individual properties). This article is about the properties of the 
field condition.

## Navigation
* [Individual properties](rest-condition-type-field#individual-properties)
* [Example](rest-condition-type-field#example)
* [More information](rest-condition-type-field#more-information)

## Individual properties
* **comparison**: Comparison type for fieldcondition. Possible values: 
"equals", "not equals", "contains", "not contains", "less", "more", "is email", 
"regexp" and "is-numeric".
* **field**: Field to compare with value
* **value**: Value to compare with field (setting this resets other-field)
* **other-field**: Other field to compare "field" with. If this variable is set 
"value" will not be used.
* **numeric-comparison**: Boolean value to indicate whether value is done numerically or not.

## Example

Let's assume, for the purposes of the example, that we have a product only 
children like and that we know which of our customers have children. This is 
indicated in the field "hasChildren" in the fields of the database profiles. 
Now we can pick a specific target for marketing, namely parents, using the field 
condition. If we set **field** to "hasChildren" and the **value** to "yes" 
we can make a selection of only parents and email this selection.

## More information
* [Fetch rule conditions](rest-get-rule-conditions)
* [Post rule conditions](rest-post-rule-conditions)
* [Condition type interest](rest-condition-type-interest)