# Ethnus
Project Setup
Backend: Node.js with Express
1.	Initialize Project
       mkdir expense-tracker
cd expense-tracker
npm init -y
2.	Install Dependencies

npm install express mongoose bcryptjs jsonwebtoken dotenv cors
npm install --save-dev nodemon eslint prettier
3.	Setup Project Structure

mkdir server client
cd server
mkdir models routes controllers middlewares config
touch server.js .env
Frontend: React
1.	Initialize React App

npx create-react-app client
cd client
npm install axios redux react-redux react-router-dom recharts
Backend Implementation
           1.Environment Variables (server/.env)
                      PORT=5000
                      MONGO_URI=your_mongo_uri
                      JWT_SECRET=your_jwt_secret
2.	Server Setup (server/server.js)

const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const dotenv = require('dotenv');

dotenv.config();

const app = express();

app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
}).then(() => console.log('MongoDB Connected'))
  .catch(err => console.log(err));

// Routes
const userRoutes = require('./routes/user');
const expenseRoutes = require('./routes/expense');

app.use('/api/users', userRoutes);
app.use('/api/expenses', expenseRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
3.	User Model (server/models/User.js)

const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
  },
  password: {
    type: String,
    required: true,
  },
});

module.exports = mongoose.model('User', UserSchema);
4.	Expense Model (server/models/Expense.js)

const mongoose = require('mongoose');

const ExpenseSchema = new mongoose.Schema({
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  date: {
    type: Date,
    required: true,
  },
  amount: {
    type: Number,
    required: true,
  },
  category: {
    type: String,
    required: true,
  },
  description: {
    type: String,
  },
});

module.exports = mongoose.model('Expense', ExpenseSchema);
5.	User Routes (server/routes/user.js)

const express = require('express');
const { registerUser, loginUser } = require('../controllers/userController');

const router = express.Router();

router.post('/register', registerUser);
router.post('/login', loginUser);

module.exports = router;
6.	Expense Routes (server/routes/expense.js) 

const express = require('express');
const { addExpense, getExpenses, updateExpense, deleteExpense } = require('../controllers/expenseController');
const auth = require('../middlewares/auth');

const router = express.Router();

router.post('/', auth, addExpense);
router.get('/', auth, getExpenses);
router.put('/:id', auth, updateExpense);
router.delete('/:id', auth, deleteExpense);

module.exports = router;
7.	User Controller (server/controllers/userController.js)
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const registerUser = async (req, res) => {
  const { username, email, password } = req.body;
  try {
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ msg: 'User already exists' });
    }
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const newUser = new User({
      username,
      email,
      password: hashedPassword,
    });
    
    await newUser.save();
    
    const token = jwt.sign({ id: newUser._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ token, user: { id: newUser._id, username: newUser.username, email: newUser.email } });
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

const loginUser = async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ token, user: { id: user._id, username: user.username, email: user.email } });
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

module.exports = { registerUser, loginUser };
8.	Expense Controller (server/controllers/expenseController.js)
const Expense = require('../models/Expense');

const addExpense = async (req, res) => {
  const { date, amount, category, description } = req.body;
  try {
    const newExpense = new Expense({
      user: req.user.id,
      date,
      amount,
      category,
      description,
    });
    
    const savedExpense = await newExpense.save();
    res.json(savedExpense);
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

const getExpenses = async (req, res) => {
  try {
    const expenses = await Expense.find({ user: req.user.id });
    res.json(expenses);
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

const updateExpense = async (req, res) => {
  const { date, amount, category, description } = req.body;
  try {
    const expense = await Expense.findById(req.params.id);
    if (!expense) {
      return res.status(404).json({ msg: 'Expense not found' });
    }
    if (expense.user.toString() !== req.user.id) {
      return res.status(401).json({ msg: 'Unauthorized' });
    }
    
    expense.date = date || expense.date;
    expense.amount = amount || expense.amount;
    expense.category = category || expense.category;
    expense.description = description || expense.description;
    
    const updatedExpense = await expense.save();
    res.json(updatedExpense);
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

const deleteExpense = async (req, res) => {
  try {
    const expense = await Expense.findById(req.params.id);
    if (!expense) {
      return res.status(404).json({ msg: 'Expense not found' });
    }
    if (expense.user.toString() !== req.user.id) {
      return res.status(401).json({ msg: 'Unauthorized' });
    }
    
    await expense.remove();
    res.json({ msg: 'Expense removed' });
  } catch (err) {
    res.status(500).json({ msg: 'Server error' });
  }
};

module.exports = { addExpense, getExpenses, updateExpense, deleteExpense };
9.	Authentication Middleware (server/middlewares/auth.js)

const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  const token = req.header('Authorization');
  if (!token) {
    return res.status(401).json({ msg: 'No token, authorization denied' });
  }
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token is not valid' });
  }
};

module.exports = auth;
Frontend Implementation
1.	User Registration and Login
o	Create a folder structure:
bash
Copy code
cd client/src
mkdir components pages redux services
o	User Service (client/src/services/userService.js)
javascript
Copy code
import axios from 'axios';

const API_URL = '/api/users/';

const register = async (userData

