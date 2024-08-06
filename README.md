# Travel Social Net - System Design

Homework for [course by System Design](https://balun.courses/courses/system_design). Travel Social Net is a social network for travelers.

### Functional requirements:
- user can create post about theirs trips  with photos, description and coordinates to the poi
- other users can write comments and rate to the post
- user can subscribe to other users to keep track on their activity
- user can view posts feed of the other user he subscribed
- user can view posts feed of the other user
- search for popular see sides and view posts about it
### Non-functional requirements:
- 10 000 000 DAU
- high availability, ~99,95%
- posts and media are always stored
- could be seasonality - more users before holidays
- CIS audience only (most text must be in Cyrillic)
- on average:
	- each user view about 50 posts every day (includes search and looking through the feeds) with 20 comments each
	- each 10th user rate one post and write one comment
	- each 100th user write a new post with ~5 photos
## Basic calculations

### Entities
#### Post (~2033B ~= 2.1KB)
**id** - int64 - 8B
**userId** - int64 - 8B
**text** - 1000 Unicode chars ~2000B
**coordinates** - 2xdouble - 8B+8B
**avgRate** - 1B (3-4 bits must be enough)

#### Photo (~210B metadata + ~2MB content)
**id** - int64 - 8B
**caption** - 100 Unicode chars ~200B
**content** ~2MB

#### Comment (~525B)
**id** - int64 - 8B
**userId** - int64 - 8B
**postId** - int64 - 8B
**text** - 250 Unicode chars ~500B

#### Rate (17B)
**userId** - int64 - 8B
**postId** - int64 - 8B
**value** - 1B (3-4 bits must be enough)

### Operations
#### Create Post
	each 100th user write a new post with 5 photos
DAU: 10 000 000 / 100
RPS: 100 000 / 86 400 ~= 1.2
w/traffic: 1.2 * (2.1KB + 5 * 210B + 5 * 2MB) ~= 12MB (data: 3.8KB  + media: 12MB)
#### Create Comment
	each 10th user rate one post and write one comment
DAU: 10 000 000 / 10
RPS: 1 000 000 / 86 400 ~= 12
w/traffic: 12 * 525B ~= 6.5MB
#### Rate Post
	each 10th user rate one post and write one comment
DAU: 10 000 000 / 10
RPS: 1 000 000 / 86 400 ~= 12
w/traffic: 12 * 17B ~= 204B (can be ignored)
##### The highest workload among write operations: 
- upload media on post creation - 12MB/s
- write comment - 6.5MB/s
#### Read Posts
	each user view about 50 posts every day (includes search and looking through the feeds) with 20 comments each
DAU: 10 000 000
RPS: 10 000 000 / 86 400 ~= 120
r/traffic: 
 - data: 120 * (2.1KB + 5 * 210B + 20 * 525B) ~= 120 * 14KB = 1.7MB
 - media: 120 * 5 * 2MB = 1.2GB

### Required resources:
#### 1 year:
##### Memory
Data: 6.5MB * 100 000 * 400 ~= 260TB
Media: 12MB * 100 000 * 400 ~= 480TB

including AFR (1%):
Data: 260 TB * 1.1 ~= 300TB
Media: 480 TB * 1.1 ~= 550TB
Total: 850 TB (cost ~$25 500)
##### Traffic:
Data: 
 - read: 1.7 MB * 100 000 * 400 ~= 68TB
 - write: 6.5MB * 100 000 * 400 ~= 260TB
Media: 
 - read: 1.2GB * 100 000 * 400 ~= 48 PB
 - write: 12MB * 100 000 * 400 ~= 480 TB
Total: ~50 PB (cost ~$5M)



