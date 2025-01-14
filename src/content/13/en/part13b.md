---
mainImage: ../../../images/part-13.svg
part: 13
letter: b
lang: en
---

<div class="content">

### Application structuring

So far, we have written all the code in the same file. Now let's structure the application a bit better. Let's create the following directory structure and files

```
index.js
util
  config.js
  db.js
models
  index.js
  note.js
controllers
  notes.js
```

The content of the files is as follows. The file `util/config.js` handles the environment variables:

```js
require('dotenv').config()

module.exports = {
  DATABASE_URL: process.env.DATABASE_URL,
  PORT: process.env.PORT || 3001,
}
```

The role of the `index.js` file is to configure and launch the application:

```js
const express = require('express')
const app = express()

const { PORT } = require('./util/config')
const { connectToDatabase } = require('./util/db')

const notesRouter = require('./controllers/notes')

app.use(express.json())

app.use('/api/notes', notesRouter)

const start = async () => {
  await connectToDatabase()
  app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`)
  })
}

start()
```

The application startup is slightly different from what we saw earlier, as we want to make sure that the database connection is established before the actual startup.

The file `util/db.js` contains the code to initialize the database:

```js
const Sequelize = require('sequelize')
const { DATABASE_URL } = require('./config')

const sequelize = new Sequelize(DATABASE_URL, {
  dialectOptions: {
    ssl: {
      require: true,
      rejectUnauthorized: false
    }
  },
});

const connectToDatabase = async () => {
  try {
    await sequelize.authenticate()
    console.log('database connected')
  } catch (err) {
    console.log('connecting database failed')
    return process.exit(1)
  }

  return null
}

module.exports = { connectToDatabase, sequelize }
```

The model corresponding to the table to be stored in the notes is stored in the file `models/note.js`:

```js
const { Model, DataTypes } = require('sequelize')

const { sequelize } = require('../util/db')

class Note extends Model {}

Note.init({
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  content: {
    type: DataTypes.TEXT,
    allowNull: false
  },
  important: {
    type: DataTypes.BOOLEAN
  },
  date: {
    type: DataTypes.DATE
  }
}, {
  sequelize,
  underscored: true,
  timestamps: false,
  modelName: 'note'
})

module.exports = Note
```

The file `models/index.js` is almost useless at this point as there is only one model in the application. As we add more models to the application, the file will become more useful because it will eliminate the need to import files defining individual models from the rest of the application.

```js
const Note = require('./note')

Note.sync()

module.exports = {
  Note
}
```

The route handlers for notes can be found in the file `controllers/notes.js`

```js
const router = require('express').Router()

const { Note } = require('../models')

router.get('/', async (req, res) => {
  const notes = await Note.findAll()
  res.json(notes)
})

router.post('/', async (req, res) => {
  try {
    const note = await Note.create(req.body)
    res.json(note)
  } catch(error) {
    return res.status(400).json({ error })
  }
})

router.get('/:id', async (req, res) => {
  const note = await Note.findByPk(req.params.id)
  if (note) {
    res.json(note)
  } else {
    res.status(404).end()
  }
})

router.delete('/:id', async (req, res) => {
  const note = await Note.findByPk(req.params.id)
  if (note) {
    await note.destroy()
  }
  res.status(204).end()
})

router.put('/:id', async (req, res) => {
  const note = await Note.findByPk(req.params.id)
  if (note) {
    note.important = req.body.important
    await note.save()
    res.json(note)
  } else {
    res.status(404).end()
  }
})

module.exports = router
```

The application structure is now good. However, we note that the router handlers that handle a single note contain a bit of repetitive code, since they all start with the line that searches for the note to be handled:

````js
const note = await Note.findByPk(req.params.id)
```

Let's refactor this into our own <i>middleware</i> and implement it in route handlers:

```js
const noteFinder = async (req, res, next) => {
  req.note = await Note.findByPk(req.params.id)
  next()
} 

router.get('/:id', noteFinder, async (req, res) => {
  if (req.note) {
    res.json(req.note)
  } else {
    res.status(404).end()
  }
})

router.delete('/:id', noteFinder, async (req, res) => {
  if (req.note) {
    await req.note.destroy()
  }
  res.status(204).end()
})

router.put('/:id', noteFinder, async (req, res) => {
  if (req.note) {
    req.note.important = req.body.important
    await req.note.save()
    res.json(req.note)
  } else {
    res.status(404).end()
  }
})
```

The route handlers now receive <i>three</i> parameters, the first being a string indicating the route and the second being the middleware `noteFinder` we defined, which retrieves the note from the database and places it in the `note` field of the `req` entity. A small amount of copypaste is eliminated and we are satisfied!

The current code for the application is available in full at [github](https://github.com/fullstack-hy/part122-notes/tree/part12-2), branch <i>part12-2</i>.

</div>

<div class="tasks">

### Tasks 13.5.-13.7.

#### Task 13.5.

Change the structure of your application to match the example above, or to follow some other similar clear convention.

#### Task 13.6.

Also implement support for changing the number of blogs' likes, i.e. the operation

- PUT api/blogs/:id (modify the like count of a blog)
#### Task 13.7.

Centralize the error handling of the application in middleware as in [part 3](/part3/data_storage_in_mongo_db_database#debugging_centralization_in_middleware). You can also enable middleware [express-async-errors](https://github.com/davidbanham/express-async-errors) as we did in [part 4](/part4/backend_testing#try-catchin-elimination).

The data returned with the error message is of little importance. However, it is likely that the frontend cannot interpret it correctly.

At this point, the situations that require error handling by the application are creating a new blog and changing the number of likes on a blog. Make sure that the error handler handles both of these appropriately.

</div>

<div class="content">

### User management

Next, add a database table `users` to the application to store the users of the application. In addition, the possibility of creating users and token-based login as in [part 4](/part4/token_based_login) is implemented. For simplicity, we will now implement the implementation so that all users have the same password `secret`.

The user-defining model in `models/user.js` is fairly straightforward

```js
const { Model, DataTypes } = require('sequelize')

const { sequelize } = require('../util/db')

class User extends Model {}

User.init({
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  username: {
    type: DataTypes.STRING,
    unique: true,
    allowNull: false
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
}, {
  sequelize,
  underscored: true,
  timestamps: false,
  modelName: 'user'
})

module.exports = User
```

The username is conditioned to be unique. The user ID could in principle have been used as the main key of the table. However, we decided to create the master key as a separate field with integer value <i>id</i>.


The file <i>models/index.js</i> expands slightly

```js
const Note = require('./note')
const User = require('./user') // highlight-line

User.sync() // highlight-line

module.exports = {
  Note, User // highlight-line
}
```

The route handlers in <i>controllers/users.js</i> that create a new user and display all users do not contain anything dramatic

```js
const router = require('express').Router()

const { User } = require('../models')

router.get('/', async (req, res) => {
  const users = await User.findAll()
  res.json(users)
})

router.post('/', async (req, res) => {
  try {
    const user = await User.create(req.body)
    res.json(user)
  } catch(error) {
    return res.status(400).json({ error })
  }
})

router.get('/:id', async (req, res) => {
  const user = await User.findByPk(req.params.id)
  if (user) {
    res.json(user)
  } else {
    res.status(404).end()
  }
})

module.exports = router
```

The router handler (file <i>controllers/login.js</i>) that handles logging is as follows:

```js
const jwt = require('jsonwebtoken')
const router = require('express').Router()

const { SECRET } = require('../util/config')
const User = require('../models/user')

router.post('/', async (request, response) => {
  const body = request.body

  const user = await User.findOne({ 
    where: { 
      username: body.username
    }
  })
  
const passwordCorrect = body.password === 'secret'
  if (!(user && passwordCorrect)) {
    return response.status(401).json({
      error: 'invalid username or password'
    })
  }

  const userForToken = {
    username: user.username, 
    id: user.id,
  }

  const token = jwt.sign(userForToken, SECRET)

  response
    .status(200)
    .send({ token, username: user.username, name: user.name })
})

module.exports = router
```

The mail request will be accompanied by a user name (<i>username</i>) and password (<i>password</i>). First, the entity corresponding to the user is retrieved from the database using the `User` method of the model [findOne](https://sequelize.org/master/manual/model-querying-finders.html#-code-findone--code-): 

```js
const user = await User.findOne({ 
  where: { 
    username: body.username
  }
})
```

In the console, we see the SQL statement corresponding to the method call

```sql
SELECT "id", "username", "name" 
FROM "users" AS "User" 
WHERE "User". "username" = 'mluukkai';
```

If the user is found and the password is correct (i.e. _secret_ for all users), `jsonwebtoken` is returned to the caller containing the user's information. To do this, we install 
dependency

```js
npm install jsonwebtoken
```

The `index.js` file will expand slightly

```js
const notesRouter = require('./controllers/notes')
const usersRouter = require('./controllers/users')
const loginRouter = require('./controllers/login')

app.use(express.json())

app.use('/api/notes', notesRouter)
app.use('/api/users', usersRouter)
app.use('/api/login', loginRouter)
```

The current code for the app is available in full at [github](https://github.com/fullstack-hy/part12-notes/tree/part12-3), branch <i>part12-3</i>.

### Inter-table join

Users can now be added to the application and users can log in, but by itself this is not yet a very useful feature. The idea is that only a logged in user can add notes, and that each note is associated with the user who created it. To do this, we need a <i>reference key</i> to store the notes in the `notes` table. 

When using Sequelize, the reference key can be specified by modifying the `models/index.js` file as follows

```js
const Note = require('./note')
const User = require('./user')

// hightlight-start
User.hasMany(Note)
Note.belongsTo(User)

Note.sync({ alter: true })
User.sync({ alter: true })
// hightlight-end

module.exports = {
  Note, User
}
```

So this [defines](https://sequelize.org/master/manual/assocs.html#one-to-one-relationships) that there is a _one to many_ relationship between the `users` and `notes` lines. We also changed the `sync` calls to change the tables if there are changes to the table definition. Now looking at the database schema from the console, it looks like this:

```js
username=> \d users
                                     Table "public.users"
  Column | Type | Collation | Nullable | Default
----------+------------------------+-----------+----------+-----------------------------------
 id | integer | not null | nextval('users_id_seq'::regclass)
 username | character varying(255) | | not null |
 name | character varying(255) | | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "notes" CONSTRAINT "notes_user_id_fkey" FOREIGN KEY (user_id) REFERENCES users(id) ON UPDATE CASCADE ON DELETE SET NULL

username=> \d notes
                                      Table "public.notes"
  Column | Type | Collation | Nullable | Default
-----------+--------------------------+-----------+----------+-----------------------------------
 id | integer | not null | nextval('notes_id_seq'::regclass)
 content | text | | not null |
 important | boolean | | | |
 date | timestamp with time zone | | | |
 user_id | integer | | | |
Indexes:
    "notes_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "notes_user_id_fkey" FOREIGN KEY (user_id) REFERENCES users(id) ON UPDATE CASCADE ON DELETE SET NULL
```

That is, a reference key `user_id` has been created in the `notes` table, which refers to the `users` row of the table.

Let us now make a change to the insertion of a new note to associate the note with a user. Before we do a proper implementation (where the join is done with a token to the user who is logged in), let's join the note to the first user found in the database:

```js

router.post('/', async (req, res) => {
  try {
    // hightlight-start
    const user = await User.findOne()
    const note = await Note.create({...req.body, userId: user.id})
    // hightlight-end
    res.json(note)
  } catch(error) {
    return res.status(400).json({ error })
  }
})
```

What is noteworthy in the code is that although there is a column `user\_id` in the notes at the database level, in the corresponding entity in the database row it is referred to in the camel case as `userId`.

Making a simple connection query is very easy. Let's change the route that shows all users so that it also shows the notes of each user:

```js
router.get('/', async (req, res) => {
  // highlight-start
  const users = await User.findAll({ 
    include: {
      model: Note
    }
  })
  // highlight-end
  res.json(users)
})
```

So the join query is done using the [include](https://sequelize.org/master/manual/assocs.html#eager-loading-example) wrapper on the query parameter.

The sql statement generated from the query is seen in the console:

```
SELECT "User". "id", "User". "username", "User". "name", "Notes". "id" AS "Notes.id", "Notes". "content" AS "Notes.content", "Notes". "important" AS "Notes.important", "Notes". "date" AS "Notes.date", "Notes". "user_id" AS "Notes.UserId" 
FROM "users" AS "User" LEFT OUTER JOIN "notes" AS "Notes" ON "User". "id" = "Notes". "user_id";
```

The end result is also as you might expect

IMAGE

_TODO: where in include is an example (e.g. notes where `important: true`)?_

### Proper insertion of notes

Incidentally, note insertion should work as in [section 4](/section4), i.e. note creation will only succeed if a valid token accompanies the request for creation. The note is stored in the list of notes created by the user identified by the token:

```js
// highlight-start
const tokenExtractor = (req, res, next) => {
  const authorization = req.get('authorization')
  if (authorization && authorization.toLowerCase().startsWith('bearer ')) {
    try {
      req.decodedToken = jwt.verify(authorization.substring(7), SECRET)
    } catch{
      res.status(401).json({ error: 'token invalid' })
    }
  } } else {
    res.status(401).json({ error: 'token missing' })
  }
  next()
}
// highlight-end

router.post('/', tokenExtractor, async (req, res) => {
  try {
    // highlight-start
    const user = await User.findByPk(req.decodedToken.id)
    const note = await Note.create({...req.body, userId: user.id, date: new Date()})
    // highlight-end
    res.json(note)
  } catch(error) {
    return res.status(400).json({ error })
  }
})
```

The token is taken and decoded from the request headers and placed in the <i>req</i> token by the <i>tokenExtractor</i> middleware. When a token is created, the <i>date</i> field indicating the time of its creation is also given a value.

### Fine-tuning

Except for error handling, our backend currently works almost identically to the Part 4 version of the same application. Before we make some extensions to the backend, let's change the routes of all notes and all users slightly.

We will add a note with information about the user who added it:

```js
router.get('/', async (req, res) => {
  const notes = await Note.findAll({ 
    attributes: { exclude: ['userId'] },
    include: {
      model: user,
      attributes: ['name']
    }
  })
  res.json(notes)
})
```

We have also [constrained](https://sequelize.org/master/manual/model-querying-basics.html#specifying-attributes-for-select-queries) which field values we want. For notes, we take all fields except `userId` and for the user associated with the note, only `name`.

Let's make a similar change to the route for all users, removing the unnecessary field `userId` from the notes associated with the user: 

```js
router.get('/', async (req, res) => {
  const users = await User.findAll({ 
    include: {
      model: note,
      attributes: { exclude: ['userId'] } // highlight-line
    }
  })
  res.json(users)
})
```

The current code for the application is available in its entirety on [github](https://github.com/fullstack-hy/part12-notes/tree/part12-4), branch <i>part12-4</i>.''

</div>

<div class="tasks">

### Tasks 13.8.-13.11.

#### Task 13.8.

Add support for users to the application. Users have the following fields in addition to their ID:

- name (string, must not be empty)
- username (string, must not be empty)

Unlike in the material, do not now prevent Sequelize from creating [timestamps](https://sequelize.org/master/manual/model-basics.html#timestamps) <i>created\_at</i> and <i>updated\_at</i> for users

All users can have the same password as the material. You can also choose to implement the password properly as in [part 4](/part4/user_management).

Implement the following routines 

- POST api/users (add new user)
- GET api/users (list all users)
- PUT api/users/:username (change user name, note that the parameter is not id but username)

Make sure that the timestamps <i>created_at</i> and <i>updated_at</i> automatically set by Sequelize work correctly when creating a user and changing the user name.

#### Exercise 13.9.

Sequelize provide a set of predefined
[validations](https://sequelize.org/master/manual/validations-and-constraints.html) for model fields, which it performs before storing the entities in the database.

It is decided to change the username creation policy so that only a valid email address is valid as username. Make a validation in connection with the creation of the ID to check this.

Modify the error handling middleware to provide a more descriptive error message in the situation (using the Sequelize error message), e.g.

```js
{
    "error": [
        "Validation isEmail on username failed"
    ]
}
```

#### Exercise 13.10.

Extend the application so that the blog is attached to a logged-in user identified by a token.

#### Exercise 13.11.

Make it possible to delete a blog only for the user who added the blog.

#### Task 13.12.

Modify the route for blogs and users so that the blogs show the user who added the blog and the user shows the user's blogs.

</div>

<div class="content">

## More queries

So far our application has been very simple in terms of queries, queries have either retrieved a single row based on the master key using the METHOD [findByPk](https://sequelize.org/master/class/lib/model.js~Model.html#static-method-findByPk) or they have retrieved all rows in the table using the method [findAll](https://sequelize.org/master/class/lib/model.js~Model.html#static-method-findAll). These are sufficient for the frontend of the application made in Section 5, but let's extend the backend so that we can also practice making slightly more complex queries.

Let's first implement the possibility to retrieve only important or non-important notes. Let's implement these using the [query-parameter](http://expressjs.com/en/5x/api.html#req.query) important:

```js
router.get('/', async (req, res) => {
  const notes = await Note.findAll({ 
    attributes: { exclude: ['userId'] },
    include: {
      model: user,
      attributes: ['name']
    },
    // highlight-start
    where: {
      important: req.query.important === "true"
    }
    // highlight-end
  })
  res.json(notes)
})
```

Now the backend can retrieve important notes with a request to http://localhost:3001/api/notes?important=true and non-important notes with a request to http://localhost:3001/api/notes?important=false

The SQL query generated by Sequelize naturally contains a where clause delimiting the rows to be returned: 

```sql
SELECT "note". "id", "note". "content", "note". "important", "note". "date", "user". "id" AS "user.id", "user". "name" AS "user.name" 
FROM "notes" AS "note" LEFT OUTER JOIN "users" AS "user" ON "note". "user_id" = "user". "id" 
WHERE "note". "important" = true;
```

Unfortunately, this implementation will not work if the request is not interested in whether the note is important or not, i.e. if the request is made to http://localhost:3001/api/notes. The fix can be done in several ways. One, but perhaps not the best way to do the correction would be as follows:

```js
const { Op } = require('sequelize')

router.get('/', async (req, res) => {
  //highlight-line
  let important = {
    [Op.in]: [true, false]
  }
  
  if ( req.query.important ) {
    important = req.query.important === "true"
  }
  //highlight-end
  
  const notes = await Note.findAll({ 
    attributes: { exclude: ['userId'] },
    include: {
      model: user,
      attributes: ['name']
    },
    where: {
      important // highlight-line
    }
  })
  res.json(notes)
})
```

The `important` object now stores the query condition. It defaults to 

```js
where: {
  important: {
    [Op.in]: [true, false]
  }
}
```

i.e. the column `important` can be `true` or `false`, using one of the many Sequelize operations [Op.in](https://sequelize.org/master/manual/model-querying-basics.html#operators). If the query parameter `req.query.important` is specified, the query will take one of two forms

```js
where: {
  important: true
}
```

or

```js
where: {
  important: true
}
```

depending on the value of the query parameter.

Extend the functionality further by allowing you to specify the required keyword when retrieving notes, e.g. a request to http://localhost:3001/api/notes?search=database will return all notes with `database` or a request to http://localhost:3001/api/notes?search=javascript&important=true will return all marked notes with `javascript`. The implementation is as follows

```js
router.get('/', async (req, res) => {
  let important = {
    [Op.in]: [true, false]
  }
  
  if ( req.query.important ) {
    important = req.query.important === "true"
  }

  const notes = await Note.findAll({ 
    attributes: { exclude: ['userId'] },
    include: {
      model: user,
      attributes: ['name']
    },
    where: {
      important,
      // highlight-start
      content: {
        [Op.substring]: req.query.search ? req.query.search : ''
      }
      // highlight-end
    }
  })
  
  res.json(notes)
})
```

Sequelizen [Op.substring](https://sequelize.org/master/manual/model-querying-basics.html#operators) constructs the query we want using the like keyword in SQL. For example, if we make a query to http://localhost:3001/api/notes?search=database&important=true we will see that the SQL query it generates is exactly as we assumed.

```sql
SELECT "note". "id", "note". "content", "note". "important", "note". "date", "user". "id" AS "user.id", "user". "name" AS "user.name" 
FROM "notes" AS "note" LEFT OUTER JOIN "users" AS "user" ON "note". "user_id" = "user". "id" 
WHERE "note". "important" = true AND "note". "content" LIKE '%database%';
```

Another bug in our application is that if we make a request to http://localhost:3001/api/notes, i.e. we want all the notes, our implementation will cause the query to cause an unnecessary wheren, which (depending on the implementation of the database engine) may unnecessarily affect the query execution time:

```sql
SELECT "note". "id", "note". "content", "note". "important", "note". "date", "user". "id" AS "user.id", "user". "name" AS "user.name" 
FROM "notes" AS "note" LEFT OUTER JOIN "users" AS "user" ON "note". "user_id" = "user". "id" 
WHERE "note". "important" IN (true, false) AND "note". "content" LIKE '%%';
```

Let's further optimize the code so that the where-conditions are used only when necessary:

```js
router.get('/', async (req, res) => {
  const where = {}

  if (req.query.important) {
    where.important = req.query.important === "true"
  } 

  if (req.query.search) {
    where.content = {
      [Op.substring]: req.query.search
    }
  }

  const notes = await Note.findAll({ 
    attributes: { exclude: ['userId'] },
    include: {
      model: user,
      attributes: ['name']
    },
    where
  })

  res.json(notes)
})
```

If the query contains search conditions, e.g. http://localhost:3001/api/notes?search=database&important=true, a query containing wheren is generated

```sql
SELECT "note". "id", "note". "content", "note". "important", "note". "date", "user". "id" AS "user.id", "user". "name" AS "user.name" 
FROM "notes" AS "note" LEFT OUTER JOIN "users" AS "user" ON "note". "user_id" = "user". "id"
WHERE "note". "important" = true AND "note". "content" LIKE '%database%';
```

If the request is not searchable http://localhost:3001/api/notes there is no unnecessary where in the query

```sql
SELECT "note". "id", "note". "content", "note". "important", "note". "date", "user". "id" AS "user.id", "user". "name" AS "user.name" 
FROM "notes" AS "note" LEFT OUTER JOIN "users" AS "user" ON "note". "user_id" = "user". "id";
```

The current code of the application is available in its entirety on [github](https://github.com/fullstack-hy/part12-notes/tree/part12-5), branch <i>part12-5</i>.

</div>

<div class="tasks">

### Tasks 13.13.-13.16

#### Task 13.13.

Implement filtering by keyword in the application for the route returning all blogs. The filtering works as follows
- GET http://localhost:3003/api/blogs?serch=react returns all blogs with the search term <i>react</i> in the <i>title</i> field, the search term is not case sensitive
- GET http://localhost:3003/api/blogs returns all blogs


[This](https://sequelize.org/master/manual/model-querying-basics.html#operators) should be useful for this and the next task.
#### Exercise 13.14.

Extend the filter to search for a keyword in the <i>title</i> and author <i>author</i> fields, i.e.

- GET http://localhost:3003/api/blogs?serch=jami returns blogs with the search term <i>jami</i> in the <i>title</i> field or <i>author</i> in the <i>author</i> field
#### Exercise 13.15.

Modify the blog route so that it returns blogs in descending order of likes. Look in [documentation](https://sequelize.org/master/manual/model-querying-basics.html) for instructions on ordering.

#### Task 13.16.

Make a route for the application http://localhost:3003/api/authors that returns the number of blogs for each author and the total number of likes. Implement the operation directly at the database level. You will most likely need the [group by](https://sequelize.org/master/manual/model-querying-basics.html#grouping) functionality, and the [sequelize.fn](https://sequelize.org/master/manual/model-querying-basics.html#specifying-attributes-for-select-queries) aggregator function.

The JSON returned by the route might look like the following, for example:

IMAGE

```
[
  {
    author: "Jami Kousa",
    articles: "3",
    likes: "10"
  },
  {
    author: "Kalle Ilves",
    articles: "1",
    likes: "2"
  },
  {
    author: "Dan Abramov",
    articles: "1",
    likes: "4"
  }
]
```

</div>
