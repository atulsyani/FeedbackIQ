# FeedbackIQ
FeedbackIQ is a full-stack web application that allows users to submit written feedback and instantly receive an AI-powered sentiment analysis. This project demonstrates proficiency in React, Node.js, RESTful APIs, SQL, and applied AI using sentiment analysis.

// === Backend: server.js ===
const express = require('express');
const cors = require('cors');
const sqlite3 = require('sqlite3').verbose();
const Sentiment = require('sentiment');

const app = express();
const port = 5000;
const sentiment = new Sentiment();
const db = new sqlite3.Database('./feedback.db');

app.use(cors());
app.use(express.json());

// Initialize DB
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS feedback (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    message TEXT,
    sentiment TEXT,
    score INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )`);
});

// POST feedback
app.post('/api/feedback', (req, res) => {
  const { message } = req.body;
  const result = sentiment.analyze(message);
  const sentimentLabel = result.score > 0 ? 'positive' : result.score < 0 ? 'negative' : 'neutral';
  db.run(
    'INSERT INTO feedback (message, sentiment, score) VALUES (?, ?, ?)',
    [message, sentimentLabel, result.score],
    function (err) {
      if (err) return res.status(500).send(err.message);
      res.status(201).json({ id: this.lastID, message, sentiment: sentimentLabel, score: result.score });
    }
  );
});

// GET all feedback
app.get('/api/feedback', (req, res) => {
  db.all('SELECT * FROM feedback ORDER BY created_at DESC', [], (err, rows) => {
    if (err) return res.status(500).send(err.message);
    res.json(rows);
  });
});

app.listen(port, () => console.log(`Server running on port ${port}`));


// === Frontend: App.jsx ===
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [message, setMessage] = useState('');
  const [feedback, setFeedback] = useState([]);

  const fetchFeedback = async () => {
    const res = await axios.get('http://localhost:5000/api/feedback');
    setFeedback(res.data);
  };

  const submitFeedback = async (e) => {
    e.preventDefault();
    if (!message) return;
    await axios.post('http://localhost:5000/api/feedback', { message });
    setMessage('');
    fetchFeedback();
  };

  useEffect(() => {
    fetchFeedback();
  }, []);

  return (
    <div style={{ padding: 20, fontFamily: 'Arial' }}>
      <h1>FeedbackIQ</h1>
      <form onSubmit={submitFeedback}>
        <textarea value={message} onChange={e => setMessage(e.target.value)} rows={4} cols={50} />
        <br />
        <button type="submit">Submit Feedback</button>
      </form>

      <h2>Previous Feedback</h2>
      {feedback.map(f => (
        <div key={f.id} style={{ marginBottom: 10 }}>
          <p><strong>{f.sentiment.toUpperCase()}</strong> — {f.message}</p>
          <small>Score: {f.score} | Date: {new Date(f.created_at).toLocaleString()}</small>
        </div>
      ))}
    </div>
  );
}

export default App;


// === package.json (backend) ===
{
  "name": "feedbackiq-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "sentiment": "^5.0.2",
    "sqlite3": "^5.1.6"
  }
}


// === package.json (frontend) ===
{
  "name": "feedbackiq-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "axios": "^1.6.7",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "^5.0.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}
