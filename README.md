### hapi-session-mongo

[**hapi**](https://github.com/hapijs/hapi) MongoDB Session Storage

## Install

## Usage
A session store plugin for hapi and MongoDB. Must have database already set up
with one user that has readWrite role. All other users in database are just for
the challenge-response mechanism and require no roles. Requires options:
- `ip` - The IP address of the database. Defaults to `127.0.0.1`.
- `port` - The port number of the database. Defaults to `27017`.
- `db` - The name of the database. Defaults to `test`.
- `name` - The name of the user with readWrite. Defaults to `undefined`.
- `pwd` - The password of the user with readWrite, also signs Iron cookie. Defaults to `undefined`.
- `ssl` - MongoDB ssl. Defaults to `false`.
- `ttl` - Time-to-live for Iron cookie. Defaults to `0`.

Also exports functions:
- `user.login(username, password, callback)` - Challenge-response user. Callback is cookie or false.
- `req.auth.session.set(session)` - Called with login to set server state.
- `user.get(session, callback)` - Session is cookie. Callback is true or false if still valid.
    to be called with the `validateFunc`.

During the `server.auth.strategy` phase `validateFunc(session, callback)` is required.

Example set up:
```javascript
var Hapi = require('hapi');

var server = Hapi.createServer('127.0.0.1', 3000);

server.pack.register({
    plugin: require('../hapi-session-mongo'),
    options: {
        db: 'users',
        name: 'sessionHandler',
        pwd: 'supersecretpassword',
        ssl: true
    }
}, function (err) {
    if (err) { console.log(err); };

    server.auth.strategy('session', 'mongo', {
        validateFunc: function(session, callback) {
            server.plugins['hapi-session-mongo'].user.get(session, function(valid) {
                return callback(null, valid);
            });
        }
    });
});

server.route([
    {
        method: 'POST',
        path: '/banned',
        handler: function (req, res) {
            server.plugins['hapi-session-mongo'].user.login(req.payload.username,
                req.payload.password, function(logged) {
                  req.auth.session.set(logged);
                  res(logged);
            });
        }
    },
    {
        method: 'GET',
        path: '/banned',
        config: {
            handler: function(req, res) {
                res('You are now logged in');
            },
            auth: 'session'
        }
    }
]);

server.start();
```

## Removing stale sessions

MongoDB 2.2 and above supports doing this via an index, see http://docs.mongodb.org/manual/tutorial/expire-data/
To enable this, run

    db.sessions.ensureIndex( { "createdAt": 1 }, { expireAfterSeconds: 3600 } )

Mongo will now remove all sessions older than an hour (every 60 seconds).
