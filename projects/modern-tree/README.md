# Modern Tree

Started on: Friday, September 19, 2025

This is the temporary code name for a new project, for building a new Java-based service that will be able to generate `SearchableIndividual` / `AggregateIndividual` objects from trees by directly reading from the site database tables, built upon the logic that we already have in the batch `treeprocessor`.

This idea came up recently when I worked on upgrading dependencies and tests for treeprocessor, and realized that we might be able to invoke its logic in different ways than how it currently consumes the tables using batch.

I initially thought the main benefit is not having two dual implementations of the same code, one in PHP, for random access, and the other in Java, for batch. The dual implementations are causing consistency issues and increased maintenance efforts.

But now I realize there might be many other benefits related to performance and cost saving. Read access to tree individuals is probably the single most heavy access pattern that we have on the entire website, it is needed everywhere. With the exception of batch matching, where we were wise enough to re-implement this in a more efficient way a decade ago, pretty much everything else still has to run through the PHP Monolith, and is using code that was written 15-10 years ago (Dori's viewers, or even older layers), and probably using inefficient query patterns. This code is a mess, very convoluted, people are afraid to touch it - it really is no man's land, despite being some of the most frequently called pieces of code that we have.

Sure, having access to just `AggregateIndividual` objects does not cover all tree access patterns, but I'm willing to bet it will cover the vast majority of them. In an essence, it's simply about providing a denormalized view to tree profiles, it makes sense that the majority of places that need data about individuals will need this denormalized view that already includes inlined relatives, events and such.

There was an attempt to build `PeopleStore`, that really is just storing a cache of a format very close to `AggregateIndividual`, but it is a very leaky abstraction - we don't fully trust the data that is available there, it's not always correct and not always available, I think it's being populated by some PHP dameon asynchrounsly after tree updates? need to look into it. I assume that to read from it, one has to go through PHP code that knows how to fallback to reading from the site database? So it doesn't allow other services to read directly from `PeopleStore` and skipping PHP entirely. If that's the case, what do we even gain from putting `PeopleStore` as a Java service between PHP and Cassandra? Why can't we just call it a cache and have PHP read directly from Cassandra? BTW, Cassandra itself is probably a poor choice for this form of storage, why do we need to store each bit as a column when we always read the entire blob? TODO: learn more about `PeopleStore`, verify that what I wrote above is true.

As a next step, I want to try and quantify how much individual reads we have in PHP that we might be able to replace with calls to a new service, how much of the total site database traffic they are, how expensive is the site DB, how much `PeopleStore` traffic we have, etc.

I also want to do a PoC in turning `treeprocessor` into a service that can be called ad-hoc while reading its data directly from a site db.

I need to try and get the dev-env to work, hopefully it contains enough to allow me to see how PHP builds SIs and works with `PeopleStore`, and since it has a local site DB perhaps I can get `treeprocessor` and the new service to run there as well? It can make development much easier. 

More ideas:
1. Use same new service to bulk-update SIs following tree updates, either for specific individuals or even for an entire trees, this can be far more efficient then how PHP is doing it today. We can use it to populate the Solr indexer feed, add delta updates for batch SIs that are consumed by matching, etc.

2. Add a more efficient way of calculating relationship paths to the same service. We recently discussed the benefits of having a pre-calculated table that has the relationship path between the root individual in a tree and each individual in the tree, this service will be able to update it.

