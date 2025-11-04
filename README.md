<html lang="nl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>IVA 3.0</title>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
  <style>
    body {
      margin: 0;
      font-family: "Segoe UI", sans-serif;
      background: #f0f2f5;
      display: flex;
      height: 100vh;
    }
    #sidebar {
      width: 250px;
      background: #1e1e2f;
      color: white;
      padding: 1rem;
      overflow-y: auto;
    }
    #main {
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    #chat {
      flex: 1;
      padding: 1rem;
      overflow-y: auto;
      background: white;
    }
    .bubble {
      margin: 0.5rem 0;
      padding: 0.75rem;
      border-radius: 0.5rem;
      max-width: 70%;
    }
    .user { background: #4f46e5; color: white; margin-left: auto; text-align: right; }
    .bot { background: #eee; color: #111; margin-right: auto; }
    img.generated { max-width: 100%; margin-top: 1rem; border-radius: 0.5rem; }
    #controls {
      display: flex;
      gap: 0.5rem;
      padding: 1rem;
      background: #f9f9f9;
      border-top: 1px solid #ccc;
    }
    input, select, button {
      font-size: 1rem;
      padding: 0.5rem;
    }
    input { flex: 1; }
    .chat-title {
      cursor: pointer;
      padding: 0.5rem;
      border-bottom: 1px solid #333;
    }
    .chat-title:hover {
      background: #2e2e3f;
    }
  </style>
</head>
<body>
  <div id="sidebar">
    <button id="login">🔐 Log in</button>
    <button id="newChat">➕ Nieuwe chat</button>
    <div id="chatList"></div>
  </div>
  <div id="main">
    <div id="chat"></div>
    <div id="controls">
      <select id="tone">
        <option value="vriendelijk">Vriendelijk</option>
        <option value="zakelijk">Zakelijk</option>
        <option value="speels">Speels</option>
      </select>
      <input id="input" type="text" placeholder="Typ je vraag..." />
      <button id="send">Verstuur</button>
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
    const db = firebase.firestore();
    const GPT_API_KEY = "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";

    let userId = null;
    let chatId = null;
    let chatRef = null;

    document.getElementById("login").onclick = () => {
      const provider = new firebase.auth.GoogleAuthProvider();
      auth.signInWithPopup(provider).then(result => {
        userId = result.user.uid;
        loadChats();
      });
    };

    document.getElementById("newChat").onclick = async () => {
      const newRef = await db.collection("users").doc(userId).collection("chats").add({ created: Date.now() });
      chatId = newRef.id;
      chatRef = newRef.collection("messages");
      document.getElementById("chat").innerHTML = "";
      loadChats();
    };

    async function loadChats() {
      const list = document.getElementById("chatList");
      list.innerHTML = "";
      const snapshot = await db.collection("users").doc(userId).collection("chats").orderBy("created", "desc").get();
      snapshot.forEach(doc => {
        const div = document.createElement("div");
        div.className = "chat-title";
        div.textContent = "Chat " + doc.id.slice(0, 6);
        div.onclick = () => switchChat(doc.id);
        list.appendChild(div);
      });
    }

    async function switchChat(id) {
      chatId = id;
      chatRef = db.collection("users").doc(userId).collection("chats").doc(chatId).collection("messages");
      document.getElementById("chat").innerHTML = "";
      const snapshot = await chatRef.orderBy("timestamp").get();
      snapshot.forEach(doc => {
        const msg = doc.data();
        addMessage(msg.text, msg.sender);
      });
    }

    function addMessage(text, sender) {
      const div = document.createElement("div");
      div.className = `bubble ${sender}`;
      div.textContent = text;
      document.getElementById("chat").appendChild(div);
      document.getElementById("chat").scrollTop = 99999;
    }

    async function saveMessage(text, sender) {
      if (!chatRef) return;
      await chatRef.add({ text, sender, timestamp: Date.now() });
    }

    function isImagePrompt(text) {
      return text.toLowerCase().includes("toon") || text.toLowerCase().includes("beeld");
    }

    document.getElementById("send").onclick = async () => {
      const text = document.getElementById("input").value.trim();
      if (!text) return;
      addMessage(text, "user");
      saveMessage(text, "user");
      document.getElementById("input").value = "";

      if (isImagePrompt(text)) {
        const imageUrl = await generateImage(text);
        if (imageUrl) {
          const img = document.createElement("img");
          img.src = imageUrl;
          img.className = "generated";
          document.getElementById("chat").appendChild(img);
        } else {
          addMessage("Kon geen afbeelding genereren.", "bot");
        }
        return;
      }

      const tone = document.getElementById("tone").value;
      const prompt = `Reageer in toon "${tone}" op: ${text}`;
      const res = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${GPT_API_KEY}`
        },
        body: JSON.stringify({
          model: "gpt-3.5-turbo",
          messages: [{ role: "user", content: prompt }]
        })
      });
      const data = await res.json();
      const reply = data.choices?.[0]?.message?.content || "Geen antwoord.";
      addMessage(reply, "bot");
      saveMessage(reply, "bot");
    };

    async function generateImage(promptText) {
      const res = await fetch("https://generativelanguage.googleapis.com/v1beta/models/gemini-pro-vision:generateContent?key=AIzaSyAyhPh_QioJMXfo-nW__fhlfb3u0SEpZwk", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          contents: [{ role: "user", parts: [{ text: `Genereer een afbeelding van: ${promptText}` }] }]
        })
      });
      const data = await res.json();
      const imagePart = data.candidates?.[0]?.content?.parts?.find(p => p.inlineData?.mimeType?.includes("image"));
      if (imagePart) {
        return "data:" + imagePart.inlineData.mime
