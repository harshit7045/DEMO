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

### 1️⃣ Naive Interval-Based Tracking (Basic Idea)
This approach logs every play and pause action and stores time intervals.

#### ✅ Pros
- Highly accurate
- Can merge overlapping intervals

#### ❌ Cons
- High overhead for merging time intervals
- Repeated comparisons
- Many reads/writes to database

#### 🧪 Example
```
User plays from 2s–5s and 10s–15s.
Then plays from 4s–11s (overlaps both).
Merged view time = 2s to 15s → 13s total unique view time.
```

---

### 2️⃣ Disjoint Set (Optimized Interval Merging)
Use graph-style merging of watched intervals using Union-Find (DSU).

#### ✅ Pros
- Less space overhead
- Efficient merging

#### ❌ Cons
- Needs more code
- Still similar complexity on the backend
- Not natively supported by frontend events

---

### 3️⃣ Goated Approach: WebSocket + Redis (Real-time at Scale)
A scalable solution for handling large users in real time.

#### 🔧 Implementation Idea:
- Frontend sends watch info every second using WebSocket.
- Redis buffers these updates to avoid bombarding the database.
- Data is flushed in intervals.

#### ✅ Pros
- Super scalable
- Great for real-time dashboards

#### ❌ Cons
- Overkill for an assignment
- Needs Redis, Socket.io, batching logic

---

### 4️⃣ ✅ Final Implemented Approach: Hashing-Based Segment Tracking
This approach is implemented in this project.

### 🧩 How It Works:
- Divide the video into 5-second segments.
- Store a binary array `segmentsWatchedArrayHash`.
- Each index represents a 5s chunk of the video.
- On crossing a segment boundary, that segment is marked as watched (set to 1).

#### 🧮 Example:
```js
Video duration = 60 seconds
Segment size = 5s
Total segments = 60 / 5 = 12
segmentsWatchedArrayHash = [0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0]
```

Only segments 3 and 4 (i.e., 15s–25s) have been watched so far.

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

This function triggers every time the video updates. It only sends a new segment update if:
- The video is currently playing
- The time crosses a 5s boundary

---

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

This backend function:
- Receives watched segment(s)
- Updates segment hash array
- Updates `lastWatchTime`
- Saves the user object

---

## 📊 UI – Progress Visualization
- Frontend maps the segment hash into a colored progress bar:
```js
{viewedSegments.map((segment, index) => (
  <div
    key={index}
    className={`h-full ${segment === 1 ? 'bg-green-500' : 'bg-red-500'}`}
    style={{ width: `${100 / viewedSegments.length}%` }}
  />
))}
```

- Green bar = watched (1)
- Red bar = not watched (0)

---

## 📌 API Endpoints Summary
| Endpoint                        | Method | Description                       |
|--------------------------------|--------|-----------------------------------|
| `/auth/updateUserViewVideo`    | POST   | Update watched segments           |
| `/auth/getUserViewedSegments`  | POST   | Fetch hash of segments watched    |
| `/auth/getlastWatchedTime`     | POST   | Fetch time for video resume       |

---



```

This is now saved in the canvas. Let me know if you want to export it as a downloadable file or convert it to PDF format for upload/submission.

