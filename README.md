# Requirements and Goals of the System
## Functional requirement
- One to one chatting
- Online/Sent/Read status
- Persistent storage for char history
## Non-functional Requirements
- Users should have real-time chat experience with minimum latency.
- Our system should be highly consistent; users should be able to see the same chat history on all their devices.
- Messenger’s high availability is desirable; we can tolerate lower availability in the interest of consistency.
## Extended Requirements
- Group Chats: Messenger should support multiple people talking to each other in a group.
- Push notifications: Messenger should be able to notify users of new messages when they are offline.
## Design constraints
### Message retention
- Facebook keeps all the data in the messaging system until user go ahead and delete it
- Whatsapp delete the message as soon as others receives the message
### Message security 
- Facebook can read the message until you turn on end to end message encryption
- Whatsapp has end to end message encryption i.e. middle tier can't read your message
### Traffic 
- Daily active users: 500 million
- Each user sends 40 messages daily
- Expected service response: 100ms
### Whatsapp Statistics
- 2 billion users
- One billion daily active users
- India is the biggest WhatsApp market in the world, with 200 million users
- 120 million WhatsApp users in Brazil
- US WhatsApp market relatively small, at 23 million
- 65 billion WhatsApp messages sent per day, or 29 million per minute
- Two billion minutes spent making WhatsApp voice and video calls per day
- 55 million WhatsApp video calls made per day, lasting 340 million minutes in total
- 85 billion hours of WhatsApp usage measured May-July 2018
- WhatsApp acquired by Facebook for $19 billion in 2014
![](https://1z1euk35x7oy36s8we4dr6lo-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/WhatsApp-Facebook-users.png)
![](https://1z1euk35x7oy36s8we4dr6lo-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/2019-social-media-users-by-platform.jpg)

# System API
## API to initiate websocket connection
### API Schema
```
POST /init/<userId>?apiKey=string&authToken
Response: 
{
    serverId: int
}
```
Following is schema to store created connection in `connections` table in database
```
{
    userId: int 4 bytes
    heartbeatTime: datetime 8 bytes
    serverId: int 4 bytes
}
```
### Storage estimate
- Daily active users: 500 million
- Number of daily active connection: 500 million
- index on userId and serverId
- Size of document: 12 bytes + 4 * 20bytes + 8 bytes = 100 bytes
- Total size per day: 500 m * 100 bytes = 50GB
### App Cache layer
- Total size per day: 50GB
- Use distributed in memory cache to store all data in cache
### Network bandwidth estimate
- Size of input payload: 4 bytes
- Size of response document: 24 bytes
- Size of request: 28 bytes ~ 30 bytes
- Total response per day: 500m * 30 bytes ~ 15GB
- Network bandwidth per second: 15GB/(24 * 24 * 60 * 60) ~ 173 kb/sec
### App Scale
- Daily active users: 500 million
- Number of daily active connection: 500 million
- Concurrent connections supported by a server: 1000
- Number of servers: 500k
- Number of connections per second: 5787 per sec ~ 6K per second

## API to heartbeat webscoket connection
```
POST /heartbeat/<userId>?apiKey=string&authToken=string
```
It will update heartbeatTime in `connections` table in database
```
{
    userId: int
    serverId: int
    heartbeatTime: datetime
}
```
Note: Same micro services will be used for `init` and `heartbeat` api
![](https://systeminterview.com/imgs/ch12/12-5.png)
![](https://systeminterview.com/imgs/ch12/12-6.png)
## API to create conversation between user1 and user2
```
POST /conversation
{
    user1: int
    user2: int
    encryptionKey: string
}
Response: 
{
    conversationId: int
}
```
Following is schema to store created conversation in `conversations` table in database
```
{
    conversationId: int 4 bytes
    user1: int 4 bytes
    user2: int 4 bytes
    encryptionKey: string 20 bytes
}
```
### Storage estimate
- Daily active users: 500 million
- Number of daily active connection: 500 million
- index on user1 and user2
- Size of document: 32 bytes + 4 * 20bytes + 2 * 4 bytes = 120 bytes
- Total size per day: 500 m * 120 bytes = 60GB
### App Cache layer
- Total size per day: 60GB
- Use distributed in memory cache to store all data in cache

### Network bandwidth estimate
- Size of input payload: 68 bytes ~ 70 bytes
- Size of response document: 24 bytes ~ 25 bytes
- Size of request: 95 bytes ~ 100 bytes
- Total response per day: 500m * 100 bytes ~ 50GB
- Network bandwidth per second: 50GB/(24 * 24 * 60 * 60) ~ 578 kb/sec
### App Scale
- Daily active users: 500 million
- Avg number of pairs: 250 million 
- Concurrent connections supported by a server: 1000
- Number of servers: 250k
- Number of connections per second: 2894 per sec ~ 3K per second
## API to send message 
```
POST /conversation/<conversationId>/message?apiKey=string
{  
    toUserId: int
    message: string,
}
```
Following is schema to store message in `messages` table in database
```
{
    messageId: int 8 bytes
    conversationId: int 4 bytes
    fromUserId: int 4 bytes
    toUserId: int 4 bytes
    message: string, 50 char ~ 100 bytes
    time: timestamp 8 bytes
    delivered: boolean 4 bytes
}
```
### How does the messenger maintain the sequencing of the messages?
- We store a timestamp with each message, which is the time the message is received by the server. 
- This will still not ensure correct ordering of messages for clients.
- The scenario where the server timestamp cannot determine the exact order of messages would look like this:
1. User-1 sends a message M1 to the server for User-2.
2. The server receives M1 at T1.
3. Meanwhile, User-2 sends a message M2 to the server for User-1.
4. The server receives the message M2 at T2, such that T2 > T1.
5. The server sends message M1 to User-2 and M2 to User-1.

So User-1 will see M1 first and then M2, whereas User-2 will see M2 first and then M1.

How to resolve client side message order issue?
- We need to keep a sequence number with every message for each client. 
- This sequence number will determine the exact ordering of messages for EACH user. 
- With this solution, both clients will see a different view of the message sequence, but this view will be consistent for them on all devices.
### Storage estimate
- Daily active users: 500 million
- Each user sends 40 messages daily
- Number of messages daily: 20 billion
- index on fromUserId and toUserId
- Size of document: 132 bytes + 7 * 20bytes + 2 * 4 bytes = 280 bytes ~ 300 bytes
- Total size per day: 20 b * 300 bytes = 6Tb
- 5 years chat history:  10,950tb ~ 11Pb  
- Storage limit per server: 1TB
- Number of shard: 11000
- Number of replicas: 3
- shardKey messageId
- Total storage: 18TB
- CAP? AP system
### Message Queue Storage
- Total size per day: 20 b * 300 bytes = 6Tb
- In 7 days: 42TB
### Network bandwidth estimate
- Size of input payload: 128 bytes ~ 130 bytes
- Size of response document: 24 bytes ~ 25 bytes
- Size of request: 154 bytes ~ 200 bytes
- Total response per day: 500m * 200 bytes ~ 100GB
- Network bandwidth per second: 100GB/(24 * 24 * 60 * 60) ~ 1.2 mb/sec
### App Scale
- Daily active users: 500 million
- Each user sends 40 messages daily
- Number of messages daily: 20 billion
- Concurrent connections supported by a server: 1000
- Number of servers: 20b/1000 ~ 200m
- Number of connections per second: 20b/24*60*60= 231481 per sec ~ 232 million per second
## API to notify user on message delivery
```
POST /conversation/<conversationId>/message/<messageId>/delivery?apiKey=string
{
    toUserId: int,
    time: timestamp
}
```
### Network bandwidth estimate
- Size of input payload: 8 bytes ~ 10 bytes
- Size of response document: 52 bytes ~ 60 bytes
- Size of request: 70 bytes
- Total response per day: 20b * 70 bytes ~ 1400GB
- Network bandwidth per second:  1400GB/(24 * 24 * 60 * 60) ~ 16 gb/sec
### App Scale
- Daily active users: 500 million
- Each user sends 40 messages daily
- Number of messages daily: 20 billion
- Concurrent connections supported by a server: 1000
- Number of servers: 20b/1000 ~ 200m
- Number of connections per second: 20b/24*60*60= 231481 per sec ~ 232 million per second
## Service discovery
- The primary role of service discovery is to recommend the best chat server for a client based on the criteria like geographical location, server capacity, etc. 
- Apache Zookeeper is a popular open-source solution for service discovery. 
- It registers all the available chat servers and picks the best chat server for a client based on predefined criteria.
![](https://systeminterview.com/imgs/ch12/12-11.png)
## API flow
- user1 create websocket connection to server using `/init` api. 
    - Following entry is added to `connections` table
    ```
    {
        userId: 'user1', // primary key
        heartbeatTime: '2020-11-18T15:36:21.117Z'
    }   
    ```
    - At this moment, when user1 is not sending data to server, they will heartbeat to each other using `/heartbeat` api which will keep updating `connections` table
    ```
    {
        userId: 'user1', // primary key
        heartbeatTime: '2020-11-18T15:36:21.117Z'
    }   
    ```    
    - This way server knows that user1 is still there
    - Also, user1 knows that server is still there
- user2 create websocket connection to server using `/init` api. 
    - Following entry is added to `connections` table
    ```
    {
        userId: 'user2', // primary key
        heartbeatTime: '2020-11-18T11:36:21.117Z'
    }   
    ```
    - At this moment, when user2 is not sending data to server, they will heartbeat to each other using `/heartbeat` api which will keep updating `connections` table
    ```
    {
        userId: 'user2', // primary key
        heartbeatTime: '2020-11-18T12:36:21.117Z'
    }   
    ``` 
    - This way server knows that user2 is still there
    - Also, user2 knows that server is still there  
- User keep calling heartbeat every one second to server then server will update it cache.
- If user die then server will stop updating user heartbeat details, other user can figure out last time user1 was online. 
- `user1` start conversation with `user2` using `/conversations` api.
    - This will create following entry in `conversations` table
    ```
    {
        conversationId: 1
        user1: 1
        user2: 2
        encryptionKey: 'rwe34532w4tergetye'
    }   
    ``` 
- `user1` sends message to server asking to deliver to `user2` using `/message` api
    - Server first write message into persistent storage as given below
    ```
        {  
            messageId: 1
            conversationId: 1 
            fromUserId: 1
            toUserId: 2
            message: 'hi',
            time: '2020-11-18T16:36:21.117Z'
            delivered: false
        }    
    ```
    - if target user is online then send messages to connected target user and mark as delivered.
    ```
        {  
            messageId: 1
            conversationId: 1 
            fromUserId: 1
            toUserId: 2
            message: 'hi',
            time: '2020-11-18T16:36:21.117Z'
            delivered: true
        }    
    ```    
    - if target user is not online then do nothing (Based on last heartbeat time)
    - if target user comes online back then check the storage if there is undelivered message then send to target user and mark message as delivered.
- Server will notify user1 that message was delivered by calling `delivery` api

# 1 to 1 chat flow
![](https://systeminterview.com/imgs/ch12/12-12.png)
1. User A sends a chat message to Chat server 1.
2. Chat server 1 obtains a message ID from the ID generator.
3. Chat server 1 sends the message to the message sync queue.
4. The message is stored in a key-value store.
5. a. If User B is online, the message is forwarded to Chat server 2 where User B is connected.
5. b. If User B is offline, a push notification is sent from push notification (PN) servers.
6. Chat server 2 forwards the message to User B. There is a persistent WebSocket connection between User B and Chat server 2.
# Message synchronization across multiple devices
![](https://systeminterview.com/imgs/ch12/12-13.png)
- When User A logs in to the chat app with her phone, it establishes a WebSocket connection with `Chat server 1`. 
- Similarly, there is a connection between the laptop and `Chat server 1`.
- Each device maintains a variable called `cur_max_message_id`, which keeps track of the latest message ID on the device. 
- Messages that satisfy the following two conditions are considered as news messages:
    - The recipient ID is equal to the currently logged-in user ID.
    - Message ID in the key-value store is larger than cur_max_message_id.
# Online presence
![](https://systeminterview.com/imgs/ch12/12-18.png)

# Additional requirement
## How to handle sending images?
- user1 will send encrypted image to `server1`
- `server1` will call `upload` micro service
- `upload` service will store picture in blob storage and get the url back
- `server1` will store url in conversation storage
## Can we optimize storing data?
- Keep only last 7 days active conversation between user in the storage server
- All old data, extract from storage, convert into blob and push to blob storage server
- If user is trying to read 7 days old data then it will retrieve from blob storage and send to user
## Search on chat history
- Search based on text message
- Building inverted index will be compute heavy
## Can we support group based messaging
- Introduce group table
- Add members and conversationId
- Remaining algorithm will be same

# Reference
https://systeminterview.com/design-a-chat-system.php