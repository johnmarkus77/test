
## Storing messages in our database

Storing messages in our database:

Everytime a user sends a message to our LLM, we will store the following in our conversation table inside database:

    1. user_id(or some unique identifier)
    2. message_content(the actual messages between user and llm)
    3. time
    4- ChabotId: The chatbot ID this message belongs to
    5. message_position (to build the chain of message histories)
    5. role (LLM or user)
    6. metadata: (Grade for the user and any othe information that comes up)


This will all be stored in the Conversations table inside our database, which has the following schema:
```
TABLE Conversation (
    user_id: INT
    message_content: TEXT (SQLs string type)
    time: TIMESTAMP
    message_position: INT
    chatbot_Id: INT
    role ENUM('LLM', 'user')
    metadata: (Grade and any other information we may need that we have not accounted for that might come up later, this is just a buffer in case we need more basically)
)
```

So essentially anytime a user or the LLM responds, we will store the message into the Conversations table in its own entry.

This might be the state of the database after a simple convo like the following:

Lets assume **John is userId 1** and he clicked on the **chatbot ID 4343** which is about Vegetables

```
John: Hello, I want to learn about Tomatoes.

LLM: Tomatoes are a great food with lots of nutrients

John: I want to grow my own tomatoes (grammar error here)

LLM: That is great, you made a small grammar error here, you can correct it by using xyz....And tomatoes are a great plan to grow!
```


####  Database state

| user_id | chatbot_id | message_content                                                                          | time                 | message_position | role | metadata          |
|---------|------------|-----------------------------------------------------------------------------------------|----------------------|------------------|------|-------------------|
| 1       | 4343       | Hello, I want to learn about Tomatoes.                                                  | 2024-10-29 10:00:00  | 1                | user |                   |
| 2       | 4343       | Tomatoes are a great food with lots of nutrients                                        | 2024-10-29 10:00:01  | 2                | LLM  |                   |
| 1       | 4343       | I want to grow my own tomatoes (grammar error here)                                     | 2024-10-29 10:00:02  | 3                | user |                   |
| 2       | 4343       | That is great, you made a small grammar error here, you can correct it by using xyz....And tomatoes are a great plan to grow! | 2024-10-29 10:00:03  | 4                | LLM  |                   |


So now we can easily query the database as follows:

Get me all of the messages from userID 1(John) and chatbot 1(Theme is Veggies)
```
SELECT *
FROM Conversation
WHERE user_id = 1 OR chatbot_id = 4343
ORDER BY message_position ASC;
```


This will return an array of the messages in perfect order!
```

[
    {
        "user_id": 1,
        "chatbot_id": 4343,
        "message_content": "Hello, I want to learn about Tomatoes.",
        "time": "2024-10-29T10:00:00Z",
        "message_position": 1,
        "role": "user",
        "metadata": {}
    },
    {
        "user_id": 2,
        "chatbot_id": 4343,
        "message_content": "Tomatoes are a great food with lots of nutrients",
        "time": "2024-10-29T10:00:01Z",
        "message_position": 2,
        "role": "LLM",
        "metadata": {}
    },
    {
        "user_id": 1,
        "chatbot_id": 4343,
        "message_content": "I want to grow my own tomatoes (grammar error here)",
        "time": "2024-10-29T10:00:02Z",
        "message_position": 3,
        "role": "user",
        "metadata": {}
    },
    {
        "user_id": 2,
        "chatbot_id": 4343,
        "message_content": "That is great, you made a small grammar error here, you can correct it by using xyz....And tomatoes are a great plan to grow!",
        "time": "2024-10-29T10:00:03Z",
        "message_position": 4,
        "role": "LLM",
        "metadata": {}
    }
]
```



## Since we store an entire row per message, the breakdown is as follows:
```

    user_id: 4 bytes
    message_content: 2500 bytes ---> 
    
    For simplicity sake lets assume the user sends 500 word long messages each time and the LLM responds with the same amount(which is a crazy amount but just to show an extreme)
    
    2,500 characters per message with each character stored as 1 byte and let us assume each word is on average 5 characters, 500 words would be approx 2,500 characters.
    
    time: 4 bytes
    message_position: 4 bytes
    
    chatbotId (To uniquely identify the messages) 4 bytes
    
    role: 1 byte
    metadata: 100 bytes (Huge estimate just to be on the safe side)
```

This means that everytime the user or the LLM responds we are storing the following amount of data:

**2,617 bytes per message.**


### Rough assumptions on user usage just for illustration purposes(Monthly use)

Let us assume each user has 10 individual chatbots that they are using for their different themes or topics 
and the user sends 1,000 messages to the LLM, and the LLM responds with 1,000 messages:

**Total use per hypothetical user:**

2,000 messages per chatbot

Total messages per user: 10 chatbots * 2,000 messages = 20,000 messages per user.

Storage per user, per month: 20,000 messages * 2,617 bytes per message = 52,340,000 bytes or about 52.34MB per user/month

Let us assume all of UCF is using our app:

52.34 MB per user * 70,000 users = 3,663,800 MB or 3,664 GB per month (3.66 TB)

Amazon's SQL database solution charges $0.125 per GB of storage:

3,660 GB * 0.125 USD/GB = approx $460 per month

### Reality: Amazon give us 20 gigs of storage for free per month:

1. Storage per message: 2,617 bytes per message

2. Total Storage availiable: 20GB = 20,000 MB = 20,000,000 KB =
20,000,000,000 bytes 

3. Max number of messages we could store for free:

20,000,000,000 bytes / 2,617 bytes per messages = **7.5 million messages per month**

7.5 million messages / 70,000 students = each user could on average send 55, 500 word long messages and get 55 LLM responds.

500 word messages * 55 = **each user can send 27,500 words per user per month** assuming we are serving the entire UCF population

(500 words is approx one page of text)






