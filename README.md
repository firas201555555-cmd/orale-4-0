
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>ORALE 3.1</title>
<style>
*{box-sizing:border-box}
body{
 margin:0;background:#07111f;color:white;font-family:Arial;
 min-height:100vh;display:flex;justify-content:center;
 align-items:center;padding:15px
}
.app{
 width:100%;max-width:650px;background:#102139;
 border:1px solid #087ca8;border-radius:22px;
 padding:20px;box-shadow:0 0 30px #00bfff25
}
h1{text-align:center;color:#39d5ff}
.sub{text-align:center;color:#9bb8ce}
.robot{text-align:center;font-size:65px;margin:15px}
.status{text-align:center;color:#57e6ad;font-size:13px}
#chat{
 height:340px;overflow-y:auto;background:#071522;
 padding:12px;border-radius:14px;margin:18px 0
}
.msg{
 padding:12px;margin:9px 0;border-radius:12px;
 max-width:90%;line-height:1.6;white-space:pre-wrap;
 overflow-wrap:anywhere
}
.user{background:#075985;margin-right:auto}
.bot{background:#263b52;margin-left:auto}
.controls,.keybox{display:flex;gap:8px;margin-top:10px}
input{
 min-width:0;flex:1;padding:13px;border:0;
 border-radius:10px;font-size:15px
}
button{
 border:0;border-radius:10px;padding:12px 16px;
 background:#08a9d5;color:white;font-size:15px;cursor:pointer
}
button:disabled{opacity:.5;cursor:wait}
.note{font-size:12px;color:#a9bdcf;line-height:1.6}
</style>
</head>
<body>
<div class="app">
 <h1>🤖 ORALE 3.1</h1>
 <div class="sub">Your Personal AI Assistant</div>
 <div class="robot">🤖</div>
 <div class="status" id="status">● جاهز للمحادثة</div>

 <div id="chat">
  <div class="msg bot">مرحبًا! أنا ORALE. اسألني ما تريد! 🚀</div>
 </div>

 <div class="controls">
  <input id="message" placeholder="اكتب رسالتك هنا...">
  <button id="send">إرسال</button>
 </div>

 <div class="keybox">
  <input id="apiKey" type="password" placeholder="مفتاح Gemini API">
  <button id="showKey">إظهار</button>
 </div>

 <p class="note">
  هذه نسخة تجريبية. لا تنشر مفتاح API داخل صفحة عامة،
  ولا تستخدم مفتاحًا مهمًا؛ فالمتصفح قد يكشفه.
 </p>
</div>

<script>
const MODELS = [
 "gemini-3.6-flash",
 "gemini-3.5-flash-lite"
];

const chat = document.getElementById("chat");
const messageInput = document.getElementById("message");
const keyInput = document.getElementById("apiKey");
const sendButton = document.getElementById("send");
const statusText = document.getElementById("status");

let history = [];
let busy = false;

function addMessage(text, type) {
 const div = document.createElement("div");
 div.className = "msg " + type;
 div.textContent = text;
 chat.appendChild(div);
 chat.scrollTop = chat.scrollHeight;
 return div;
}

document.getElementById("showKey").onclick = () => {
 keyInput.type = keyInput.type === "password" ? "text" : "password";
};

async function askGemini(model, key, message) {
 const controller = new AbortController();
 const timer = setTimeout(() => controller.abort(), 40000);

 try {
  const response = await fetch(
   `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent`,
   {
    method: "POST",
    headers: {
     "Content-Type": "application/json",
     "x-goog-api-key": key
    },
    body: JSON.stringify({
     systemInstruction: {
      parts: [{
       text: "أنت ORALE، مساعد ذكي وودود. أجب بوضوح وباللغة التي يستعملها المستخدم."
      }]
     },
     contents: [
      ...history,
      {role:"user",parts:[{text:message}]}
     ],
     generationConfig: {
      temperature:0.7,
      maxOutputTokens:700
     }
    }),
    signal: controller.signal
   }
  );

  const data = await response.json();

  if (!response.ok) {
   const error = new Error(
    data.error?.message || "HTTP " + response.status
   );
   error.status = response.status;
   throw error;
  }

  const answer = data.candidates?.[0]?.content?.parts
   ?.map(p => p.text || "").join("");

  if (!answer) throw new Error("لم يصل رد نصي من النموذج.");

  return answer;
 } finally {
  clearTimeout(timer);
 }
}

async function sendMessage() {
 if (busy) return;

 const message = messageInput.value.trim();
 const key = keyInput.value.trim();

 if (!message) return;

 if (!key) {
  addMessage("ضع مفتاح Gemini في الخانة أولًا.", "bot");
  return;
 }

 addMessage(message, "user");
 messageInput.value = "";
 busy = true;
 sendButton.disabled = true;

 const loading = addMessage("جاري التفكير... 🤔", "bot");
 statusText.textContent = "● جاري الاتصال بـ Gemini";

 try {
  let answer = "";
  let lastError = null;

  for (const model of MODELS) {
   try {
    statusText.textContent = "● تجربة " + model;
    answer = await askGemini(model, key, message);
    break;
   } catch (error) {
    lastError = error;

    // جرّب النموذج البديل عند الضغط أو عدم توفر النموذج
    if (![404, 429, 500, 502, 503, 504].includes(error.status)
        && error.name !== "AbortError") {
     throw error;
    }
   }
  }

  if (!answer) throw lastError || new Error("تعذر الاتصال.");

  loading.remove();
  addMessage(answer, "bot");

  history.push(
   {role:"user",parts:[{text:message}]},
   {role:"model",parts:[{text:answer}]}
  );

  if (history.length > 12) history = history.slice(-12);

  statusText.textContent = "● ORALE متصل";

 } catch (error) {
  loading.remove();

  let explanation = error.message;

  if (error.name === "AbortError") {
   explanation = "انتهت مهلة الانتظار. حاول مرة أخرى.";
  }

  addMessage("تعذر الاتصال: " + explanation, "bot");
  statusText.textContent = "● لم يكتمل الاتصال";
 } finally {
  busy = false;
  sendButton.disabled = false;
  messageInput.focus();
 }
}

sendButton.addEventListener("click", sendMessage);

messageInput.addEventListener("keydown", event => {
 if (event.key === "Enter") sendMessage();
});
</script>
</body>
</html>
