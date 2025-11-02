<html lang="nl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>IVA Test</title>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
  <style>
    body { font-family: sans-serif; padding: 2rem; background: #f4f7f9; }
    .chat { height: 300px; overflow-y: auto; border: 1px solid #ccc; padding: 1rem; margin-bottom: 1rem; background: #fff; }
    .bubble { margin: 0.5rem 0; padding: 0.5rem; border-radius: 0.5rem; max-width: 80%; }
    .user { background: #4f46e5; color: white; text-align: right; margin-left: auto; }
    .bot { background: #eee; color: #111; margin-right: auto; }
  </style>
</head>
<body>
  <h1>IVA Test</h1>
  <button id="login">🔐 Log in met Google</button>
  <select id="tone">
    <option value="vriendelijk">Vriendelijk</option>
    <option value="zakelijk">Zakelijk</option>
  </select>
  <div id="chat" class="chat"></div>
  <input id="input" type="text" placeholder="Typ je vraag..." />
  <button id="send">Verstuur</button>

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

    document.getElementById("login").onclick = () => {
      const provider = new firebase.auth.GoogleAuthProvider();
      auth.signInWithPopup(provider).then(() => alert("Ingelogd!")).catch(console.error);
    };

    const chat = document.getElementById("chat");
    const input = document.getElementById("input");
    const send = document.getElementById("send");
    const tone = document.getElementById("tone");

    function addMessage(text, sender) {
      const div = document.createElement("div");
      div.className = `bubble ${sender}`;
      div.textContent = text;
      chat.appendChild(div);
      chat.scrollTop = chat.scrollHeight;
    }

    send.onclick = async () => {
      const text = input.value.trim();
      if (!text) return;
      addMessage(text, "user");
      input.value = "";
      const prompt = `Reageer in toon "${tone.value}" op: ${text}`;
      const res = await fetch("https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=AIzaSyDYCpPo7jRa8vJDbH3P5R1eopFo5koGPAo", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ contents: [{ role: "user", parts: [{ text: prompt }] }] })
      });
      const data = await res.json();
      const reply = data.candidates?.[0]?.content?.parts?.[0]?.text || "Geen antwoord.";
      addMessage(reply, "bot");
    };
  </script>
</body>
</html>
