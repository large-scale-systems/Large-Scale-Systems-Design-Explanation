## What this system looks like?

* This system provides us to share text with other users. This is typically social networking system. When users use this system, they post
character sets(maximum 150 characters) and they are called as "tweets". Notice that in Twitter system, if you register this
site, you can post and read tweets, on the other hand, if you are not register (or not login) you can only read tweets. Moreover, 
users reach this system any platform like a desktop application or mobile application.

## Requirements and goals of Twitter

* There are typically 3 types of requirements. First one is functional requirements, the second one is nonfunctional requirements and the 
third one is extended requirements. During the development process, we should determine all three requirements clearly.
Users can register and login.

* Users can log out.
* When users log in, they can share or read tweets.
* When users don't register or log in, they can only read tweets.
* Users can follow other users.
* Users can mark some tweets as a favourite.
* The system should be highly reliable. This means that no of post should be lost.
* The system should be highly available. This means that users should reach this system whenever they want.
* The system should return the requests with minimum latency. This means that users should reach the posts nearly real lime.
* The system can be monitored. For this purpose, we can keep the location of users, or location of shared tweets.
* During the development of the system, REST or SOAP API's can be used. (REST API is a little bit good option)
* When users post tweets, these tweets can contain any file or photo. Notice that, if we design a system like Twitter, we can determine the acceptable latency like as a 300 ms.
* Consistency can be acceptable if the system gives the request properly later.
* Users can search the tweets during the travel the site.
* Finally, people can answer the tweets.

## Capacity and Estimation

If we assume that daily, 100 Million new tweets come to this system, and notice that maximum tweet size is 150 character,

Daily storage = 100 Million * (140 * 2 + 20) = 100 Million * 300 bytes = 30 GB per day. (20 Byte for metadata of tweet)
If we think about 10 years later, 30 GB * 30 * 12 * 10 = 30 GB * 3600 = 108 TB for text tweet. Additionaly if we assume that, 
If any photo = 100 KB and any video is 2 MB and ratio is (5:1:1);

100 KB * 20 Million * 30 * 12 * 10 = 200 Million * 3600 KB = 2 GB * 360 = 720 TB for photos
2 MB * 20 Million * 30 * 12 * 10 = 400 Million * 360 MB = approximately 4*10^12 byte * 36 = 144 * 10^12 byte for videos.

This is a huge data. We need to sharding.

Additionally, we can calculate bandwidth estimate. Notice that for banding, we should calculate the read and write tweet capacity in a 
second. Moreover, caching mechanism should be used to develop this system. If we think that the general approximation of caching 
mechanism is %20 for daily storage, we can easily calculate the necessary memory for caching.

## API's

To develop this site, we can use REST API for mobile application and desktop application. (Notice that we can also use SOAP API's)
There are several main API's.

* createTweet
* deleteTweet
* updateTweet
* getTweet
* searchTweet
* markTweet
* followUser
* login
* logout
...etc

Notice that, the key parameter should be used almost every API because of controlling registration and monitoring system. 
(We can obtain information about users with key). A successful create API will return URL demonstrated by string and failed API return 
HTTP_Response error.

## Think about high-level system design

The system is read-heavy system and huge data will be stored for this system. To provide this purpose, we should use multiple application
servers. (to provide minimum latency) If we use multiple application servers we can use the load balancer for sharing incoming request.
We can use robin prenciples for sharing incoming request for the load balancer, but round robin has some lack of properties. If the 
server is busy or not ready to return a response, the load balancer will continue to send the request to this server. This condition 
is not good so we can develop more intelligent sharing principles to handle this situation.
Also notice that we should have a database server, file storage server and metadata server for reducing the system load.

As explained later, we can use multiple cache servers and database servers for providing partitioning. Additionally, the load balancer 
will be used between application servers and cache servers and application servers and file servers and application servers
and metadata servers.

## Database

We need to store information about users and tweets. Notice that we can use SQL or NoSQL but if we use NoSQL then we should keep (user, 
tweet)

* **Tweet (TweetID:bigInt, Content, CreationDate, Tweet Location, NumberOfFavourites) - (Except Content all of the information will be kept in metadata storages)**
* **Users (UserID:bigInt, Name, Email, Age, LastLogin)**
* **UserFollow (UserID1, UserID2)**
* **UserFavourite (UserID, TweetID, FavouriteDate)**

Additionally, when creating TweetID or UserID we can use Key generation service. Key generation service creates all the possible ID's 
and stores the KGS Databases before. There are two tables. First one is usedKeys second one is nonUsedKeys. notice that, any of two 
servers can wish to key at the same time. To handle this problem, if any key used before, the database server should wish to new key 
from the application server
and application server should wish to the new key for KGS database server. Additionally, Some keys can be kept in over the application 
servers. But if we think about when the server fails, we will lose all the keys to the application server. Notice that if we have 
enough key for this system. (8 character longs. and base64 encoding) there are approximately billions of keys here and losing keys 
are not important.
As a brief, if the server fails, server wishes to new keys from the KGS database storages.

## Data Sharding

There will be huge data kept in database servers and we have a motto that system should give a request with acceptable latency. 
To providing this purpose we use partitioning. (shard data.) There are some options for sharing. First one is hash based on userID, 
second one is hash based on tweetID, the third one is hash based on tweetcreationdate etc.. If we use hash based on userID.

The system can be unbalanced since some users may be hot or some users post tweets very little. If we use hash based on tweetID, 
for creating timeline generation, the system should keep track of some process. (Like bring tweets from different servers). This 
causes higher latency. If we use hashing with creation time, we can quickly obtain the new or hot tweets. On the other hand, obtaining 
latest tweets causes higher latencies. Briefly, we can use hash based on tweetID and tweetCreationTime for taking advantage of both of 
them.

## Caching

We can use LRU (least recently used) principles for caching mechanism. Additionally, we can use hottest tweets or users for caching. 
We can use MemCache for this operation. As you know, caching operation provide us to obtain data quickly. Notice that we should determine
the number of necessity caching servers. As we explained before, we can try to use %20 principles.

## Detailed explanation of high-level system design

Clients applications are routed to application servers by the load balancer.
Application servers quickly look the caching servers by the load balancer whether data is here or not.
If data is not here, then application server looks to DB Shard. (We should determine the Sharding number)
When data is found to DB Shard, LRU caching mechanism will be used and cache server will be updated.
Notice that file storage or metadata servers connect the application servers at the same time.

## Data replication

To provide high reliability, we can keep data more than one servers. The best idea for replication is every always-on server, two or 
more copy servers should be used. Additionally, we can use this idea that always-on servers can be used for writing data and copy servers
can be used for only reading operations. This helps system to reduce traffic loads. Moreover, if always-on servers (primary) fail, one 
of the secondary(copy) servers behave like an always-on server.

## Monitoring

To measure the durability of the system, monitoring operation is more crucial.By monitoring operation we can quickly notice the
more needed replication, sharing, caching or load balancing.
* We can keep new tweets in a day.
* we can measure latency.
* we can keep the location of the tweets.
* how many tweets serving at the same time?
etc....
