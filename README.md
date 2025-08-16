# WhatsApp Server

This project is a WhatsApp server built in Go using the powerful `whatsmeow` library. It acts as a bridge between the WhatsApp platform and your own applications, allowing you to programmatically send and receive messages, handle media, and store conversation history. It exposes a simple REST API for easy integration with any programming language or system.

This server is ideal for building chatbots, notification systems, or any application that requires interaction with WhatsApp users.

## Features

- **Connect to WhatsApp**: Establishes a persistent connection to the WhatsApp Web API by scanning a QR code. Session information is stored locally, so you only need to authenticate once.
- **Send and Receive Messages**: Handles both incoming and outgoing messages in real-time. It supports text messages as well as various media types.
- **Message Storage**: All incoming and outgoing messages are archived in a local SQLite database (`messages.db`), allowing for conversation history retrieval and analysis.
- **Media Handling**: Automatically downloads media (images, videos, audio, documents) from incoming messages and saves them to the `store` directory. It also supports sending media files via the API.
- **REST API**: A clean and simple REST API is exposed on port `8080` for sending messages and downloading media, making it easy to integrate with other services.
- **Status Updates**: The server can listen for and store status updates from your contacts, saving them to the database and downloading any associated media.
- **History Sync**: On initial connection, it can sync a portion of your recent chat history from your phone.
- **Automatic Media Downloading**: Any incoming message containing media (images, videos, audio, documents) will have its content automatically downloaded and saved to the `store` directory, organized by chat.

## How It Works

The application follows a straightforward architecture:

1.  **Initialization**: On startup, it initializes a `whatsmeow` client and connects to the WhatsApp Web servers.
2.  **Authentication**: If it's the first run, it generates a QR code for you to scan with your phone. On subsequent runs, it uses the stored session data in `store/whatsapp.db` to reconnect automatically.
3.  **Event Handling**: The server listens for events from WhatsApp, such as incoming messages, presence updates, and history sync data. An event handler processes these events.
4.  **Database Interaction**: Incoming messages and chat information are parsed and stored in the `messages.db` SQLite database.
5.  **Media Handling**: When a message with media is received, the server automatically triggers a download. The media is saved to a directory corresponding to the chat JID (e.g., `store/1234567890@s.whatsapp.net/`).
6.  **REST API Server**: An HTTP server runs concurrently, listening for API requests on port `8080`. API handlers interact with the `whatsmeow` client to send messages or trigger media downloads.

## Automatic Media Downloading

The server is configured to automatically download all media from incoming messages. This feature is handled in the background and requires no manual intervention.

- **Storage Location**: Media files are saved in the `store` directory. Inside `store`, a subdirectory is created for each chat JID. For example, media from the chat with `1234567890@s.whatsapp.net` will be saved in `store/1234567890@s.whatsapp.net/`.
- **Status Media**: Media from status updates are also downloaded and stored in a special directory: `store/statuses/<sender_jid>/`.
- **Manual Downloads**: While downloads are automatic, the `/api/download` endpoint is still available if you need to re-download a file or verify its path.

## Prerequisites

- [Go](https://golang.org/doc/install) (version 1.18 or higher) installed on your system.

## Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/Okramjimmy/whatsapp_server.git
    cd whatsapp_server
    ```
    *(Replace `your-username` with the actual repository owner's username if you forked it).*

2.  **Install dependencies:**
    This command will download all the necessary libraries defined in the `go.mod` file.
    ```bash
    go mod tidy
    ```

## Running the Application

1.  **Run the server:**
    ```bash
    go run main.go
    ```

2.  **Scan the QR Code:**
    On the first launch, a QR code will be rendered in your terminal. To link the server to your WhatsApp account:
    - Open WhatsApp on your phone.
    - Go to **Settings** > **Linked Devices**.
    - Tap on **Link a Device** and scan the QR code shown in the terminal.

    Once the connection is successful, you will see a "Successfully connected and authenticated!" message. The server is now running and will start processing messages.

## API Usage

The server exposes a REST API on `http://localhost:8080`.

### Send a Message

- **Endpoint**: `POST /api/send`
- **Description**: Sends a text message or a media file to a specified recipient.
- **Request Body**:
  ```json
  {
    "recipient": "1234567890", // Phone number (without '+') or a full JID (e.g., "1234567890@s.whatsapp.net")
    "message": "Hello, this is a test message!", // Caption for media or the text message
    "media_path": "/path/to/your/media.jpg" // Optional: absolute path to a media file on the server's filesystem
  }
  ```
- **Example using `curl`**:
  ```bash
  curl -X POST -H "Content-Type: application/json" -d '{
    "recipient": "1234567890",
    "message": "Check out this image!",
    "media_path": "/Users/Desktop/Pictures/cat.jpg"
  }' http://localhost:8080/api/send
  ```
- **Success Response (`200 OK`)**:
  ```json
  {
    "success": true,
    "message": "Message sent to 1234567890"
  }
  ```
- **Error Response (`500 Internal Server Error`)**:
  ```json
  {
    "success": false,
    "message": "Error sending message: <error_details>"
  }
  ```

### Download Media

- **Endpoint**: `POST /api/download`
- **Description**: Downloads the media attachment from a specific message.
- **Request Body**:
  ```json
  {
    "message_id": "MESSAGE_ID_FROM_DB",
    "chat_jid": "CHAT_JID_FROM_DB" // e.g., "1234567890@s.whatsapp.net"
  }
  ```
- **Example using `curl`**:
  ```bash
  curl -X POST -H "Content-Type: application/json" -d '{
    "message_id": "ABCDEFGHIJKL",
    "chat_jid": "1234567890@s.whatsapp.net"
  }' http://localhost:8080/api/download
  ```
- **Success Response (`200 OK`)**:
  ```json
  {
    "success": true,
    "message": "Successfully downloaded image media",
    "filename": "image_20231027_123456.jpg",
    "path": "/Users/Desktop/Projects/whatsapp_server/store/1234567890@s.whatsapp.net/image_20231027_123456.jpg"
  }
  ```
- **Error Response (`500 Internal Server Error`)**:
  ```json
  {
    "success": false,
    "message": "Failed to download media: <error_details>"
  }
  ```

## Database Schema

The application uses a SQLite database (`store/messages.db`) with two main tables:

### `chats` table
Stores information about each conversation.
- `jid` (TEXT, PRIMARY KEY): The unique JID of the chat (e.g., `1234567890@s.whatsapp.net` or `group-id@g.us`).
- `name` (TEXT): The name of the contact or group.
- `last_message_time` (TIMESTAMP): The timestamp of the last message in the conversation.

### `messages` table
Stores individual messages.
- `id` (TEXT): The unique ID of the message.
- `chat_jid` (TEXT): The JID of the chat this message belongs to (foreign key to `chats.jid`).
- `sender` (TEXT): The sender's JID.
- `content` (TEXT): The text content of the message.
- `timestamp` (TIMESTAMP): When the message was sent.
- `is_from_me` (BOOLEAN): `true` if the message was sent from the connected account.
- `media_type` (TEXT): The type of media (`image`, `video`, `audio`, `document`).
- `filename` (TEXT): The local filename of the downloaded media.
- `url` (TEXT): The direct URL to the media on WhatsApp's servers.
- `media_key` (BLOB): The key required to decrypt the media.
- `file_sha256` (BLOB): The SHA256 hash of the media file.
- `file_enc_sha256` (BLOB): The SHA256 hash of the encrypted media file.
- `file_length` (INTEGER): The length of the file in bytes.

## Project Structure

```
.
├── go.mod              # Go module file defining dependencies
├── go.sum              # Go module checksums
├── main.go             # The main application entry point and logic
└── store/              # Directory for storing persistent data
    ├── messages.db     # SQLite database for storing chats and messages
    └── whatsapp.db     # SQLite database used by whatsmeow to store the session
```

## Configuration

Currently, the configuration is hardcoded in `main.go`. For example, the REST API port is set to `8080`. To change it, you need to modify this line in the `main` function:
```go
startRESTServer(client, messageStore, 8080) // Change 8080 to your desired port
```

## Contributing

Contributions are welcome! If you'd like to improve the project, please feel free to fork the repository and submit a pull request. You can also open an issue to report bugs or suggest new features.

## License

This project is open-source and available under the [MIT License](LICENSE).
