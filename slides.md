# Recommendations Engines with Redis and Ruby

!SLIDE middle
# Evan Light
## [@elight](http://twitter.com/elight)
## [evan.light@rackspace.com](mailto:evan.light@rackspace.com)

!SLIDE
# Recommendation Engines
#### &nbsp;
# with Redis and Ruby

!SLIDE middle
}}} images/ruby_dcamp.png

!SLIDE 
}}} images/rackspace.jpg
!NOTE
# Developer advocate
# Straw poll
# Dev trial

!SLIDE
# Agenda
## Context
## Solution
## Redis-related tangents
## Lessons learned

!SLIDE top-left
# Context
## &nbsp;
## Soccer social network
## Display popular & relevant Posts
## Near real-time

!SLIDE top-left
# What the client asked for
## A recommendation engine!

!SLIDE
# Suggests possible choices for the user
##### &nbsp;
# based on previous chocies by the user.

!SLIDE
# Approximation
!NOTE
# Inaccurate
# Based on assumptions
# Canonically rooted in statstical methods except...

!SLIDE top-left
# Statistics and I
}}} images/i_have_no_idea_what_im_doing.jpg
!NOTE
# This is new
# I'm scared
# Therefore this will be exciting!

!SLIDE
# Why not statistical methods?
!NOTE
# Because I like being paid
# I failed to convince the client to pay for a few research days
# Oops

!SLIDE
# Why Ruby?
!NOTE
# The client asked for it

!SLIDE
# The nouns
## User
## Post
## Comment
## **Team**
## **Player**

!SLIDE
# The verbs

!SLIDE
# Submitting a Post
## Polymorphic tags
### Teams
### Players

!SLIDE
# Commenting on a Post 
## Tag (by citation) other Users

!SLIDE
# Other verbs
## Favoriting a Team or Player
## Liking a Post

!SLIDE top-left
# Given this...
}}} images/domain.png
!NOTE
# A "Post" can be any arbitrary unit of content.  This system uses Facebook-like Posts.

!SLIDE top-left
# ... and maybe this...
}}} images/user-tag.png

!SLIDE top-left
# ... we want *this*!
}}} images/scored-user-posts.png
!NOTE
# Subjective (per User) recommendations

!SLIDE top-left
# That "maybe"
}}} images/user-tag.png


!SLIDE
# We start with ActiveRecord...
#### &nbsp;
## <img src="images/troll.jpg"/>

!SLIDE
# Ok, ActiveRecord::Observer
!NOTE
# This IS for a Rails app...
# Knowledge (our nouns + verbs) can be captured as AR callbacks

!SLIDE
# Callbacks
## Post / Comment / Favorite / Like 
### create/destroy

!SLIDE
# We want two kinds of Posts
## Popular 
## Relevant
!NOTE
# "Popular" is fairly objective and shared across all users
# "Relevant" is subjective per user

!SLIDE
# Popularity
## Comments
## Likes
!NOTE
# Scalar value per Post

!SLIDE
# Measuring User interest
## **Favoriting** DC United
## **Submitting** a Post tagged with DC United
## **Liking** ...
## **Commenting** on ...
## Being **mentioned** in a Comment on ...
!NOTE
# It's all about the tags!!!!

!SLIDE top-left
# Ok, so I used to follow DC United...
}}} images/dc_united.jpg

!SLIDE
# Relevance (cont'd)
## Given any individual expression of interest
## For each associated Tag
## For each User interested in Tag
## (Re)score the User's interest in Tag
!NOTE
# Lacking a machine learning background...

!SLIDE bottom-left
# <span style="font-size: 1.5em">O(n<sup>2</sup>)</span>
}}} images/im_so_sorry.png

!SLIDE
}}} images/kobayashi-maru.jpg
!NOTE
# 3 day Post window "cheat" :D

!SLIDE
# [Resque](http://github.com/resque/resque)

!SLIDE
# [Redis](http://redis.io) in 120 seconds or Less
#### (Don't time me)
## Key/value store
## In memory
## Few but adequate persistent options

!SLIDE
# Redis (cont'd)
## "value" can itself be a data structure:
### List
### Hash
### Set
### Sorted set

!SLIDE
# Redis (cont'd)
## Time to Live (TTLs)

!SLIDE
# AR::Observer push events to Resque

!SLIDE
# Example event
# &nbsp;
```javascript
{
  event_type: "post",
  action:     "create",
  id:         post_id_here,
  user_id:    poster_id_here,
  tags:       ["Player1", "Team1"]
}
```

!SLIDE top-left
}}} images/share\ the\ load.jpg
!NOTE


!SLIDE
}}} images/calculator_workflow.png

!SLIDE
# *Calculator*
## Resque worker
## Strategy
## Handles all persistence
!NOTE
# Refactored Calculator early into three classes

!SLIDE
# Persistence
## Hide Redis behind abstraction
## Duck-typeable

!SLIDE
# Trendingness Calculator
## (Re)compute individual Post's popularity
### Input
#### Event
### Output
#### New score for Post
!NOTE
# Same interface as a Resque worker
# Different behavior per event type

!SLIDE
# Post trendingness in Redis
## Stored as key-value pairs
## 3 day TTL when set

!SLIDE
# *User Interest Calculator*
## (Re)compute User's interest in Tags
### Input
#### Event
### Output
#### Updated User interest per Tag
!NOTE
# Same interface as a Resque worker
# Different behavior per event type

!SLIDE
# User interest in Redis
## Hash per User
## Key: User ID
## Field names: Tags
## Values: scalar representation of User's interest by Tag
## Never expire by intent

!SLIDE bottom-left
# Post Score Calculator
}}} images/recompute_all_the_users.jpg

!SLIDE
# *Post Score Calculator*
## Recompute User interests for all effected users
## Input
### Event 
### User's interests per Tag from User Interest Calculator
## O(n<sup>2</sup>) 

!SLIDE
# But [inverted indices](http://en.wikipedia.org/wiki/Inverted_index) helped
## Index content to location of content
## Redis Set of Post IDs keyed by Tag
## Redis Set of interested User IDs key by Tag
!NOTE
# Fast
# Prevented numerous RDBMS queries

!SLIDE
# User Post scores in Redis
## Hash
## Key: User ID
## Field: Post ID
## Value: User interest score

!SLIDE top-left
# What is this?
}}} images/calculator_workflow.png

!SLIDE
}}} images/series_of_queues.png

!SLIDE
# Design considerations

!SLIDE
## Aggressively optimized to reduce DB queries

!SLIDE
## Cache interesting primary keys in Redis Sets
## Reduces need for RDBMS queries

!SLIDE
## For example
### User's IDs interested in a particular Tag
### Post IDs related to a particular Tag

!SLIDE
## Break up *Calculator*
## Wrap each *X Calculator* in a Resque worker
## Each *X Calculator* becomes Resque worker
## Allows *X Calculator* to scale independently

!SLIDE
# Ideal?
## Not really

!SLIDE
# Ideal enough?
## It ran successfully in production!
## Customer was happy!

!SLIDE
# Would it scale?
## Limited by Redis (memory)
## Limited by O(n<sup>2</sup>)
## Also, pruning 
#### (more on that later)

!SLIDE
# Emergent behavior
## Everything is "taggable" in app
## Most AR object lifecycle queued as events
## Engine scored **every** Tag relationship...
## ... including how interested I am in other people!

!SLIDE
# Side business as dating site? ;-)

!SLIDE
# Lesson learned
## Statistical methods
### Because O(n<sup>2</sup>) burnses us preciousssss
!NOTE
# Some arguments are worth having

!SLIDE
# Lesson learned
## Prefer straight K-V over Hash
### No per field TTL on Hash
### Replace with convoluted key names for namespacing

!SLIDE top-left
}}} images/calculator_workflow.png
# Extract smaller workers
!NOTE
## Case statements internally
## Classic polymorphic smell

!SLIDE
# Lesson learned
## Less chattiness with Redis
### Redis Lua scripting
#### Wasn't an option ~2 years ago
#### &nbsp;
### Pipelining
!NOTE
# AWS network calls can still take ~2ms
# Even fast constant time calls to Redis add up!

!SLIDE
# Lesson learned
## Pruning
### Set of Post IDs keyed by Tag
!NOTE
# This will just grow and grow
# Post IDs need to be removed as they outlive their 3 day window
# Each entry could have a timestamp
# Prune per insertion for expired timestamps

!SLIDE
# Lesson learned
## Runtime faster than Ruby
## &nbsp;
# <img src="images/troll.jpg"/>
!NOTE
# Seriously, this wasn't a problem

!SLIDE
# Conclusion

!SLIDE
# Redis is terrific
### As long as you have enough RAM
!NOTE
# Saved us from numerous RDBMS queries
# Minimal impact on overall app performance

!SLIDE
# Inverted indices
## Powerful in K-V data stores

!SLIDE
# Make constraints work for you
!NOTE
# 3 day window made O(n2) suck far far far less

!SLIDE
# Monolithic, but easily divisable, worker

!SLIDE middle
# Evan Light
## [@elight](http://twitter.com/elight)
## [evan.light@rackspace.com](mailto:evan.light@rackspace.com)
