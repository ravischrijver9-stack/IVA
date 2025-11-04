<html lang="nl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>IVA — AI Platform</title>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
  <style>
    body { font-family: sans-serif; background: #f4f7f9; padding: 2rem; }
    .chat { height: 400px; overflow-y: auto; border: 1px solid #ccc; padding: 1rem; background: #fff; margin-bottom: 1rem; }
    .bubble { margin: 0.5rem 0; padding: 0.5rem; border-radius: 0.5rem; max-width: 80%; }
    .user { background: #4f46e5; color: white; text-align: right; margin-left: auto; }
    .bot { background: #eee; color: #111; margin-right: auto; }
    img.generated { max-width: 100%; margin-top: 1rem; border-radius: 0.5rem; }
  </style>
</head>
<body>
  <h1>IVA</h1>
  <button id="login">🔐 Log in met Google</button>
  <select id="tone">
    <option value="vriendelijk">Vriendelijk</option>
    <option value="zakelijk">Zakelijk</option>
    <option value="speels">Speels</option>
  </select>
  <div id="chat" class="chat"></div>
  <input id="input" type="text" placeholder="Typ je vraag..." />
  <button id="send">Verstuur</button>

  <script>
    const firebaseConfig = {
      apiKey: "JOUW_FIREBASE_API_KEY",
      authDomain: "JOUW_FIREBASE_PROJECT.firebaseapp.com",
      projectId: "JOUW_FIREBASE_PROJECT"
    };
    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();
    const db = firebase.firestore();

    const GEMINI_API_KEY = "JOUW_GEMINI_API_KEY";

    let userId = null;
    let chatId = "default";
    let chatRef = null;

    document.getElementById("login").onclick = () => {
      const provider = new firebase.auth.GoogleAuthProvider();
      auth.signInWithPopup(provider).then(result => {
        userId = result.user.uid;
        chatRef = db.collection("users").doc(userId).collection("chats").doc(chatId).collection("messages");
        loadMessages();
      });
    };

    async function loadMessages() {
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
      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ contents: [{ role: "user", parts: [{ text: prompt }] }] })
      });
      const data = await res.json();
      const reply = data.candidates?.[0]?.content?.parts?.[0]?.text || "Geen antwoord.";
      addMessage(reply, "bot");
      saveMessage(reply, "bot");
    };

    async function generateImage(promptText) {
      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-pro-vision:generateContent?key=${GEMINI_API_KEY}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          contents: [{ role: "user", parts: [{ text: `Genereer een afbeelding van: ${promptText}` }] }]
        })
      });
      const data = await res.json();
      const imagePart = data.candidates?.[0]?.content?.parts?.find(p => p.inlineData?.mimeType?.includes("image"));
      if (imagePart) {
        return "data:" + imagePart.inlineData.mimeType + ";base64," + imagePart.inlineData.data;
      }
      return "";
    }
  </script>
</body>
</html>
