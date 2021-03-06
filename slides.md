# Recommendation Engines with Redis and Ruby


!SLIDE
# Recommendation Engines
# with Redis and Ruby


!SLIDE middle
# Evan Light
## **Code Janitor**
## [@elight](http://twitter.com/elight)
## [evan.light@tripledogdare.net](mailto:evan.light@tripledogdare.net)


!SLIDE middle
}}} images/stuff_i_do.png


!SLIDE
}}} images/kyck.png


!SLIDE
# So a recommendation engine...


!SLIDE
# ... based on the user's interests...


!SLIDE
# ... and social proof


!SLIDE
}}} images/no_idea.jpg
!NOTE
# If you were expecting a true machine learning talk, that ain't me!


!SLIDE
# Exciting!
# So let's begin!


!SLIDE
# The domain


!SLIDE top-left
# Given something like...
}}} images/user-tag.png


!SLIDE top-left
# ... and this...
}}} images/domain.png
!NOTE
# A "Post" can be any arbitrary unit of content.  This system uses Facebook-like Posts.


!SLIDE top-right
# ... we want *this*!
}}} images/scored-user-posts.png
!NOTE
# Subjective (per User) recommendations


!SLIDE
# Trendingness
## "Objective" ranking


!SLIDE
# Constraints
## Near real-time
## Effective even w/ small data sets


!SLIDE
# Goal
## Calculate scalar User interest
## for Tags
## on a per event basis
!NOTE
# Lacking a machine learning background...


!SLIDE
# User event driven
## Liking/Favoriting
## Posting
## Commenting
!NOTE
# Straightforward. Data is entered via "events"


!SLIDE
## Offload processing of events to queue
## **ActiveRecord::Observer feed into Resque**
!NOTE
# How many folks are familiar with Resque?


!SLIDE bottom-left
# Engine design


!SLIDE
}}} images/calculator_workflow.png


!SLIDE
# *Calculator*
## Pop an event off Resque
## "Magic" happens!
## Persists Post scores


!SLIDE
# "Magic"


!SLIDE
# *Trendingness Calculator*
## **Input**: The event
## (Re)compute Post's trendingness score
## **Output**: The trendingness score for the Post


!SLIDE
## *Trendingness Calculator* does not persist its output
## Nor any other *Foo Calculator*
## **TDD made this painfully (literally) obvious**


!SLIDE
# Tangent: On persistence
## Caller responsible for persistence
## Trendingness stored as key-value pairs
## Incrementing trendingness score trivially fast


!SLIDE
# On persistence (cont'd)
## Abstracted Redis dependencies into *RedisStore*
## Any class duck typing *RedisStore* could provide persistence
## *Foo Calculators* only depend on a single persistence interface


!SLIDE
# *User Interest Calculator*
## **Input**: The event
## For originating User:
## **Output**: User's updated interests per Tag


!SLIDE
# *Post Score Calculator*
## **Input**: The event plus User's interests per Tag
## Locate Posts related to these Tags
## Locate Users interested in these Tags
#### O(n<sup>2</sup>) == *slow!*
## **Output**: new Post scores per interested User


!SLIDE top-left
# What is this?
}}} images/calculator_workflow.png


!SLIDE
}}} images/series_of_queues.png


!SLIDE
# Performance "optimizations"


!SLIDE
## Expected RDBMS to be under heavy load
## <u>*Prematurely*</u> optimized to reduce DB queries


!SLIDE
## Cache interesting primary keys in Redis Sets
## Reduces need for expensive database queries


!SLIDE
## For example
### User's IDs interested in a particular Tag
### Post IDs related to a particular Tag


!SLIDE
## Break up *Calculator*
## Wrap each *Foo Calculator* in a Resque worker
## Each *Foo Calculator* becomes Resque worker
## Allows *Foo Calculator* to scale independently


!SLIDE
# Ideal?
## Not really


!SLIDE
# Ideal enough?
## Customer was happy!
## It's still in production!


!SLIDE
# Emergent behavior
## Everything is "taggable" in app
## Most AR object lifecycle queued as events
## Engine scored **every** Tag relationship...
## ... including how interested I am in other people!


!SLIDE
# Side business as dating site? ;-)


!SLIDE
# Coulda woulda shoulda
## Statistical methods
## Perhaps stored Post scores per User differently
### Redis Hash of User ID -> Post ID -> Score
## Redis pruner for non key-values


!SLIDE
# Even more coulda woulda shoulda
## Less chattiness with Redis
### (Redis Lua scripting)
## Runtime faster than Ruby
## <img src="images/troll.jpg"/>
!NOTE
# AWS network calls can still take ~2ms
# Even O(1) calls to Redis add up!


!SLIDE middle
# Evan Light
## **Code Janitor**
## [@elight](http://twitter.com/elight)
## [evan.light@tripledogdare.net](mailto:evan.light@tripledogdare.net)
