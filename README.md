# Customer API
A sample design for a RESTful API using RAML that contains a single resource, customers, and allows the following:
* List customers
* Create a new customer
* Update a customer
* Deletes a customer

The API supports the following consumer use cases:
1. A consumer may periodically (every 5 minutes) consume the API to enable it (the consumer) to maintain a copy of the provider API's customers (the API represents the system of record)
2. A mobile application used by customer service representatives that uses the API to retrieve and update the customers details
3. Simple extension of the API to support future resources such as orders and products 

## Design Decisions
The following section details the design decisions made while designing the API.
### Data Replication
One of the API use cases allows consumers to maintain a local copy of the customer list. Since the number of customers can be quite large, the idea of pulling the full list of customers regularly is ruled out. An alternate approach is to provide a way for the consumer to retrieve incremental changes made to the customer list.
#### Candidate Solutions
* One approach is to provide an out-of-band initial load of the data to the consumer using, for example, a file, and then use the API to retrieve changes made since a certain timestamp. A downside of this approach is that it doesn't scale to a large number of consumers since preparing the initial load is a manual step that is customised for every consumer. However, this step can also be automated using a separate tool. Another challenge is that taking a static snapshot of a constantly-changing set of data can be tricky. There has to be a cutoff time after which all changes happening to the data should not be included in the snapshot. The consumer would then have to retrieve changes that occurred since that time.
* Another possible solution to implement the initial load using the API itself. The consumer can request a snapshot of the data as of a particular timestamp. The data would then be paginated to avoid returning a very large response. This approach suffers from a similar drawback to the previous one, which is that a snapshot of the data has to be taken and persisted on the server and made available for that particular consumer. This, again, does not scale with a large number of consumers.
#### Selected Solution
The selected solution, although a bit more complex, is the most robust and puts the least load on the server. It involves:
* Maintaining changes to the customer data on the server as a series of events (CustomerCreated and CustomerUpdated). For more complex data structures, more fine-grained events can be used, but these two simple events will suffice for this example. Note that if the system was not designed in this manner from the start, creating a set of initial CustomerCreated events from existing data should still be straightforward.
* Expose an `events` API that would be used by the consumer to pull changes to the data every few minutes.
* A consumer does not need to pull an initial snapshot of the data separately since it can page through the changes in reverse-chronological order until it has consumed all the events. This approach is similar to the one used in the Twitter timeline API.
* Every event would have an id that increases chronologically. A consumer uses the following parameters when requesting change events:
    * `count` - the number of events to return. When requesting the first batch, the consumer would only use this parameter.
    * `max_id` - the maximum id of the events to return. The consumer would use that to retrieve the batch prior to the last one they retrieved. This is how a consumer would iterate through the batches until they have retrieved the full customer list.
    * `since_id` - the consumer would use this parameter to request the latest changes that occurred since the last event they received. The consumer would use that when making the call every 5 minutes.
* To recap, a consumer using the API for the first time, would perform the following steps:
    * Make an initial call to the events API to retrieve the latest set of events.
    * Continue to iterate through the event list using the max_id parameter until all events have been consumed.
    * At that point, the consumer has retrieved the full list of customers as of a certain point in time.
    * From that point onwards, the consumer would call the API periodically using the since_id parameter to retrieve the latest change events.
##### Related Decisions
* During the initial load, and regardless of the direction in which the consumer iterates through the list, as long as the server is not maintaining any information about the consumer, it's inevitable that the consumer would retrieve duplicate events. For example, if a customer is updated more than once, the consumer only needs to retrieve the latest state of the customer once, not for every time it changed. To reduce this overhead, the events API will not return the actual data. Rather it will only return the event type (crate or update), the customer id and the version of the customer after the event. Since the consumer will iterate through the list in reverse-chronological order, in the case of a duplicate event, the version of the customer internally stored by the consumer will be higher than the version in the event. In this case, the consumer will ignore that event. If the consumer does not have the customer, then it will request the latest version of the customer separately.
* In order to optimise the retrieval of multiple customers and to reduce network round trips, we will allow the retrieval of multiple customers in a single call using the `ids` query parameter that will have a comma-separated list of ids.
### Search - GET vs POST
In order for CS representatives to update customer details, they need to be able to search for customers. Typically a customer would provide their first and last names and either a date of birth or a mobile number. The API would then return matching customers up to a maximum number of records to avoid loading the server or overwhelming the client. A decision to be made here is which HTTP verb to use for search. Ideally, searches should use GET since they do not cause any state changes on the server. However, GET has a limit on the amount of information that can be sent in the URL, so advanced searches with extensive search criteria might exceed that limit. In such cases, a POST could be used with the criteria provided in the body and the server would respond with the URL that the consumer can use to retrieve the search results using GET. A downside of this approach is that the search request cannot be bookmarked or cached. Also, the server has to manage the state of the search results and decide how long the URL will be available.
Considering our use case and the fact that there's a limit to the number of parameters that can be used for search, using query string parameters with a GET request should suffice. The API would also allow using a `sort` parameter to sort the results.
### Search - Separate API
Another design decision is whether to provide a different API for search. This is mostly done when the search requirements are complex. In our case, since our search requirements are pretty straightforward, using the base `/customers` resource with query parameters should be enough.
This means, of course, that the same URL could not be used for simply listing customers, which also fits with our overall approach.
### Search - Reducing Response Size
To reduce the load on the network, especially for mobile devices, the search API will provide the following capabilities:
* Pagination: Using the `count` parameter, the client can determine how many records to return per page and the `page` parameter to iterate through the pages. Iteration requires the results to be deterministically sorted, so a `sort` parameter will also be provided.
* Subset of data: Although in our case, the customer object is quite small, in a real-life scenario, it's likely that the customer will contain many attributes. In the case of search, the user does not usually need to see all the attributes to determine whether the search result is what they're looking for. Hence, using the `attributes` parameter, the consumer can determine which attributes of the customer object needs to be returned.
### Update - Updating Subset of Data
Again, in a real-life scenario, the customer object can be quite large. A CS representative may need to update only a subset of the customer's attributes. If we perform the updates using the `PUT` verb, the consumer must return the full object, which is not very efficient in terms of network resources.
#### Candidate Solutions
Despite being very common, using `PUT` to update resources is very CRUD-like and can cause the business logic and rules to creep into the client code. An alternate approach usually referred to as "REST without PUT" requires modeling the changes to the resources as independent Process Resources. So, instead of `PUT`ing the `customer` resource to change the customer's name, a `POST` to a `customer-change-of-name` resource would initiate the change of name process. The same can be done for other kinds of updates.
This solution provides many benefits including separating the update process from the customer resource, effectively allowing for long-running back office processes to be used before the update is actually done to the customer, e.g. an approval may be required. It also allows for eventual consistency since creating the process resource does not necessarily mean that the change would be reflected on the customer resource being returned from the next `GET` request. However, since in our case, it's actually the CS representative who's doing the update, no approval is likely to be required. Also, the modest requirements of our API does not warrant such a complex solution.
#### Selected Solution
Fitting with our simple requirements is a simple CRUD solution. The solution will allow clients to update particular attributes using the `PATCH` HTTP verb so that the whole `customer` resource does not need to be returned.
