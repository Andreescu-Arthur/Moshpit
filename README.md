# Moshpit

Moshpit is a simple photo-sharing mobile app where people can post pictures and short captions, follow friends, and comment on moments. It is designed to be easy to use for non-technical people: open the app, take or select a photo, write a caption, and post. Moshpit works even when the phone is temporarily offline — posts and comments created offline are saved locally and uploaded automatically when the device reconnects. The interface is clean and focused on photos so anyone can start sharing instantly.

---

## Domain details (entities to be persisted)
Below are the application entities that will be stored. Each field includes a description.

### 1) `User`
- **userId** (string / UUID) — unique identifier for the user.
- **username** (string) — public display name.
- **email** (string) — contact email (used for account recovery).
- **profilePictureUrl** (string) — URL pointing to profile photo.
- **bio** (string) — short user bio.
- **createdAt** (datetime) — account creation time.
- **lastSeenAt** (datetime) — last time app synced or used.

### 2) `Post`
- **postId** (string / UUID) — unique post identifier.
- **userId** (string) — the author’s userId.
- **imageUrl** (string) — URL for the uploaded image (or local path if pending).
- **caption** (string) — user-entered text for the post.
- **likesCount** (integer) — cached number of likes.
- **commentsCount** (integer) — cached number of comments.
- **createdAt** (datetime) — time post was created.
- **syncStatus** (enum) — `pending | uploading | synced | failed` (used for offline/queueing).

### 3) `Comment`
- **commentId** (string / UUID) — comment identifier.
- **postId** (string) — parent post id.
- **userId** (string) — who wrote the comment.
- **text** (string) — comment text.
- **createdAt** (datetime) — when comment created.
- **syncStatus** (enum) — `pending | synced | failed`.

### 4) `Like`
- **likeId** (string / UUID) — unique like id.
- **postId** (string) — the post that was liked.
- **userId** (string) — who liked it.
- **createdAt** (datetime) — when like was made.
- **syncStatus** (enum) — `pending | synced | failed`.

### 5) `Notification`
- **notificationId** (string / UUID) — unique id.
- **userId** (string) — recipient user id.
- **message** (string) — "Anna liked your post".
- **seen** (boolean) — whether user viewed the notification.
- **createdAt** (datetime) — timestamp.

---

## CRUD operations (detailed per entity)

### Entity: `Post`
- **Create (Add Post)**  
  - *What the user does*: Taps “+”, picks or takes a photo, writes a caption, taps “Post”.  
  - *What is stored*: A `Post` with `syncStatus = pending` saved in local DB, and a file reference for the image.  
  - *Example input JSON*: `{ "userId":"u123", "caption":"Sunset!", "imageLocalPath":"file://...", "createdAt":"2025-10-01T12:00:00Z" }`  
  - *User feedback*: Immediately show the new post in Feed with a small “uploading” spinner or “Saved locally — uploading”.

- **Read (View Feed)**  
  - *What the user does*: Opens app or pulls to refresh.  
  - *What is stored / shown*: App reads cached posts from local DB and displays them; concurrently attempts to fetch fresh feed from server.  
  - *User feedback*: Shows posts instantly from cache, then replaces/updates from server results when available.

- **Update (Edit caption)**  
  - *What the user does*: Edits caption on a post they own and taps “Save”.  
  - *What is stored*: Updates local `Post` record and sets `syncStatus = pending` if offline. On success `syncStatus = synced`.  
  - *Example input*: `{ "postId":"p123", "caption":"New caption" }`  
  - *User feedback*: “Saved” toast and local post updated; if offline, show “Saved locally — will sync.”

- **Delete (Remove post)**  
  - *What the user does*: Taps delete on their post and confirms.  
  - *What is stored*: Locally mark `deleted = true` or remove the record and queue a delete request.  
  - *User feedback*: Post removed from feed immediately; show “Undo” briefly and “Will delete when online” if offline.

### Entity: `Comment`
- **Create (Add comment)**  
  - User types comment and taps send. Saved locally with `syncStatus = pending`, shown in comment list immediately.  
- **Read**  
  - Comment list loads from local DB; refreshes from server when online.  
- **Update / Delete**  
  - Owner can edit or delete; changes queued if offline.

### Entity: `Like`
- **Create (Like)**  
  - Tap like button. Increment shown locally and queue update to server.  
- **Delete (Unlike)**  
  - Tap again to remove; update locally and queue.

### Entity: `User` and `Notification`
- Typical CRUD: register (create), read profile, edit bio (update), deactivate account (delete). Notifications are created by server and synced to local DB.

---

## Persistence details (local DB vs server)
Which CRUD ops are persisted locally and on the server.

- **Operations persisted locally and on server**:
  1. **Create Post** — saved to local DB immediately; queued and uploaded to server.
  2. **Update Post (edit caption)** — saved locally; update request queued to server.
  3. **Delete Post** — deletion stored locally and queued to delete on server.
  4. **Create Comment** — saved locally and synced to server.
  

- **Read operations**: cached locally (feed and comments) so app can show content offline; server is source of truth when online.

- **Sync policy**:
  - Local writes are enqueued in a local `sync_queue` table with `actionType`, `payload`, `attempts`, `createdAt`.  
  - Background sync worker retries every N seconds while online.  
  - On successful server response, local record is updated (server IDs replaced if needed) and `sync_queue` entry removed.

---

## Offline behavior — separate scenario for each CRUD operation

> The app uses a local DB plus a `sync_queue`. The UI always reads from local DB.

### Create (offline)
- **Scenario**: User creates a new post while offline.
- **What happens**:
  1. App saves the image file locally and inserts a `Post` with `syncStatus = pending` in local DB.
  2. An entry is added to `sync_queue` with action `CREATE_POST` and the local post payload.
  3. UI shows the post immediately in the feed with an “uploading” badge.
  4. When device is online, the sync worker uploads the image, calls `POST /api/posts`, server returns `postId` and imageUrl. App replaces local temp id with server id, updates `syncStatus = synced`, and removes queue item.
- **User-facing message**: “Saved locally — will upload when online.” If upload fails after retries, show “Upload failed — tap to retry”.

### Read (offline)
- **Scenario**: User opens feed with no connectivity.
- **What happens**:
  1. App reads cached posts and comments from local DB.
  2. If cache is empty, app displays “No internet — try again” and a helpful tip to open app while connected once.
- **User-facing message**: Cached content visible; indicator “Offline — using cached posts.”

### Update (offline)
- **Scenario**: User edits a caption while offline.
- **What happens**:
  1. Local `Post.caption` updated and `syncStatus = pending-update`.
  2. A `UPDATE_POST` entry is created in `sync_queue`.
  3. When online the update is processed
### Delete (offline)
- **Scenario**: User deletes their post while offline.
- **What happens**:
  1. Post is removed from feed locally (or flagged `deleted`).
  2. A `DELETE_POST` entry is added to `sync_queue`.
  3. When online, sync worker issues `DELETE /api/posts/:id`. If server reports post already deleted, queue entry removed silently.
- **User-facing message**: “Deleted — will remove from server when online”


