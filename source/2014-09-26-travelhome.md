Meet TravelHome! A site for people to rent out lodging (think airbnb, couchsurfing).  In this
post we'll walk you through some of our core functionality and highlight some
particularly interesting or difficult problems we encountered along the way.

With TravelHome, users can function both as hosts and travelers.  This means that
there are no special permissions required for users who wish to create listings for
others to book, all users who register for the site have the option to create and manage
listings.

An example interaction involving both a traveler and a host may look something like this:

- A traveler comes to the site and browses available listings
- They request to book a listing, at which point the host is notified by email
- The host has the option to confirm or deny this booking request
- Should the request be accepted, confirmation is sent to both host and traveler
and a a listing has been successfully booked!

To recap here are some of the core abilities of users:

Hosts can:

- Create and manage listings
- Confirm and deny booking requests
- See a history of successful visits

Travelers can:

- Browse all listings or listings for a specific host
- Filter by things like price and available dates
- Maintain multiple booking requests over a single order

### Legacy Code
As a class this was the first project we've worked on that was built on top of an existing
codebase.  Our last project was [Dinner Dash](http://tutorials.jumpstartlab.com/projects/dinner_dash.html),
an online ordering platform for a restaurant.  We were given the task of
[pivoting](http://tutorials.jumpstartlab.com/projects/the_pivot.html) Dinner Dash,
and TravelHome was born.  

It was critical for us to review the code thoroughly not only to familiarize ourselves,  
but also to determine which parts of the code we would be able to repurpose for our
application and which parts could be safely dropped.

Part of the challenge in using code written by other developers is not knowing which parts are actually
functional and if that code was being tested properly.  We were very fortunate in that we inherited a well-structured
and tested codebase.  As a result, we were able to quickly repurpose the existing implementation and tests.

### The actions of many users interacting with one object:
Something that makes TravelHome unique is the idea of requesting to book several listings
from multiple hosts in a single order.  From a technical standpoint this means that the ‘state’ of
an order is dependent on the actions of multiple users.  An order can only be ‘completed’ by the
traveler, but before that event can be triggered each host must confirm or deny their specific request.

In order to better control the flow of these actions they were pulled out (as much as possible) into their
own controllers. I thought of it a bit like a set of dominos; one event triggers another, which triggers
another until the appropriate sequence has taken place. The other piece of the puzzle that was fun to work
on was connecting the users via email at the right time. State specific notifications needed to be sent to both
travelers and hosts, and it was an interesting challenge to line those events up correctly.

### Multitenancy
One of the main goals for this project was [multitenancy](https://en.wikipedia.org/wiki/Multitenancy).
One of the best examples of a multitenancy application is none other than Amazon.  Multiple "stores"
can sell their products on Amazon's platform, and users can maintain a single order
across multiple sellers.  This presents a lot of interesting problems, one example being sub-orders (invoices):
Users aren't the only ones that need a record of their past orders, sellers do as well.  But,
sellers are understandably only interested in the items a user purchased from their store.  Additionally,
you need to be able to give certain users the ability to manage a store, and it's often
more than one (a store can have multiple "employee" users).

TravelHome was a bit different in that some of our most interesting problems weren't related
to multitenancy.  We only have one concept of a user (besides a site-wide platform admin), and
users can function both as hosts and travelers.  There aren't special priveleges involved in
managing a "store", because as it is now only one user (the host) will ever be in charge of a
particular listing.

### Yay For Dates!
For us, one of the trickiest pieces was dealing with dates.  The application we inherited was for a restaurant,
and associating menu items with an order was fairly straightforward.  In this implementation a
menu item was added to a particular order via a join table, and there was no extra baggage.

With TravelHome, dates, or availabilities more accurately, became an integral part of dealing
with users posting and booking listings.  Each listing needs an associated
range of dates that indicate when the property is available.  Further, we needed to keep
track of which available dates for each listing had been succesfully booked by a user, as
we could no longer display them as being available for other travelers.

We ended up implementing that through three data models: listings, availabilites, and listing
availabilities (the latter of which being a simple join table).  Modeling listing data was
straightforward, but the key to this particular solution was the availabilities table.  It had
two important fields: one for a single date, and the other was a foreign key representing an
entry in a table holding data about listings that were booked by a user.

This gave us the information we needed to implement a system that would only display truly
availble dates for a listing.  Currently, the presence of a non-nil value for that foreign key
indicates that that date is no longer available for a listing, and we can then update which dates
travelers can select for that listing.

Onto some code examples: here is a method we wrote to "unavailablefy" a date:

```ruby
def make_dates_unavailable(listing, order_listing, requested_dates)
  requested_dates.each do |date|
    availability = listing.availabilities.find_by(date: date)
    availability.order_listing_id = order_listing.id
    availability.save
  end
end
```

Here we take three arguments: a listing object, an order listing object, and a collection of requested
dates.  First we iterate over each date in the collection, and using
the availabilities association we store the corresponding availability object in a variable.
We then change that availability's order_listing_id to the id of the order listing passed
as an argument and save the entry.

Next we needed to create a scope to easily check if an availability was reserved or unresered:

```ruby
class Availability < ActiveRecord::Base
  # ...

  scope :unreserved, -> { where(order_item_id: nil) }
end
```

And there you have it!  Here are some links to find the team on github and twitter:

- Marc Garreau [github](https://github.com/MarcGarreau) | [twitter](https://twitter.com/MarcGarreau)
- Emily Davis [github](https://github.com/emilyadavis303) | [twitter](https://twitter.com/emilyadavis303)
- Gustavo Villagrana [github](https://github.com/GusVilla303) | [twitter](https://twitter.com/GusVilla303)
- Eric Fransen [github](https://github.com/ericfransen) | [twitter](https://twitter.com/eric_fransen)
- Will Faurot [github](https://github.com/wfro) | [twitter](https://twitter.com/will_faurot)
