
# Instagram Newsfeed System Design

## 1. Deciding Requirements

### Functional Requirements
- **Posts**
  - Texts
  - Image
  - Video
- **Follow/Unfollow** other users
- **News feed**
  - In the reverse chronological order: Newest to oldest
- **Like & Comment**
- **User notifications**

### Non-Functional Requirements
- **Availability** - 99.999% uptime
- **Eventual Consistency** - It's okay if a post takes 1-2 seconds to appear.
- **Latency** - Newsfeed should load in 1-2 seconds on Home click.
- **Scalability** - Handle global scale (200M DAU, 2.1B MAU)
- **Extensibility** - Easy to add features like comment replies, recommendations, ads
- **Usability** - Fast rendering, high-quality user experience

---

## 2. Capacity Planning

### DAU / MAU
- DAU: 500M
- MAU: 2B

### Throughput

#### Write Throughput
1. **Creating Posts**
   - 10% of DAU post daily: 50M create requests/day

2. **Follow/Unfollow**
   - 1 follow/week/user = 71.4M follow requests/day

3. **Comments/Likes**
   - 3 activities/user/day = 1.5B activities/day

#### Read Throughput
- Each user reads 100 posts/day → 500M users → **50B read requests/day**

### Storage Estimation

#### Posts
- 50M posts/day with size assumptions:
  - Video: 20MB (20%)
  - Image: 0.5MB (60%)
  - Text: 100KB (20%)
- Total: ~216TB/day → ~750PB in 10 years

#### Follow Activity
- 16 bytes/follow → ~4TB in 10 years

#### Comments/Likes
- 216 bytes/activity → ~1.1PB in 10 years

### Memory Estimation (Cache)
- 1% of daily storage → 2.16TB/day

### Network/Bandwidth Estimation
- **Ingress**: 216 TB/day → 2.5 GB/s
- **Egress**: 216 PB/day → 2.5 TB/s

---

## 3. API Design

### Create Text Post

```http
POST /v1/posts
{
  "UserId": "1234",
  "Text": "Excited for my Europe Trip!",
  "HashTags": ["travel", "fun"]
}
```

### Create Image Post

```http
POST /v1/posts
{
  "UserId": "1234",
  "MediaUrl": "https://blob.core.microsoft/strgact/container1/abc.jpg",
  "Description": "Feeling Relaxed!!!",
  "HashTags": ["travel", "fun"]
}
```

### Commenting on a Post

```http
POST /v1/comments
{
  "UserId": "1234",
  "PostId": "1423",
  "Comment": "Looking great"
}
```

### Follow / Unfollow

```http
POST /v1/follow
{
  "FollowerId": "12345",
  "FolloweeId": "1423"
}
```

### Read NewsFeed

```http
GET /v1/feeds/{userId}
```

---

## 4. High-Level Design (HLD)

### Follow/Unfollow User
1. Client → API Gateway → Follow Service → Follow DB (Graph DB)

### Create Text Post
1. Client → API GW → PostWriter Service → PostsDB

### Read Posts
1. Client → API GW → NewsFeed Reader → FollowDB → PostsDB → Sorted → Client

### Optimized Post + Read Flow
1. Client creates post → PostsDB
2. PostWriter → MQ → NewsFeed Service → Fetch post + followers → Update FeedsDB + Cache

### Create Image/Video Post
1. Client requests presigned URL → Uploads to Object Storage
2. Gets media URL → Sends post to PostWriter
3. PostWriter → PostsDB → MQ → NewsFeed Reader → FeedsDB + Cache

### Read NewsFeed
1. Client → API GW → NewsFeed Reader → Feed Cache → Response (includes media URLs)

### Comment on Post
1. Client → API GW → Comment Service → CommentsDB
2. Comment Service → MQ → Notification Service

### Like on Post
1. Client → API GW → Likes Service → LikesDB + Likes Cache
2. Likes Service → MQ → Notification Service

> **Likes count in Newsfeed** is fetched live from Likes Cache.

---

## 5. Deep Dive Insights

### DB Selection
1. **PostsDB** - NoSQL (high volume, unstructured)
2. **FollowDB** - Graph DB (relationships)
3. **FeedsDB** - NoSQL (userId → feed mapping)
4. **CommentsDB** - NoSQL (nested comments)
5. **LikesDB** - NoSQL (likes/reactions)

### DB Modeling

#### PostsDB

```json
{
  "postId": "unique_post_id",
  "userId": "user_Id",
  "text": "post text",
  "mediaUrls": ["url1", "url2"],
  "timestamp": "1720748150"
}
```

#### FollowDB

```json
{
  "userId": "user1_Id",
  "followers": [
    {"followerId": "user2_Id", "timestamp": "1720748150"}
  ],
  "followees": [
    {"followeeId": "user3_Id", "timestamp": "1720748150"}
  ]
}
```

#### FeedsDB

```json
{
  "userId": "user_Id",
  "feedItems": [
    {
      "postId": "unique_post_id",
      "userId": "user_Id",
      "text": "post text",
      "mediaUrls": ["url1", "url2"],
      "timestamp": "1720748150"
    }
  ]
}
```

#### CommentsDB

```json
{
  "commentId": "unique_comment_id",
  "userId": "user_Id",
  "postId": "post_id",
  "comment": "comment text",
  "timestamp": "1720748150"
}
```

#### LikesDB

```json
{
  "likeId": "unique_like_id",
  "userId": "user_Id",
  "postId": "post_id",
  "timestamp": "1720748150"
}
```

### Pre-Signed URLs
- Secure, temporary URLs for uploading media to object storage (e.g., S3, Azure Blob)
- Speeds up uploads, avoids direct server load

### Media Processing
- After upload, media is converted into multiple formats and resolutions
- Enables optimized delivery based on device & network

---

## Handling Celebrity Posts

> If a user like Ronaldo posts, generating feeds for 600M followers instantly is inefficient.

**Solution**: Combine cached feeds with **on-demand updates** for high-profile users.

- Only push to active/online users
- Others can pull during login or refresh
- Reduces system strain and improves performance

---

© Designed with scalability, resilience, and usability in mind.
