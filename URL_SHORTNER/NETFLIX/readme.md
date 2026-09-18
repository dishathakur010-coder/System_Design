# 🎬 Netflix — System Design

A scalable system design for a Netflix-like video streaming platform that supports millions of users browsing, searching, discovering, and streaming movies and TV shows.

---

## 📌 Overview

Netflix is a large-scale video streaming platform where users can:

- Create accounts and profiles
- Browse movies and TV shows
- Search for content
- View movie details and thumbnails
- Receive personalized recommendations
- Play, pause, and resume videos
- Maintain watch history
- Continue watching previously started content

The system is designed to handle a large number of concurrent users while providing low-latency responses and reliable video streaming.

The architecture separates:

- Application requests
- Backend services
- Application data
- Video storage
- Content delivery

This allows different parts of the system to scale independently.

---

## 🏗️ High-Level Architecture

```text
                         USER
                  Mobile / TV / Browser
                           |
                           v
                          DNS
                           |
                           v
                    LOAD BALANCER
                           |
                           v
              +--------------------------+
              |      NETFLIX BACKEND     |
              |                          |
              |  User Service            |
              |  Movie/Catalog Service   |
              |  Search Service          |
              |  Recommendation Service  |
              |  Playback Service        |
              +------------+-------------+
                           |
                    +------+------+
                    |             |
                    v             v
                DATABASE        CACHE
                                    
                           |
                           v
                    VIDEO STORAGE
                           |
                           v
                          CDN
                           |
                           v
                         USER
```

---

## ⚙️ Functional Requirements

1. Users should be able to register and log in.
2. Users should be able to create and manage profiles.
3. Users should be able to browse movies and TV shows.
4. Users should be able to search for content.
5. Users should be able to view movie and series details.
6. The system should provide personalized recommendations.
7. The system should display thumbnails, posters, and artwork.
8. Users should be able to play movies and episodes.
9. Users should be able to pause and resume playback.
10. The system should maintain watch history.
11. The system should support Continue Watching.
12. The system should support different video qualities.
13. The system should deliver video content efficiently.

---

## 🚀 Non-Functional Requirements

### Scalability

The system should support millions of users and a large number of concurrent requests.

### High Availability

The system should continue operating even if some servers or components fail.

### Low Latency

Browsing, searching, and loading movie information should be fast.

### Reliable Streaming

Videos should be delivered smoothly with minimal interruptions.

### Fault Tolerance

Failure of one server or service should not bring down the entire platform.

### Security

User accounts, authentication, and content access should be protected.

### Data Durability

Important information such as user data and watch history should not be lost.

### Efficient Content Delivery

Large video files should be delivered efficiently without putting unnecessary load on backend servers.

---

# 🔌 API Design

## Get Home Page

```text
GET /v1/home
```

Returns the content required to display the user's home page.

Example response:

```json
{
  "continueWatching": [],
  "trending": [],
  "recommended": [],
  "popular": []
}
```

---

## Search Content

```text
GET /v1/search?q=Wednesday
```

Returns movies and series matching the search query.

---

## Get Movie Details

```text
GET /v1/movies/{movieId}
```

Example:

```text
GET /v1/movies/501
```

Returns information about a movie.

Example response:

```json
{
  "movieId": "501",
  "title": "Example Movie",
  "genre": "Drama",
  "duration": 120
}
```

---

## Get Recommendations

```text
GET /v1/recommendations
```

Returns personalized recommendations for the user.

---

## Start Playback

```text
POST /v1/playback
```

Example request:

```json
{
  "movieId": "501",
  "profileId": "101"
}
```

The Playback Service processes the request and provides the information required to start streaming.

---

## Update Watch Progress

```text
POST /v1/watch-progress
```

Example:

```json
{
  "movieId": "501",
  "position": 2100
}
```

This allows the system to remember where the user stopped watching.

---

# 🧩 Main Components

## 1. DNS

DNS helps the user's device locate the Netflix service.

```text
User
  |
  v
Netflix Domain
  |
  v
DNS
  |
  v
Netflix Service
```

DNS does not store movie information or deliver the video.

Its main responsibility is helping the client find where to send its request.

---

## 2. Load Balancer

The Load Balancer distributes incoming requests among multiple backend servers.

```text
                 LOAD BALANCER
                /      |      \
               v       v       v
          Server 1 Server 2 Server 3
```

This prevents a single server from handling all requests.

It also allows the backend to scale horizontally.

---

## 3. Netflix Backend

The backend contains the application's business logic.

Different responsibilities can be separated into different services.

```text
NETFLIX BACKEND

User Service
Movie / Catalog Service
Search Service
Recommendation Service
Playback Service
```

---

## 4. User Service

The User Service handles user-related operations.

It can manage:

- User accounts
- Login
- Authentication
- Profiles
- Subscription information
- User preferences

---

## 5. Movie / Catalog Service

The Catalog Service manages information about movies and TV shows.

For example:

```text
Movie ID
Title
Description
Genre
Release Date
Duration
Actors
Artwork
```

The Catalog Service manages information **about the movie**.

The actual movie video is stored separately.

---

## 6. Search Service

The Search Service handles searches made by users.

Example:

```text
User searches "Wednesday"
            |
            v
      Search Service
            |
            v
      Matching Content
            |
            v
           User
```

A search index can be used to quickly find matching titles.

---

## 7. Recommendation Service

The Recommendation Service provides personalized content recommendations.

It can use information such as:

- Watch history
- Previously watched content
- Genres
- User interactions
- Similar content

Example:

```text
User Activity
      |
      v
Recommendation Service
      |
      v
Recommended Movies
      |
      v
     User
```

---

## 8. Playback Service

The Playback Service handles requests related to playing a movie or episode.

For example:

```text
User clicks Play
       |
       v
Playback Service
       |
       v
Checks playback information
       |
       v
Provides information required for streaming
```

The Playback Service handles the application-side playback logic.

The actual video can be delivered through the CDN.

---

# 🗄️ Database Design

The database stores structured application data.

## USER

```text
USER
-------------------------
user_id          PRIMARY KEY
email            UNIQUE
password_hash
created_at
subscription_id
```

## PROFILE

```text
PROFILE
-------------------------
profile_id       PRIMARY KEY
user_id          FOREIGN KEY
name
preferences
```

## MOVIE

```text
MOVIE
-------------------------
movie_id         PRIMARY KEY
title
description
genre
release_date
duration
```

## WATCH_HISTORY

```text
WATCH_HISTORY
-------------------------
profile_id       FOREIGN KEY
movie_id         FOREIGN KEY
position
last_watched
```

Example:

```text
Profile ID : 101
Movie ID   : 501
Position   : 35 minutes
```

This information can be used to implement Continue Watching.

## ARTWORK

```text
ARTWORK
-------------------------
artwork_id       PRIMARY KEY
movie_id         FOREIGN KEY
image_url
type
```

A movie can have multiple artwork options.

---

# 🔗 Database Relationships

```text
USER
  |
  +---- PROFILE
          |
          +---- WATCH_HISTORY
          |
          +---- USER PREFERENCES


MOVIE
  |
  +---- ARTWORK
  |
  +---- VIDEO FILES
```

---

# ⚡ Caching Strategy

Netflix has a large number of frequently accessed movies and metadata.

A cache stores frequently requested information so that it can be returned faster.

For example:

```text
movie:501 → Movie Metadata
```

Instead of every request going directly to the database:

```text
User
  |
  v
Backend
  |
  v
Database
```

the system can first check the cache:

```text
User
  |
  v
Backend
  |
  v
Cache
  |
  v
Movie Metadata
```

### Cache Miss

If the requested data is not present in the cache:

```text
Backend
   |
   v
Cache Miss
   |
   v
Database
   |
   v
Update Cache
   |
   v
Return Data
```

### Benefits

- Faster response time
- Reduced database load
- Better performance
- Helps handle traffic spikes

---

# 🔄 Browse Flow

When a user opens the Netflix home page:

```text
User
  |
  v
DNS
  |
  v
Load Balancer
  |
  v
Catalog Service
  |
  +----> Cache
  |
  +----> Database
  |
  v
Movie Metadata
  |
  v
User
```

The Catalog Service retrieves movie information.

Frequently accessed information can be served from the cache.

---

# 🔎 Search Flow

When a user searches for a movie:

```text
User
  |
  v
Load Balancer
  |
  v
Search Service
  |
  v
Search Index
  |
  v
Matching Movies
  |
  v
User
```

For example:

```text
User searches:
"Wednesday"

        |
        v

Search Service

        |
        v

Matching Movies / Series

        |
        v

Results shown to User
```

---

# ⭐ Recommendation Flow

The recommendation system uses user activity to generate personalized recommendations.

```text
User Activity
      |
      v
Watch History
      |
      v
Recommendation Service
      |
      v
Recommended Content
      |
      v
User
```

The Recommendation Service can consider:

- Watch history
- Genres
- User interactions
- Similar content
- Previously watched content

---

# 🖼️ Thumbnail Selection

A movie can have multiple thumbnails or artwork options.

```text
Movie A
  |
  +---- Thumbnail 1
  |
  +---- Thumbnail 2
  |
  +---- Thumbnail 3
  |
  +---- Thumbnail 4
```

The system can select suitable artwork based on factors such as:

- User
- Context
- Device
- Personalization
- Experimentation

The flow can be:

```text
User
  |
  v
Recommendation Service
  |
  v
Select Content
  |
  v
Artwork / Thumbnail Selection
  |
  v
CDN
  |
  v
User
```

### Important Distinction

**Recommendation Service**

```text
Which content should be shown?
```

**Artwork Selection**

```text
Which image should represent that content?
```

**CDN**

```text
How should the image be delivered efficiently?
```

---

# 💾 Video Storage

The actual movie and episode files are stored separately from the application database.

A movie may have multiple quality versions:

```text
Movie A
  |
  +---- 1080p
  |
  +---- 720p
  |
  +---- 480p
```

The video can also be divided into smaller segments:

```text
Movie
  |
  +---- Segment 1
  +---- Segment 2
  +---- Segment 3
  +---- Segment 4
  +---- ...
```

This allows the client to receive the video progressively.

---

# ▶️ Video Playback Flow

When the user clicks **Play**:

```text
User
  |
  v
Load Balancer
  |
  v
Playback Service
  |
  v
Video Storage
  |
  v
CDN
  |
  v
Video Segments
  |
  v
User
```

The Playback Service handles the application-side playback request.

The CDN delivers the actual video content.

---

# 🌐 CDN Strategy

A CDN (Content Delivery Network) is used to efficiently deliver large media content.

The CDN can deliver:

- Video segments
- Thumbnails
- Posters
- Images

Example:

```text
                    VIDEO STORAGE
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      CDN Region 1   CDN Region 2   CDN Region 3
          |              |              |
          v              v              v
        Users          Users          Users
```

### Why CDN?

If every user downloads video directly from central storage:

```text
Millions of Users
       |
       v
Central Video Storage
```

the storage and network would receive enormous traffic.

Using a CDN:

```text
Video Storage
      |
      v
     CDN
   /  |  \
  v   v   v
Users Users Users
```

allows content to be delivered from locations closer to users.

---

# 📺 Adaptive Video Quality

The same content can be available in different qualities:

```text
1080p
720p
480p
```

The client can select an appropriate quality depending on:

- Network speed
- Available bandwidth
- Device capabilities

Example:

```text
Fast Internet
      |
      v
    1080p


Medium Internet
      |
      v
     720p


Slow Internet
      |
      v
     480p
```

This helps maintain smoother playback.

---

# 📊 Capacity Estimation

A Netflix-like system has two very different types of traffic:

### Application Traffic

Includes:

- Login requests
- Search requests
- Browse requests
- Movie metadata requests
- Recommendation requests
- Watch-progress updates

### Video Traffic

Includes:

- Video segments
- Different video qualities
- Large amounts of bandwidth

Video traffic is significantly larger than normal API traffic.

Therefore, video delivery should primarily be handled by the CDN.

---

# 💾 Storage Estimation

The system stores two major types of data.

## Application Data

```text
Users
Profiles
Movie Metadata
Watch History
Recommendations
```

## Media Data

```text
Movies
Episodes
Video Segments
Thumbnails
Posters
Artwork
```

Video files represent the majority of the storage requirement.

Therefore, large media files should be stored in dedicated scalable storage rather than the normal application database.

---

# 🖥️ Server Estimation

The application layer can be horizontally scaled.

```text
                 Load Balancer
                 /     |     \
                v      v      v
          Backend  Backend  Backend
          Server 1 Server 2 Server 3
```

Additional servers can be added as traffic increases.

Different backend services can also be scaled independently.

For example:

```text
Search Service
 ├── Instance 1
 ├── Instance 2
 └── Instance 3
```

---

# 📈 Scalability

## Horizontal Scaling

Instead of relying on a single server:

```text
Client
  |
  v
Single Server
```

multiple servers can be used:

```text
             Load Balancer
             /     |     \
            v      v      v
        Server 1 Server 2 Server 3
```

More servers can be added when traffic increases.

---

## Database Scaling

As the amount of data increases, the database can use:

- Replication
- Partitioning
- Sharding

---

## Cache Scaling

Caching reduces repeated database requests for popular data.

```text
Users
  |
  v
Backend
  |
  v
Cache
  |
  v
Database
```

---

## CDN Scaling

The CDN can distribute content across multiple locations.

```text
                  Video Storage
                       |
          +------------+------------+
          |            |            |
          v            v            v
       CDN A         CDN B        CDN C
          |            |            |
          v            v            v
        Users        Users        Users
```

---

# 🛡️ Reliability

The system should avoid having a single point of failure.

### Backend Reliability

Use multiple backend instances.

```text
Load Balancer
     |
     +---- Server 1
     |
     +---- Server 2
     |
     +---- Server 3
```

If one server fails, other servers can continue serving requests.

### Database Reliability

Database replication can be used to maintain copies of important data.

### Video Storage Reliability

Video files should be stored using durable storage with redundancy.

### CDN Reliability

Multiple CDN locations can help continue content delivery even if one location experiences problems.

---

# 🔐 Security

The system should protect user accounts and content.

Security measures can include:

- HTTPS/TLS
- Authentication
- Authorization
- Secure password storage
- Rate limiting
- API validation
- Access control
- DDoS protection
- Monitoring and logging

---

# 🧩 Design Decisions

## Why use a Load Balancer?

A single server cannot efficiently handle all requests at large scale.

The Load Balancer distributes traffic across multiple backend servers.

---

## Why use a Cache?

Frequently requested data can be served faster from cache.

This reduces database load and improves response time.

---

## Why use a CDN?

Video files are very large.

The CDN allows content to be delivered efficiently from locations closer to users.

---

## Why separate Video Storage from Database?

The normal database stores structured application information.

Video files are much larger and require specialized scalable storage.

Therefore:

```text
Application Data
       |
       v
   Database


Video Files
       |
       v
Video Storage
```

---

## Why use multiple Backend Services?

Different services have different responsibilities.

For example:

```text
User Service
Catalog Service
Search Service
Recommendation Service
Playback Service
```

This allows them to be developed, maintained, and scaled independently.

---

## Why separate Playback from Video Delivery?

The Playback Service handles playback-related application logic.

The CDN handles actual video delivery.

```text
Playback Service
       |
       v
Handles Playback Logic


CDN
       |
       v
Delivers Video
```

This prevents backend servers from becoming responsible for transferring all video data.

---

# 📌 Core Design Principle

The main design principle is to separate **application logic from media delivery**.

### Application Requests

```text
Client
  |
  v
DNS
  |
  v
Load Balancer
  |
  v
Backend Services
  |
  +---- Cache
  |
  +---- Database
```

### Video Delivery

```text
Client
  |
  v
Playback Service
  |
  v
Video Storage
  |
  v
CDN
  |
  v
Client
```

This separation allows the application layer and video delivery layer to scale independently.

---

# 🗺️ Complete System Flow

```text
                         USER
                           |
                           v
                          DNS
                           |
                           v
                    LOAD BALANCER
                           |
                           v
                   NETFLIX BACKEND
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     User Service      Catalog Service   Search Service
                           |
                           v
                  Recommendation Service
                           |
                    +------+------+
                    |             |
                    v             v
                DATABASE        CACHE
                                    
                           |
                           v
                    PLAYBACK SERVICE
                           |
                           v
                    VIDEO STORAGE
                           |
                           v
                          CDN
                           |
                           v
                         USER
```

---

# 📂 Project Structure

```text
Netflix-System-Design/
│
├── README.md
│
└── netflix_architecture_like_reference.drawio
```

### `README.md`

Contains the documentation and explanation of the Netflix system design.

### `netflix_architecture_like_reference.drawio`

Contains the complete system architecture diagram created using draw.io / diagrams.net.

---

# 🛠️ Tools Used

- **draw.io / diagrams.net** — System architecture diagram
- **GitHub** — Repository and documentation
- **Markdown** — Project documentation

---

# 🎯 Learning Objectives

This project demonstrates the following system design concepts:

- Client-server architecture
- DNS
- Load balancing
- Backend services
- Microservices
- Database design
- Caching
- CDN
- Video storage
- Video streaming
- Adaptive video quality
- Recommendation systems
- Thumbnail selection
- Horizontal scaling
- Database replication
- Database sharding
- Capacity estimation
- Fault tolerance
- High availability
- Security

---

# 🎤 Viva Summary

The overall flow of the Netflix system can be explained as:

```text
User
 ↓
DNS
 ↓
Load Balancer
 ↓
Backend Services
 ↓
Database / Cache
```

For video playback:

```text
User
 ↓
Playback Service
 ↓
Video Storage
 ↓
CDN
 ↓
User
```

The backend handles **application logic and metadata**, while the database and cache handle application data.

The actual video files are stored separately, and the CDN is responsible for efficiently delivering video segments, thumbnails, and other media to users.

The architecture uses load balancing, caching, CDN, horizontal scaling, and redundant components to support a large number of concurrent users while maintaining availability and performance.

---

# ⭐ Conclusion

The Netflix system design separates different responsibilities into independent components.

The **Load Balancer** distributes application requests, the **Backend Services** handle business logic, the **Database** stores structured application data, the **Cache** provides fast access to frequently requested information, **Video Storage** stores large media files, and the **CDN** delivers video and images efficiently to users.

This architecture allows the system to scale horizontally and handle large amounts of traffic while maintaining reliable and efficient video streaming.
