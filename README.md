Let's build an all-in-one Personalized Health Tracker with features like fitness tracking, meal logging, sleep analysis, and mental well-being checks. This project will use a full stack approach with a Node.js backend, a React frontend, and MongoDB for data storage.

Backend: Node.js & MongoDB
Set Up Project:

shell
mkdir health-tracker
cd health-tracker
npm init -y
npm install express mongoose bcryptjs jsonwebtoken
Create Server (server.js):

javascript
const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const app = express();
const PORT = process.env.PORT || 3000;

mongoose.connect('mongodb://localhost:27017/health-tracker', { useNewUrlParser: true, useUnifiedTopology: true });

const userSchema = new mongoose.Schema({
    username: String,
    password: String,
    fitnessData: [{ date: Date, steps: Number, caloriesBurned: Number }],
    mealData: [{ date: Date, mealType: String, calories: Number, description: String }],
    sleepData: [{ date: Date, hoursSlept: Number, sleepQuality: String }],
    mentalWellBeing: [{ date: Date, mood: String, notes: String }]
});

const User = mongoose.model('User', userSchema);

app.use(express.json());

app.post('/register', async (req, res) => {
    const { username, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({ username, password: hashedPassword });
    await user.save();
    res.send({ message: 'User registered successfully' });
});

app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const user = await User.findOne({ username });
    if (!user || !await bcrypt.compare(password, user.password)) {
        return res.status(400).send({ message: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user._id }, 'secretKey');
    res.send({ token });
});

app.get('/data', async (req, res) => {
    const token = req.headers['authorization'];
    const decoded = jwt.verify(token, 'secretKey');
    const user = await User.findById(decoded.id);
    res.send(user);
});

app.post('/data', async (req, res) => {
    const token = req.headers['authorization'];
    const decoded = jwt.verify(token, 'secretKey');
    const user = await User.findById(decoded.id);

    const { fitnessData, mealData, sleepData, mentalWellBeing } = req.body;

    if (fitnessData) user.fitnessData.push(fitnessData);
    if (mealData) user.mealData.push(mealData);
    if (sleepData) user.sleepData.push(sleepData);
    if (mentalWellBeing) user.mentalWellBeing.push(mentalWellBeing);

    await user.save();
    res.send(user);
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
Frontend: React
Set Up Project:

shell
npx create-react-app health-tracker-frontend
cd health-tracker-frontend
npm install axios
Create Components: App.js:

javascript
import React from 'react';
import { BrowserRouter as Router, Route, Routes, Link } from 'react-router-dom';
import Register from './components/Register';
import Login from './components/Login';
import Dashboard from './components/Dashboard';

const App = () => {
    return (
        <Router>
            <nav>
                <Link to="/register">Register</Link>
                <Link to="/login">Login</Link>
                <Link to="/dashboard">Dashboard</Link>
            </nav>
            <Routes>
                <Route path="/register" element={<Register />} />
                <Route path="/login" element={<Login />} />
                <Route path="/dashboard" element={<Dashboard />} />
            </Routes>
        </Router>
    );
};

export default App;
Register.js:

javascript
import React, { useState } from 'react';
import axios from 'axios';

const Register = () => {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        await axios.post('http://localhost:3000/register', { username, password });
        setUsername('');
        setPassword('');
    };

    return (
        <form onSubmit={handleSubmit}>
            <input 
                type="text" 
                value={username} 
                onChange={(e) => setUsername(e.target.value)} 
                placeholder="Username" 
            />
            <input 
                type="password" 
                value={password} 
                onChange={(e) => setPassword(e.target.value)} 
                placeholder="Password" 
            />
            <button type="submit">Register</button>
        </form>
    );
};

export default Register;
Login.js:

javascript
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        const response = await axios.post('http://localhost:3000/login', { username, password });
        localStorage.setItem('token', response.data.token);
        setUsername('');
        setPassword('');
    };

    return (
        <form onSubmit={handleSubmit}>
            <input 
                type="text" 
                value={username} 
                onChange={(e) => setUsername(e.target.value)} 
                placeholder="Username" 
            />
            <input 
                type="password" 
                value={password} 
                onChange={(e) => setPassword(e.target.value)} 
                placeholder="Password" 
            />
            <button type="submit">Login</button>
        </form>
    );
};

export default Login;
Dashboard.js:

javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const Dashboard = () => {
    const [data, setData] = useState(null);
    const [fitnessData, setFitnessData] = useState({ steps: '', caloriesBurned: '' });
    const [mealData, setMealData] = useState({ mealType: '', calories: '', description: '' });
    const [sleepData, setSleepData] = useState({ hoursSlept: '', sleepQuality: '' });
    const [mentalWellBeing, setMentalWellBeing] = useState({ mood: '', notes: '' });

    useEffect(() => {
        const fetchData = async () => {
            const token = localStorage.getItem('token');
            const response = await axios.get('http://localhost:3000/data', {
                headers: { 'Authorization': token }
            });
            setData(response.data);
        };
        fetchData();
    }, []);

    const handleSubmit = async (type, data) => {
        const token = localStorage.getItem('token');
        await axios.post('http://localhost:3000/data', { [type]: data }, {
            headers: { 'Authorization': token }
        });
        const response = await axios.get('http://localhost:3000/data', {
            headers: { 'Authorization': token }
        });
        setData(response.data);
    };

    return (
        <div>
            <h1>Dashboard</h1>
            <h2>Fitness Data</h2>
            <form onSubmit={(e) => { e.preventDefault(); handleSubmit('fitnessData', fitnessData); }}>
                <input
                    type="number"
                    placeholder="Steps"
                    value={fitnessData.steps}
                    onChange={(e) => setFitnessData({<div align="center">
    <picture>
        <source media="(prefers-color-scheme: light)" srcset="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg">
        <source media="(prefers-color-scheme: dark)" srcset="https://user-images.githubusercontent.com/7659/174594559-0b3ddaa7-e75b-4f10-9dee-b51431a9fd4c.svg">
        <img src="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg" alt="Dependabot" width="336">
    </picture>
</div>

## Dependabot Demo Repository

This repo contains some projects with outdated dependencies. Fork it to try out
Dependabot :dependabot:!

### Enabling Security Updates

- In your fork, click the **Settings** tab
- In the left hand side navigation, click **Code security and analysis**
- Enable **Dependabot security updates** or **Grouped security updates**
- Dependabot will now start creating PRs for detected security vulnerabilities
- Go into the **Security** tab and click **Dependabot** in the left hand side navigation to see what Dependabot is working on

<img width="929" alt="screenshot showing Dependabot working on Security Updates" src="https://github.com/dependabot/demo/assets/886768/9295c61a-631b-4c56-9c00-ff078874f362">

After about 5 minutes you should see some PRs open. Merge them and the Securty Alerts will close 🎉

### Enabling Version Updates

This demo includes a `dependabot.yml` which configures [Version Updates](https://docs.github.com/github/administering-a-repository/keeping-your-dependencies-updated-automatically), but forks don't automatically start with Dependabot enabled.

The enable Dependabot on your fork:
- Click the **Insights** tab
- In the left hand side navigation, click **Dependency Graph**
- Click on the **Dependabot** tab
- Click on the **Enable Dependabot** button
- After a moment, refresh the page and you should see Dependabot hard at work

<img width="917" alt="screenshot showing Dependabot working on Version Updates" src="https://github.com/dependabot/demo/assets/886768/4adf5727-255a-4ae1-97f7-70e94dc1134b">

After a few minutes, you should get some more PRs!
