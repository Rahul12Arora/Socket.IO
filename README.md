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


