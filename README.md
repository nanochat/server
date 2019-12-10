NanoChat is the perfect solution for those looking for a chat that :

- you can install on your own server
- ensures end-to-end encryption
- is open source
- keeps it simple

[Mattermost](https://mattermost.com/) or [Rocket.Chat](https://rocket.chat/) might be good alternatives otherwise.

# Features
## Backend

- installation requires only one static binary with no dependencies
- data is stored in a [file-based database](https://github.com/etcd-io/bbolt)
- creating users can only be done on the server
- all communications with frontends are encrypted

## Frontend

Frontend features are deliberately limited to :

- edit the server to connect to
- create/import/export private/public keys
- create/delete a conversation with one user or a group of users
- send/edit/delete a message with text and/or a file

# How it works
## Misc

- plain private keys should NEVER be stored locally in frontends. Only their AES encrypted version should. Plain private keys should only be stored in RAM.
- plain private keys should NEVER be sent to the server. Only their AES encrypted version should and ONLY if the user has requested so.
- private key passwords should NEVER be stored locally in frontends. private key passwords should NEVER be sent to the server.
- when receiving an encrypted response from the server, the frontend should ALWAYS validate the `created_at` field
- public/private keys are specific to a server and a username
- random key generation should be STRONG

## Endpoints and models

 See [api.yml](api.yml).

## Encrypted communication with the server

By default, all communications are encrypted and are sent through one of the following 2 options:

- `POST /encrypted`
- `GET /websocket`

Either way, the body to encrypt should always follow the `Body` model. To encrypt the body:

- generate a random key
- AES encrypt the body with the key
- RSA encrypt the key with the target public key then, if available, with its own private key
- send all those information using the `EncryptedBody` model

## Unencrypted communications with the server

Only for a few cases, the communication is not encrypted.

### Getting the server public key

Use `GET /references`

## Create a user

Creating a user must be done on the server using the installation binary. It will create a user in the DB and output a `complete_token` with a limited lifespan. At that point, only the `username` is required.

## Complete a user

The frontend needs to send the `user.complete` body with the `CompleteUser` payload.

Here's a design proposal:

```
Token
+-------------+
|             |
+-------------+

Display name
+-------------+
|             |
+-------------+

Password
+-------------+
|             |
+-------------+

Advanced Mode
+---+
|   |
+---+
```

By default:

- the "Advanced Mode" is disabled
- the user provides the complete token, a display name and a password
- the frontend generates RSA public/private keys
- the frontend AES encrypt the private key with the password
- the frontend stores the encrypted private key locally
- the frontend doesn't send the private key in the message

Checking "Advanced Mode" adds the following design:

```
Store private key on server
+---+
|   |
+---+

Import public/private keys
+-------------+
|    Open     |
+-------------+
```

Checking "Store private key on server" adds the following design:

```
Sync Password
+-------------+
|             |
+-------------+
```

If the "Store private key on server" option is checked, the frontend sends the encrypted private key as well as the sync password in the message.

The "Import" option allows the user to select his own public/private keys files. The frontend still needs to AES encrypt the private key with the password before storing it locally or sending it in the message.

## Login

No login is needed with the server since all communications are encrypted with the user's private key.

However, a login is mandatory in the frontend to decrypt the encrypted private key. It works as follow:

```
Server
+-------------+
|             |
+-------------+

Username
+-------------+
|             |
+-------------+
```

With that information, the frontend tries to retrieve the encrypted private key locally.

If it fails, it adds the following design:

```
Sync Password
+-------------+
|             |
+-------------+
```

The frontend sends a `user.sync` body with the `SyncUser` payload.

The response is a `PrivateKeyUser` payload.

If all is successfull, the frontend adds the following design:

```
Password
+-------------+
|             |
+-------------+
```

The frontend then tries to AES decrypt the encrypted private key with the password.

If the decryption fails, the login failed. If the decryption succeeds, the login succeeded.

## Update a user

The frontend needs to send the `user.update` body with the `UpdateUser` payload.

All fields are optional.

In case the user is updating his public/private keys, the design should be similar to when completing a user.

## Delete a user

The frontend needs to send the `user.delete` body with no payload.

The deletion is immediate and irreversible.

## Create a conversation

The frontend needs to send the `conversation.create` body with the `CreateConversation` payload.

## Update a conversation

The frontend needs to send the `conversation.update` body with the `Conversation` payload.

All fields are optional except the `id`.

## Leave a conversation

The frontend needs to send the `conversation.leave` body with the `BaseConversation` payload.

## Delete a conversation

The frontend needs to send the `conversation.delete` body with the `BaseConversation` payload.

## Get a conversation

The frontend needs to send the `conversation.get` body with the `BaseConversation` payload.

Response is a body with the `MessageConversation` payload.

## List all conversations

The frontend needs to send the `conversation.list` body with no payload.

Response is a body with the `List` payload, each item being a `MessageConversation`

## Create a message

The frontend needs to generate a random key and AES encrypt the text and the file.

Then for each recipient of the conversation, it needs to RSA encrypt the key with its private key and the recipient public key.

Finally the frontend needs to send the `message.create` body with the `CreateMessage` payload.

If `file` is not provided, `text` is mandatory. If `file` is provided, `text` is optional.

## Update a message

Same key manipulation as in `Create`.

The frontend needs to send the `message.update` body with the `UpdateMessage` payload.

If `file` is not provided, `text` is mandatory. If `file` is provided, `text` is optional.

## Delete a message

The frontend needs to send the `message.delete` body with the `BaseMessage` payload.

## Export/Import public/private keys

Frontends should provide a way to export/import public/private keys in a `.json` file that the user should be able to AES encrypt with a password.