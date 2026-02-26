# fun-notes

This is simple note taking application, but it does have a CLI support


## Project Idea (Simple prototype not for scaling)

### Backend

- We will have a note room and for every note room we will have a note-room-id
- We will create a different table for note room (#note-room id, note-room name, currently active user, total notes)
- This will be a simple note taking application (#ID, title, body, isImportant, isCompleted, currentIndex, note-room-id)
- We will create a index on the note-room-id as we need to fetch all notes for a note-room
- We will use postgres with persistance for the backend
- We will use a websocket connection to talk to the backend
- We will use redis queue for making web socket stateless
- We will publish the notes to redis queue (note-room-id)

### Frontend

- We will use react for frontend
- At first we will ask user to enter note-room-id to connect, or it can create a new note room with name with auto generated ID
- A table to where we can list all the rooms and thir attributes with websocket connection to see live active users and notes count
- We can swap or drag and drop the notes in the notes list to change its order

### CLI

- Now we will create a CLI using bubble tea to show the notes in CLI
- Drag and drop facility
- We will try to mimic exactly same facility what we have in the frontend



### Improvements

- Local Caching in cli using sqlite and try to sync with the backend
- Local Caching in UI using browser cache and try to sync with the backend
- Conflict resolution if found
- Scaling
- Authentication and Authorization


## Tech Stack

- Golang
- 



## Refferences

- https://stackblitz.com/edit/draggable-mui-list?file=App.tsx