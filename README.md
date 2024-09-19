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

### Required resources (1 year):

#### Avarage estimate
##### Memory
Data: 11KB * 100 000 * 400 ~= 440GB
Media: 12MB * 100 000 * 400 ~= 480TB

including AFR (1%):
Data: 440GB * 1.1 ~= 485GB
Media: 480TB * 1.1 ~= 550TB

Total: 550TB

##### Traffic:
Data: 
 - read: 3.2MB * 100 000 * 400 ~= 128TB
 - write: 11KB * 100 000 * 400 ~= 260GB
Media: 
 - read: 60GB * 100 000 * 400 ~= 2 400PB
 - write: 12MB * 100 000 * 400 ~= 480TB
Total: ~2 400 PB

#### Disks
##### Media (photos)

###### **Hot media (last 2 monthes):**

*Using SSD (nVME)*

**Сapacity** = 550TB / 6 = 92TB

**Disks_for_capacity** = 92TB / 30TB ~= 3.1

**Disks_for_throughput** = 72GB/s / 3GB/s = 24

**Disks_for_iops** = 30 000 / 10 000 = 3

**Disks** = max(ceil(3.1), ceil(20), ceil(3)) = 24

###### **Cold media (all medias):**
	We expect that there will be 5 times fewer requests for cold photos

*Using HDD:*

**Сapacity** = 550TB

**Disks_for_capacity** = 550TB / 32TB = 17.2

**Disks_for_throughput** = 14.4GB/s / 0.1GB/s = 144

**Disks_for_iops** = 6 000 / 100 = 60

**Disks** = max(ceil(17.2), ceil(144), ceil(60)) = 144

##### Data

*Using SSD (SATA)*

**Сapacity** = 485GB

**Disks_for_capacity** = 485GB / 100TB = 0

**Disks_for_throughput** = 3.2MB/s / 500MB/s = 0

**Disks_for_iops** = 7500 / 1 000 = 7.5

**Disks** = max(ceil(0), ceil(0), ceil(7.5)) = 8
