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
