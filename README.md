# Socket.IO
Socket IO

<h3>Initialization & Connection</h3>

```
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const bodyparser = require('body-parser');
const passport = require('passport');
const nocache = require('nocache');
const fileUpload = require('express-fileupload');
const session = require('express-session');
const http = require('http');
const socketIo = require('socket.io');

const app = express();

// Configure session middleware
app.use(session({
  secret: 'secret',
  resave: false,
  saveUninitialized: true
}));

const port = process.env.PORT || 5003;

app.engine('html', require('ejs').renderFile);
app.set('view engine', 'html');

//Adding middleware
const corsOptions = {
  origin: ["https://erp.decorpot.com", "https://inventoryerp.decorpot.com", "https://quotationerp.decorpot.com","https://customer.decorpot.com", "http://localhost:3000"],
  methods: 'GET,HEAD,PUT,PATCH,POST,DELETE',
  credentials: true, // Enable credentials (e.g., cookies, authorization headers)
  optionsSuccessStatus: 204, // Some legacy browsers (IE11, various SmartTVs) choke on 204
};

app.use(cors());
app.use(bodyparser.json({ limit: '50mb', extended: true }));
app.use(nocache());
app.use(express.static(__dirname + '/dist/quotationTool/'));
app.use(passport.initialize());
app.use(passport.session());
app.use(fileUpload());

// Initialize routes and other startup scripts
require('./config/passport')(passport);
require('./startup/routes')(app);
require('./startup/db')();
require('./startup/dbView')
require('./startup/redis');
require('./scheduler/setup');

if (process.env.NODE_ENV == 'production') {
    require('./startup/prod')(app);
}

// Create HTTP server and bind it with the Express app
const server = http.createServer(app);

// Attach Socket.IO to the server
const io = socketIo(server, {
  cors: {
    origin: "https://localhost:3000"
  }});

io.on('connection', (socket) => {
    // console.log('A user connected');

    console.log("Socket Id connected",socket.id)

    socket.emit('message',"hello from server dear user");

    socket.on('disconnect', () => {
        console.log('user disconnected');
    });

    //this works
    socket.on('message',(message)=>{
      console.log(`socket with id ${socket.id} sent a message ${message}`);
      socket.broadcast.emit("message", message);
      db.collection('chatMessage', insert(message))
    })
    socket.on('chatMessageDelete')

});

// Listen on the server, not the app
server.listen(port, () => {
    console.log('Server started at port ' + port);
});
```

<h3>Schema For Chat App</h3>

```
"use strict";
const mongoose = require("mongoose");
const Schema = mongoose.Schema;
const constants = require("../utils/constants");

const ChatMessageSchema = new Schema({
  //Ticket Identifiers
  authorDetails: {
    type: Schema.Types.ObjectId,
    ref: "User",
  },
  DMdetails:{
    type: String,
  },

  // Details about this chat was part of which group
  GroupDetails: {
    type: Schema.Types.ObjectId,
    ref: "chatGroup",
  },
  messageBody: {
    type: String,
  },
});

ChatMessageSchema.plugin(require("mongoose-timestamp"));
ChatMessageSchema.plugin(require("mongoose-delete"), {
  overrideMethods: true, 
  deletedAt: true,
});

module.exports = mongoose.model("ChatMessages", ChatMessageSchema);
```

```
"use strict";
const mongoose = require("mongoose");
const Schema = mongoose.Schema;
const constants = require("../utils/constants");

const ChatGroupSchema = new Schema({
  //Ticket Identifiers
  groupTitle: {
    type: String,
    required: false,
  },
  groupDp: {
    type: String,
  },
  groupMemberDetails: [
    {
        memberId:{
            type: Schema.Types.ObjectId,
            ref: "User",
        },
        joinedOn: {
            type: Date,
        },
    }
  ],
  //Tickte Specific Input Info
  groupCreatedOn: {
    type: String,
    required: false,
  },
  groupCreatedBy: {
    type: String,
    required: false,
  },
  groupAdmins:[
    {
        type: Schema.Types.ObjectId,
        ref: "User",
    }
  ],
});

ChatGroupSchema.plugin(require("mongoose-timestamp"));
ChatGroupSchema.plugin(require("mongoose-delete"), {
  overrideMethods: true,
  deletedAt: true,
});

module.exports = mongoose.model("ChatGroup", ChatGroupSchema);
```

Client-Side/FrontEnd

**Socket.js**

```
import { io } from 'socket.io-client';

// "undefined" means the URL will be computed from the `window.location` object
const URL = process.env.NODE_ENV === 'production' ? 'http://localhost:5003' : 'http://localhost:5003';

export const socket = io(URL);
```


```
import React, { useState, useEffect } from 'react';
import { socket } from '../socket';

export function SocketMessage() {
  const [value, setValue] = useState('');
  const [recievedMessages, setRecievedMessages] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {    
    socket.on('connect', () => {
      console.log('Client is Connected to server');
    });

    socket.on('disconnect', () => {
      console.log('Client Disconnected from server');
    });

    // socket.on('message',(message)=>{
    //   setRecievedMessages([...recievedMessages, message]);
    //   console.log('message recieved from server : ',message)
    // })

    socket.on('message', (message) => {
      setRecievedMessages(prevMessages => [...prevMessages, message]);
      console.log('Message received from server:', message);
    });

    // Clean up the socket connection when the component unmounts
    return () => socket.disconnect();
  }, []);

  //send message
  function onSubmit(event) {
    event.preventDefault();
    setIsLoading(true);

    socket.timeout(5000).emit('message', value, () => {
      setIsLoading(false);
    });

  }

  //connect and disconnect
  function connect() {
    socket.connect();
  }

  function disconnect() {
    socket.disconnect();
  }


  return (
    <>
    <form onSubmit={ onSubmit }>
      <input onChange={ e => setValue(e.target.value) } />

      <button type="submit" disabled={ isLoading }>Submit</button>
    </form>

    <div>
      <button onClick={ connect }>Connect</button>
      <button onClick={ disconnect }>Disconnect</button>
    </div>

    <div>
      {recievedMessages.length>0 && recievedMessages.map((el)=>{
        return <div>{el}</div>
      })}

    </div>

    </>
  );
}
```
