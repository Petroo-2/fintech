# fintech
 Backend Setup (Node.js + Express)
Initialize Project
```bash
mkdir fintech-backend
cd fintech-backend
npm init -y
npm install express mongoose bcryptjs jsonwebtoken cors dotenv
```

Basic Server (server.js)
```javascript
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

// MongoDB Connection
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/transactions', require('./routes/transactions'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

User Authentication (JWT)
routes/auth.js
```javascript
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  const { email, password } = req.body;
  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({ email, password: hashedPassword });
    await user.save();
    res.status(201).json({ message: 'User registered' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });
    if (!user) return res.status(400).json({ error: 'User not found' });

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) return res.status(400).json({ error: 'Invalid credentials' });

    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ token });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

Transaction Handling
models/Transaction.js
```javascript
const mongoose = require('mongoose');

const TransactionSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  amount: { type: Number, required: true },
  type: { type: String, enum: ['deposit', 'withdrawal', 'transfer'], required: true },
  recipient: { type: String }, // For transfers
  date: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Transaction', TransactionSchema);
```

routes/transactions.js
```javascript
const express = require('express');
const auth = require('../middleware/auth');
const Transaction = require('../models/Transaction');
const router = express.Router();

// Get user transactions (protected route)
router.get('/', auth, async (req, res) => {
  try {
    const transactions = await Transaction.find({ userId: req.user.id });
    res.json(transactions);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Add new transaction
router.post('/', auth, async (req, res) => {
  const { amount, type, recipient } = req.body;
  try {
    const transaction = new Transaction({ userId: req.user.id, amount, type, recipient });
    await transaction.save();
    res.status(201).json(transaction);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

---

 Frontend (React.js)
Initialize Project
```bash
npx create-react-app fintech-frontend
cd fintech-frontend
npm install axios react-router-dom @material-ui/core chart.js
```

Auth Context (JWT Storage)
context/AuthContext.js
```javascript
import { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    } else {
      delete axios.defaults.headers.common['Authorization'];
    }
  }, [token]);

  const login = async (email, password) => {
    const res = await axios.post('http://localhost:5000/api/auth/login', { email, password });
    localStorage.setItem('token', res.data.token);
    setToken(res.data.token);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export default AuthContext;
```

Dashboard with Transactions
components/Dashboard.js`
```javascript
import React, { useEffect, useState, useContext } from 'react';
import axios from 'axios';
import AuthContext from '../context/AuthContext';
import { Bar } from 'react-chartjs-2';

const Dashboard = () => {
  const { token } = useContext(AuthContext);
  const [transactions, setTransactions] = useState([]);

  useEffect(() => {
    if (token) {
      axios.get('http://localhost:5000/api/transactions')
        .then(res => setTransactions(res.data))
        .catch(err => console.error(err));
    }
  }, [token]);

  const data = {
    labels: transactions.map(t => new Date(t.date).toLocaleDateString()),
    datasets: [{
      label: 'Transaction Amount',
      data: transactions.map(t => t.amount),
      backgroundColor: transactions.map(t => 
        t.type === 'deposit' ? 'green' : t.type === 'withdrawal' ? 'red' : 'blue'
      ),
    }]
  };

  return (
    <div>
      <h2>Transactions</h2>
      <Bar data={data} />
      <ul>
        {transactions.map(tx => (
          <li key={tx._id}>
            {tx.type} - ${tx.amount} {tx.recipient && `to ${tx.recipient}`}
          </li>
        ))}
      </ul>
    </div>
  );
};

export default Dashboard;
```

---

 Mobile (React Native)
Initialize Project
```bash
npx react-native init FintechApp
cd FintechApp
npm install axios @react-navigation/native @react-navigation/stack react-native-chart-kit
```

Transaction Screen
screens/Transactions.js
```javascript
import React, { useState, useEffect, useContext } from 'react';
import { View, Text, FlatList } from 'react-native';
import axios from 'axios';
import AuthContext from '../context/AuthContext';
import { BarChart } from 'react-native-chart-kit';

const TransactionsScreen = () => {
  const { token } = useContext(AuthContext);
  const [transactions, setTransactions] = useState([]);

  useEffect(() => {
    if (token) {
      axios.get('http://localhost:5000/api/transactions')
        .then(res => setTransactions(res.data))
        .catch(err => console.error(err));
    }
  }, [token]);

  return (
    <View>
      <BarChart
        data={{
          labels: transactions.map(t => new Date(t.date).toLocaleDateString()),
          datasets: [{ data: transactions.map(t => t.amount) }]
        }}
        width={300}
        height={220}
        yAxisLabel="$"
        chartConfig={{
          backgroundColor: '#1cc910',
          backgroundGradientFrom: '#eff3ff',
          backgroundGradientTo: '#efefef',
          decimalPlaces: 2,
          color: (opacity = 1) => `rgba(0, 0, 0, ${opacity})`,
        }}
      />
      <FlatList
        data={transactions}
        keyExtractor={item => item._id}
        renderItem={({ item }) => (
          <Text>
            {item.type}: ${item.amount}
          </Text>
        )}
      />
    </View>
  );
};

export default TransactionsScreen;
