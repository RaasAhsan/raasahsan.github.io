---
title: "Effective Fallbacks"
date: 2020-07-05T22:51:38-05:00
draft: false
---

Faults in distributed systems are inherently inevitable, and so as system designers, we make an effort to build resilience mechanisms into our systems to protect users of our services from observing those faults. This idea is captured by the notion of *fault tolerance*. A system is fault tolerant with respect to a domain of faults if it continues to operate normally even in the presence of those faults. 

Fault tolerance can often be costly, difficult or impossible to implement, so many systems opt to achieve a weaker form of fault tolerance known as *graceful degradation*. A system exhibits graceful degradation with respect to a domain of faults if it continues to operate at a potentially degraded level of service in the presence of those faults.

Fallbacks are a type of resilience pattern that achieve graceful degradation for a system. Generally, a fallback is employed in conjunction with some unreliable or fallible component in a system. If that component fails to produce a response, then the system invokes the fallback, which produces a response that is typically imprecise or stale, but cheaper or easier to compute (the properties of the fallback vary with the nature of the system). Crucially, it enables the system to proceed. 

## Searching for books
As a concrete example, let's consider a system architecture for an e-commerce website that sells books. Users can enter keywords in a search bar to find books they are interested in reading. The main web service calls the search service is for every search query. The search service incorporates submitted keywords as well as user preferences, user search history and user purchase history to find books that are appropriate to show the user.

What happens if the search service experiences an outage that lasts for 10 minutes? Users wouldn't be able to find books they want to buy and read. To avoid the risk of losing thousands of sales, we design a fallback for the search service. The goal of the fallback is to make a best effort to return relevant books that may not be faithful to the search query and relevant user data.

The fallback calls for a batch job: we determine the top N most popular keywords, pre-calculate a set of M relevant books for each one, and save the result set in a cache. The job runs on a regular basis as trends change over time and new books are released daily. If the web service fails to get a response from the search service, it falls back to the cached result sets.

The following Java code illustrates the behavior of the web service:
```java
public class SearchController {
    public Listing search(String query, UserId userId) {
        try {
            Books books = searchService.getBooksForSearch(query, userId);
            // images? ratings? testimonials? another service that is necessary to hit?
            // what if search returns book IDs and we have to query a book database for metadata
            return new Listing(books);
        } catch (Exception e) {
            Books fallbackBooks = cachedBooksDatabase.getBooksForSearch(query);
            return new Listing(fallbackBooks);
        }
    }
}
```

## The trouble with fallbacks
Despite their conceptual simplicity, fallbacks tend to be highly controversial in the practice of distributed systems design<sup>1</sup>. Some common criticisms on fallbacks are stated below:
* Fallbacks are difficult or impossible to test.
* If the fallback itself fails, it provides no value.
* Systems with fallbacks are harder to reason about.
* Many fallbacks can create an unpredictable system.

FIX: Instead, just use circuit breakers and retries etc
Is a wrong answer better than no answer?
how fallbacks interact with other systems 

I tend to agree with these criticisms. The third point may seem superficial, but it is a completely legitimate concern. For example, a system with fallbacks for 4 independent components admits 16 possible execution paths. The fallback for one system can interact with another system in non-trivial ways. I've also encountered several incidents where a fallback was the root cause of an outage because it wasn't working properly<sup>2</sup>.

That being said, it is hard to deny that there are scenarios where fallbacks are appropriate. The goal then should be to design fallbacks that are easy to understand, maintain and debug.

In the following sections, we will discuss concerns around designing effective and reliable fallbacks. This is by no means a definitive guide as it only encompasses learnings from past experiences.

## Justification
* Consider the criticality of the component
* What domain of faults do we want to protect against?
* Is the component inherently unreliable?
* Does the component experience intermittent failures or prolonged outages

By building a fallback, you are adding a lot of complexity to your system. More code, cyclomatic complexity. branch coverage, mentally keep track of fallback

## Regular testing
Testing is arguably the most crucial, the most neglected, and the most difficult part in maintaining fallbacks. By nature, failures in most systems happen infrequently, so a fallback may never be invoked. If we do not regularly verify that a fallback works, we have no way of guaranteeing that it continues to work as the system evolves. 

To continue our running example, imagine the business asks us to build a feature that forces us to change the book data model in such a way that it breaks compatibility. We forget that the fallback cache stores data with the old model because that we rarely touch that infrastructure. Eventually, the system experiences an outage but because the cache entries aren't compatible with the new data model, the service fails to load them, so users can't search for books.

How can we be confident our fallback works in production when it is needed? Testing the fallback in production is the most fool-proof way to do it, but it may be difficult to force the system to exercise that behavior.

## Scope and surface area
Minimize the number of fallbacks
Fallback for a system rather than for a component
at the edge of your system
number of distinct execution paths through a system - cyclomatic complexity

There are some cases where it may make sense to introduce fallbacks for individual components. In the book store example, imagine we build a user rating feature. Ratings for each book should be rendered in the search listing, so the web service queries a user rating service to fetch the ratings. If the user rating service experiences a prolonged outage, should we rely on our primary fallback or should we return the listing without any ratings?

I think this can be a highly debatable topic, but ultimately it feels like it depends on the business, the domain and the nature of the system. My interpretation of the problem is described below:
1. The former approach minimizes complexity in the system and encourages us to constantly improve the reliability of each component. If any given component fails, we return the cached listing even though we may have enough data to produce a more optimal one. Perhaps the fallback cache generation job can incorporate ratings into the results as well.
2. The latter approach optimizes for the best user experience in the presence of faults. However, we pay in the form of the complexity that an additional fallback brings.

## Visibility

## Compositional resiliency
Fallbacks tend to pair well with other resilience mechanisms like timeouts, retries and circuit breakers. Imagine that we observe that a handful of requests to the book service occasionally time out because of network congestion issues. 

cascading failures

## Simplicity
Avoid replicating behavior.
Focus more on building reliable systems.
Reserve fallbacks for the most disastrous failures.
Build reliable fallbacks, but try not to rely on them
fail fast



* Failover, design other fault tolerant systems like queueing emails in a database
* Focus on building more reliable services
* Avoid fallbacks if possible
* Cover as much surface area with a fallback as possible, rather than for individual components
* Use fallbacks in conjunction with other resilience mechanisms
* Keep fallbacks as simple as possible
* Test fallbacks regularly
* Logging and monitoring fallbacks
* Justify fallbacks. Try to use them for inherently unreliable services, and clearly define their scope
* Anecdote: fallback has caused failures in our systems
* Media rights fallback
* Media services/activation services

## Footnotes
- 1: 
- 2: It's important to acknowledge whether there were any instances where the fallback *did* prevent an outage in the past before discounting it.
