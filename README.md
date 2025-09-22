# To-Do-List-API
What you get

server.js — app entry (Express + middleware + routes)
models/Todo.js — Mongoose todo schema
routes/todos.js — CRUD + filters + pagination endpoints
package.json dependencies + scripts
example .env variables and curl examples

1) package.json
Create package.json (or run npm init -y) then install packages:
npm install express mongoose dotenv body-parser cors express-validator

# optional dev:
npm install --save-dev nodemon
package.json (snippet):

{
  "name": "todo-api",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "body-parser": "^1.20.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "express-validator": "^6.14.3",
    "mongoose": "^7.0.0"
  }
}

2) .env
Create .env at project root:

PORT=4000
MONGO_URI=mongodb://localhost:27017/todo_db


(Replace MONGO_URI with your remote MongoDB connection string if needed.)

3) models/Todo.js
Create folder models/ and file Todo.js:

// models/Todo.js
const mongoose = require('mongoose');

const TodoSchema = new mongoose.Schema({
  title: { type: String, required: true, trim: true },
  description: { type: String, default: '', trim: true },
  completed: { type: Boolean, default: false },
  dueDate: { type: Date, default: null },
  priority: { type: String, enum: ['low','medium','high'], default: 'medium' },
  tags: [{ type: String }],
  // optional user field (if you add auth later)
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', default: null }
}, { timestamps: true });

module.exports = mongoose.model('Todo', TodoSchema);

4) routes/todos.js
Create folder routes/ and file todos.js:

// routes/todos.js
const express = require('express');
const router = express.Router();
const { body, param, query, validationResult } = require('express-validator');
const Todo = require('../models/Todo');

// helper
const handleErrors = (res, err) => res.status(500).json({ error: err.message || 'Server error' });

// POST /api/todos  - create a todo
router.post('/',
  body('title').isLength({ min: 1 }).withMessage('Title is required'),
  body('priority').optional().isIn(['low','medium','high']),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

    try {
      const todo = new Todo(req.body);
      await todo.save();
      return res.status(201).json(todo);
    } catch (err) {
      return handleErrors(res, err);
    }
  }
);

// GET /api/todos - list with filtering & pagination
// query: ?page=1&limit=10&completed=true&priority=high&q=search
router.get('/',
  query('page').optional().toInt(),
  query('limit').optional().toInt(),
  async (req, res) => {
    try {
      const page = Math.max(1, req.query.page || 1);
      const limit = Math.min(100, req.query.limit || 20);
      const skip = (page - 1) * limit;

      const filter = {};
      if (req.query.completed !== undefined) filter.completed = req.query.completed === 'true';
      if (req.query.priority) filter.priority = req.query.priority;
      if (req.query.tag) filter.tags = req.query.tag;
      if (req.query.q) {
        const q = req.query.q;
        filter.$or = [
          { title: new RegExp(q, 'i') },
          { description: new RegExp(q, 'i') }
        ];
      }

      const [items, total] = await Promise.all([
        Todo.find(filter).sort({ createdAt: -1 }).skip(skip).limit(limit).lean(),
        Todo.countDocuments(filter)
      ]);

      res.json({ page, limit, total, items });
    } catch (err) {
      handleErrors(res, err);
    }
  }
);

// GET /api/todos/:id - get single
router.get('/:id', param('id').isMongoId(), async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  try {
    const t = await Todo.findById(req.params.id);
    if (!t) return res.status(404).json({ error: 'Not found' });
    res.json(t);
  } catch (err) { handleErrors(res, err); }
});

// PUT /api/todos/:id - update
router.put('/:id',
  param('id').isMongoId(),
  body('priority').optional().isIn(['low','medium','high']),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });
    try {
      const updated = await Todo.findByIdAndUpdate(req.params.id, req.body, { new: true });
      if (!updated) return res.status(404).json({ error: 'Not found' });
      res.json(updated);
    } catch (err) { handleErrors(res, err); }
  }
);

// PATCH /api/todos/:id/toggle - toggle completed
router.patch('/:id/toggle', param('id').isMongoId(), async (req, res) => {
  try {
    const todo = await Todo.findById(req.params.id);
    if (!todo) return res.status(404).json({ error: 'Not found' });
    todo.completed = !todo.completed;
    await todo.save();
    res.json(todo);
  } catch (err) { handleErrors(res, err); }
});

// DELETE /api/todos/:id
router.delete('/:id', param('id').isMongoId(), async (req, res) => {
  try {
    const t = await Todo.findByIdAndDelete(req.params.id);
    if (!t) return res.status(404).json({ error: 'Not found' });
    res.json({ success: true });
  } catch (err) { handleErrors(res, err); }
});

// DELETE /api/todos  (bulk delete by filter) e.g. ?completed=true
router.delete('/', async (req, res) => {
  try {
    const filter = {};
    if (req.query.completed !== undefined) filter.completed = req.query.completed === 'true';
    const r = await Todo.deleteMany(filter);
    res.json({ deletedCount: r.deletedCount });
  } catch (err) { handleErrors(res, err); }
});

module.exports = router;

5) server.js
Root server.js:

// server.js
require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const cors = require('cors');

const todosRoute = require('./routes/todos');

const PORT = process.env.PORT || 4000;
const MONGO_URI = process.env.MONGO_URI;

const app = express();

app.use(cors());
app.use(bodyParser.json({ limit: '2mb' }));
app.use(bodyParser.urlencoded({ extended: true }));

// health
app.get('/', (req, res) => res.send({ status: 'ok', env: process.env.NODE_ENV || 'dev' }));

// routes
app.use('/api/todos', todosRoute);

// global error handler (basic)
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
});

// start
mongoose.connect(MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => {
    console.log('MongoDB connected');
    app.listen(PORT, () => console.log(`Server listening on ${PORT}`));
  })
  .catch(err => {
    console.error('Failed to connect to MongoDB', err);
    process.exit(1);
  });

6) Quick test with curl
Start server: npm run dev (if using nodemon) or npm start.

Create a todo:

curl -X POST http://localhost:4000/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Finish assignment","description":"Finish ToDo API","dueDate":"2025-09-30","priority":"high","tags":["school","backend"]}'

List todos (first page):
curl "http://localhost:4000/api/todos?page=1&limit=10"


Get single todo:
curl http://localhost:4000/api/todos/<id>


Update:
curl -X PUT http://localhost:4000/api/todos/<id> -H "Content-Type: application/json" -d '{"title":"Updated"}'


Toggle complete:
curl -X PATCH http://localhost:4000/api/todos/<id>/toggle


Delete:
curl -X DELETE http://localhost:4000/api/todos/<id>


Search by keyword:
curl "http://localhost:4000/api/todos?q=assignment"


Filter by priority:

curl "http://localhost:4000/api/todos?priority=high"
