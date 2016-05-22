
Data Transformation

Effective data transformation is pivotal to delivering high quality data assets in Data products. The document
covers some mandates that we need to follow while we do so. These mandates coule lie either in a technical landscape
or exists as "data thing'. Hence these can be catagorised as more of data or more of technical.


Highlighting a few points here. 

Currently what is existing:

* Wrong usage of data types, resulting in errors that are difficult to catch and they propagate.
* No logical grouping with in a data asset, resulting in heavy dependency on an external documentation. 
* No standards around how to encode binary values.
* Improper partitioning of data assets, resulting the performance of all downstream usages.
* Improper usage/creation of persistent staging data layer, affecting the quality of business views.
* Improper encoding of values and the resulting entropy in data compression.
* No standards around how to represent a relationship data asset.
* No standards on how to partition a time variant data asset.
* No proper data dictionary from a customer perpective.
* Improper usage of toolings while development resulting in nulls in data assets.
* Considerations while bringing all fields from source data to raw views or business views.
* How do we define optional or non-optional fields as the data exist in parquet format
* When do we need to push a transformation rule as a reusable component? Do we support UDFs?
* Reloading history? A tricky thing.
* When to consider normalised or denormalised data asset?


#### Wrong usage of data types, resulting in errors that are difficult to catch and they propagate.

* Amount: 
>Any amount field has to be represented as "BigDecimal". How to represent parquet decimal in thrift structure is yet to be considered.

* Date: 
> Any date field is has to be of the type date if you are using hive 0.12+. However the current version supports only
TimeStamp as the data type. Again,the representation of this in thrift structure is yet to be considered. Also timezone seems to be a bit of tricky, and we could consider representing date as a struct with actual date and timezone in it.

* Amount and Date are tricky enough, and will be taken for further discussions in various design forums.

* Defining data type for non-transformed fields:
> The source data type has to be the same as that of the field in a business view, if the field is just a direct mapping of source field. However the correctness of source data type has to be takne into account. Ex:- A source thrift might have  amount field, and we needn't carry forward this mistake.

* Categorical Values:
> Although there is no concept of enums in hive, the thrift type should be of the type enum for certain categorical values.
> Following this standard will be really handy for downstream such as feature developers. To justify more on this, a federated-developer or feature-developer may not have thorough knowledge on what could be the possible values for a particular categorical field, and hence an enum would make his/her life easy. Usage of enums and examples are discussed
below.

#### Standards around how to encode binary values.

> Fields with binary value should be optional bool or required bool. This ensures the customer/end-user not to get baffled by the various possibilities of values for a field. This also ensures type-safety
and avoids possible that can occur with strings. However, if the the tag is not found applicable to the whole population of data assets,then it should be optional bool in your thrift structure. For an end user/customer, this will remain as true or false itself but possible values include true, false or null. 

Told that, there shouldn't be a hard-core rule that it should be bool itself. Sometimes, bool doesn't work if proper tagging 
of a dataset is not possible with your business rule. This means if your business rule requires a clear distinction between
Not Applicable (possible null in the above case) with "Not identified", then you are most probably dealing with a catagorical
value that is an enum of four values in your code base. This again, has no direct impact on the end user, but brings in more
safety that inturn makes the data less error prone. Here, schema allows you to push only one of the finite set of values.

enum {
  true,
  false,
  Not Applicable,
  Not Found
}

In our case, we could also go for a definite finite set of values for a binary feature where you need to distinguish between UNKNOWN and NOT APPLICABLE. These limited set of values can be represented as an enum consisting of Y, N, UNKNOWN, and NOT APPLICABLE. In this way, we can bring in some safety/standard so that developers/federated-developers can use only a value from this finite set.
In Hive, it would be serialized as String itself.

Something like:

enum Flags {
Y,
N,
UNKNOWN,
NOT APPLICABLE
}

struct MyBusinessView {
1: required Flags flag
}
you could also give some default value like

struct MyBusinesView {
1: required Flags flag= Flags.UNKNOWN
}
And for an end user, the query would look like ..say count the rows with flag="Y" is as simple as

select count(*) from my_business_view where flag='Y'

And for developers out there

We won't be allowed to do MyBusinessView("Y") instead, we have to do MyBusinessView(Flags.Y)


When does a field become "optional"?
For developers out there, its easy for you to change "optional" or "required" identifiers. But this has
performance implications, size implications and it indirectly affects the customers who use your data as well.

A simple theory that we follow here is
If the possible values for a field includes "null", then you could probably replace it will optional fields.

What was the alternative theory?
We tend to give empty string if a value doesn't exist. This is ridiculous, unsafe, and doesn't make any sense at all.

Example:
  struct YourData {
    1: optional string derived_field_that_can_be_null
    2
    : required string field_with_no_transformation_from_source 
  }
For any derived fields, let us avoid values such as '', "" etc. This doesn't make much sense since it is more safer
to say `"field is null"` in hive or `field == None` in scala. 
However, direct mapping of source fields into business views shouldn't involve change of identifier. This results
in weird debugging steps.

Example:
  In clickstream data set, If data type of field `evar` (in clickstream data set)  is "required" and is the same in 
  source thrift, this shouldn't be changed to optional in business views. 
  This is regardless of the possiblity of evar having empty string as one of its values from adobe.
  

Advantages:
 Going with optional fields for nullable fields instead of "" results in better space optimisation. 
 At the sametime, when you directly map a source field into your business view, the data type should never be changed since
 SME, or customers might be more comfortable with not changing the semantics/schema of source field.
 
 
#### No logical grouping with in a data asset, resulting in heavy dependency on an external documentation.

A proper logical grouping of data set with in a business view clearly makes life easier for an end user or a customer.
The major advantage is that your schema should explain what the data is all about. Logical grouping of fields with in a business
view is one way of doing it.

Logical grouping can be applied to various scenarios. For example, you can group tags/events that actually catagorises the data
as. Logical grouping is quite handy with struct.

An example is present in user_thajaf.clickstream where logical grouping is done.
  
  
Point 3:  
  
Events/Tags/Features with BinaryValues :- 

Point 4:

Partition structure is clearly based on the size of data set that you are dealing with. Operational metadata management
should never be drive the design of your partition. You have to consider two important factors while you partition the data

1. Size of the data:- Let us assume that minimum size of a partition should be 3gb. For a customer who runs query
against a partition with size 0.5GB and 3GB would never feel any significant difference in terms of performance.
The main reason behind this design consideration is to comply with the following rules:
`Lets don't have too many number of small files`
`Let us don't have too many number of partitioning resulting in Heap space exception`
`Let us don't have too many number of partitions making it difficult for updating the partition`

2. Do you really need a mutation? 
 Your business view job should be designed in such a way that we should avoid mutating the data. 
 For a customer this simply means, the query result from a partition should never change with time.
 
 select count(*) from sampletable where partition="SomeMutablePartition" === select count(*) from sampletable where partition="SomeMutablePartition"
  
 
 Bringing some amount of practicality into this aspect, there can be instances where you need to break these rules. But this
 should be clearly justified 
 Example of such an instance: Data size can't be estimated, and hence size is not a factor in designing partition. Also,
 if the partition is not hourly, it would result in mutation.
 
 If you try to comply with the above set of rules, you would merely end up with either monthly or daily partitions for most
 of the data sets.
 
Point 5:
Improper staging data sets. Why? For Performance? For speed? 
What are the impacts?
Customer Impact? Yes
Developer Impact? Yes
Operational instability > Yes
Error Propagation > Yes

Come on!! We are playing with big data, and for mere performance (say from 7min to 4 min) if you are persisting some
staging data then it doesn't make any sense. However if there is a significant performance impact, you are ready to go
with a staging layer data set. You can assume it to be  pre-processed data set, or a finite set of result that you could
use for future dates etc.

This becomes completely unmanageable when you are dealing with time variant data set. For those who are confused with this terminology,
a time variant data means anything that changes its state or value with respect to time.

An example is the account balance for a person during the period T1 to T2 is different from T2 to current-day.
This means we have a set of active and inactive records, where active record corresponds to T2-current_day or
presumably T2-future_date, and inactive data corresponds to T1 to T2.

Now let us assume that you are parsing a data (eg: SAP BP data set) and you are interested only in the active record
of all customers. For example, business is not interested in a customer's previous balance but only his active balance.
It is evident that this information is present in the source data set as different partitions. In this case, let us assume
the data is daily partitioned.

Customer1 | 500 | 2014-01-01 | 2016-05-06 | day1-partition
Customer2 | 500 | 2014-05-06 | 9999-12-12 | day1-partition
Customer3 | 500 | 2014-06-01 | 9999-12-12 | day1-partition

Assume that incoming data set is delta.





 




