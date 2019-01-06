# Persist Data

## Architect bakes in easy to use, first-class DynamoDB support for its speed and flexibility

Durable persistence of structured data is the foundation of most applications. `@architect/data` is a very thin wrapper for `DynamoDB` and `DynamoDB.DocumentClient` that reads a `.arc` file and returns a client for creating, modifying, deleting and querying data from DynamoDB (aka Dynamo).

In this guide you will build a simple note taking application with Dynamo and `.arc`.


## Generating the Data Layer

Given the following `.arc` file:

```arc
@app
testapp

@http
get /
get /notes/:noteID

post /login
post /logout
post /notes
post /notes/:noteID
post /notes/:noteID/del

@tables
accounts
  accountID *String

notes
  accountID *String
  noteID **String
```

Running `npx create` will generate routes and tables to model our persistence needs. The `accounts` table defined above will have an `accountID` partition key, while the `notes` table will have an `accountID` partition key and a unique `noteID`. This is one way to model a "many-to-many" relationship in Dynamo. 

So, at this point, `npx create` will create the following Dynamo tables:

- `testapp-staging-accounts`
- `testapp-production-accounts`
- `testapp-staging-notes`
- `testapp-production-notes`


## Implementing an Admin Interface

Now let's create a basic interface for this notes app. First, let's create a basic shared layout in `src/shared`, which will make it available to all functions (more on [sharing code across functions here](/guides/sharing-common-code)):

```bash
mkdir src/shared/views
touch src/shared/views/layout.js
touch src/shared/views/_header.js
```

```javascript
// src/shared/layout.js
let auth = require('./_header')

module.exports = function layout(params={}) {
  let body = params.body || 'hello world'
  let title = params.title || '@architect/data demo'
  let url = params.path
  return `
<!doctype html>
<html lang=en>
  <head>
    <meta charset=utf-8>
    <meta name=viewport content=width=device-width,initial-scale=1,shrink-to-fit=no>
    <link rel=stylesheet 
      href=https://stackpath.bootstrapcdn.com/bootstrap/4.1.0/css/bootstrap.min.css
      integrity=sha384-9gVQ4dYFwwWSjIDZnLEWnxCjeSWFphJiwGPXr1jddIhOegiu1FwO5qRGvFXOdJZ4 
      crossorigin=anonymous>
    <title>${title}</title>
  </head>
  <body>
    ${auth({url})}
    ${body}
  </body>
</html>
`
}
```

The layout module itself accepts an optional params object with `body`, `title` and `url` keys, and returns an HTML document as a string. Truly, string interpolation is the purest essence of web development. And thus for now we'll just use Bootstrap. 🤷🏽‍♀️

Implement the HTML partial `_header` control:

```javascript
// src/shared/_header.js
module.exports = function _header({url}) {
  if (path.includes('logout')) {
    return `
<form action=${url} method=post>
  <button type=submit class="btn btn-primary float-right m-4">Logout</button>
</form>`
  }
  else {
    return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>

    <form action=${url} method=post>
      <div class=form-group>
        <label for=email>Email address</label>
        <input type=email class=form-control name=email placeholder="Enter email">
      </div>
      <div class=form-group>
        <label for=password>Password</label>
        <input type=password class=form-control name=password placeholder="Password">
      </div>
      <button type=submit class="btn btn-primary float-right">Login</button>
    </form>

  </div>
</div>
`
  }
}
```

The `_header` module accepts a parameters object with the key `url`. If the URL includes the text "logout" it renders a logout button. Otherwise it renders a login form. Vanilla stuff.

Next, include the layout into your home route:

```javascript
// src/html/get-index/index.js
let arc = require('@architect/functions')
let layout = require('@architect/shared/views/layout')
let url = arc.http.helpers.url

function route(req, res) {
  let body = '&nbsp;'
  let title = 'welcome home'
  let path = url(req.session.account ? '/logout' : '/login')
  let html = layout({body, title, path})
  res({html})
}

exports.handler = arc.http(route)
```


## Implementing Login

For now, let's just hardcode credentials:

```javascript
// src/html/post-login/index.js
let arc = require('@architect/functions')
let url = arc.http.helpers.url

function route(req, res) {
  let location = url('/')
  let session = {}
  let authorized = req.body.email === 'b@brian.io' && req.body.password === 'lolwut'
  if (authorized) {
    session.account = {name: 'brianleroux', accountID: 'fake-account-id'}
  }
  res({session, location})
}

exports.handler = arc.http(route)
```


## Implementing Logout

We'll implement the logout handler too:

```javascript
// src/http/post-logout/index.js
let arc = require('@architect/functions')
let url = arc.http.helpers.url

function route(req, res) {
  res({
    session: {},
    location: url('/'),
  })
}

exports.handler = arc.http(route)
```

This wipes the current session and redirects back to `/`.


## Protecting Routes

To ensure no bad actors start posting notes, we can lock down the other routes with some basic middleware. 

```bash
mkdir src/shared/middleware
touch src/shared/middleware/auth.js
```

This auth middleware will check for `req.session.account`. If it exists, execution is passed to the next function in the middleware chain. If it does not exist, the response is redirected `/`. (Later in the guide we'll incorporate this into routes we want to protect.)

```javascript
// src/shared/middleware/auth.js
let arc = require('@architect/functions')
let url = arc.http.helpers.url

module.exports = function auth(req, res, next) {
  // if the current session is logged in just continue to the next function
  if (req.session.account) {
    next()
  }
  else {
    // otherwise boot them back to the home page
    res({
      location: url('/')
    })
  }
}
```

> 🏄‍♀️ Read more about middleware and sessions in the [HTTP Functions](/guides/http) guide


## Write a Note

Ok, it's almost time to start creating some notes. First, let's modify the index route with its own HTML partial to render a form for creating notes when logged in:

```javascript
// src/html/get-index/_form
module.exports = function form({url}) {
  return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>

    <form action=${url} method=post>
      <div class=form-group>
        <input type=text class=form-control name=title placeholder="Enter title" required>
      </div>
      <div class=form-group>
        <textarea class=form-control placeholder="Enter text"></textarea>
      </div>
      <button type=submit class="btn btn-primary float-right">Save</button>
    </form>

  </div>
</div>
`
}
```

And add it to the index handler:

```javascript
// src/html/get-index/index.js
let arc = require('@architect/functions')
let layout = require('@architect/shared/views/layout')
let url = arc.http.helpers.url
let form = require('./_form')

function route(req, res) {
  let title = 'welcome home'
  let body = req.session.account? form({url:url('/notes')}) : '&nbsp;'
  let path = req.session.account? url('/logout') : url('/login')
  let html = layout({body, title, path})
  res({html})
}

exports.handler = arc.http(route)
```

Now implement the post handler. We'll use the `hashids` library to help create keys for our notes.

```bash
cd src/html/post-notes
npm i hashids
```

And then in the handler:

```javascript
// src/html/post-notes/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let auth = require('@architect/shared/middleware/auth')
let url = arc.http.helpers.url
let Hashids = require('hashids')
let hashids = new Hashids

async function route(req, res) {
  try {
    // get the note.title and note.body from the form post
    let note = req.body
    // create the partition and sort keys
    note.accountID = req.session.account.accountID
    note.noteID = hashids.encode(Date.now())
    // save the note
    let result = await data.notes.put(note)
    // log it to stdout
    console.log(result)
  }
  catch(e) {
    console.log(e)
  }
  res({
    location: url('/')
  })
}

exports.handler = arc.http(auth, route)
```

Requiring `@architect/data` reads your app's `.arc` manifest and generates a data access client from it. Working with `.arc` this way means:

- You can immediately start working with a database without any configuration
- No syntax errors due to misspelled string identifiers for table names
- No configuration code to map table names to execution environments, ensuring staging and production data cannot get mixed up

The following API was generated from the `.arc` file above:

- `data._db` - an instance of `DynamoDB` from the `aws-sdk`
- `data._doc` - an instance of `DynamoDB.DocumentClient` from the `aws-sdk`
- `data._name` - a helper for returning an environment appropriate table name
- `data.accounts.get` - get an account row
- `data.accounts.query` - query accounts
- `data.accounts.scan` - return all accounts in pages of 1MB
- `data.accounts.put` - write an account row
- `data.accounts.update` - update an account row
- `data.accounts.delete` - delete an account row
- `data.notes.get` - get a note
- `data.notes.query` - query notes
- `data.notes.scan` - return all notes in pages in 1MB
- `data.notes.put` - write a note
- `data.notes.update` - update a note
- `data.notes.delete` - delete a note

In addition to providing some extra safety, the generated code also saves on boilerplate! 

All generated methods accept a params object as a first parameter and an optional callback. If the callback is not supplied, a Promise is returned.

It is important to note this is a powerful client that allows you to read and write data indiscriminately. Security and access control to your Dynamo tables is determined by your domain specific application business logic. (As such, notice the `shared/middleware/auth` handler has been added to protect the route.)

Extra credit:

- Sanitize inputs with XSS
- Validate input; you probably can do without a library


## Show All Notes

For now, lets just modify the home route to get all the notes and pass them into the `_form` HTML partial:

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let layout = require('@architect/shared/views/layout')
let url = arc.http.helpers.url
let form = require('./_form')

async function route(req, res) {
  let title = 'welcome home'
  let notes = await data.notes.scan({})
  let body = req.session.account? form({url:url('/notes'), notes}) : '&nbsp;'
  let path = req.session.account? url('/logout') : url('/login')
  let html = layout({body, title, path})
  res({html})
}

exports.handler = arc.http(route)
```

Inside the partial we can just dump the JSON for now:

```javascript
// src/html/get-index/_form
module.exports = function form({url, notes}) {
  return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>
    <form action=${url} method=post>
      <div class=form-group>
        <input type=text class=form-control name=title placeholder="Enter title" required>
      </div>
      <div class=form-group>
        <textarea class=form-control placeholder="Enter text"></textarea>
      </div>
      <button type=submit class="btn btn-primary float-right">Save</button>
    </form>
  </div>
</div>

<div class="card mt-4 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>
    <pre>${JSON.stringify(notes, null, 2)}</pre>
  </div>
</div>
`
}
```

Now as we add notes, we can see them populating the database.


## Show a Note 

Let's clean up the debugging JSON with an HTML representation of the note data:

```javascript
// src/html/get-index/_form

function note({title, body, href}) {
  return `
<div class="card mt-4 mr-auto ml-auto mb-1 w-25">
  <div class=card-header><a href=${href}>${title}</a></div>
  <div class=card-body>${body}</div>
</div>
`
}

module.exports = function form({url, notes}) {
  return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>
    <form action=${url} method=post>
      <div class=form-group>
        <input type=text class=form-control name=title placeholder="Enter title" required>
      </div>
      <div class=form-group>
        <textarea class=form-control name=body placeholder="Enter text"></textarea>
      </div>
      <button type=submit class="btn btn-primary float-right">Save</button>
    </form>
  </div>
</div>

${notes.map(note).join('\n')}
`
}
```

The form partial now maps over notes, applying an internal function `note` to generate HTML.

We'll use this opportunity to tidy up the home page logic by breaking out the authenticated and unauthenticated responses into separate middleware:

```javascript
// src/html/get-index/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let layout = require('@architect/shared/views/layout')
let url = arc.http.helpers.url
let form = require('./_form')

// logic for authenticated visitors
async function authorized(req, res, next) {
  if (!req.session.account) {
    next()
  }
  else {
    // get all the notes
    let title = 'welcome home'
    let all = await data.notes.scan({})

    // add href to each note for the template link
    let notes = all.Items.map(function addHref(note) {
      note.href = url(`/notes/${note.noteID}`)
      return note
    })
  
    // disambiguate URLs for envs
    let createUrl = url('/notes')
    let logoutUrl = url('/logout')
  
    // interpolate the template data 
    let body = form({url: createUrl, notes})
    let html = layout({body, title, path: logoutUrl})

    res({html})
  }
}

// shown for unauthenticated visitors
function unauthorized(req, res) {
  let title = 'welcome home'
  let body = '&nbsp;'
  let path = url('/login')
  let html = layout({body, title, path})
  res({html})
}

exports.handler = arc.http(authorized, unauthorized)
```

🆒 Now let's implement `get /notes/:noteID`.

```javascript
// src/html/get-notes-000noteID/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let layout = require('@architect/shared/views/layout')
let auth = require('@architect/shared/middleware/auth')
let url = arc.http.helpers.url

async function route(req, res) {
  let title = 'welcome home'
  let noteID = req.params.noteID
  let accountID = req.session.account.accountID
  let note = await data.notes.get({noteID, accountID})
  let body = `<pre>${JSON.stringify(note, null, 2)}</pre>`
  let path = url('/logout')
  let html = layout({body, title, url})
  res({html})
}

exports.handler = arc.http(auth, route)
```


## Edit a Note

Lets make the detail page show the current note in an edit form.

```javascript
// src/http/get-notes-000noteID/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let layout = require('@architect/shared/views/layout')
let auth = require('@architect/shared/middleware/auth')
let url = arc.http.helpers.url
let form = require('./_form')

async function route(req, res) {
  let title = 'welcome home'
  // wrangle the data
  let noteID = req.params.noteID
  let accountID = req.session.account.accountID
  let note = await data.notes.get({noteID, accountID})
  note.href = req._url(`/notes/${noteID}`)
  // build out the templates
  let body = form(note)
  let path = url('/logout')
  let html = layout({body, title, path})
  // send the response
  res({html})
}

exports.handler = arc.http(auth, route)

```

And then the form partial itself:

```javascript
// src/html/get-notes-000noteID/_form
module.exports = function form({noteID, href, title, body}) {
  return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>
    <form action=${href} method=post>
      <input type=hidden name=noteID value=${noteID}>
      <div class=form-group>
        <input
          type=text
          class=form-control
          name=title
          placeholder="Enter title"
          value="${title}"
          required>
      </div>
      <div class=form-group>
        <textarea
          class=form-control
          name=body
          placeholder="Enter text">${body}</textarea>
      </div>
      <button type=submit class="btn btn-primary float-right">Save</button>
    </form>
  </div>
</div>
`
}
```

And lets implement the update action.

```javascript
// src/html/post-notes-000noteID/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let auth = require('@architect/shared/middleware/auth')
let url = arc.http.helpers.url

async function route(req, res) {
  try {
    let note = req.body
    note.accountID = req.session.account.accountID
    note.updated = new Date(Date.now()).toISOString()
    // save the note
    let result = await data.notes.put(note)
    // log it to stdout
    console.log(result)
  }
  catch(e) {
    console.log(e)
  }
  res({
    location: url('/')
  })
}

exports.handler = arc.http(auth, route)

```

This is cheating a little bit. We're directly overwriting the note record with `put`. A more complex example would probably use `update`. 

It can be helpful to inspect the data using the repl. To do that, first install `@architect/data` into the root of your project:

```bash
npm i @architect/data
```

And now `npx repl` opens a repl into your Dynamo schema running locally and in-memory. If you are running the app with `npx sandbox` in another tab, it connects to that database.

Try starting the repl and running: `data.notes.scan({}, console.log)` to see all the current notes. The repl can attach itself to the `staging` and `production` databases also by setting the appropriate `NODE_ENV` environment variable flag. 


## Delete a Note

Finally, let's add a delete button to our edit form:

```javascript
// src/html/get-notes-000noteID/_form
module.exports = function form({noteID, href, title, body}) {
  return `
<div class="card mt-5 mr-auto ml-auto mb-1 w-25">
  <div class=card-body>
    <form action=${href} method=post>
      <input type=hidden name=noteID value=${noteID}>
      <div class=form-group>
        <input 
          type=text 
          class=form-control 
          name=title 
          placeholder="Enter title" 
          value="${title}"
          required>
      </div>
      <div class=form-group>
        <textarea 
          class=form-control 
          name=body 
          placeholder="Enter text">${body}</textarea>
      </div>
      <button type=submit class="btn btn-primary float-right">Save</button>
    </form>
    <form action=${href}/del method=post>
      <button type=submit class="btn btn-danger float-right mr-2">Delete</button>
    </form>
  </div>
</div>
`
}
```

And implement a delete route:

```javascript
// src/http/post-notes-000noteID-del/index.js
let arc = require('@architect/functions')
let data = require('@architect/data')
let url = arc.http.helpers.url

async function route(req, res) {
  let noteID = req.params.noteID
  let accountID = req.session.account.accountID
  await data.notes.delete({
    noteID, accountID
  })
  res({
    location: url('/')
  })
}

exports.handler = arc.http(route)

```

> 🎩 Tip: `data._db` and `data._doc` return instances of `DynamoDB` and `DynamoDB.DocumentClient` for directly accessing your data; use `data._name` to resolve the table names with the app name and environment prefix.


## Go farther:

- [Example code repo](https://github.com/arc-repos/arc-example-persist-data)
- [Official aws-sdk DynamoDB docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html)
- [Official aws-sdk `DynamoDB.DocumentClient` docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html)
- [DynamoDB best practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Read the `@architect/data` reference](/reference/data)

<hr>


## Next: [Logging & Monitoring](/guides/logging)
