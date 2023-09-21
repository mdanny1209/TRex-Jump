# TRex-Jump
TRex-Jump game for telegram bot

You know TRex-Jump? Of course, you already know that. If this is your first time hearing that, then that's because you just don't know the name of that. You have played the game when you can't access site in `Google chrome`? Yeah, the dinosaure appears, you can press `up`(or `space`) and `down` to avoid the obstacles like birds and cactus. Let's play that game in telegram with telegram bot.

Here, I'll describe how to create a telegram bot game with node.js.
This is a step-by-step guide with html5 game open source.

## Requirements
You need to have Node.js installed.
If you didn't install, then [click here](https://nodejs.org/en/download) to download node.js and start installation.


## How to create telegram bot game
### 1. How to create telegram bot
A telegram bot is a special type of user that is not a human but a computer program that can serve companies or brands with many features such as sending out information, reminders, playing tunes, ordering, and more. A bot can post a message in a group or channel.

Here, we'll learn how to make a telegram bot.

Let's start the funny journey. Are you ready? You opened the telegram? If then, let's start.

```
/newbot
```

In order to create a game, we must first create an inline bot. We do this by talking to BotFather and sending the command `/newbot`

BotFather is a bot on Telegram that manages all the bots that you create via your account on Telegram. You can reach him by searching `@BotFather` on Telegram and you should see this profile. To see what he can do, send /start or /help in the chat with BotFather, and you should see a list of commands that he has.

Then, we are asked to enter a name and a username for our bot and we are given an API token. We need to save it as we will need it later.

We can also complete our bot info by changing its description (which will be shown when a user enters a chat with our bot under the “What can this bot do?” section) with

```
/setdescription
```

And also set its picture, in order to make it distinguishable from the chat list. The image must be square and we can set it with the following command:

```
/setuserpic
```

We can also set the about text, which will appear on the bot’s profile page and also when sharing it with other users

```
/setabouttext
```

Our bot has to be inline in order to be able to use it for our game. In order to do this, we simply have to execute the following and follow the instructions

```
/setinline
```

Finally finished creating bot.

### 2. Creating our game

Now that we have our inline bot completely set up, it’s time to ask BotFather to create a game:

```
/newgame
```

We simply follow the instructions and finally we have to specify a `short name` for our game. This will act as a unique identifier for it, which we will `need` later along with our bot API token

### 3. Getting T-Rex game source code
As Chromium is open source, some users have extracted the T-Rex game from it and we can easily find its source code online.

In order to make the game, I have used the code available in this GitHub repo, so go ahead and clone it:

```
git clone https://github.com/wayou/t-rex-runner.git
```

### 4. Setting up dependencies
First, go into the cloned folder and move all its files into a new folder called “public”

```
mkdir public && mv * public/.
```

And init the project

```
npm init
```

You can fill the requested info as you want (you can leave the default values), leave the entry point as index.js

We will need Express and node-telegram-bot-api in order to easily interact with Telegram’s API

```
npm install express --savenpm install node-telegram-bot-api --save
```

We are going to add a start script, since it’s necessary in order to deploy the game to Heroku. Open package.json and add the start script under the scripts section:

```
"scripts": {
"test": "echo \"Error: no test specified\" && exit 1",
"start": "node index.js"
}
```

### 5. Coding our server
Now that we have all dependencies set up, it’s time to code the server for our bot. Go ahead and create the index.js file:

```
const express = require("express");
const path = require("path");
const TelegramBot = require("node-telegram-bot-api");
const TOKEN = "YOUR_API_TOKEN_GOES_HERE";
const server = express();
const bot = new TelegramBot(TOKEN, { polling: true } );
const port = process.env.PORT || 5000;
const gameName = "SHORT_NAME_YOUR_GAME";
const queries = {};
```

The code above is pretty straightforward. We simply require our dependencies and set the token we got from BotFather and also the short name we defined as the game identifier. Also, we set up the port, initialize Express and declare a queries empty object. This will act as a map to store the Telegram user object under his id, in order to retrieve it later.

Next, we need to make the contents of the public directory available as static files

```
server.use(express.static(path.join(__dirname, 'public')));
```

Now we are going to start defining our bot logic. First, let’s code the `/help` command

```
bot.onText(/help/, (msg) => bot.sendMessage(msg.from.id, "This bot implements a T-Rex jumping game. Say /game if you want to play."));
```

We have to specify the command as a regex on the first parameter of onText and then specify the bot’s reply with sendMessage. Note we can access the user id in order to reply by using msg.from.id

When our bot receives the `/start` or `/game` command we are going to send the game to the user using bot.sendGame

```
bot.onText(/start|game/, (msg) => bot.sendGame(msg.from.id, gameName));
```

Now the user will be shown the game’s title, his high score and a button to play it, but the play button still doesn’t work. So, we are going to implement its logic

```
bot.on("callback_query", function (query) {
  if (query.game_short_name !== gameName) {
    bot.answerCallbackQuery(query.id, "Sorry, '" + query.game_short_name + "' is not available.");
  } else {
    queries[query.id] = query;
    let gameurl = "https://YOUR_URL_HERE/index.html?  id="+query.id;
    bot.answerCallbackQuery({
      callback_query_id: query.id,
      url: gameurl
    });
  }
});
```

When the user clicks the play button Telegram sends us a callback. In the code above when we receive this callback first we check that the requested game is, in fact, our game, and if not we show an error to the user.

If all is correct, we store the query into the queries object defined earlier under its id, in order to retrieve it later to set the high score if necessary. Then we need to answer the callback by providing the game’s URL. Later we are going to upload it to Heroku so you’ll have to enter the URL here. Note that I’m passing the id as a query parameter in the URL, in order to be able to set a high score.

Right now we have a fully functional game but we still are missing high scores and inline behavior. Let’s start with implementing inline and offering our game:

```
bot.on("inline_query", function(iq) {
  bot.answerInlineQuery(iq.id, [ { type: "game", id: "0", game_short_name: gameName } ] );
});
```

Last, we are going to implement the high score logic:

```
server.get("/highscore/:score", function(req, res, next) {
  if (!Object.hasOwnProperty.call(queries, req.query.id)) return   next();
  let query = queries[req.query.id];
  let options;
  if (query.message) {
    options = {
      chat_id: query.message.chat.id,
      message_id: query.message.message_id
    };
  } else {
    options = {
      inline_message_id: query.inline_message_id
    };
  }
  bot.setGameScore(query.from.id, parseInt(req.params.score),  options,
  function (err, result) {});
});
```

In the code above, we listen for URLs like `/highscore/300?id=5721`. We simply retrieve the user from the queries object given its id (if it exists) and the use bot.setGameScore to send the high score to Telegram. The options object is different if the user is calling the bot inline or not, so we check both situations as defined in the Telegram Bot API

The last thing we have to do on our server is to simply listen in the previously defined port:

```
server.listen(port);
```

### 6. Modifying T-Rex game
We have to modify the T-Rex game we cloned from the GitHub repo in order for it to send the high score to our server.

Open the `index.js` file under the public folder, and at the top of it add the following lines in order to retrieve the player id from the url:

```
var url = new URL(location.href);
var playerid = url.searchParams.get("id");
```

Last, we are going to locate the `setHighScore` function and add the following code to the end of it, in order to submit the high score to our server:

```
// Submit highscore to Telegram
var xmlhttp = new XMLHttpRequest();
var url = "https://YOUR_URL_HERE/highscore/" + distance  +
"?id=" + playerid;
xmlhttp.open("GET", url, true);
xmlhttp.send();
```

### 7. Deploying to CloudFlare

Our game is complete, but without uploading it to a server we can’t test it on Telegram, and `CloudFlare` provides us a very straightforward way to upload it.

Login or sign up if you have no account in `CloudFlare`

So what can you see now after login?
There's a side bar in the left of the page and there, you can find `Workers & Pages` section. Select it.

After that, you can find the `Create application` button in the main page. Of course, the next step is clicking the button.

In the `Create an application` page, click the `Pages` tab and click `Upload assets` button.

Then you can see the `Deploy a site by uploading your project` page.

There, the site provide the domain for your project. If you type the project name like `trex-jump`, then you can see  `trex-jump-6ok.pages.dev` domain which is generated. That is the domain for our html5 game.

Upload your project assets which located in `public` folder.

After clicking `Deploy site` button, then you can access the game page with the generated domain.

Change our URL placeholders with the actual URL (replace with your own):

Replace the URL with the setHighScore function

```
var url = "https://trex-jump.pages.dev/highscore/" + distance +
"?id=" + playerid;
```

And also on the callback on the server:

```
let gameurl = "https://trex-jump.pages.dev/index.html?id="+query.id;
```

Finally, let’s start our node server:

```
npm start
```

After that, you can start the game by sending `/start` in the bot you generated.

### 8. Share the link to friends

Now, it's finished, the remain is to share the game to your friends. Of course, you can do that by sending the bot name to your friends or invite the bot to the group channels and type the command. 

Another way to share this game? Yes. Where there's a will, there's a way.

In this part, I'll describe how to share the game to your friend directly.

```
<button onclick="TelegramGameProxy.shareScore()">Share</button>
```

Add this code inside the `<body>` of `index.html`.
(If you already deployed the asset, then you'll be needed to modify it)

After that, you can see the `Share` button and when you click that, then you can send message that involves the game to your friends.

## Conclusion

So the lesson is finished. It's funny and simple, isn't it?
Anyone can try even if he/she doesn't know how to code. Just copy and paste the code in the lesson.

If you have some problem while doing that, you can refer my code in the repository.

If you want more help, then please contact me.

- Gmail     : mdanny1209@gmail.com
- Discord   : danny1209
- Telegram  : mdanny1122
