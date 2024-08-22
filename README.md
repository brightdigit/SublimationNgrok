# SublimationNgrok

Share your local development server easily with your Apple devices via Ngrok.

```mermaid
sequenceDiagram
    participant DevServer as Development Server
    participant Sub as Sublimation (Server)
    participant Ngrok as Ngrok (https://ngrok.com)
    participant KVdb as KVdb (https://kvdb.io)
    participant SubClient as Sublimation (Client)
    participant App as iOS/watchOS App
    
    DevServer->>Sub: Start development server
    Sub->>Ngrok: Request public URL
    Ngrok-->>Sub: Provide public URL<br/>(https://abc123.ngrok.io)
    Sub->>KVdb: Store URL with bucket and key<br/>(bucket: "fdjf9012k20cv", key: "dev-server",<br/>url: https://abc123.ngrok.io)
    App->>SubClient: Request server URL<br/>(bucket: "fdjf9012k20cv", key: "dev-server")
    SubClient->>KVdb: Request URL<br/>(bucket: "fdjf9012k20cv", key: "dev-server")
    KVdb-->>SubClient: Provide stored URL<br/>(https://abc123.ngrok.io)
    SubClient-->>App: Return server URL<br/>(https://abc123.ngrok.io)
    App->>Ngrok: Connect to development server<br/>(https://abc123.ngrok.io)
    Ngrok->>DevServer: Forward request to local server
```

## Overview

Ngrok is a fantastic service for setting up local development server for outside access. Let's say you need to share your local development server because you're testing on an actual device which can't access your machine via your local network. You can run `ngrok` to setup an https address which tunnels to your local development server:

```bash
> vapor run serve -p 1337
> ngrok http 1337
```
Now you'll get a message saying your vapor app is served through ngrok:

```
Forwarding https://c633-2600-1702-4050-7d30-cc59-3ffb-effa-6719.ngrok.io -> http://localhost:1337 
```

With Sublimation you save the address (such as `https://c633-2600-1702-4050-7d30-cc59-3ffb-effa-6719.ngrok.io`) to a key-value storage and pull that address from your Apple device during development.

### Cloud Setup

If you haven't already setup an account with ngrok and install the command-line tool via homebrew. Next let's setup a key-value storage with kvdb.io which is currently supported. _If you have another service, please create an issue in the repo. Your feedback is helpful._ 

Sign up at kvdb.io and get a bucket name you'll use. You'll be using that for your setup. Essentially there are three components you'll need:

* **ngrok executable path**
    - if you installed via homebrew it's `/opt/homebrew/bin/ngrok` but you can find out using: `which ngrok` after installation
* your kvdb.io **bucket name**
* your kvdb.io **key**
    - you just need to pick something unique for your server and client to use

Save these somewhere in your shared configuration for both your server and client to access, such as an `enum`:

```swift
public enum SublimationConfiguration {
  public static let bucketName = "fdjf9012k20cv"
  public static let key = "my-"
}
```

### Server Setup

When creating your `Sublimation` object you'll want to use the provided convenience initializers `TunnelSublimatory.init(ngrokPath:bucketName:key:application:isConnectionRefused:ngrokClient:)` to make it easier for **ngrok** integration with the `TunnelSublimatory`:

```swift
let tunnelSublimatory = TunnelSublimatory(
  ngrokPath: "/opt/homebrew/bin/ngrok", // path to ngrok executable
  bucketName: SublimationConfiguration.bucketName, // "fdjf9012k20cv"
  key: SublimationConfiguration.key, // "dev-server"
  application: { myVaporApplication }, // pass your Vapor.Application here
  isConnectionRefused: {$.isConnectionRefused}, // supplied by `SublimationVapor`
  transport: AsyncHTTPClientTransport() // ClientTransport for Vapor
)

let sublimation = Sublimation(sublimatory: tunnelSublimatory)
```

### Client Setup

For the client, you'll need to import the `SublimationKVdb` module and retrieve the url via:

```swift
import SublimationKVdb

let hostURL = try await KVdb.url(
  withKey: SublimationConfiguration.key, 
  atBucket: SublimationConfiguration.bucketName
) 
```