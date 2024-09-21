# Travel Social Net - System Design

Homework for [course by System Design](https://balun.courses/courses/system_design).
Travel Social Net is a social network for travelers.

### Functional requirements:
- user can create post about theirs trips  with photos, description and coordinates to the poi
- other users can write comments and rate to the post
- user can subscribe to other users to keep track on their activity
- user can view posts feed of the other user he subscribed
- user can view posts feed of the other user
- search for popular sights and view posts about it

### Non-functional requirements:
- 10 000 000 DAU
- high availability, ~99,95%
- posts and media are always stored
- could be seasonality - more users before holidays (~20% users more)
- CIS audience only (most text must be in Cyrillic)
- on average:
	- each user view about 50 posts every day (includes search and looking through the feeds) with 20 comments each (search could take up to 3-5 seconds, view post details - 1 second, loading media - maximum 1 second per photo) in 10 locations
	- each 10th user rate one post and write one comment (max 1 second for both operations)
	- each 100th user write a new post with ~5 photos (max 1 second for write operations and 1 sec for every image upload)
	- each 1000th user register a new location

## Design overview

### Level 1. High level design
![High level design](/puml/c4-l1.png "High level design")

We assume that user registration is out of our current implementation. 
It could be used ready-made implementation of the SSO service (e.g. Keycloak) or third-party openid auth service (e.g. VK-ID or Yandex-Passport).

### Level 2. Subsystems design
![Subsystems design](/puml/c4-l2.png "Subsystems design")

### Critical flows and subsystem details

#### Create Post
![Create Post](/puml/c4-l2-create_post.png "Create Post")

User's client upload images to the media service first with expiration date +1 day. File storage will remove file automatically if expiration date will be still defined 
(e,g, if creation will be interrupted or failed). Next client call post create method. On successful creation post service notify media service to remove expiration date.
We are using MongoDB to persist Posts because there isn't complicated and tricky relations, no need in the strict transactions and it's easier to keep photos as 
the inner nested array of the media details. We assume that 80% of the posts users view are not older than 2 months. So posts younger than 2 monthes are kept in the hot fast
db and after the 2 monthes they will be removed from the hot db and persisted in the cold db.

#### Comments and Rates
![Comments and Rates](/puml/c4-l2-post_reactions.png "Comments and Rates")

User can write comments and rate posts and reactions service is responsible for this functionality (it could be devided into separated services but I didn't find the reason for now).
Rates are persisted and requested from posts so it's not needed to update rate in the feed on any reaction. Reaction service notify psot service about avarage rate updates through
message queue. It could be aggregated and debounced.

#### Subscription
![Subscription](/puml/c4-l2-subscription.png "Subscription")

Subscription service keep user-to-user subscriptions and statistics how many users has subscriptions and followers. THis information is used to detect celebrities.

#### Feed
![Feed](/puml/c4-l2-feed.png "Feed")

Feed service is responsible for building user's feeds. Feed service keeps full posts for every feed. Celebrity users' posts are kept as the separated list,
service mixes them in the prepared feed in the runtime. We assume that 80% users view less than 50 posts in the feed, if user try to read more, feed service 
requests the post service for older posts. Feed service rebuild feeds on every subscription and post creation, so it's easier to use in-memory DB like Redis.

## Basic calculations

### Entities
#### Location (225B)
- **id** - int64 - 8B
- **caption** - 100 Unicode chars ~200B
- **coordinates** - 2 x double - 8B+8B
- **avgRate** - 1B (3-4 bits must be enough)

#### Post (~2025B ~= 2.1KB)
- **id** - int64 - 8B
- **userId** - int64 - 8B
- **text** - 1000 Unicode chars ~2000B
- **locationId** - int64 - 8B
- **avgRate** - 1B (3-4 bits must be enough)

#### Photo (~210B metadata + ~2MB content)
- **id** - int64 - 8B
- **caption** - 100 Unicode chars ~200B
- **content** ~2MB

#### Comment (~525B)
- **id** - int64 - 8B
- **userId** - int64 - 8B
- **postId** - int64 - 8B
- **text** - 250 Unicode chars ~500B

#### Rate (17B)
- **userId** - int64 - 8B
- **postId** - int64 - 8B
- **value** - 1B (3-4 bits must be enough)

### Operations
#### Create Location
	each 1000th user create a new location
DAU: 10 000 000 / 1000 = 10 000

RPS: 10 000 / 86 400 ~= 0.12

w/traffic: 0.12 * 225B ~= 23B

#### Create Post
	each 100th user write a new post with 5 photos
DAU: 10 000 000 / 100 = 100 000

RPS: 100 000 / 86 400 ~= 1.2

w/traffic: 1.2 * (2.1KB + 5 * 210B + 5 * 2MB) ~= 12MB (data: 3.8KB  + media: 12MB)

#### Create Comment
	each 10th user rate one post and write one comment
DAU: 10 000 000 / 10 = 1 000 000

RPS: 1 000 000 / 86 400 ~= 12

w/traffic: 12 * 525B ~= 6.3KB/s

#### Rate Post
	each 10th user rate one post and write one comment
DAU: 10 000 000 / 10 = 1 000 000

RPS: 1 000 000 / 86 400 ~= 12

w/traffic: 12 * 17B ~= 204B (can be ignored)

##### The highest workload among write operations: 
 - upload media on post creation - 12MB/s
 - write post - 3.8KB
 - write comment - 6.3KB/s


#### Read Locations
	each user view about 10 locations every day (includes search and looking through the feeds)
DAU: 10 000 000

RPS: 10 000 000 / 86 400 * 10 ~= 1200

r/traffic: 
 - data: 1200 * 225B ~= 270KB/s

#### Read Posts
	each user view about 50 posts every day (includes search and looking through the feeds)
DAU: 10 000 000

RPS: 10 000 000 / 86 400 * 50 ~= 6000

r/traffic: 
 - data: 120 * 50 * (2.1KB + 5 * 210B) ~= 120 * 50 * 14KB = 1.7MB/s
 - media: 120 * 50 * 5 * 2MB = 60GB/s

#### Read Posts' Comments
	each user view 20 comments for every 50 viewed post
DAU: 10 000 000

RPS: 10 000 000 / 86 400 * 50 / 50 ~= 120

r/traffic: 
 - data: 120 * (20 * 525B) ~= 1.5MB/s

#### Read Rates
	each user view rate for every 50 viewed post
DAU: 10 000 000

RPS: 10 000 000 / 86 400 * 50 / 50 ~= 120

r/traffic: 
 - data: 120 * 17B ~= 2KB

##### The highest workload among read operations: 
 - download post media - 60GB/s
 - read posts - 1.7MB/s
 - read comments - 1.5MB/s

### Avarage resources estimate for 1 year (full system)
#### Traffic:
Data: 
 - read: 3.2MB * 100 000 * 400 ~= 128TB
 - write: 11KB * 100 000 * 400 ~= 260GB
Media: 
 - read: 60GB * 100 000 * 400 ~= 2 400PB
 - write: 12MB * 100 000 * 400 ~= 480TB
Total: ~2 400 PB

#### Memory
Data: 11KB * 100 000 * 400 ~= 440GB
Media: 12MB * 100 000 * 400 ~= 480TB

including AFR (1%):
Data: 440GB * 1.01 ~= 445GB
Media: 480TB * 1.01 ~= 484.5TB

Total: ~485TB

### Required resources for 1 year by services:

#### Media

##### Traffic:
 - read: 60GB * 100 000 * 400 ~= 2 400PB
 - write: 12MB * 100 000 * 400 ~= 480TB

##### Memory
###### **Hot media (last 2 monthes):**

*Using SSD (nVME)*

**Сapacity** = 485TB / 6 = 81TB

**Disks_for_capacity** = 81TB / 30TB ~= 2.7

**Disks_for_throughput** = 72GB/s / 3GB/s = 24

**Disks_for_iops** = 30 000 / 10 000 = 3

**Disks** = max(ceil(2.7), ceil(24), ceil(3)) = 24

**Hosts** = disks / disks_per_host = 24 / 1

**Hosts_with_replication** = hosts * replication_factor = 24 * 2 = 48


###### **Cold media (all medias):**
	We expect that there will be 5 times fewer requests for cold photos

*Using HDD:*

**Сapacity** = 485TB

**Disks_for_capacity** = 550TB / 32TB = 15.2

**Disks_for_throughput** = 14.4GB/s / 0.1GB/s = 144

**Disks_for_iops** = 6 000 / 100 = 60

**Disks** = max(ceil(15.2), ceil(144), ceil(60)) = 144

**Hosts** = disks / disks_per_host = 144 / 1

**Hosts_with_replication** = hosts * replication_factor = 144 * 2 = 288

#### Posts service
##### Traffic:
Traffic:
 - write:
   - Posts: 1.2 * (2KB + 5 * 0.5KB) ~= 3.6KB/sec
   - Locations: 0.2 * 0.25KB ~= 0.05KB/sec
   - **Total**: ~3.7KB/sec
 - read: 
   - Posts: 120 * 50 * 3.6KB ~= 21.6MB/s
   - Locations: 1200 * 0.25KB ~= 300KB/sec
   - **Total**: ~22MB/s

##### Memory
*Using SSD (SATA)*

**Сapacity**: 3.7KB * 100 000 * 400 ~= 148GB
including AFR (1%): ~150GB

**Disks_for_capacity** = 150GB / 100TB = 0.002

**Disks_for_throughput** = 26MB/s / 500MB/s = 0.05

**Disks_for_iops** = 7500 / 1 000 = 7.5

**Disks** = max(ceil(0.002), ceil(0.05), ceil(7.5)) = 8

**Hosts** = disks / disks_per_host = 8 / 1

**Hosts_with_replication** = hosts * replication_factor = 8 * 2 = 16

#### Reactions service

##### Traffic:
Traffic:
 - write: 12 * 2KB ~= 24KB/sec
 - read: 120 * 20 * 2KB ~= 5MB/sec

##### Memory
*Using SSD (SATA)*

**Сapacity**: 
 - reactions: 24KB * 100 000 * 400 ~= 960GB
 - stats: (1.2 * 24B + 0.2 * 32B) * 100 000 * 400 ~= 1.5GB
 - **Total**: 962GB
including AFR (1%): ~972GB

**Disks_for_capacity** = 972GB / 100TB = 0.01

**Disks_for_throughput** = 5.25MB/s / 500MB/s = 0.01

**Disks_for_iops** = 240 / 1 000 = 0.25

**Disks** = max(ceil(0.01), ceil(0.01), ceil(0.25)) = 1

**Hosts** = disks / disks_per_host = 1

**Hosts_with_replication** = hosts * replication_factor = 1 * 2 = 2

#### Subscriptions service

##### Traffic:
Traffic:
 - write: 12 * 16B ~= 192B/sec
 - read: 120 * 16B ~= 2KB/sec

##### Memory
*Using SSD (SATA)*

**Сapacity**: 200M * 24B + 200M * 100 * 16B ~= 325GB
including AFR (1%): ~360GB

**Disks_for_capacity** = 360GB / 100TB = 0.004

**Disks_for_throughput** = 2.2KBMB/s / 500MB/s = 0.0..1

**Disks_for_iops** = 123 / 1 000 = 0.12

**Disks** = max(ceil(0.01), ceil(0.01), ceil(0.12)) = 1

**Hosts** = disks / disks_per_host = 1

**Hosts_with_replication** = hosts * replication_factor = 1 * 2 = 2
