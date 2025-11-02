<!DOCTYPE html>
<html lang="nl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>IVA — Jouw AI Platform</title>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
  <style>
    body { font-family: sans-serif; background-color: #f4f7f9; margin: 0; padding: 2rem; }
    .container { max-width: 900px; margin: auto; background: white; padding: 2rem; border-radius: 0.5rem; box-shadow: 0 0 10px rgba(0,0,0,0.1); display: flex; gap: 2rem; }
    .sidebar { width: 200px; border-right: 1px solid #ddd; padding-right: 1rem; }
    .chat-area { flex: 1; display: flex; flex-direction: column; }
    h1 { text-align: center; color: #4f46e5; margin-bottom: 1rem; }
    .chat { height: 400px; overflow-y: auto; border: 1px solid #ddd; padding: 1rem; background: #f9fafb; border-radius: 0.5rem; margin-bottom: 1rem; }
    .chat-bubble { padding: 0.75rem; margin: 0.5rem 0; border-radius: 0.5rem; max-width: 80%; }
    .user { background-color: #4f46e5; color: white; text-align: right; margin-left: auto; }
    .bot { background-color: #e5e7eb; color: #111827; margin-right: auto; }
    .input-row { display: flex; gap: 0.5rem; }
    input, select { flex: 1; padding: 0.5rem; border: 1px solid #ccc; border-radius: 0.25rem; }
    button { padding: 0.5rem 1rem; background-color: #4f46e5; color: white; border: none; border-radius: 0.25rem; cursor: pointer; }
    .chat-list { list-style: none; padding: 0; margin: 0; }
    .chat-list li { padding: 0.5rem; cursor: pointer; border-bottom: 1px solid #eee; }
    .chat-list li:hover { background-color: #f0f0f0; }
    .logout { background: none; color: #dc2626; text-decoration: underline; margin-top: 1rem; cursor: pointer; }
  </style>
</head>
<body>
  <h1>IVA — Jouw AI Platform</h1>
  <div class="container">
    <div class="sidebar">
      <button id="new-chat">➕ Nieuwe chat</button>
      <ul id="chat-list" class="chat-list"></ul>
      <button id="logout-btn" class="logout">Uitloggen</button>
    </div>
    <div class="chat-area">
      <div id="auth-section">
        <p>Log in om IVA te gebruiken:</p>
        <button id="login-btn">🔐 Log in met Google</button>
      </div>

      <div id="chat-section" style="display:none;">
        <select id="tone">
          <option value="vriendelijk">Vriendelijk</option>
          <option value="zakelijk">Zakelijk</option>
          <option value="speels">Speels</option>
          <option value="poëtisch">Poëtisch</option>
          <option value="analytisch">Analytisch</option>
        </select>
        <div id="chat" class="chat"></div>
        <div class="input-row">
          <input id="input" type="text" placeholder="Typ je vraag..." />
          <button id="send">Verstuur</button>
        </div>
      </div>
    </div>
  </div>

  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyAebutJDPHTFPhGbUOb__ZuRWXJFNrIupE",
      authDomain: "ivar-4be46.firebaseapp.com",
      projectId: "ivar-4be46",
      storageBucket: "ivar-4be46.appspot.com",
      messagingSenderId: "887327729474",
      appId: "1:887327729474:web:3658723ea0271a78070adb"
    };

    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();

    const loginBtn = document.getElementById("login-btn");
    const logoutBtn = document.getElementById("logout-btn");
    const authSection = document.getElementById("auth-section");
    const chatSection = document.getElementById("chat-section");
    const chatList = document.getElementById("chat-list");
    const newChatBtn = document.getElementById("new-chat");

    let currentChatId = null;
    let chats = {};
    let ivaMemory = [];

    loginBtn.addEventListener("click", () => {
      const provider = new firebase.auth.GoogleAuthProvider();
      auth.signInWithPopup(provider).catch(console.error);
    });

    logoutBtn.addEventListener("click", () => {
      auth.signOut();
    });

    auth.onAuthStateChanged(user => {
      if (user) {
        authSection.style.display = "none";
        chatSection.style.display = "block";
        loadChats();
      } else {
        authSection.style.display = "block";
        chatSection.style.display = "none";
        chatList.innerHTML = "";
        document.getElementById("chat").innerHTML = "";
      }
    });

    function loadChats() {
      chatList.innerHTML = "";
      chats = {};
      for (let i = 1; i <= 3; i++) {
        const id = "chat-" + i;
        chats[id] = { name: "Chat " + i, messages: [] };
        const li = document.createElement("li");
        li.textContent = chats[id].name;
        li.onclick = () => switchChat(id);
        chatList.appendChild(li);
      }
      switchChat("chat-1");
    }

    function switchChat(id) {
      currentChatId = id;
      ivaMemory = chats[id].messages;
      renderChat();
    }

    function renderChat() {
      const chat = document.getElementById("chat");
      chat.innerHTML = "";
      ivaMemory.forEach(msg => {
        const bubble = document.createElement("div");
        bubble.className = `chat-bubble ${msg.sender}`;
        bubble.textContent = msg.text;
        chat.appendChild(bubble);
      });
      chat.scrollTop = chat.scrollHeight;
    }

    const input = document.getElementById("input");
    const send = document.getElementById("send");
    const tone = document.getElementById("tone");

    send.addEventListener("click", async () => {
      const text = input.value.trim();
      if (!text) return;
      addMessage(text, "user");
      input.value = "";
      const reply = await fetchIVAReply(text);
      addMessage(reply, "bot");
    });

    input.addEventListener("keydown", e => {
      if (e.key === "Enter") send.click();
    });

    function addMessage(text, sender) {
      ivaMemory.push({ text, sender });
      chats[currentChatId].messages = ivaMemory;
      renderChat();
    }

    async function fetchIVAReply(message) {
      const context = ivaMemory.map(m => `${m.sender}: ${m.text}`).join(" | ");
      const selectedTone = tone.value;
      const prompt = `
Je bent IVA, een slimme AI-chatbot met toon "${selectedTone}".
Je praat met ${auth.currentUser?.displayName || "de gebruiker"}.
Context: ${context}
Gebruiker zegt: "${message}"
Reageer in het Nederlands, met de gekozen toon.
      `.trim();

      const response = await fetch("https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=AIzaSyDYCpPo7jRa8vJDbH3P5R1eopFo5koGPAo", {
