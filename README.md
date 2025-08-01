loan-shield-v4/
│── render.yaml
│── README.md
│
├── backend/
│   ├── package.json
│   ├── server.js
│   ├── .env.example
│   ├── models/
│   │   ├── User.js
│   │   └── Proof.js
│   └── middleware/
│       └── errorHandler.js
│
├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── .env.example
│   └── src/
│       ├── App.jsx
│       ├── Splash.jsx
│       ├── index.css
│       └── components/
│           ├── Login.jsx
│           ├── LoanCalculator.jsx   ✅ नया UI
│           ├── About.jsx
│           ├── Contact.jsx
│           └── Footer.jsx
│
├── mobile/
│   ├── package.json
│   ├── App.js
│   ├── screens/
│   │   ├── WarningScreen.js
│   │   ├── LoginScreen.js
│   │   ├── Dashboard.js
│   │   └── EvidenceScreen.js
│   └── utils/
│       ├── crypto.js
│       └── autoReply.js
│
└── go-service/                ✅ नया Go microservice
    ├── main.go
    └── go.mod
MONGO_URI=mongodb+srv://<anandrao727>:<Ar@15592>@cluster.mongodb.net/loanShield
JWT_SECRET=supersecretjwtkey
PORT=5000
GO_SERVICE_URL=https://loan-verifier-go.onrender.com
import express from "express";
import bodyParser from "body-parser";
import cors from "cors";
import mongoose from "mongoose";
import jwt from "jsonwebtoken";
import dotenv from "dotenv";
import helmet from "helmet";
import rateLimit from "express-rate-limit";
import fetch from "node-fetch";   // Go service call

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
  message: "⚠️ Too many requests, please try later."
});
app.use(limiter);

// DB connect
mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log("✅ MongoDB Connected"))
  .catch(err => console.error("❌ DB Error:", err));

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
      const token = jwt.sign({ mobile: user.mobile }, process.env.JWT_SECRET, { expiresIn: "2h" });
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
    const proofs = await Proof.find({ mobile: req.user.mobile }).sort({ timestamp: -1 });
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
      body: JSON.stringify({ amount, duration, rate })
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
        "Authorization": `Bearer ${token}`
      },
      body: JSON.stringify({ amount: parseFloat(amount), duration: parseInt(duration), rate: parseFloat(rate) }),
    });
    const data = await res.json();
    setResult(data);
  };

  return (
    <div className="p-6 bg-white rounded shadow w-96 space-y-3">
      <h2 className="text-lg font-bold">📊 Loan Calculator</h2>
      <input type="number" placeholder="Loan Amount" value={amount} onChange={(e) => setAmount(e.target.value)} className="w-full border p-2 rounded" />
      <input type="number" placeholder="Duration (months)" value={duration} onChange={(e) => setDuration(e.target.value)} className="w-full border p-2 rounded" />
      <input type="number" placeholder="Interest Rate (%)" value={rate} onChange={(e) => setRate(e.target.value)} className="w-full border p-2 rounded" />
      <button onClick={handleVerify} className="w-full bg-blue-600 text-white py-2 rounded">Calculate EMI</button>
      {result && (
        <div className="mt-3 text-sm bg-gray-100 p-3 rounded">
          <p>{result.message}</p>
          <p>EMI: <b>{result.data?.emi?.toFixed(2)}</b></p>
        </div>
      )}
    </div>
  );
}
import { useState } from "react";
import Splash from "./Splash";
import Login from "./components/Login";
import LoanCalculator from "./components/LoanCalculator";

import React, { useState } from "react";
import { View, Text, TextInput, TouchableOpacity, StyleSheet } from "react-native";
import AsyncStorage from "@react-native-async-storage/async-storage";

export default function LoanCalculatorScreen() {
  const [amount, setAmount] = useState("");
  const [duration, setDuration] = useState("");
  const [rate, setRate] = useState("");
  const [result, setResult] = useState(null);
  const [message, setMessage] = useState("");

  const handleVerify = async () => {
    const token = await AsyncStorage.getItem("jwt");
    if (!token) {
      setMessage("❌ Please login first");
      return;
    }

    try {
      const res = await fetch("http://10.0.2.2:5000/verify-loan", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${token}`
        },
        body: JSON.stringify({
          amount: parseFloat(amount),
          duration: parseInt(duration),
          rate: parseFloat(rate)
        })
      });

      const data = await res.json();
      setMessage(data.message);
      setResult(data.data);
    } catch (err) {
      console.error("Error:", err);
      setMessage("❌ Go Service unavailable");
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>📊 Loan Calculator</Text>

      <TextInput
        style={styles.input}
        placeholder="Loan Amount"
        keyboardType="numeric"
        value={amount}
        onChangeText={setAmount}
      />
      <TextInput
        style={styles.input}
        placeholder="Duration (months)"
        keyboardType="numeric"
        value={duration}
        onChangeText={setDuration}
      />
      <TextInput
        style={styles.input}
        placeholder="Interest Rate (%)"
        keyboardType="numeric"
        value={rate}
        onChangeText={setRate}
      />

      <TouchableOpacity style={styles.button} onPress={handleVerify}>
        <Text style={styles.buttonText}>Calculate EMI</Text>
      </TouchableOpacity>

      {message ? <Text style={styles.message}>{message}</Text> : null}
      {result && (
        <View style={styles.resultBox}>
          <Text style={styles.resultText}>EMI: {result.emi?.toFixed(2)}</Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: "#f9f9f9" },
  title: { fontSize: 22, fontWeight: "bold", marginBottom: 20, textAlign: "center" },
  input: { borderWidth: 1, borderColor: "#ccc", borderRadius: 8, padding: 10, marginBottom: 10 },
  button: { backgroundColor: "green", padding: 12, borderRadius: 8, marginTop: 10 },
  buttonText: { color: "#fff", fontWeight: "bold", textAlign: "center" },
  message: { marginVertical: 10, fontSize: 14, color: "blue", textAlign: "center" },
  resultBox: { marginTop: 15, padding: 12, backgroundColor: "#fff", borderRadius: 8 },
  resultText: { fontSize: 16, fontWeight: "bold", color: "black" },
});

import React, { useState } from "react";
import { NavigationContainer } from "@react-navigation/native";
import { createStackNavigator } from "@react-navigation/stack";

import LoginScreen from "./screens/LoginScreen";
import Dashboard from "./screens/Dashboard";
import EvidenceScreen from "./screens/EvidenceScreen";
import LoanCalculatorScreen from "./screens/LoanCalculatorScreen";  // ✅ New screen

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
            <Stack.Screen name="LoanCalculator" component={LoanCalculatorScreen} /> {/* ✅ Add here */}
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}

export default function App() {
  const [showSplash, setShowSplash] = useState(true);
  const [token, setToken] = useState(localStorage.getItem("jwt"));

  if (showSplash) return <Splash onFinish={() => setShowSplash(false)} />;

  if (!token) return <Login setToken={setToken} />;

  return (
    <div className="p-6 space-y-6">
      <h2 className="text-lg font-bold text-green-700">✅ Logged In</h2>
      <p className="text-gray-700">अब आप Evidence upload / fetch कर सकते हैं और Loan calculate भी कर सकते हैं।</p>
      <LoanCalculator />
    </div>
  );
}
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
services:
  - type: web
    name: loan-shield-backend
    env: node
    rootDir: backend
    buildCommand: npm install
    startCommand: node server.js
    envVars:
      - key: MONGO_URI
        sync: false
      - key: JWT_SECRET
        sync: false
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
    buildCommand: go build -o app main.go
    startCommand: ./app
