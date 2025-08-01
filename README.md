life-shield/
│── render.yaml
│── README.md
│
├── backend/                 # Node.js + Express + MongoDB + JWT
│   ├── package.json
│   ├── server.js
│   ├── .env.example
│   ├── models/
│   │   ├── User.js
│   │   └── Proof.js
│   └── middleware/
│       └── errorHandler.js
│
├── frontend/                # React (Vite + Tailwind + JWT Auth)
│   ├── package.json
│   ├── vite.config.js
│   ├── .env.example
│   └── src/
│       ├── App.jsx
│       ├── Splash.jsx
│       ├── index.css
│       └── components/
│           ├── Login.jsx
│           ├── LoanCalculator.jsx
│           ├── About.jsx
│           ├── Contact.jsx
│           └── Footer.jsx
│
├── mobile/                  # React Native (Expo + JWT Auth)
│   ├── package.json
│   ├── App.js
│   ├── screens/
│   │   ├── WarningScreen.js
│   │   ├── LoginScreen.js
│   │   ├── Dashboard.js
│   │   └── EvidenceScreen.js
│   │   └── LoanCalculatorScreen.js
│   └── utils/
│       ├── crypto.js
│       └── autoReply.js
│
└── go-service/              # Go microservice
    ├── main.go
    └── go.mod
# Loan Verifier Go Service

### Run locally
```bash
cd go-service
go run main.go
go build -o app main.go
./app
---

### ⚡ Render.yaml (final snippet)
```yaml
- type: web
  name: loan-verifier-go
  env: go
  rootDir: go-service
  buildCommand: go build -o app main.go
  startCommand: ./app
import express from "express";
import bodyParser from "body-parser";
import cors from "cors";
import mongoose from "mongoose";
import jwt from "jsonwebtoken";
import dotenv from "dotenv";
import helmet from "helmet";
import rateLimit from "express-rate-limit";
import fetch from "node-fetch"; // ✅ ensure added in package.json

import User from "./models/User.js";
import Proof from "./models/Proof.js";
import errorHandler from "./middleware/errorHandler.js";

dotenv.config();
const app = express();

// Security
app.use(helmet());
app.use(cors());
app.use(bodyParser.json());

// Rate Limit
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "⚠️ Too many requests, please try later.",
});
app.use(limiter);

// DB connect
mongoose
  .connect(process.env.MONGO_URI)
  .then(() => console.log("✅ MongoDB Connected"))
  .catch((err) => console.error("❌ DB Error:", err));

// JWT Middleware
function verifyToken(req, res, next) {
  const token = req.headers["authorization"]?.split(" ")[1];
  if (!token) return res.status(401).json({ message: "❌ Token missing" });

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "❌ Invalid token" });
    req.user = decoded;
    next();
  });
}

// Login
app.post("/login", async (req, res) => {
  const { mobile, identifier, consent } = req.body;
  if (!consent) return res.status(400).json({ message: "⚠️ पहले चेतावनी स्वीकार करें।" });

  const user = await User.findOne({ mobile });
  if (!user) return res.status(403).json({ message: "❌ Loan user नहीं मिला।" });

  if (user.blocked) return res.status(403).json({ message: "❌ आप पहले से ब्लॉक हैं।" });

  if (user.loanActive && (user.email === identifier || user.branchCode === identifier)) {
    if (user.loanEvidence) {
      const token = jwt.sign({ mobile: user.mobile }, process.env.JWT_SECRET, {
        expiresIn: "2h",
      });
      return res.json({ message: "✅ Login Successful", token });
    } else {
      user.blocked = true;
      await user.save();
      return res.status(403).json({ message: "❌ Evidence missing. Blocked!" });
    }
  } else {
    user.blocked = true;
    await user.save();
    return res.status(403).json({ message: "❌ Wrong details. Blocked!" });
  }
});

// Save Proof
app.post("/save-proof", verifyToken, async (req, res, next) => {
  try {
    const { encryptedPDF } = req.body;
    const proof = new Proof({ mobile: req.user.mobile, encryptedPDF });
    await proof.save();
    res.json({ message: "📂 Proof saved" });
  } catch (err) {
    next(err);
  }
});

// Get Proof
app.get("/get-proof", verifyToken, async (req, res, next) => {
  try {
    const proofs = await Proof.find({ mobile: req.user.mobile }).sort({
      timestamp: -1,
    });
    res.json(proofs);
  } catch (err) {
    next(err);
  }
});

// 🆕 Loan Verification (Go microservice)
app.post("/verify-loan", verifyToken, async (req, res) => {
  try {
    const { amount, duration, rate } = req.body;
    const response = await fetch(process.env.GO_SERVICE_URL + "/check-loan", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ amount, duration, rate }),
    });
    const data = await response.json();
    res.json({ message: "✅ Loan Verified via Go Service", data });
  } catch (err) {
    console.error("❌ Go service error:", err);
    res.status(500).json({ message: "Go service unavailable" });
  }
});

app.use(errorHandler);
app.listen(process.env.PORT || 5000, () => console.log("🚀 Backend running"));
import { useState } from "react";
import Splash from "./Splash";
import Login from "./components/Login";
import LoanCalculator from "./components/LoanCalculator";

export default function App() {
  const [showSplash, setShowSplash] = useState(true);
  const [token, setToken] = useState(localStorage.getItem("jwt"));

  if (showSplash) return <Splash onFinish={() => setShowSplash(false)} />;

  if (!token) return <Login setToken={setToken} />; // ✅ Fixed

  return (
    <div className="p-6">
      <h2 className="text-lg font-bold text-green-700">✅ Logged In</h2>
      <p className="mt-2 text-gray-700">
        अब आप Evidence upload / fetch कर सकते हैं और Loan calculate भी कर सकते हैं।
      </p>
      <LoanCalculator />
    </div>
  );
}
import { useState } from "react";

export default function LoanCalculator() {
  const [amount, setAmount] = useState("");
  const [duration, setDuration] = useState("");
  const [rate, setRate] = useState("");
  const [result, setResult] = useState(null);

  const handleVerify = async () => {
    const token = localStorage.getItem("jwt");
    const res = await fetch(`${import.meta.env.VITE_API_URL}/verify-loan`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({
        amount: parseFloat(amount),
        duration: parseInt(duration),
        rate: parseFloat(rate),
      }),
    });
    const data = await res.json();
    setResult(data);
  };

  return (
    <div className="p-4 bg-white rounded shadow mt-4 space-y-3 w-80">
      <h3 className="text-lg font-bold">📊 Loan Calculator</h3>
      <input
        type="number"
        placeholder="Loan Amount"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        className="w-full border p-2 rounded"
      />
      <input
        type="number"
        placeholder="Duration (months)"
        value={duration}
        onChange={(e) => setDuration(e.target.value)}
        className="w-full border p-2 rounded"
      />
      <input
        type="number"
        placeholder="Interest Rate (%)"
        value={rate}
        onChange={(e) => setRate(e.target.value)}
        className="w-full border p-2 rounded"
      />
      <button
        onClick={handleVerify}
        className="w-full bg-green-600 text-white py-2 rounded"
      >
        Calculate EMI
      </button>
      {result && (
        <div className="mt-2 text-sm">
          <p>{result.message}</p>
          <p>
            EMI: <strong>{result.data?.emi?.toFixed(2)}</strong>
          </p>
        </div>
      )}
    </div>
  );
}
import React, { useState } from "react";
import { NavigationContainer } from "@react-navigation/native";
import { createStackNavigator } from "@react-navigation/stack";

import LoginScreen from "./screens/LoginScreen";
import Dashboard from "./screens/Dashboard";
import EvidenceScreen from "./screens/EvidenceScreen";
import LoanCalculatorScreen from "./screens/LoanCalculatorScreen";

const Stack = createStackNavigator();

export default function App() {
  const [token, setToken] = useState(null);

  return (
    <NavigationContainer>
      <Stack.Navigator>
        {!token ? (
          <Stack.Screen name="Login">
            {(props) => <LoginScreen {...props} setToken={setToken} />}
          </Stack.Screen>
        ) : (
          <>
            <Stack.Screen name="Dashboard" component={Dashboard} />
            <Stack.Screen name="Evidence" component={EvidenceScreen} />
            <Stack.Screen name="LoanCalculator" component={LoanCalculatorScreen} />
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
package main

import (
	"encoding/json"
	"log"
	"math"
	"net/http"
)

type LoanCheckRequest struct {
	Amount   float64 `json:"amount"`
	Duration int     `json:"duration"`
	Rate     float64 `json:"rate"`
}

type LoanCheckResponse struct {
	EMI float64 `json:"emi"`
	Ok  bool    `json:"ok"`
}

func calculateEMI(amount float64, duration int, rate float64) float64 {
	monthlyRate := rate / (12 * 100)
	n := float64(duration)
	emi := (amount * monthlyRate * math.Pow(1+monthlyRate, n)) /
		(math.Pow(1+monthlyRate, n) - 1)
	return emi
}

func loanHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Only POST allowed", http.StatusMethodNotAllowed)
		return
	}
	var req LoanCheckRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	emi := calculateEMI(req.Amount, req.Duration, req.Rate)
	resp := LoanCheckResponse{EMI: emi, Ok: true}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}

func main() {
	http.HandleFunc("/check-loan", loanHandler)
	log.Println("✅ Go Loan Service running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
services:
  - type: web
    name: loan-shield-backend
    env: node
    rootDir: backend
    buildCommand: npm install
    startCommand: node server.js
    envVars:
      - key: MONGO_URI
        value: mongodb+srv://<username>:<password>@cluster.mongodb.net/loanShield
      - key: JWT_SECRET
        value: supersecretjwtkey
      - key: PORT
        value: 5000
      - key: GO_SERVICE_URL
        value: https://loan-verifier-go.onrender.com

  - type: web
    name: loan-shield-frontend
    env: static
    rootDir: frontend
    buildCommand: npm install && npm run build
    staticPublishPath: dist
    envVars:
      - key: VITE_API_URL
        value: https://loan-shield-backend.onrender.com

  - type: web
    name: loan-verifier-go
    env: go
    rootDir: go-service
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type LoanCheckRequest struct {
	Amount   float64 `json:"amount"`
	Duration int     `json:"duration"`
	Rate     float64 `json:"rate"`
}

type LoanCheckResponse struct {
	EMI float64 `json:"emi"`
	Ok  bool    `json:"ok"`
}

func calculateEMI(amount float64, duration int, rate float64) float64 {
	monthlyRate := rate / (12 * 100)
	n := float64(duration)

	emi := (amount * monthlyRate * pow(1+monthlyRate, n)) / (pow(1+monthlyRate, n) - 1)
	return emi
}

func pow(x, y float64) float64 {
	res := 1.0
	for i := 0; i < int(y); i++ {
		res *= x
	}
	return res
}

func loanHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Only POST allowed", http.StatusMethodNotAllowed)
		return
	}

	var req LoanCheckRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	emi := calculateEMI(req.Amount, req.Duration, req.Rate)
	resp := LoanCheckResponse{EMI: emi, Ok: true}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}

func main() {
	http.HandleFunc("/check-loan", loanHandler)
	log.Println("✅ Go Loan Service running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
module loan-verifier

go 1.20

    buildCommand: go build -o app .
    startCommand: ./app

curl -X POST https://loan-verifier-go.onrender.com/check-loan \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 500000,
    "duration": 60,
    "rate": 10
  }'
{
  "emi": 10624.58,
  "ok": true
}