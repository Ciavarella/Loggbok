#### Frontend

##### HTML, CSS, Planning

Today i started with some research of what i am going to need to be able to create the extension. I have a basic idea of what i want to achieve. I know i have to implement the Spotify API so i registered a project for developers on Spotify. After that i started a Github repository and started with a basic template written in HTML and added some CSS.

---

#### Frontend

##### HTML, CSS

Today i worked with some HTML and CSS for the template. The template is just going to make a request to Spotify so the user can sign in with Spotify and i will then get a token to make requests and control the Spotify player.

---

#### Extension

##### Node

Today i started with the extension for Visual Studio Code. To start with the extension i began to setup a basic extension with a tutorial. I then started to change things and added new features. I did this to check that everything worked and that i could run the extension. I added a commands to start the extension.

---

#### Frontend, Backend

##### HTML, CSS, Node

Today i started to work with the backend of the application to implement OAuth with the Spotify API. Implemented a backend with Node and Express. I also added more styling (CSS) to the template site. To authenticate with Spotify the application needed to be deployed so i deployed both the backend and the template to Heroku. I did not choose Heroku for a specific reason, i just wanted to deploy it quickly and i can always change where the application should be deployed.

This is some of the backend code that uses the Spotify API to authenticate.

```javascript
app.get('/callback', (req, res) => {
  let code = req.query.code || null;
  let authOptions = {
    url: 'https://accounts.spotify.com/api/token',
    form: {
      code: code,
      redirect_uri,
      grant_type: 'authorization_code'
    },
    headers: {
      Authorization:
        'Basic ' +
        Buffer.from(
          process.env.SPOTIFY_CLIENT_ID +
            ':' +
            process.env.SPOTIFY_CLIENT_SECRET
        ).toString('base64')
    },
    json: true
  };
  request.post(authOptions, (error, response, body) => {
    const access_token = body.access_token;
    const uri = 'https://mpm-template.herokuapp.com/';
    res.redirect(uri + '?access_token=' + access_token);
  });
});
```

---

#### Extension

##### Node

Today i continued with the extension. I worked alot with how i was going to get the tokens from Spotify and decided that it was best to show a placeholder where the user can enter his / her token. I do not have a way to save the token yet but am planning to that later. I implemented a lot of features such as listening to when keys is pressed, added a counter to show how many seconds of playtime is left, Play / pause request to play and pause the music on Spotify.

Example of some of the code that i added to the extension.

```javascript
/**
 * This method is called when the extension is activated.
 * The initSpotify method is called after a message is presented.
 * @param {vscode.ExtensionContext} context
 */
function activate(context) {
  let disposable = vscode.commands.registerCommand('extension.mpm', function() {
    vscode.window.showInformationMessage('Music Per Minute');
  });

  requestSpotifyAccess();
  showTokenPlaceholder();
  setInterval(decrementCounter, 1000);
  vscode.workspace.onDidChangeTextDocument(checkInput);
  context.subscriptions.push(disposable);
}
exports.activate = activate;

/**
 * Opens the Spotify helper to authenticate access
 */
const requestSpotifyAccess = () => {
  vscode.commands.executeCommand(
    'vscode.open',
    vscode.Uri.parse('https://mpm-template.herokuapp.com/index.html')
  );
};

/**
 * Shows a inputfield where the user enters the Spotify token.
 */
const showTokenPlaceholder = () => {
  vscode.window
    .showInputBox({
      ignoreFocusOut: true,
      prompt: 'Enter your Spotify token here'
    })
    .then(usertoken => {
      token = usertoken;
    });
};
```

---

#### Backend, Frontend

##### Node, JavaScript (Vanilla)

Today i changed some structure and added refreshtoken in the backend. Before i only sent the access token that is required to make request to Spotify. The accesstoken expires after one hour but i also get a refreshtoken that i can use to get a new access token. After i added the refreshtoken together with the access token i needed to change the method that copies the URL parameters in the frontend template.

```javascript
request.post(authOptions, (error, response, body) => {
  const access_token = body.access_token;
  const refresh_token = body.refresh_token;
  const uri = 'https://mpm-template.herokuapp.com/index.html';
  res.redirect(
    uri + '?access_token=' + access_token + '?refresh_token=' + refresh_token
  );
});
```

---

#### Extension, Backend

##### Node

#### Extension

Today i continued with the extension. I thought of some ways so store the access & refreshtokens. I wanted to store them so i can use the extension more than once and not being forced to request a new token each time and paste it in the placeholder. Localstorage is not an option since the extention is not run in the browser. The VS Code API has a similar method that is called globalState where you can store keys and values. This is where i split the url parameters to be one accesstoken and one refreshtoken and then save them in th globalstate.

```javascript
/**
 * Shows a inputfield where the user enters the Spotify token.
 * Set the acccess_token and refresh_token in the globalstate.
 */
const showTokenPlaceholder = () => {
  vscode.window
    .showInputBox({
      ignoreFocusOut: true,
      prompt: 'Enter your Spotify token here'
    })
    .then(usertoken => {
      const str = usertoken;
      let pos = str.indexOf('?refresh_token=');
      const refresh_token = str.slice(pos + 15);
      const access_token = str.slice(0, pos);
      token = access_token;
      context.globalState.update('api_key', token);
      context.globalState.update('refresh_key', refresh_token);
      pauseMusic();
    })
    .catch(err => console.log(err));
};
```

#### Backend

I also added route to the backend to handle a refreshtoken. It is used when the accesstoken has expired and i have to request a new accesstoken using a refreshtoken.

```javascript
app.get('/refresh_token', function(req, res) {
  let refresh_token = req.query.refresh_token;
  let authOptions = {
    url: 'https://accounts.spotify.com/api/token',
    form: {
      grant_type: 'refresh_token',
      refresh_token: refresh_token
    },
    headers: {
      Authorization:
        'Basic ' +
        Buffer.from(
          process.env.SPOTIFY_CLIENT_ID +
            ':' +
            process.env.SPOTIFY_CLIENT_SECRET
        ).toString('base64')
    },
    json: true
  };
  request.post(authOptions, function(error, response, body) {
    if (!error && response.statusCode === 200) {
      let access_token = body.access_token;
      res.send({
        access_token: access_token
      });
    }
  });
});
```

---

#### Extension

##### Node

Today i found some problems that i did not think about when starting the project. The play and pause request work fine when i run the extension, however there is a problem with the requests if the Spotify application is not started (obviously). I needed to work out how to check if Spotify is started and if it is not then i want to start it.

I wrote this method to check playback devices and if there are no devices i then run a exec command that starts Spotify for the user.

```javascript
/**
 * Checks if Spotify is open.
 * Checks the OS of the computer and sets a exec command.
 * Exec command starts Spotify.
 */
const checkPlaybackDevice = () => {
  const bearer = 'Bearer ' + token;
  fetch('https://api.spotify.com/v1/me/player/devices', {
    method: 'GET',
    headers: {
      Authorization: bearer
    }
  })
    .then(res => res.json())
    .then(data => {
      if (data.devices[0] == undefined) {
        let os = process.platform;
        let command = '';
        if (os === 'darwin') {
          command = 'open -a spotify';
        } else if (os === 'win32') {
          let windowsUser = getWindowsUser();
          command = `start C:\Users\\${windowsUser}\AppData\Roaming\Spotify\Spotify.exe`;
        } else if (os === 'linux') {
          command = 'spotify';
        }
        cp.exec(command, (err, stdout, stderr) => {
          if (err) {
            return;
          }
          checkPlaybackDevice();
        });
      } else {
        let deviceId = data.devices[0].id;
        activateSpotify(deviceId);
      }
    });
};

/**
 * Gets the username of a Windows user.
 */
const getWindowsUser = async () => {
  return await username();
};
```

When Spotify is started it is not active so i have to initiate it, this is so the play & pause request is going to work.

```javascript
/**
 * Initiates Spotify
 */
const activateSpotify = deviceId => {
  const bearer = 'Bearer ' + token;
  let device = {
    device_ids: [deviceId]
  };
  fetch('https://api.spotify.com/v1/me/player', {
    method: 'PUT',
    headers: {
      Authorization: bearer,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(device)
  });
};
```

---

#### Frontend, Backend

##### React, CSS, Node

#### Frontend

Today i started with a dashboard, the dashboard is going to be a webpage where you can login and see statistics of your sessions of the extension. It will also have personal settings that the extension is going to use. I started with setting up a basic structure and a simple view.

```javascript
import React, { Component } from 'react';
import { BrowserRouter as Router, Switch, Route } from 'react-router-dom';
import Start from './Views/Start';

class App extends Component {
  render() {
    return (
      <React.Fragment>
        <Switch>
          <Route exact path='/' component={Start} />
        </Switch>
      </React.Fragment>
    );
  }
}

export default App;
```

I started with a basic view and then added a music visualizer that is shown when the user uploads a music file.

```javascript
import React, { Component } from 'react';
import Header from '../Components/Header';
import '../index.css';
import MusicVisualizer from '../Utils/Music-Visualizer/MusicVisualizer';

class Start extends Component {
  render() {
    return (
      <div className='start-page'>
        <Header />
        <MusicVisualizer />
      </div>
    );
  }
}

export default Start;
```

#### Backend

I also made some changes to the backend to handle both the extension and the dashboard. The approach i went with was to hade different routes in the backend. I added a /auth route to handle the Spotify Authentication.

```javascript
const express = require('express');
const app = express();
const cors = require('cors');
const auth = require('./Routes/auth');
const port = process.env.PORT || 8080;

app.use(cors());
app.get('/', (req, res) => res.send('Nothing here yet'));
app.use('/auth', auth);

app.listen(port, () => console.log(`Express listening to port: ${port}`));

module.exports = app;
```

---

#### Frontend

##### React

Today i restructured my project, im no longer in need of the template that handles the authentication to Spotify. That is unnecessary when the dashboard can handle it on a a separate route. I removed the template and added a route to the dashboard to handle it for me.

```javascript
<Route exact path='/auth/redirect' component={AuthRedirect} />
<Route exact path='/auth/login' component={AuthLogin} />
```

This view is the template that was used before. It is now used when the user is redirected from the backend and has the tokens in the url. It also has a copytoken method that copies the token from the url, this is used to paste the token into the extention.

```javascript
import React, { useRef } from 'react';
import spotifyIcon from '../Assets/spotifyIcon.png';
import { getParams } from '../Utils/helpers';
import '../Styles/AuthRedirect.css';

const AuthRedirect = () => {
  const textInputRef = useRef(null);
  const params = getParams(window.location);

  const copyToken = e => {
    textInputRef.current.select();
    document.execCommand('copy');
    e.target.focus();
  };

  return (
    <div className='mpm-helper-container'>
      <img className='spotifyLogo' src={spotifyIcon} />
      {params && params.access_token && (
        <div className='tokenContainer'>
          <input
            className='token'
            ref={textInputRef}
            value={params.access_token}
            readOnly
          />
          <button className='btn' onClick={copyToken}>
            Copy to clipboard
          </button>
        </div>
      )}
    </div>
  );
};

export default AuthRedirect;
```

---

#### Frontend, Backend

##### React, Node

#### Backend

Today i fixed backed sign in with Spotify, not sure how to handle the sign in yet but i am looking into it. I did some research how to handle it, i do not want to make the user copy & paste the token when signing in to the dashboard.

I wanted to just send the response but that is not possible beacuse the user is redirected to Spotify to allow the application access.

```javascript
// This does not work.
res.send({
  access_token: access_token,
  refresh_token: refresh_token
});

// This works, this makes me get the token in the url.
res.redirect(
  uri + '?access_token' + access_token + '?refresh_token' + refresh_token
);
```

#### Frontend

I also worked with some frontend, i made some components that i will use when the user has signed in. This still need a lot of work but i made some components to start with and added a route.

```javascript
const SignedIn = props => {
  console.log('mocking', mock);
  return (
    <div className='signedInContainer'>
      <h1>signed in view</h1>
      <Footer />
    </div>
  );
};

export default SignedIn;
```

---

#### Backend, Frontend

##### Node, React

#### Backend

Today i did a lot of backend work. I implemented sign in through the dashboard and also added a database. Once the user has signed in i wanted to store the user in my database and if the user already exsist then the user should just be signed in. To do this i first structured the backend with callbacks, this later became a problem so i refactored everything and decided to use axios. I also implemented jwt tokens to handle the sign in.

My first try was not that pretty:

```javascript
request.post(authOptions, (error, response, body) => {
  const bearer = 'Bearer ' + body.access_token;
  let parameters = {
    url: 'https://api.spotify.com/v1/me',
    headers: {
      Authorization: bearer,
      'Content-Type': 'application/json'
    }
  };
  request.get(parameters, (error, response, userbody) => {
    const bodyjson = JSON.parse(userbody);
    User.findOne({ email: bodyjson.email }, (err, user) => {
      if (err) {
        console.error(err);
      } else {
        if (!user) {
          User.create(
            {
              username: bodyjson.display_name,
              email: bodyjson.email,
              spotifyId: bodyjson.id
            },
            (err1, newUser) => {
              if (err1) {
                return handleError(err1);
              }

              const token = jwt.sign(
                body.access_token,
                body.refresh_token,
                newUser
              );
              process.env.NODE_ENV === 'development'
                ? 'http://localhost:3000'
                : 'https://mpm-dashboard.herokuapp.com';
              res.redirect(dashboard_uri + '?token=' + token);
            }
          );
        } else {
          const token = jwt.sign(body.access_token, body.refresh_token, user);
          process.env.NODE_ENV === 'development'
            ? 'http://localhost:3000'
            : 'https://mpm-dashboard.herokuapp.com';
          res.redirect(dashboard_uri + '?token=' + token);
        }
      }
    });
  });
```

This is after i refactored it:

```javascript
  try {
    const getUserToken = await axios.post(
      tokenUrl,
      querystring.stringify(formData),
      {
        method: 'post',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
          Authorization:
            'Basic ' +
            Buffer.from(
              process.env.SPOTIFY_CLIENT_ID +
                ':' +
                process.env.SPOTIFY_CLIENT_SECRET
            ).toString('base64')
        }
      }
   );

    const { access_token, refresh_token } = getUserToken.data;
    const meUrl = 'https://api.spotify.com/v1/me';

    const getUserByToken = await axios({
      method: 'GET',
      url: meUrl,
      headers: {
        Authorization: 'Bearer ' + access_token
      }
    });

    const { email, display_name, id } = getUserByToken.data;
    let user = await User.findOne({ email }).exec();

    if (!user) {
      user = await User.create({
        username: display_name,
        email: email,
        spotifyId: id
      });
    }

    const token = jwt.sign(access_token, refresh_token, user);

    const dashboard_uri =
      process.env.NODE_ENV === 'development'
        ? 'http://localhost:3000'
        : 'https://mpm-dashboard.herokuapp.com';

    res.redirect(dashboard_uri + '?token=' + token);
  } catch (error) {
    console.error('error', error);
  }
});
```

#### Frontend

I also did some work clientside on the dashboard, mostly how to handle the authentication. This is the first time i used useContext and useEffect in React.

```javascript
import React, { useContext, useEffect } from 'react';
import { AuthContext } from '../Auth/AuthContext';

const AuthRedirectApi = () => {
  const { isAuth, signin } = useContext(AuthContext);

  useEffect(() => {
    const params = window.location.search;
    const token = params.split('=')[1];
    if (!token) {
      window.location.replace('/');
    } else {
      signin(token);
      window.location.replace('/');
    }
  }, []);
  return null;
};

export default AuthRedirectApi;
```

I also created an Authprovider to wrap the routes and a protected route.

---

#### Backend

##### Node

Today i had some problems with the database, at first i wanted to use a object database but after setting it up and using it for a while i realized that it would probably be better with an SQL database. It would have been better beacuse i needed relations. I then decided to change the database to an SQL database. I choose postgres with sequelize instead of mongodb with mongoose. So i started to remove moongose and mongo db and changed it.

```javascript
'use strict';
const fs = require('fs');
const path = require('path');
const Sequelize = require('sequelize');
const basename = path.basename(__filename);
const sequelize = new Sequelize(process.env.DB_CONNECTION);
let db = {};

fs.readdirSync(__dirname)
  .filter(file => {
    return (
      file.indexOf('.') !== 0 && file !== basename && file.slice(-3) === '.js'
    );
  })
  .forEach(file => {
    let model = sequelize['import'](path.join(__dirname, file));
    db[model.name] = model;
  });

Object.keys(db).forEach(modelName => {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize.sync();
db.Sequelize = Sequelize;

module.exports = db;
```

This also forced me to change my models.

```javascript
module.exports = (sequelize, DataTypes) => {
  const User = sequelize.define('User', {
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true
    },
    username: {
      type: DataTypes.STRING,
      allowNull: false
    },
    email: {
      type: DataTypes.STRING,
      allowNull: false
    },
    spotifyId: {
      type: DataTypes.STRING,
      allowNull: false
    }
  });

  User.associate = models => {
    User.hasMany(models.Session, {
      as: 'Sessions',
      foreignKey: 'id'
    });
  };

  return User;
};
```

---

#### Backend, Frontend, Extension

##### Node, React

#### Backend

Today i did a lot of backend work to make everything work. I can now create session and update the session if there already is a session with the same id.

```javascript
router.post('/', async (req, res) => {
  const { totalTime, pausedTimes, sessionId } = req.body;
  const { email, display_name, id } = req.body.user;
  let user = await DB.User.findOne({ where: { email: email } });

  if (!user) {
    user = await DB.User.create({
      username: display_name,
      email: email,
      spotifyId: id,
      createdAt: new Date(),
      updatedAt: new Date()
    });
  }

  let session = await DB.Session.upsert(
    {
      id: sessionId,
      totalTime: totalTime,
      pausedTimes: pausedTimes,
      userId: user.id,
      createdAt: new Date(),
      updatedAt: new Date()
    },
    {
      returning: true
    }
  );

  res.json({ user, session });
});
```

#### Extension

I can now send the users sessiondata from the extension. When the user starts the extension the sessionId is null, that triggers the backend to create a row in the sessions table. When i get the response back i store the id and update the row. This is beacuse the method is called every minute.

```javascript
/**
 * Sends data to the backend and stores it in the database
 */
const sendData = async () => {
  let user = context.globalState.get('user');
  let data = {
    sessionId: sessionId,
    totalTime: userTotalTime,
    pausedTimes: pauseCount,
    user: user
  };
  let res = await fetch(`${baseUrl}/extension`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
  });

  let sessionData = await res.json();
  sessionId = sessionData.session[0].id;
};
```

#### Frontend

I worked alot with the frontend, how to handle the sign in and have protected routes.

```javascript
import React, { useContext, useEffect, useState } from 'react';
import { Redirect, Route } from 'react-router-dom';
import { AuthContext } from './AuthContext';
import Loader from '../Components/Loader';

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const PrivateRoute = ({ component: Component, ...rest }) => {
  const [isLoading, toggleLoading] = useState(true);
  const { isAuth, signin } = useContext(AuthContext);
  const token = localStorage.getItem('token');

  useEffect(() => {
    async function sleepytime() {
      signin(token);
      await sleep(1400);
      toggleLoading(false);
    }
    if (token) {
      sleepytime();
    } else {
      toggleLoading(false);
    }
  }, []);

  return isLoading ? (
    <Loader />
  ) : (
    <Route
      {...rest}
      render={props =>
        isAuth === true ? <Component {...props} /> : <Redirect to='/' />
      }
    />
  );
};

export default PrivateRoute;
```

I also did more frontend work with my signed in view.

```javascript
const Dashboard = () => {
  const { user } = useContext(AuthContext);
  const [sessions, setData] = useState([]);

  useEffect(() => {
    const fetchData = async () => {
      const res = await api(`/extension/${user.id}`);
      const sessions = await res.json();
      sessions.map(session => {
        let convertedTime = convertTime(session.totalTime);
        session.totalTime = convertedTime;
      });
      setData(sessions);
    };
    fetchData();
  }, []);

  const convertTime = time => {
    let hours = Math.floor(time / 3600);
    let minutes = Math.floor((time % 3600) / 60);
    let seconds = Math.floor((time % 3600) % 60);
    return {
      hours,
      minutes,
      seconds
    };
  };

  return (
    <div className='signedInContainer'>
      <Header username={user.username} />
      <div className='sessionContainer'>
        {sessions.map(session => (
          <div key={session.id} className='sessionCard'>
            <p>{new Date(session.createdAt).toDateString()}</p>
            <div className='sessionTime'>
              <p>{session.totalTime.hours} hours</p>
              <p>{session.totalTime.minutes} minutes</p>
              <p>{session.totalTime.seconds} seconds</p>
            </div>
            <p>Paused times: {session.pausedTimes}</p>
          </div>
        ))}
      </div>
      <Footer />
    </div>
  );
};

export default Dashboard;
```

The frontend still needs a lot of work but i can now sign in from the dashboard, get all the data from the signed in user, send data from the extension to the correct user. There is still a lot of styling that needs to be done and also some more features.

---

#### Frontend

##### React

Today i wanted to present the data fetched a little bit prettier so i decided to have a chart. I choose recharts. This made it a lot easier to make a chart. The chart only displays the played time and the paused time in a piechart.

```javascript
<div key={session.id} className='sessionCard'>
  <p>{new Date(session.createdAt).toDateString()}</p>
  <Chart data={chartData} />
</div>
```

Im going have a const with colors and planning on adding a way to change the color later. Either by sending them in as a prop to the component or by having a way to change theme on the webpage.

```javascript
import { PieChart, Pie, Cell } from 'recharts';

const COLORS = ['#B749A7', '#6E5AAD'];

const Chart = props => {
  return (
    <PieChart width={400} height={200}>
      <Pie
        dataKey='value'
        startAngle={180}
        endAngle={0}
        data={props.data}
        cx={200}
        cy={150}
        outerRadius={100}
        fill='#8884d8'
        label={({ cx, cy, midAngle, innerRadius, outerRadius, index }) => {
          const RADIAN = Math.PI / 180;
          const radius = 25 + innerRadius + (outerRadius - innerRadius);
          const x = cx + radius * Math.cos(-midAngle * RADIAN);
          const y = cy + radius * Math.sin(-midAngle * RADIAN);

          return (
            <text
              x={x}
              y={y}
              fill='#8884d8'
              textAnchor={x > cx ? 'start' : 'end'}
              dominantBaseline='central'
            >
              {props.data[index].name}
            </text>
          );
        }}
      >
        {props.data.map((entry, index) => (
          <Cell key={`slice-${index}`} fill={COLORS[index % COLORS.length]} />
        ))}
      </Pie>
    </PieChart>
  );
};
```

---

#### Frontend

##### React

Today i wanted to add a way to change the theme of the webpage. I did not know how to approach this and tried some different things. The firt thing i tried was making a ThemeContext and wrapping components.

This was my first approach

```javascript
import React from 'react';
export const themes = {
  light: 'light',
  dark: 'dark'
};

export const ThemeContext = React.createContext(themes.light);
```

```javascript
const theme = useContext(ThemeContext);

<ThemeContext.Provider value={colorTheme} />;
```

The problem was that every HTML element was going to use the theme as a classname and i had to send it as a prop everywhere so i removed it.

The second approach i took was having a button (or something) that toggles it and stores the theme in localstorage, this is beacuse i wanted to have the same theme saved for the user even if they reload the page. I then prefix the css witch makes it easy to add more themes later on if that is what i want to do. So the button just adds a classname to the body element.

```css
.link {
  text-decoration: none;
  color: #000;
}

body.dark .link {
  text-decoration: none;
  color: #1db954;
}
```

This worked fine until i discovered that i had some trouble with the canvas music visualizer. The bars did not change colors and i tried to figure out how to make it work. I struggled a lot with it beacuse the component does not render again. I then made tought "what if i listen to changes in the DOM tree?". I found MutationObserver that maybe could help me make it work, and it did!

```javascript
let targetNode = document.getElementById('body');
const config = { attributes: true };

const callback = (mutationsList, observer) => {
  for (const mutation of mutationsList) {
    if (mutation.target.className == '') {
      fillStyleColor = '#fff';
    } else {
      fillStyleColor = '#1b1c21';
    }
  }
};

let observer = new MutationObserver(callback);
observer.observe(targetNode, config);

let canvas = document.querySelector('#canvas');
let ctx = canvas.getContext('2d');
ctx.fillStyle = fillStyleColor;

if (fillStyleColor === '#fff') {
  r = barHeight + 25 * (i / bufferLength);
  g = 5 * (i / bufferLength);
  b = 177;
} else {
  r = 5 * (i / bufferLength);
  g = barHeight + 150 * (i / bufferLength);
  b = 50;
}
```

This listens to changes on the body element and sets different color based on the theme.

---

#### Frontend, Backend

##### React, Node

#### Frontend

Today i worked a bit more with my piecharts. I added a piechart that shows a sum of all the sessions combined. I also removed some errors that i had. I had ssome small things that i wanted to refactor so i also did that. Mostly on the musicvisualizer. The dashboard now shows all the sessions in small cards and one total sessions. The totalsessions still needs some styling but the functionality works.

I found some small bugs here and there but were able to fix them quite quick.

#### Backend

To get the total sum of all the sessions for a user i choose to create a separate route in the backend that returns that for me.

```javascript
router.get('/total/:id', async (req, res) => {
  let totalSessions = await DB.Session.findOne({
    attributes: [
      [
        DB.Sequelize.fn('sum', DB.Sequelize.col('pausedTimes')),
        'pausedTimesSum'
      ],
      [DB.Sequelize.fn('sum', DB.Sequelize.col('musicTime')), 'musicTimeSum'],
      [DB.Sequelize.fn('sum', DB.Sequelize.col('totalTime')), 'totalTimeSum']
    ],
    where: { userId: req.params.id }
  });
  res.json(totalSessions);
});
```

There is still a lot of work to be done but for now i am i happy with my progress.

---

#### Frontend, Backend, Documentation

#### React, Node

#### Backend

Today i worked with adding more features. I want to add user settings that the user can change and the extension changes based on the user settings. I wanted to add how much time the user gets per keypress in the extension (now it is hardcoded to 1 second) and if the user wants to enable "hard mode". If hard mode is enabled it will reset the counter of time left to 0 when the user presses backspace. (Now it is hardcoded to decrement by 1).

I needed to update the user model to have support for settings. I did not know if i was going to make another table or just add a column inside the users table. I decided to add a column in the users table that is JSON.

```javascript
  settings: {
    type: DataTypes.JSON,
    allowNull: true
  }
```

I wanted to allow null beacuse if the user does not set any settings it will be defaulted from the extension.

I then needed to add routes for the settings in the backend.

```javascript
router.post('/settings/:id', async (req, res) => {
  let settings = req.body;
  let user = await DB.User.update(
    {
      settings: settings
    },
    {
      where: { id: req.params.id },
      returning: true
    }
  );
  res.json(user);
});

router.get('/settings/:id', async (req, res) => {
  let user = await DB.User.findOne({
    attributes: ['settings'],
    where: { id: req.params.id }
  });

  res.json(user);
});
```

I had some small fixes to do with moving routes to the right path.

#### Frontend

I did some work with the footer by adding icons and switching the colors when the theme is changed.

```javascript
const Footer = () => {
  return (
    <div className='footer'>
      <a href='https://github.com/Ciavarella' id='github' />
      <a href='https://facebook.com/victor.ciavarella' id='facebook' />
      <a href='https://instagram.com/vciavarella/' id='instagram' />
    </div>
  );
};
```

I fixed some things with the dashboard that has not working correctly.

I then added a new view for the user settings.

```javascript
import React, { useContext, useState } from 'react';
import { AuthContext } from '../../Auth/AuthContext';
import Header from '../../Components/Header';
import Footer from '../../Components/Footer';
import '../../Styles/Settings.css';

const baseUrl =
  process.env.NODE_ENV === 'development'
    ? 'http://localhost:8080'
    : 'https://mpm-node-backend.herokuapp.com';

const Settings = () => {
  const { user } = useContext(AuthContext);
  const [keyValue, setKeyValue] = useState();
  const [hardModeValue, setHardMode] = useState(false);

  const setPressValue = e => {
    let data = e.target.value;
    setKeyValue(data);
  };

  const setHardModeValue = e => {
    const value = JSON.parse(e.target.value);
    let data = value === false ? true : false;
    setHardMode(data);
  };

  const saveSettings = async () => {
    const values = {
      keypress: keyValue,
      hardcore: hardModeValue
    };
    fetch(`${baseUrl}/dashboard/settings/${user.id}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values)
    })
      .then(res => res.json())
      .then(data => {
        const { keypress, hardcore } = data[1][0].settings;
      });
  };

  return (
    <div className='settingsContainer'>
      <Header username={user.username} />
      <div className='settingsCard'>
        <div className='form'>
          <div className='keyPress'>
            <p>Seconds / Keypress</p>
            <input
              type='text'
              name='keypress'
              placeholder='Keypress'
              onChange={setPressValue}
            />
          </div>
          <div className='hardMode'>
            <p>Hard mode</p>
            <input
              type='checkbox'
              name='hardcore'
              value={hardModeValue}
              onClick={setHardModeValue}
            />
          </div>
        </div>
        <button onClick={saveSettings} className='settingsBtn'>
          Save
        </button>
      </div>
      <Footer />
    </div>
  );
};

export default Settings;
```

It still needs some work but it works fine for now.

#### Documentation

I also wrote a lot of documentation on all the backend routes and the Spotiy routes that i use.

The documentation has all the backend paths and routes, parameters that is needed and response.

I still need to add documentation for the frontend paths that is used.

---

#### Deploy, Buggfixes

#### Buggfixes

Today i worked i some small fixes but my main focus was on deploying the application even though it is not done yet to test the features in production. I fixed some border radius on an input, changed the input from text to number etc.

#### Deploy

I have been deploying before to Heroku but when i deployed today i had some issues with the buildpack. First i had to figure out that it was the buildpack that did not work correctly and i hade a npm script that was not working properly in production beacuse i had it configured for development. Also paths that were specified to development.

I did not achive as much as i wanted today beacuse i had a lot of issues finding what was wrong but i think im still on schedule.

---

#### Extension, Backend, Frontend

#### Node, React

#### Frontend

I had some issues when getting the users settings from the database, i realized that when i insert the user settings to the database i send it as a string and not a number.

This sent the value as a string.

```javascript
let data = e.target.value;
setKeyValue(data);
```

Parse the value and then set it, it is now a number.

```javascript
let value = JSON.parse(e.target.value);
setKeyValue(value);
```

#### Backend

Today i worked more with the backend to make the extension work properly towards the backend. I had a method that could get the users settings. The user id was recived as a url parameter which did not work for me beacuse i had no access to users id from the extension.

This method worked fine but from the extension i needed to make another request to the backend to store the database id just to get the settings.

```javascript
/**
 *  Not currently used.
 *  Get user settings by user id.
 *  */
router.get('/settings/:id', async (req, res) => {
  let user = await DB.User.findOne({
    attributes: ['settings'],
    where: { id: req.params.id }
  });

  res.json(user);
});
```

I had the user that was returned from Spotify that included the users email. I then decided to send the email as a headers parameter to not send one more request.

```javascript
/**
 * Get user settings by email.
 * */
router.get('/settings', async (req, res) => {
  const { email } = req.headers;

  let user = await DB.User.findOne({
    attributes: ['settings'],
    where: { email: email }
  });

  res.json(user);
});
```

#### Extension

Today i worked with getting the user settings from the backend to set as counter. I want the user to set the amount of seconds per keypress that will be added to the playtime. The user can also enable hardmode that sets the playtime to 0 and pauses the music if the user presses backspace.

If the user does not have any settings it defaults to 1 second per keypress and disables hard mode.

```javascript
/**
 * Gets the users settings by email based on the user in globalState.
 * If the user has no settings it will default to the default values.
 * */
const getUserSettings = async () => {
  let user = context.globalState.get('user');

  if (user !== undefined) {
    let res = await fetch(`${baseUrl}/extension/settings`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        email: user.email
      }
    });
    let settings = await res.json();
    if (settings !== null) {
      context.globalState.update('keypress', settings.settings.keypress);
      context.globalState.update('hardmode', settings.settings.hardcore);
      keypressTime = settings.settings.keypress;
      hardMode = settings.settings.hardcore;
    } else {
      keypressTime = 1;
      hardMode = false;
    }
  } else {
    keypressTime = 1;
    hardMode = false;
  }
};
```

I have some problems with adding time and removing time from the playtime when i check the key that is pressed.

```javascript
/**
 * Adds to the counter.
 */

myEmitter.on('keystroke', () => {
  counter = counter + keypressTime;
});

/**
 * Removes from the counter if the counter not is on 0.
 */
myEmitter.on('backspace', () => {
  if (hardMode === true) {
    counter = 0;
  }
  if (counter !== 0) {
    counter = counter - 2;
  }
});

/**
 * Will check if the key pressed is backspace.
 * Call an event emitters to check time and keystroke.
 */
const checkInput = event => {
  if (event.contentChanges[0].text === '') {
    myEmitter.emit('backspace');
  }
  myEmitter.emit('check');
  myEmitter.emit('keystroke');
};
```

This is where im having some trouble. First i check if backspace is pressed. If it is it will decrement correctly but then it calls the keystroke event emitter that will add to the counter. This worked fine when i hardcoded the keypressed value to 1. I need to figure this out and make some changes to make it work

---

#### Frontend, Backend, Extension, Deploy

#### React, Node

#### Backend

I only did some small things in on my backend. I updated dependencies, added some comments and changed the url that the backend redirects to. I choose to host my client on a different host so i needed to update the url.

#### Frontend

I did a lot of small fixes on the frontend. I added settings and signout to the header when the user is signed in. I also added a about page. When i had added it i realised that it would probably have the description on the startpage and not just a logo. So i changed it so there is a description on the start page. I then removed the about page. I made some more small changes, changed the font, set the default theme to be the light theme, changed color of the loader when the theme is dark and changed the favicon.

It was not big changes but i think it made the feel of the application feel a lot better.

#### Extension

Yesterday i had some problems with the checkInput method. I don't really know what i was thinking but today i took another look at it and just added a else statement and it worked fine after that.

Before changes:

```javascript
const checkInput = event => {
  if (event.contentChanges[0].text === '') {
    myEmitter.emit('backspace')
  }
  myEmitter.emit('check')
  myEmitter.emit('keystroke')
```

After changes:

```javascript
  const checkInput = event => {
    if (event.contentChanges[0].text === '') {
      myEmitter.emit('backspace')
    } else {
      myEmitter.emit('keystroke')
    }
    myEmitter.emit('check')
  }
}
```

Yesterday i implemented methods to get the users settings, the if statements when to play or pause music broke beacuse the counter could be different values. I then needed to update the if statements.

```javascript
/**
 * Compares the counter and the previous value.
 * Checks if it should play or pause music.
 * Calls the play or pause method.
 */
myEmitter.on('check', () => {
  if (counter === 0 && prevCount === 0) {
    return;
  } else if (counter > 0 && prevCount < counter) {
    let expires = context.globalState.get('expires');
    let now = Date.now() / 1000;
    if (now > expires) {
      requestNewToken();
    } else {
      playMusic();
    }
  } else if (counter === 0 && counter < prevCount) {
    let expires = context.globalState.get('expires');
    let now = Date.now() / 1000;
    if (now > expires) {
      requestNewToken();
    } else {
      pauseMusic();
    }
  }
});
```

I also added a global variable that is called isPlaying and default it to false. This is beacuse when calling the play method i have a if statement that returns early if isPlaying is set to true. This is beacuse i dont want to make requests all the time to spotify when a user is pressing a key.

#### Deploy

I had both my frontend and backend deployed on Heroku but i wanted to use my own domain "ciavarella.dev". So i decided to host it on Netlify. i then deployed the frontend / dashboard to Netlify. The backend is still deployed on Heroku. I had to do some configuration for this and re-deploy multiple times to test things. I had some problems with react router on the deployed version. I did som research and found out that i needed to add a \_redirects file to the public folder.

\_redirects

```
/*    /index.html   200
```

I think i am ready to try and publish my extension and have some friends try it out and come with some feedback.

#### Deploy part 2

So i tried to publish the extension on marketplace and publishing it was not a problem, the publishing part did go smooth. I got an issue from a user that had downloaded the extension and had some problems. I was really happy with the feedback and tried to debug the issues that the user had. I tried to fix some bugs and published it again. There are probably some more issues that needs to be done but i am really happy with the progress i made.

---

#### Extension

#### Node, TypeScript

#### Extension

I have published my extension and everything is working fine, however i want to add more features to make it more user friendly. Currently there is no stop function that stops the extension. I have it set up so the user has to run a command in the command palette to activate the extension but i also want to and a command so the user can stop the extension.

I have been working on a stop command for the extension but there is still some work do before i publish a new release. I am also adding some settings so that the user does not need to sign in on the dashboard to change the settings. I want the user to be able to do it directly through VsCode settings.

I made some major refactoring of the code. My first version was basically just trying to make it work. Now that im close to being finished (or so i think, but probably not) i want to refactor it. I choose to use TypeScript instead of regular JavaScript.

This part keeps track of when the extension is activated or when it's stopped.

```javascript
import * as dotenv from 'dotenv';
import * as vscode from 'vscode';
import MusicPerMinute from './Mpm';

let documentChangeListenerDisposer: any = null;
let enabled = false;
let Mpm: MusicPerMinute;
/**
 * This method is called when the extension is activated.
 * @param {vscode.ExtensionContext} context
 */
export function activate(context: vscode.ExtensionContext) {
  Mpm = new MusicPerMinute(context);

  vscode.workspace.onDidChangeConfiguration(Mpm.onDidChangeConfiguration);
  Mpm.onDidChangeConfiguration();

  const disposable = vscode.commands.registerCommand('extension.mpm', () => {
    vscode.window.showInformationMessage('Music Per Minute Started!');
    if (!enabled) {
      Mpm.init();
    }
  });

  const stopMe = vscode.commands.registerCommand('extension.mpm.stop', () => {
    vscode.window.showInformationMessage('Stopped Music Per Minute!');
    deactivate();
  });

  context.subscriptions.push(disposable);
  context.subscriptions.push(stopMe);
}

/**
 * Method called when stopping the extension.
 */
export function deactivate() {
  if (documentChangeListenerDisposer) {
    documentChangeListenerDisposer.dispose();
    documentChangeListenerDisposer = null;
    enabled = false;
  }
}
```

This is after refactoring it to TypeScript, there is still some more refactoring needed, probably add some utils function or make a class that handles all the counters that i have.

```javascript
  /**
   * Gets the user and stores the user in config.
   */
  public async getUser(): Promise<void> {
    const bearer = 'Bearer ' + this.token;

    const res = await fetch('https://api.spotify.com/v1/me', {
      method: 'GET',
      headers: {
        Authorization: bearer,
        'Content-Type': 'application/json'
      }
    });

    const user = await res.json();

    this.ctx.globalState.update('user', user);
  }
```

Im still having some trouble with the extension when running it locally after i refactored it and added the stop method. I don't think it is a major problem but i need to look it over and also go through the settings.

---

#### Extension

#### Node, TypeScript

#### Extension

Today i have implemented the stop function, also so i can start it again without closing or restarting VsCode. Now the user can stop the extension with a command in the command palette.

I also have fixed so the user can change their settings directly in VsCode and does not need to go to the dashboard to change the settings. This is more user friendly but if the user wants to change it from the dashboard then that also works.

I fixed some things with the deactivate method aswell. I fixed so the counter no longer is visible and it stops the counter.

This is what it looked like before.

```javascript
/**
 * Method called when stopping the extension.
 */
export function deactivate() {
  if (documentChangeListenerDisposer) {
    documentChangeListenerDisposer.dispose();
    documentChangeListenerDisposer = null;

    enabled = false;
  }
}
```

This is after

```javascript
/**
 * Method called when stopping the extension.
 */
export function deactivate() {
  Mpm.dispose();
  vscode.workspace.getConfiguration('mpm').update('enabled', false);
  enabled = false;
}

  public dispose() {
    this.disposable.dispose();
    clearInterval(this.intervalId);
    clearInterval(this.decrementTimer);
    clearInterval(this.sendDataTimer);
    this.item.text = '';
    this.item.hide();
    this.counter = 0;
  }

```

For now im happy with the extension, it could probably be improved i a couple of ways but i have published a new version and waiting to see if i get some feedback from users.

Also created a ER diagram.

---

#### Extension, Documentation

#### Node

I had some fixes that needed to be done, there was a problem with sign in. The problem was that the user has to sign in with Spotify again after the user already has approved the application. The only time the user shold approve the application and sign in with Spotify is the first time the user uses the extension. This was not the case.

The issue was that the extension is a class and when the extension starts it initiates a new instance of the class and passes a context. So when the user stops and start the extension again the class recives a new context. The api key and refreshtoken was saved in that specific context so i could not get the users api key, so the user has to sign in again.

To solve the problem i choose to not save the users api key and refresh token in globalstate anymore and decided to save it in settings instead.

```javascript
// Get values
const key = await vscode.workspace.getConfiguration('mpm').get('api_key');

// Update values
vscode.workspace.getConfiguration('mpm').update('api_key', accessToken, true);

vscode.workspace
  .getConfiguration('mpm')
  .update('refresh_key', refreshToken, true);
```

This solved my problem so i always have access to the values no matter the instance.

#### Documentation

I have also worked with submitting my assignment with all documentation, making sure everything is up to date and that everything looks neat and clean.
