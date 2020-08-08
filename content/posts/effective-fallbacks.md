---
title: "Effective Fallbacks"
date: 2020-07-05T22:51:38-05:00
draft: false
---

Faults in distributed systems are inherently inevitable, and so as system designers, we make an effort to build resilience mechanisms into our systems to protect users of our services from observing those faults. This idea is captured by the notion of *fault tolerance*. A system is fault tolerant with respect to a domain of faults if it continues to operate normally even in the presence of those faults. 

Fault tolerance can often be costly, difficult or impossible to implement, so many systems opt to achieve a weaker form of fault tolerance known as *graceful degradation*. A system exhibits graceful degradation with respect to a domain of faults if it continues to operate at a potentially degraded level of service in the presence of those faults.

Fallbacks are a type of resilience pattern that achieve graceful degradation for a system. Generally, a fallback is employed in conjunction with some unreliable or fallible component in a system. If that component fails to produce a response, then the system invokes the fallback, which produces a response that is typically imprecise or stale, but cheaper or easier to compute (the properties of the fallback vary with the nature of the system). Crucially, it enables the system to proceed. 

## Searching for books
As a concrete example, let's consider a fallback for an e-commerce store that sells books. Users can enter keywords in a search bar to find books they are interested in reading. The main web service calls the search service is for every search query. The search service incorporates submitted keywords as well as user preferences, user search history and user purchase history to find books that are appropriate to show the user.

What happens if the search service experiences an outage that lasts for an hour? Users wouldn't be able to find books they want to buy and read, and our service loses thousands of sales. To avoid a scenario like this, we design a fallback for the search service. The goal of the fallback is to make a best effort to return relevant books that may not be faithful to the search query and may not incorporate user data.

The fallback calls for a batch job: we determine the 10000 most popular keywords, pre-calculate a set of 100 relevant books for each one, and save the result set for each keyword in a database. The job runs on a regular basis as trends change over time and new books are released daily. If the web service fails to get a response from the search service, it falls back to the cached result sets.

The following Java code demonstrates the behavior of the web service:
```java
public class SearchController {
    public Listing search(String query, UserId userId) {
        try {
            Books books = searchService.getBooksForSearch(query, userId);
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

FIX: Instead, just use circuit breakers and retries etc
Is a wrong answer better than no answer?
how fallbacks interact with other systems 

I tend to agree with these criticisms. The third point may seem superficial, but it's a completely legitimate concern. For example, a system with fallbacks for 4 independent components admits 16 possible execution paths. The fallback for one system can interact with another system in non-trivial ways. I've also encountered several incidents where a fallback itself was the root cause of an outage. If those fallbacks didn't exist to begin with, the incidents wouldn't have happened or at the very least wouldn't have been as severe<sup>2</sup>.

That being said, there are many scenarios where fallbacks *are* appropriate. The goal then should be to design fallbacks that are easy to understand, easy to maintain and easy to debug.

In the following sections, we will discuss guidelines for designing effective and reliable fallbacks.

## Justify the fallback
scope the domain of faults we want to protect against

## Maximize surface area
Minimize the number of fallbacks
Fallback for a system rather than for a component
at the edge of your system

## Regularly test the fallback


## Instrument visibility into fallbacks


## Keep fallbacks simple


## Pair fallbacks with other resilience mechanisms





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
- 2: It's important to acknowledge whether there were any instances where the fallback *did* prevent an outage before discounting it.
