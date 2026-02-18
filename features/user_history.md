# Chat History

The chat history, called sessions, is the concept used to represent the user chat history.

The flow goes as follow:
* user clicks in the front-end "new chat"
* front-end sends a request to the backend to create an ephemeral session (to not polute the backend with empty chat sessions)
* user starts chatting and gets a response
* front-end sends to save the session along with the prompt/responses

* when the user clicks on chat history, the front-end fetches the history (document type session) from the cosmosdb database

# CosmosDB

The chat history needs to be in a container created by you. It uses 2 keys:
`/session_id` (first) and `/entra_oid` (second)

Once created, it can be configured to be used in code using an environment variable called
`COSMOSDB_CHATHISTORY_CONTAINER`.

## chathistory (container)

The container that contains the chat history has 2 types of documents
* session
* message_pair

### session document

The session document type is used to store the title, created date and updated date

Structure:
```json
{
    "id": "sessionId-session. The sessionId is the session id value.",
    "title": "the title of the chat",
    "session_id": "the session id",
    "entra_oid": "the object id of the authentifed user",
    "created_date": "The session created date",
    "updated_date": "The session last updated date",
    "type": "The type of the document, in this case session",
    "version": "The document version, in case changes are made over time"
}
```

### session document

The message_pair document type is used to store the actual prompts/responses

Structure:
```json
{
    "id": "sessionId-index. The sessionId is the session id value. The index represents the order of the pair, starting at 0",
    "session_id": "the session id",
    "entra_oid": "the object id of the authentifed user",
    "prompt": "The session created date",
    "response": "The session last updated date",
    "version": "The document version, in case changes are made over time",
    "type": "The type of the document, in this case message_pair",
}
```
### new parameter
 "COSMOSDB_USERPROFILE_DATABASE_NAME": "UserInformation",
    "COSMOSDB_CHATHISTORY_CONTAINER":"chat_history_session",
    "COSMOSDB_GROUP_MANAGEMENT_CONTAINER":"group_management",