# Customer API
A sample design for a RESTful API using RAML that contains a single resource, customers, and allows the following:
* List customers
* Create a new customer
* Update a customer
* Delete a customer

The API supports the following consumer use cases:
1. A consumer may periodically (every 5 minutes) consume the API to enable it (the consumer) to maintain a copy of the provider API's customers (the API represents the system of record)
2. A mobile application used by customer service representatives that uses the API to retrieve and update the customers details
3. Simple extension of the API to support future resources such as orders and products 

## Design Decisions
The following section details the different design decisions made while designing the API.

### Data Replication
The API allows consumers to maintain a local copy of the customer list. Since the number of customers can be quite large, the idea of pulling the full list of customers regularly is ruled out. An alternate approach is to provide a way for the consumer to retrieve incremental changes made to the customer list.

#### Selected Solution
The selected approach to providing a data replication capability aims to make this
a natural extension of the API rather than a separate API with its own resources.
By allowing the consumer to filter customers using the "since" attribute to
retrieve only those customers created or updated since the last sync simplifies
the replication process.

Initially, the customer will need to retrieve an initial snapshot of all the
customers. This process is optimised by allowing the consumer to expand the
"addresses" child collection so that it doesn't need to perform additional network
trips to retrieve them.

To cater for the fact that the server and consumer clocks could be out of sync
the "Last-Modified" header is provided so that it can be used by the consumer
as the value of the "since" parameter. As explained in the documentation (see
customer.doc.raml), the consumer needs to use the value of the "Last-Modified"
header returned with the first page as changes to the data could be happening
while the initial load is in-progress.

The maximum allowed pageSize parameter is set to 1000 due to the small size of
the customer resource. Ideally, the consumer would only use this maximum value
during the initial load process or if the consumer only pulls updates at a very
low frequency (e.g. once a day).

The consumer also has the ability to sort the resources using the "sortBy"
parameter. If not provided, a default field is used for sorting. Naturally, a
consumer should maintain the same value for the "sortBy" parameter while paginating
through the collection.

#### Other Candidate Solutions

* One approach is to provide an out-of-band initial load of the data to the consumer using, for example, a file, and then use the API to retrieve changes made since a certain timestamp. A downside of this approach is that it doesn't scale to a large number of consumers since preparing the initial load is a manual step that is customised for every consumer. However, this step can also be automated using a separate tool. Another challenge is that taking a static snapshot of a constantly-changing set of data can be tricky. There has to be a cutoff time after which all changes happening to the data should not be included in the snapshot. The consumer would then have to retrieve changes that occurred since that time.

* Another approach is maintaining changes to the customer data on the server as a series of events (CustomerCreated and CustomerUpdated). For more complex data structures, more fine-grained events can be used, but these two simple events will suffice for this example. Note that if the system was not designed in this manner from the start, creating a set of initial CustomerCreated events from existing data should still be straightforward.

  Expose an `events` API that would be used by the consumer to pull changes to the data every few minutes. A consumer does not need to pull an initial snapshot of the data separately since it can page through the changes in reverse-chronological order until it has consumed all the events. This approach is similar to the one used in the Twitter timeline API.

 Every event would have an id that increases chronologically. A consumer uses the following parameters when requesting change events:
    * `count` - the number of events to return. When requesting the first batch, the consumer would only use this parameter.
    * `max_id` - the maximum id of the events to return. The consumer would use that to retrieve the batch prior to the last one they retrieved. This is how a consumer would iterate through the batches until they have retrieved the full customer list.
    * `since_id` - the consumer would use this parameter to request the latest changes that occurred since the last event they received. The consumer would use that when making the call every 5 minutes.

  A consumer using the API for the first time, would perform the following steps:
    * Make an initial call to the events API to retrieve the latest set of events.
    * Continue to iterate through the event list using the max_id parameter until all events have been consumed.
    * At that point, the consumer has retrieved the full list of customers as of a certain point in time.
    * From that point onwards, the consumer would call the API periodically using the since_id parameter to retrieve the latest change events.

  Although this approach is quite robust, it's more complex and requires the consumer to learn a new (events) resource separate from customer. It could be more suitable for resources that change at a very fast pace.

### Search
In order for CS representatives to update customer details, they need to be able to search for customers. Typically a customer would provide their first and last names and/or a mobile number. The API would then return matching customers up to a maximum number of records to avoid loading the server or overwhelming the client.

#### GET vs POST
A decision to be made here is which HTTP verb to use for search. Ideally, searches should use GET since they do not cause any state changes on the server. However, GET has a limit on the amount of information that can be sent in the URL, so advanced searches with extensive search criteria might exceed that limit. In such cases, a POST could be used with the criteria provided in the body and the server would respond with the URL that the consumer can use to retrieve the search results using GET. A downside of this approach is that server has to manage the state of the search results and decide how long the URL will be available.

Considering our use case and the fact that there's a limit to the number of parameters that can be used for search, a single query string parameter with a GET request should suffice. The query string will accept a set of search terms. This approach has the advantage that it is expandable to allow for more operators (other than equal) in the future. For example, if dates are used for search the "< - less than" or "> - greater than" operators could be supported without making changes to the interface or breaking existing consumers.

#### Separate Resource?
Another design decision is whether to provide a different resource for search. This is mostly done when the search requirements are complex. In our case, since our search requirements are pretty straightforward, using the base `/customers` resource with query parameters should be enough.

#### Reducing Response Size
To reduce the load on the network, especially for mobile devices, the search API will provide the following capabilities:
* Pagination: Using the `pageSize` parameter, the client can determine how many records to return per page and the `pageIndex` parameter to iterate through the pages. Iteration requires the results to be deterministically sorted, so a `sortBy` parameter is provided.
* Subset of data: Although in our case, the customer object is quite small, in a real-life scenario, it's likely that the customer will contain many attributes. In the case of search, the user does not usually need to see all the attributes to determine whether the search result is what they're looking for. Hence, using the `attributes` parameter, the consumer can determine which attributes of the customer object needs to be returned. If the consumer needs to retrieve the full
representation of the customer, it can perform a separate GET request using the "self" link.

### Update - Updating Subset of Data
Again, in a real-life scenario, the customer object can be quite large. A CS representative may need to update only a subset of the customer's attributes. If we perform the updates using the `PUT` verb, the consumer must return the full object, which is not very efficient in terms of network resources.

#### Selected Solution
Fitting with our simple requirements is a simple CRUD solution. The solution will allow clients to update particular attributes using the `PATCH` HTTP verb so that the whole `customer` resource does not need to be returned.

#### Candidate Solutions
Despite being very common, using `PUT` to update resources is very CRUD-like and can cause the business logic and rules to creep into the client code. An alternate approach usually referred to as "REST without PUT" requires modeling the changes to the resources as independent Process Resources. So, instead of `PUT`ing the `customer` resource to change the customer's name, a `POST` to a `customer-change-of-name` resource would initiate the change of name process. The same can be done for other kinds of updates.
This solution provides many benefits including separating the update process from the customer resource, effectively allowing for long-running back office processes to be used before the update is actually done to the customer, e.g. an approval may be required. It also allows for eventual consistency since creating the process resource does not necessarily mean that the change would be reflected on the customer resource being returned from the next `GET` request. However, since in our case, it's actually the CS representative who's doing the update, no approval is likely to be required. Also, the modest requirements of our API does not warrant such a complex solution.

### Extending the API
Care has been taken to ensure that the API can be extended to cover additional resources such as Orders and Products without a lot of additional work.

#### Resource Types
Resource types and traits have been used extensively to capture design
patterns that can be applied to any new resources. This does not only reduce the effort required to add new resources, but also ensures that the API is consistent and consumers familiar with one resource can easily start consuming another.

Behavior that is expected to apply to all collections and entities was captured in resource types, while also allowing for certain methods to be optional (e.g. some resources may not allow deletion or patching).

Traits were used to capture optional behavior that can be selectively applied to collection and entity methods that need it.

#### Links
All resources, whether collections or entities support links. These links can be used in the future to link new resources to existing ones.
For example, an "orders" link could be added to the customer resource that could be used to retrieve orders for that customer. Although orders would not be a sub-resource of customer as they would deserve their own separate resource URL, the "orders" link would have the value of the orders resource with a filter on the customer id.
