📂 project-root
 ┣ 📄 .env.example
 ┣ 📄 server.js
 ┣ 📂 models
 ┃ ┣ User.js
 ┃ ┗ Proof.js
 ┣ 📂 middleware
 ┃ ┗ errorHandler.js
 ┣ 📂 src
 ┃ ┣ App.jsx
 ┃ ┗ components
 ┃   ┗ Login.jsx
 ┣ 📂 screens
 ┃ ┣ LoginScreen.js
 ┃ ┗ EvidenceScreen.js
 ┣ 📄 package.json
 ┗ 📄 README.md


---

🔹 .env.example

MONGO_URI=mongodb+srv://<username>:<password>@cluster0.mongodb.net/loanShield
JWT_SECRET=supersecretjwtkey
PORT=5000
VITE_API_URL=http://localhost:5000


---

🔹 models/User.js

import mongoose from "mongoose";

const userSchema = new mongoose.Schema({
  mobile: String,
  email: String,
  branchCode: String,
  loanActive: Boolean,
  loanEvidence: Boolean,
  blocked: { type: Boolean, default: false }
});

export default mongoose.model("User", userSchema);


---

🔹 models/Proof.js

import mongoose from "mongoose";

const proofSchema = new mongoose.Schema({
  mobile: String,
  encryptedPDF: String,
  timestamp: { type: Date, default: Date.now }
});

export default mongoose.model("Proof", proofSchema);


---

🔹 middleware/errorHandler.js

export default function errorHandler(err, req, res, next) {
  console.error("❌ Error:", err.message);
  res.status(500).json({ message: "Server Error" });
}


---

🔹 server.js

import express from "express";
import bodyParser from "body-parser";
import cors from "cors";
import mongoose from "mongoose";
import jwt from "jsonwebtoken";
import dotenv from "dotenv";
import User from "./models/User.js";
import Proof from "./models/Proof.js";
import errorHandler from "./middleware/errorHandler.js";

dotenv.config();
const app = express();
app.use(cors());
app.use(bodyParser.json());

// DB connect
mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log("✅ MongoDB Connected"))
  .catch(err => console.error("❌ DB Error:", err));

// 🔑 JWT Middleware
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

app.use(errorHandler);
app.listen(process.env.PORT || 5000, () => console.log("🚀 Backend running"));


---

🔹 src/App.jsx

import { useState } from "react";
import Splash from "./Splash";
import Login from "./components/Login";

export default function App() {
  const [showSplash, setShowSplash] = useState(true);
  const [token, setToken] = useState(localStorage.getItem("jwt"));

  if (showSplash) {
    return <Splash onFinish={() => setShowSplash(false)} />;
  }

  if (!token) {
    return <Login setToken={setToken} />;
  }

  return (
    <div className="p-6">
      <h2 className="text-lg font-bold text-green-700">✅ Logged In</h2>
      <p className="mt-2 text-gray-700">
        अब आप Evidence upload / fetch कर सकते हैं।
      </p>
    </div>
  );
}


---

🔹 src/components/Login.jsx

import { useState } from "react";

export default function Login({ setToken }) {
  const [mobile, setMobile] = useState("");
  const [identifier, setIdentifier] = useState("");
  const [message, setMessage] = useState("");

  const handleLogin = async () => {
    const res = await fetch(`${import.meta.env.VITE_API_URL}/login`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ mobile, identifier, consent: true }),
    });
    const data = await res.json();
    setMessage(data.message);

    if (data.token) {
      localStorage.setItem("jwt", data.token);
      setToken(data.token);
    }
  };

  return (
    <div className="p-6 bg-white rounded shadow w-80 space-y-3">
      <input
        type="text"
        placeholder="Mobile Number"
        className="w-full border p-2 rounded"
        value={mobile}
        onChange={(e) => setMobile(e.target.value)}
      />
      <input
        type="text"
        placeholder="Email / Branch Code"
        className="w-full border p-2 rounded"
        value={identifier}
        onChange={(e) => setIdentifier(e.target.value)}
      />
      <button
        onClick={handleLogin}
        className="w-full bg-green-600 text-white py-2 rounded"
      >
        Login
      </button>
      {message && <p className="mt-2 text-sm">{message}</p>}
    </div>
  );
}


---

🔹 screens/LoginScreen.js

(React Native)

import React, { useState } from "react";
import { View, Text, TextInput, Button, StyleSheet } from "react-native";
import AsyncStorage from "@react-native-async-storage/async-storage";

export default function LoginScreen({ setToken }) {
  const [mobile, setMobile] = useState("");
  const [identifier, setIdentifier] = useState("");
  const [message, setMessage] = useState("");

  const handleLogin = async () => {
    const res = await fetch("http://10.0.2.2:5000/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ mobile, identifier, consent: true }),
    });
    const data = await res.json();
    setMessage(data.message);

    if (data.token) {
      await AsyncStorage.setItem("jwt", data.token);
      setToken(data.token);
    }
  };

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        placeholder="Mobile Number"
        value={mobile}
        onChangeText={setMobile}
      />
      <TextInput
        style={styles.input}
        placeholder="Email / Branch Code"
        value={identifier}
        onChangeText={setIdentifier}
      />
      <Button title="Login" onPress={handleLogin} />
      {message ? <Text style={styles.text}>{message}</Text> : null}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: "center", padding: 20 },
  input: { borderWidth: 1, borderColor: "#ccc", padding: 10, marginVertical: 5 },
  text: { marginTop: 10, color: "blue" },
});


---

🔹 screens/EvidenceScreen.js

import React, { useState } from "react";
import { View, Text, TextInput, TouchableOpacity, FlatList, StyleSheet } from "react-native";
import AsyncStorage from "@react-native-async-storage/async-storage";

export default function EvidenceScreen() {
  const [content, setContent] = useState("");
  const [evidences, setEvidences] = useState([]);
  const [message, setMessage] = useState("");

  const saveEvidence = async () => {
    const token = await AsyncStorage.getItem("jwt");
    const res = await fetch("http://10.0.2.2:5000/save-proof", {
      method: "POST",
      headers: { "Content-Type": "application/json", "Authorization": `Bearer ${token}` },
      body: JSON.stringify({ encryptedPDF: content }),
    });
    const data = await res.json();
    setMessage(data.message);
    fetchEvidence();
  };

  const fetchEvidence = async () => {
    const token = await AsyncStorage.getItem("jwt");
    const res = await fetch("http://10.0.2.2:5000/get-proof", {
      headers: { "Authorization": `Bearer ${token}` }
    });
    const data = await res.json();
    setEvidences(data);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>📂 Loan Evidence</Text>
      <TextInput
        style={[styles.input, { height: 80 }]}
        placeholder="Enter Evidence (Encrypted PDF Text)"
        value={content}
        onChangeText={setContent}
        multiline
      />
      <TouchableOpacity style={styles.button} onPress={saveEvidence}>
        <Text style={styles.buttonText}>💾 Save</Text>
      </TouchableOpacity>
      <TouchableOpacity style={[styles.button, { backgroundColor: "purple" }]} onPress={fetchEvidence}>
        <Text style={styles.buttonText}>📥 Fetch</Text>
      </TouchableOpacity>
      {message ? <Text style={styles.message}>{message}</Text> : null}
      <FlatList
        data={evidences}
        keyExtractor={(item, index) => index.toString()}
        renderItem={({ item }) => (
          <View style={styles.card}>
            <Text>{item.encryptedPDF}</Text>
          </View>
        )}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: "#f9f9f9" },
  title: { fontSize: 20, fontWeight: "bold", marginBottom: 10 },
  input: { borderWidth: 1, borderColor: "#ccc", borderRadius: 8, padding: 10, marginBottom: 10 },
  button: { backgroundColor: "green", padding: 12, borderRadius: 8, marginBottom: 10 },
  buttonText: { color: "#fff", fontWeight: "bold", textAlign: "center" },
  message: { marginVertical: 10, fontSize: 14, color: "blue" },
  card: { padding: 10, backgroundColor: "#fff", borderRadius: 8, marginBottom: 10 },
});
