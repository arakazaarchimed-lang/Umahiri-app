meta name="viewport" content="width=device-width, initial-scale=1">
<style>
  body { font-family: Arial, sans-serif; background: #f0f2f5; margin: 0; }
  h1 { background: #1877f2; color: white; margin: 0; padding: 16px; font-size: 24px; }
  .box, .post { background: white; margin: 12px; padding: 12px; border-radius: 10px; box-shadow: 0 1px 3px rgba(0,0,0,0.2); }
  input, textarea { width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 8px; box-sizing: border-box; font-size: 16px; margin-bottom: 8px; }
  textarea { height: 70px; }
  button { background: #1877f2; color: white; border: none; padding: 8px 16px; border-radius: 6px; margin-top: 4px; font-size: 16px; }
  .siba { background: #e4e6eb; color: #333; margin-left: 8px; }
  #msg { color: red; }
</style>

<h1>Inshuti</h1>

<div class="box" id="auth">
  <h3 id="authTitle">Injira</h3>
  <input id="izina" placeholder="Izina ryawe" style="display:none">
  <input id="email" type="email" placeholder="Email">
  <input id="pass" type="password" placeholder="Ijambo ry'ibanga (byibura 6)">
  <button id="authBtn">Injira</button>
  <button id="toggle" class="siba">Nta konti? Iyandikishe</button>
  <p id="msg"></p>
</div>

<div id="app" style="display:none">
  <div class="box">
    <div style="margin-bottom:8px">Muraho, <b id="uwo"></b>
      <button id="out" class="siba">Sohoka</button>
    </div>
    <textarea id="inyandiko" placeholder="Utekereza iki?"></textarea>
    <button id="shira">Shira</button>
  </div>
  <div id="posts"></div>
</div>

<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/13.0.0/firebase-app.js";
  import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, onAuthStateChanged, signOut, updateProfile } from "https://www.gstatic.com/firebasejs/13.0.0/firebase-auth.js";
  import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp, doc, updateDoc, deleteDoc, increment } from "https://www.gstatic.com/firebasejs/13.0.0/firebase-firestore.js";

  const firebaseConfig = {
    apiKey: "AIzaSyCdYfenKsmzjdlWnC1ETD9ntrVSW0aGVX0",
    authDomain: "inhuti-app.firebaseapp.com",
    projectId: "inhuti-app",
    storageBucket: "inhuti-app.firebasestorage.app",
    messagingSenderId: "299282318533",
    appId: "1:299282318533:web:d83b52122350fdd2ef9cac"
  };

  const app = initializeApp(firebaseConfig);
  const auth = getAuth(app);
  const db = getFirestore(app);
  const postsRef = collection(db, "posts");

  const $ = (id) => document.getElementById(id);
  let kwiyandikisha = false;
  let stop = null;

  function ikosa(e) {
    const c = e.code || "";
    if (c.includes("email-already-in-use")) return "Iyo email isanzwe ikoreshwa.";
    if (c.includes("invalid-email")) return "Email ntiyanditse neza.";
    if (c.includes("weak-password")) return "Ijambo ry'ibanga rigufi cane (byibura inyuguti 6).";
    if (c.includes("invalid-credential") || c.includes("wrong-password") || c.includes("user-not-found")) return "Email canke ijambo ry'ibanga si vyo.";
    return "Ikosa: " + e.message;
  }

  $("toggle").addEventListener("click", function () {
    kwiyandikisha = !kwiyandikisha;
    $("izina").style.display = kwiyandikisha ? "block" : "none";
    $("authTitle").textContent = kwiyandikisha ? "Iyandikishe" : "Injira";
    $("authBtn").textContent = kwiyandikisha ? "Iyandikishe" : "Injira";
    $("toggle").textContent = kwiyandikisha ? "Ufise konti? Injira" : "Nta konti? Iyandikishe";
    $("msg").textContent = "";
  });

  $("authBtn").addEventListener("click", async function () {
    $("msg").textContent = "";
    const email = $("email").value.trim();
    const pass = $("pass").value;
    try {
      if (kwiyandikisha) {
        const izina = $("izina").value.trim();
        if (izina === "") {
          $("msg").textContent = "Andika izina ryawe.";
          return;
        }
        const cred = await createUserWithEmailAndPassword(auth, email, pass);
        await updateProfile(cred.user, { displayName: izina });
        $("uwo").textContent = izina;
      } else {
        await signInWithEmailAndPassword(auth, email, pass);
      }
    } catch (e) {
      $("msg").textContent = ikosa(e);
    }
  });

  $("out").addEventListener("click", function () {
    signOut(auth);
  });

  $("shira").addEventListener("click", async function () {
    const inyandiko = $("inyandiko").value.trim();
    if (inyandiko === "") {
      alert("Andika ikintu mbere!");
      return;
    }
    const u = auth.currentUser;
    try {
      await addDoc(postsRef, {
        izina: u.displayName || u.email,
        uid: u.uid,
        inyandiko: inyandiko,
        likes: 0,
        time: serverTimestamp()
      });
      $("inyandiko").value = "";
    } catch (e) {
      alert("Ikosa: " + e.message);
    }
  });

  function erekana(snap) {
    const aho = $("posts");
    aho.innerHTML = "";
    snap.forEach(function (d) {
      const p = d.data();

      const div = document.createElement("div");
      div.className = "post";

      const izina = document.createElement("b");
      izina.textContent = p.izina;

      const para = document.createElement("p");
      para.textContent = p.inyandiko;

      const like = document.createElement("button");
      like.textContent = "Like (" + (p.likes || 0) + ")";
      like.onclick = function () {
        updateDoc(doc(db, "posts", d.id), { likes: increment(1) });
      };

      div.appendChild(izina);
      div.appendChild(para);
      div.appendChild(like);

      if (auth.currentUser && p.uid === auth.currentUser.uid) {
        const siba = document.createElement("button");
        siba.textContent = "Siba";
        siba.className = "siba";
        siba.onclick = function () {
          deleteDoc(doc(db, "posts", d.id));
        };
        div.appendChild(siba);
      }

      aho.appendChild(div);
    });
  }

  onAuthStateChanged(auth, function (user) {
    if (user) {
      $("auth").style.display = "none";
      $("app").style.display = "block";
      $("uwo").textContent = user.displayName || user.email;
      const q = query(postsRef, orderBy("time", "desc"));
      stop = onSnapshot(q, erekana, function (e) {
        $("posts").textContent = "Ikosa: " + e.message;
      });
    } else {
      $("auth").style.display = "block";
      $("app").style.display = "none";
      if (stop) { stop(); stop = null; }
      $("posts").innerHTML = "";
    }
  });
</script># Umahiri-app
