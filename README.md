

# 📘 README – Video Progress Tracker for SDE Intern Assignment

## 📌 Overview
This project is a full-stack video lecture tracking system built for the **SDE Intern Assignment**. It enables accurate measurement of how much of a lecture video a user has truly watched. Unlike traditional methods that mark a video as completed simply because it ended, this system tracks which parts of the video were **uniquely viewed** by the user.

---

## 🎯 Objective
- Track **real user progress** by identifying **unique segments** watched.
- **Resume** from the last watched time even after logout or refresh.
- Prevent cheating by skipping or repeatedly watching the same part.
- Store and visually present user engagement using segment-based progress.

---

## 🏗️ System Setup

### 🔧 Prerequisites
- Node.js (v16+)
- MongoDB (for persistent storage)
- Redis (optional, for scale optimization)

### ⚙️ Installation
```bash
# Clone the repository
git clone <your-repo-link>

# Backend setup
cd backend
npm install
npm run dev

# Frontend setup (new terminal)
cd frontend
npm install
npm run dev
```

> Set `VITE_API_BASE_URL` in the `.env` file to point to your backend, e.g., `http://localhost:4002/api`

---

## 🧠 Approaches Considered

### 1⃣ Naive Interval-Based Tracking (Basic Idea)
This approach logs every play and pause action and stores time intervals.

#### ✅ Pros
- **Highly accurate**: Tracks every viewable segment and ensures precise progress measurement.
- **Can merge overlapping intervals**: Multiple play intervals can be combined to avoid redundant storage of duplicate segments.

#### ❌ Cons
- **High overhead for merging intervals**: Merging intervals can be computationally expensive and lead to increased processing time, especially when dealing with large datasets.
- **Repeated comparisons**: For each new play, the system must compare the new intervals with previous ones to merge them.
- **Frequent database writes**: This can lead to performance issues due to constant interaction with the database.

#### 🧪 Example
```
User plays from 2s–5s and 10s–15s.
Then plays from 4s–11s (overlaps both).
Merged view time = 2s to 15s → 13s total unique view time.
```

---

### 2⃣ Disjoint Set (Optimized Interval Merging)
This approach uses a graph-style merging of watched intervals through the **Union-Find** (Disjoint Set Union, DSU) data structure.

#### ✅ Pros
- **Less space overhead**: The DSU reduces the amount of data stored, as it effectively handles intervals by grouping them.
- **Efficient merging**: Efficiently merges overlapping intervals without redundant data.

#### ❌ Cons
- **Complex code**: Implementing the DSU logic can be complicated and requires extra coding effort.
- **Backend complexity**: The backend logic would need to handle interval merging, which adds more complexity.
- **Frontend challenges**: Integrating the DSU logic with frontend events, like video plays, could be cumbersome.

Given that the project is intended to be completed within 2 days, this approach is **too cumbersome** to implement in the given timeframe, so we decided to use a simpler approach.

---

### 3⃣ WebSocket + Redis (Real-time at Scale)
A scalable solution for handling large users in real-time using **WebSockets** and **Redis**.

#### 🔧 Implementation Idea:
- The frontend sends watch info every second using **WebSocket**.
- **Redis** buffers these updates to avoid bombarding the database.
- Data is flushed to the database in intervals to maintain real-time tracking.

#### ✅ Pros
- **Super scalable**: This approach is highly efficient for large-scale applications and ensures real-time progress tracking.
- **Real-time dashboards**: Excellent for tracking live progress on user dashboards in real time.

#### ❌ Cons
- **Overkill for this assignment**: The implementation requires a lot of additional setup and is more suited to production-level systems.
- **Requires Redis and Socket.io**: Adding Redis and WebSockets introduces complexity and is outside the scope of a simple assignment.
- **Batching logic needed**: Flushing data in intervals requires careful implementation of batching logic.

Due to the complexity and the limited time available, this approach was not feasible for this project.

---

### 4⃣ ✅ Final Implemented Approach: Hashing-Based Segment Tracking
This is the approach implemented in this project, which divides the video into **fixed-size segments** and uses a **hashing mechanism** to track which segments have been watched.

#### 🧹 How It Works:
- Divide the video into **5-second segments**.
- Store a binary array `segmentsWatchedArrayHash`.
- Each index of the array corresponds to a 5-second chunk of the video.
- When the user crosses a segment boundary, that segment is marked as watched (set to 1).

#### ✅ Pros of Hashing Approach:
- **Low overhead**: By dividing the video into fixed-size chunks and using a binary array, this approach minimizes memory and storage requirements.
- **Simple to implement**: Hashing-based segment tracking is straightforward and easy to code, making it ideal for quick development.
- **Efficient**: The method is computationally efficient, requiring minimal database interaction.
- **Scalable**: It scales well to handle large video files and long-term tracking without performance degradation.
- **Accurate**: It tracks user progress at a granular level, ensuring that users can’t cheat by skipping parts of the video.
- **Real-time updates**: User progress is updated in real-time, enabling accurate tracking even during long sessions.

#### 🧮 Example:
```js
Video duration = 60 seconds
Segment size = 5s
Total segments = 60 / 5 = 12
segmentsWatchedArrayHash = [0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0]
```
In this example, only segments 3 and 4 (15s–25s) have been watched so far.

---

## 🔍 Hashing Implementation Breakdown

### 🧠 Frontend: `getCurrentVideoTime()` Logic
```js
const getCurrentVideoTime = () => {
  if (videoRef.current && isPlaying) {
    const currentSegment = Math.floor(videoRef.current.currentTime / 5);

    if (videoRef.current.currentTime % 5 < 0.5) {
      fetch(`${API_BASE_URL}/auth/updateUserViewVideo`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          videoId: selectedVideo,
          viewedSegments: [currentSegment]
        })
      })
      .then(response => response.json())
      .then(data => {
        if (data.success) {
          console.log(`Segment ${currentSegment} marked as watched`);
        }
      });
    }
  }
};
```

### 🧠 Backend: Segment Update Logic
```js
updateUserViewVideo: async (req, res) => {
  const { videoId, viewedSegments } = req.body;
  const userIdOrEmail = req.user.user;

  const user = await userModel.findById(userIdOrEmail);
  const videoEntry = user.videos.find(v => v.videoId.toString() === videoId);

  viewedSegments.forEach(segmentIndex => {
    if (segmentIndex >= 0 && segmentIndex < videoEntry.segmentsWatchedArrayHash.length) {
      videoEntry.segmentsWatchedArrayHash[segmentIndex] = 1;
      videoEntry.lastWatchTime = segmentIndex * 5;
    }
  });

  await user.save();

  res.status(200).json({
    success: true,
    updatedSegments: videoEntry.segmentsWatchedArrayHash,
    lastWatchTime: videoEntry.lastWatchTime
  });
}
```

---

## 📊 UI – Progress Visualization
- The frontend maps the segment hash into a colored progress bar:
```js
{viewedSegments.map((segment, index) => (
  <div 
    key={index}
    className={`h-full ${segment === 1 ? 'bg-green-500' : 'bg-red-500'}`}
    style={{ width: `${100 / viewedSegments.length}%` }}
  />
))}
```

- **Green bar** = watched (1)
- **Red bar** = not watched (0)

### 📸 Segment Progress Bar Screenshot
![Segment Progress Bar](https://drive.google.com/uc?export=view&id=1F6ZVsdwuCzrDKqCpa8hrTDVkoOAe6rYX)  
**Legend**: Green represents watched parts of the video, while red represents unwatched parts.

---

## 📌 API Endpoints Summary
| Endpoint                        | Method | Description                       |
|--------------------------------|--------|-----------------------------------|
| `/auth/updateUserViewVideo`    | POST   | Update watched segments           |
| `/auth/getUserViewedSegments`  | POST   | Fetch hash of segments watched    |
| `/auth/getlastWatchedTime`     | POST   | Fetch time for video resume       |

---

