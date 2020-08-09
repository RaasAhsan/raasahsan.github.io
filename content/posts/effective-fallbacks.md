---
title: "Effective Fallbacks"
date: 2020-08-08T20:00:00-05:00
draft: false
---

Faults in distributed systems are inherently inevitable, and so as system designers, we make an effort to build resilience mechanisms into our systems to protect users of our services from observing those faults. This idea is captured by the notion of *fault tolerance*. A system is fault tolerant with respect to a domain of faults if it continues to operate normally even in the presence of those faults. 

Fault tolerance can often be costly, difficult or impossible to implement, so many systems opt to achieve a weaker form of fault tolerance known as *graceful degradation*. A system exhibits graceful degradation with respect to a domain of faults if it continues to operate at a potentially degraded level of service in the presence of those faults.

Fallbacks are a type of resilience pattern that achieve graceful degradation for a system. Generally, a fallback is employed in conjunction with some unreliable or fallible component in a system. If that component fails to produce a response, then the system invokes the fallback, which produces a response that is typically imprecise or stale, but cheaper or easier to compute (the properties of the fallback vary with the nature of the system). Crucially, it enables the system to proceed. 

## Searching for books
As a concrete example, let's consider a system architecture for an e-commerce website that sells books. Users can enter keywords in a search bar to find books they are interested in reading. The main web service calls the search service for every search query. The search service incorporates submitted keywords as well as user preferences, user search history and user purchase history to find books that are appropriate to show the user.

Unfortunately, our search service relies on a third-party catalog who is unreliable and is known to experience prolonged outages. If an outage occurs, users cannot find books they are interested in purchasing. We would like to avoid the risk of losing thousands of sales during outages. We could build our own catalog but search technology is not our strong suit. Instead, we design a fallback for the search service,  whose goal is to make a best effort to return relevant books that may not be faithful to the search query and personalization data.

The fallback calls for a batch job: we determine the top N most popular keywords, pre-calculate a set of M relevant books for each one, and save the result set in a network cache. The job runs on a regular basis as trends change over time and new books are released daily. If the web service fails to get a response from the search service, it falls back to the cached result sets.

The following Java code illustrates the behavior of the web service:
```java
public class SearchController {
    public Listing search(String query, UserId userId) {
        List<Book> books;
        try {
            List<BookId> bookIds = searchService.getBooksForSearch(query, userId);
            books = bookService.getBooks(bookIds);
        } catch (Exception e) {
            books = cachedBooksDatabase.getBooksForSearch(query);
        }
        return new Listing(books);
    }
}
```

## The trouble with fallbacks
Despite their conceptual simplicity, fallbacks tend to be highly controversial in the practice of distributed systems design<sup>1</sup>. Some common criticisms on fallbacks are stated below:
1. Fallbacks are difficult or impossible to test. If they remain untested, it is impossible to know whether they continue to work correctly.
2. Fallbacks often fail themselves in non-trivial ways.
3. Systems with fallbacks become unpredictable and are harder to reason about.

I agree with these criticisms. I have encountered several incidents where a fallback was a root cause of an outage. In one incident, the fallback for a component was incompatible with a downstream service. In another incident, the fallback was called in parallel with the happy path, which ended up overloading a static resource. There was no evidence that either of these fallbacks provided any value, and the code behind them was rather complex. 

That being said, there are situations where fallbacks are appropriate. The goal then should be to design fallbacks that are easy to understand, maintain and debug.

## Discussion
In the following sections, we will discuss several concerns around designing effective and reliable fallbacks. We will frame these concerns around the book store example described above, but the details for another system may look completely different. This is by no means a definitive guide as it encompasses lessons learned from systems I have worked with, which are certainly biased in some ways.

### Justification
Before designing a fallback, we should justify its value. Some guiding questions are listed below:
1. Are there better ways to solve the problem? For example, timeouts, circuit breakers, retries, or failover.
2. How critical is the component and the service it provides? If it fails, what is the impact?
3. Is the component inherently unreliable? Does it frequently experience prolonged outages, or does it intermittently return failures?
4. What is the domain of faults we want to protect the system against?

If we ultimately decide to build a fallback, we should be cognizant of the benefits and the costs.

### Regular testing
Testing is arguably the most crucial, the most neglected, and the most difficult part in maintaining fallbacks. By nature, failures in most systems happen infrequently, so a fallback may never be invoked. If we do not regularly verify that a fallback works, we have no way of guaranteeing that it continues to work as the system evolves. 

To continue the book store example, imagine we are asked to build a feature that forces us to change the book data model in such a way that it breaks compatibility. We forget that the fallback cache stores data with the old model because we rarely touch that infrastructure. Eventually, the system experiences an outage but because the cache entries aren't compatible with the new data model, the service fails to load them, so users can't search for books.

How can we be confident our fallback works in production when it is needed? Testing the fallback in a live environment is the most fool-proof way to do it, but it may be unwieldy to force the system to exercise that behavior. The strategies listed below can be used in conjunction with automated testing:
1. Introduce testing users which force different kinds of behavior in the system. 
2. Support request-level configuration that can introduce special behavior on-demand.
3. Occasionally invoke the fallback in the happy path to ensure that it works. If it is appropriate, perhaps even return the fallback response for a percentage of traffic.

### Scope
In general, there are two locations at which a fallback can be positioned: in front of an individual component or in front of the entire system (or possibly a group of components). 

Consider the book store example, where the fallback covers the scope of the entire search controller. If we observe a failure in the book or search services, the fallback is invoked. Instead, what if we designed two separate fallbacks, one for the book service, and one for the search service?

The system-level fallback seems preferable for several reasons:
1. It covers a larger failure domain than component-level fallbacks. Imagine the book store web service performed extra processing before returning a listing, which could also fail.
2. It minimizes the cyclomatic complexity of the search function. The system-level fallback admits 3 distinct execution paths whereas the component-level fallbacks admit 5. Fallbacks can be fallible too!
3. Only one fallback needs to be tested rather than two.
4. From a domain modeling perspective, it makes sense to design a fallback for a high-level system rather than a low-level component. 

However, there are some cases where it may make sense to introduce fallbacks for individual components. Let's consider two scenarios in the book store example.
1. We design a rating feature where users can submit reviews for books they have read. Ratings for each book should be rendered in the search listing, so the web service additionally queries a rating service to return rating data. If that service is unavailable, should we rely on the primary fallback or should we return the listing without ratings?
2. We begin publishing search events for analytics purposes to a stream processor. If that service is unavailable, should we rely on the primary fallback or introduce a component-level fallback (suppress the failure and/or asynchronously retry)?

At a high level, the two approaches are (1) to rely on the system-level fallback or (2) to introduce a component-level fallback. The former approach minimizes complexity in the system and encourages us to constantly improve the reliability of each individual component. If a component fails, we return the cached listing even though we may have enough data to produce a more optimal one. On the other hand, the latter approach optimizes for the best user experience in the presence of faults. This marginal reliability comes at the cost of additional complexity that another fallback brings.

I think this can be a highly debatable topic, but ultimately it depends on the domain, the nature and the criticality of the system and its components.

### Compositional resiliency
Fallbacks tend to pair well with other resilience mechanisms. For example, a common pattern is to protect a network call with a timeout, circuit breaker, and retry. If the circuit breaker trips, it starts returning failures. We can introduce the fallback in front of that. The effect of this is that intermittent failures will be immediately retried, but during prolonged outages the system will resort to the fallback. 

### Visibility
Given the complexity that a fallback, like any other resilience mechanism, introduces to a system, it is important to instrument monitoring, logging and tracing to help understand the behavior of a system when a fallback is invoked.

## Conclusion
Fallbacks can be an effective technique for improving reliability, but they must be designed with diligence and caution. To wrap up the post, here is some advice I encourage others to think about when building fallbacks:
1. Design simple fallbacks that can produce a useful result if all else fails.
2. Regularly exercise fallback behavior in an evolving system.
3. Build reliable fallbacks, but avoid relying on them.
4. Minimize the number of fallbacks in a system to minimize its complexity.

## Reference
- 1: [Avoiding Fallback in Distributed Systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/)
