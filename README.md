# Building HIPAA Compliant End-to-End Encrypted Chat with Stream React Chat and Virgil Security

As healthcare technology advances, easy real-time communication is becoming
increasingly important to any modern healthcare application. It's important to
provide experiences patients expect while protecting their privacy.

In this tutorial we will walk through how to create a full, end-to-end encrypted
chat client using Stream Chat's React widgets combined with Virgil Security. By
combining
[Virgil Security](https://virgilsecurity.com/end-to-end-encrypted-chat) with
[Stream Chat](https://getstream.io/chat/) allows developers to integrate
world-class chat while protecting PHI and patient communications to stay HIPAA
compliant.

## What is end-to-end encryption?

End-to-end encryption means that messages sent between two people can only be
read by them. To do this the message is encrypted before it leaves a user's
device and can only be decryptedby the person you sent it to.

That means we can safely use Stream Chat as a backend by preventing anyone
except the intended parties from reading those messages. No one in your company,
nor any cloud provider you use can read these messages. Even if a malicious
person gained access to the database containing the messages all they would see
is encrypted text, called ciphertext.

Virgil Security is a vendor that allows us to accomplish this via public/private
key technology. Virgil provides a platform and JavaScript SDK that allows us to
securely create, store, and provide best-in-class end-to-end encryption.

# Let's build a HIPAA compliant chat application using React!

To build this application we're going to rely on three libraries,
[Stream React Chat](https://www.npmjs.com/package/stream-chat-react),
[Virgil SDK](https://www.npmjs.com/package/virgil-sdk) and
[Virgil Crypto](https://www.npmjs.com/package/virgil-crypto). We'll combine
these so encryption happens in the browser before sending a message and
decryption and verification happens in the receiver's browser.

The application preforms following steps accomplish full end-to-end encryption:

1. A user authenticates with your backend.
2. User's app requests a Stream auth token and api key from the backend. The
   browser creates a
   [Stream Chat Client](https://getstream.io/chat/docs/#init_and_users) for that
   user.
3. User's app requests a Virgil auth token from the backend and registers with
   Virgil. This generates their private and public key. The private key is
   stored locally and the public key is stored in Virgil.
4. The user decides who they want to chat with and the app creates and joins a
   [Stream Chat Channel](https://getstream.io/chat/docs/#initialize_channel).
5. The app asks Virgil for receiver's public key.
6. The user types a message and sends it to stream. Before sending, the app
   passes the receiver's public key to Virgil to encrypt the message. The
   message is relayed through Stream Chat to the receiver. Stream only stores
   ciphertext, meaning they can never see what was typed.
7. The other user receives and decrypts the message. When the message is
   received, app decrypts the message using using the Virgil SDK. Virgil SDK
   verifies the message is authentic by using the sender's public key.

This looks fairly involved but luckily Stream and Virgil do the heavy lifting
for us. As a developer using these services, we just need to wire them up
correctly.

All source code for this application is available on 
[GitHub](https://github.com/psylinse/stream-encrypted-chat). The code is split
between the react frontend contained in the `frontend` folder and the node.js
express backend is found in `backend`. See the `README.md` in each folder to see
installing and running instructions.

Let's walk through and look at the code needed for each step. 

## Prerequisites

Basic knowledge of React and Node.js is required to follow this tutorial. This
code is intended to run locally on your machine.

You will need an account with [Stream](https://getstream.io/accounts/signup/) 
and [Virgil](https://dashboard.virgilsecurity.com/signup). Once you've created
your accounts, place your credentials in `backend/.env`. You can use 
`backend/.env.example` as a reference for what credentials are required. 

This tutorial uses the following package versions:

* Node 11.14.0
* Yarn 1.17.0
* Stream Chat 0.13.3
* Stream Chat React 0.6.26
* Virgil Crypto 3.2.0
* Virgil SDK 5.3.0
* Virgil e3Kit 0.5.3
* Express 4.17.1

Except for `node` and `yarn`, all of these dependencies are declared in 
`backend/package.json` and `frontend/package.json`. 

## 0. Setup the Backend
In order for our React frontend to interact with Stream and Virgil, the
application provides three endpoints:

* `POST /v1/authenticate`: This endpoint generates an auth token that allows the
  React frontend to communicate with `/v1/stream-credentials` and
  `/v1/virgil-credentials`. To keep things simple, this endpoint simply allows
  the user to simulate and user. The frontend simply tells the backend who it
  wants to authenticate as. If you build off of this code, this would need to be
  rewritten to be a real authentication endpoint.
* `POST /v1/stream-credentials`: This returns the data required for the React
  app to establish a session with Stream. In order return this info we need to
  tell Stream this user exists and ask them to create a valid auth token:
  ```javascript
  // backend/src/controllers/v1/stream-credentials.js
  exports.streamCredentials = async (req, res) => {
      const data = req.body;
      const apiKey = process.env.STREAM_API_KEY;
      const apiSecret = process.env.STREAM_API_SECRET;

      const client = new StreamChat(apiKey, apiSecret);

      const user = Object.assign({}, data, {
          id: `${req.user.identity}`,
          role: 'admin',
          image: `https://robohash.org/${req.user.identity}`,
      });
      const token = client.createToken(user.id);
      await client.updateUsers([user]);
      res.status(200).json({ user, token, apiKey });
  }
  ```
  
  The response payload has this shape:
  ```json
    {
      "apiKey": "<string>",
      "token": "<string>",
      "user": {
        "id": "<string>",
        "role": "<string>",
        "image": "<string>"
      }
    } 
  ```
  * `apiKey` is the stream account identifier for your Stream instance. Needed
    to identify what account your frontend is trying to connect with.
  * `token` JWT token to authorize the frontend with Stream.
  * `user`: This object contains the data that the frontend needs to connect and
    render the user's view.

* `POST /v1/virgil-credentials`: This returns the JWT token used to connect the
  frontend to Virgil. In order for the user to have a valid auth token for
  Virgil, we use the Virgil Crypto SDK to generate this for us: 
  ```javascript
  // backend/src/controllers/v1/virgil-credentials.js
  const virgilCrypto = new VirgilCrypto();
  
  const generator = new JwtGenerator({
  	appId: process.env.VIRGIL_APP_ID,
  	apiKeyId: process.env.VIRGIL_KEY_ID,
  	apiKey: virgilCrypto.importPrivateKey(process.env.VIRGIL_PRIVATE_KEY),
  	accessTokenSigner: new VirgilAccessTokenSigner(virgilCrypto)
  });
  
  
  exports.virgilCredentials = async (req, res) => {
  	const virgilJwtToken = generator.generateToken(req.user.identity);
  
  	res.json({ token: virgilJwtToken.toString() });
  };
  ```
  
## 1. User authenticates With Backend
Now that we have our backend setup and running we can authenticate with the
backend. If you're running the application you'll be presented with a screen
like so:

-- INSERT SCREENSHOT --

This is a simple react form that takes what is typed, stores it in the state as
`identity` and authenticates with that information. Once we have the data from
the form we authenticate against the backend:

```javascript
// frontend/src/StartChat.js
post("http://localhost:8080/v1/authenticate", { identity: this.state.identity })
  .then(res => res.authToken)
  .then(this._connect);
```

Upon creating an identity with an `authToken` we're now ready to connect to
Stream and Virgil.

## 2. User connects to Stream
Using the credentials from step `1.` we can request Stream credentials from the
backend and connect our frontend client to Stream:

```javascript
// frontend/src/StartChat.js
const response = await post("http://localhost:8080/v1/stream-credentials", {}, backendAuthToken);

const client = new StreamChat(response.apiKey);
client.setUser(response.user, response.token);
```

This initializes the `StreamChat` instance we'll need later and tells Stream
what user is connected.

## 3. User connects to Virgil

## 4. Create stream chat channel

## 5. Lookup Virgil public keys

## 6. Sender encrypts message and sends it via Stream

## 7. Receiver decrypts and reads message

# Where to go from here
This tutorial is intended to get you up and running as fast as possible. Because
of this, some critical functionality may be missing for your application. Here
are some tips of what to do next for your app.

* Build real user registration and protect identity registration. This tutorial
  simplified registration and retrieving valid tokens to interact with Stream
  and Virgil.
* Backup user's private keys. Using Virgil's `eThree.backupPrivateKey(pwd)` will
  securely store the private key for restoration on any device.
* Integrate user image and file uploads. This functionality is hidden in this
  app via CSS. You can look at hooking into Stream React Chat's
  [MessageInput](https://getstream.github.io/stream-chat-react/#messageinput) or
  use as a jumping off point to build your own chat widget.

# ? Contact Stream more upselling ?

If you want to get started with Stream and Virgil, contact us here!