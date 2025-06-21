<script lang="ts">
  import { Clerk } from "@clerk/clerk-js";
  import { onMount } from "svelte";
  import Typed from 'typed.js';
  import { tick } from "svelte"; // make sure this is at the top

  let user = null;
  let token = "";

  const clerk = new Clerk("pk_test_bm9ibGUtc2F0eXItMTkuY2xlcmsuYWNjb3VudHMuZGV2JA");

  async function initClerk() {
    await clerk.load();
    if (clerk.user) {
      user = clerk.user;
      token = await clerk.session?.getToken();
    }
  }

  async function signIn() {
    await clerk.openSignIn();
    await initClerk();
  }

  async function signOut() {
    await clerk.signOut();
    user = null;
    token = "";
  }

  onMount(() => {
    initClerk();
  });

  let subject = "Polity";
  let count = 5;
  let questions = [];
  let answers: Record<number, string> = {};
  let submitted = false;
  let loading = false;
  let error = "";
  let timeLeft = 0;
  let timerInterval: any;
  let currentIndex = 0; // Track which question is being shown
  let quizStarted = false;
  let showMockTest = false;
  let step = 1; // 1 = subject select, 2 = question count, 3 = quiz

function startTimer() {
  timeLeft = count * 72; // 2 minutes per question
  clearInterval(timerInterval); // Clear any existing timer
  timerInterval = setInterval(() => {
    if (timeLeft > 0) {
      timeLeft--;
    } else {
      clearInterval(timerInterval);
      if (!submitted) {
        alert("⏰ Time's up! Submitting your answers.");
        submitAnswers(); // auto-submit when time is up
      }
    }
  }, 1000);
}

  const subjects = [
    "Polity",
    "Economy",
    "Modern History",
    "Ancient & Medieval History",
    "Art & Culture",
    "Science & Technology",
    "Environment & Ecology",
    "Current Affairs",
    "Previous Year Questions",
    "Physics" // <-- add this
  ];

async function fetchQuestions() {
  loading = true;
  submitted = false;
  error = "";

  try {
const res = await fetch(`/api/questions?subject=${subject}&count=${count}`);

    if (!res.ok) {
  const html = await res.text();  // <-- get full error response
  console.error("Unexpected HTML response:", html);
  throw new Error("Failed to fetch questions");
}

    questions = await res.json();
    answers = {};
    quizStarted = true;
    currentIndex = 0;
    startTimer();

    // ✅ Trigger MathJax after DOM update
    await tick();
    if (window.MathJax && typeof window.MathJax.typesetPromise === "function") {
      await window.MathJax.typesetPromise();
    }

  } catch (err) {
    console.error(err);
    error = err.message || "Failed to load questions.";
  } finally {
    loading = false;
  }
}

  async function submitAnswers() {
  clearInterval(timerInterval); // Stop timer on submit

  for (let i = 0; i < questions.length; i++) {
    const q = questions[i];
    const selected = answers[i];
    if (!selected) continue;

    await fetch(`https://backend.sunny-vanamala4.workers.dev/api/submit`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}` // 🔐 Attach the Clerk JWT
      },
      body: JSON.stringify({
        subject,
        question: q.question,
        selected_option: selected,
        correct_option: q.correct_option,
      }),
    });
  }

  submitted = true;
}


function resetQuiz() {
  quizStarted = false;
  submitted = false;
  questions = [];
  answers = {};
  timeLeft = 600;
  error = "";
}


  let typedElement: HTMLSpanElement;

  onMount(() => {
    const typed = new Typed(typedElement, {
      strings: [
        "Crack UPSC with confidence.",
        "Practice. Analyze. Improve.",
        "Daily mock tests. Real results."
      ],
      typeSpeed: 50,
      backSpeed: 30,
      loop: true
    });

    return () => typed.destroy();
  });

  function formatTime(t: number) {
    const m = Math.floor(t / 60);
    const s = String(t % 60).padStart(2, '0');
    return `${m}:${s}`;
  }

  function selectOption(opt: string) {
    answers[currentIndex] = opt;
  }
</script>

<!-- Header Bar: Centered page name and auth buttons -->
<nav class="main-nav">
  <div class="nav-content">
    <div class="nav-title">
      <span style="font-size:1.7rem;vertical-align:middle;">🎓</span>
      UPSC Master
    </div>
    <div class="nav-auth">
      {#if user}
        <span>👋 {user.firstName}</span>
        <button on:click={signOut}>Sign out</button>
      {:else}
        <button on:click={signIn}>Sign In</button>
      {/if}
    </div>
  </div>
</nav>

{#if step === 1}
  <div class="subjects-landing">
    <div class="subjects-section">
      <h2 class="subjects-title">Choose a Subject</h2>
      <div class="subjects-grid">
        {#each subjects as s}
          <div
            class="subject-card {subject === s ? 'selected' : ''}"
            on:click={() => { subject = s; step = 2; }}
          >
            {s}
          </div>
        {/each}
      </div>
    </div>
  </div>
{:else if step === 2}
  <div class="subjects-landing">
    <div class="subjects-section">
      <h2 class="subjects-title">How many questions?</h2>
      <div class="question-count-select">
        <input type="number" min="1" max="100" bind:value={count} class="question-count-input" />
        <button class="start-btn" on:click={() => { step = 3; fetchQuestions(); }}>Start Quiz</button>
      </div>
    </div>
  </div>
{:else}
  {#if questions.length && !submitted}
    <div class="progress-bar">
      <div class="progress" style="width: {(currentIndex + 1) / questions.length * 100}%"></div>
    </div>
  {/if}

  <div class="main-card">
    {#if questions.length && !submitted}
      <!-- Question Display -->
      <div class="bg-white p-6 rounded-xl shadow-md mb-6">
        <div class="text-sm text-gray-500 mb-2">
          Question {currentIndex + 1} of {questions.length} · {subject} · ⏰ {formatTime(timeLeft)}
        </div>

        <div class="text-xl font-semibold mb-4">
          {@html questions[currentIndex]?.question}
        </div>

        <div class="space-y-2">
          {#each questions[currentIndex]?.options as option, i}
            <button
              class="block w-full text-left px-4 py-2 rounded-lg border border-gray-300 hover:bg-blue-50"
              on:click={() => selectOption(String.fromCharCode(65 + i))}
              class:selected={answers[currentIndex] === String.fromCharCode(65 + i)}
            >
              <span class="font-bold">{String.fromCharCode(65 + i)}.</span>
              <span class="ml-2">{@html option}</span>
            </button>
          {/each}
        </div>
      </div>
      <div class="q-actions">
        <button on:click={() => currentIndex--} disabled={currentIndex === 0}>Previous</button>
        {#if currentIndex < questions.length - 1}
          <button on:click={() => currentIndex++}>Next Question</button>
        {:else}
          <button class="submit-btn" on:click={submitAnswers}>Submit Quiz</button>
        {/if}
      </div>
    {:else if !submitted}
      <div class="quiz-controls">
        <select bind:value={subject}>
          {#each subjects as s}
            <option value={s}>{s}</option>
          {/each}
        </select>
        <input type="number" bind:value={count} min="1" max="100" placeholder="Number of Questions" />
        <button on:click={fetchQuestions}>Start Quiz</button>
      </div>
    {/if}

    {#if loading}
      <p class="message">⏳ Loading questions...</p>
    {/if}
    {#if error}
      <p class="message error">⚠️ {error}</p>
    {/if}

    {#if submitted}
      <p class="message">✅ Answers submitted!</p>
      <h2 style="text-align:center; color:#424245;">📘 Review with Explanations</h2>
      {#each questions as q, i}
        <div class="question-block">
          <div class="question-text">{i + 1}. {q.question}</div>
          <div class="review-answer">
            <span>🟡 Your Answer: {answers[i] !== undefined ? answers[i] : "Not attempted"}</span>
            <span>✅ Correct Answer: {q.correct_option}</span>
          </div>
          <div class="explanation">📖 {q.explanation}</div>
        </div>
      {/each}
      <button class="submit-btn" on:click={resetQuiz} style="margin-top: 2rem;">🆕 Take Another Test</button>
    {/if}
  </div>
{/if}

<footer class="footer-bar">
  <div>
    <span>© {new Date().getFullYear()} QuizApp | Made with ❤️ for UPSC aspirants</span>
  </div>
  <div>
    <a href="#">Contact</a> | <a href="#">Privacy</a>
  </div>
</footer>

<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&family=Space+Grotesk:wght@700&display=swap" rel="stylesheet">

<style>
  :global(body) {
  font-family: 'Inter', 'Space Grotesk', Arial, sans-serif;
  background: #111;
  color: #f3f4f6;
  margin: 0;
  padding: 0;
  min-height: 100vh;
  letter-spacing: 0.01em;
}

  :global(body) {
  font-family: 'Inter', 'Segoe UI', Roboto, Arial, sans-serif;
  background: linear-gradient(135deg, #f8fafc 0%, #e0e7ff 100%);
  color: #232526;
  margin: 0;
  padding: 0;
  min-height: 100vh;
}

.main-nav {
  width: 100vw;
  background: #111;
  border-bottom: 1px solid #222;
  box-shadow: none;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 100;
}
.nav-content {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
  padding: 0 2rem;
}
.nav-title {
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  font-size: 1.7rem;
  font-weight: 700;
  color: #fff;
  letter-spacing: 1px;
  text-align: center;
  flex: 1;
  display: flex;
  align-items: center;
  gap: 0.7rem;
}
.nav-auth {
  display: flex;
  align-items: center;
  gap: 1rem;
}
.nav-auth button {
  background: #222;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.5rem 1.2rem;
  font-weight: 600;
  font-size: 1rem;
  transition: background 0.2s;
}
.nav-auth button:hover {
  background: #333;
}
.nav-auth span {
  color: #232526;
  font-weight: 500;
}

.landing-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  padding-top: 100px;
  padding-bottom: 60px;
}
.landing-card {
  width: 100%;
  max-width: 700px;
  margin: 0 auto;
  text-align: center;
}
.landing-title {
  font-size: 2rem;
  font-weight: 700;
  margin-bottom: 0.3rem;
  color: #232526;
}
.landing-subtitle {
  color: #888;
  font-size: 1.1rem;
  margin-bottom: 2.5rem;
}
.quiz-setup-card {
  background: #fff;
  border-radius: 12px;
  border: 1.5px solid #e6e6e6;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  padding: 2.5rem 2rem 2rem 2rem;
  margin: 0 auto;
  max-width: 500px;
}
.quiz-title {
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 0.2rem;
  color: #232526;
}
.quiz-desc {
  color: #888;
  font-size: 1.08rem;
  margin-bottom: 1.5rem;
}
.quiz-section-label {
  font-weight: 600;
  margin-bottom: 0.7rem;
  margin-top: 1.2rem;
}
.question-count-row {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1.2rem;
  margin-bottom: 1rem;
}
.question-count-row button {
  background: #f8f9fa;
  border: 1px solid #e6e6e6;
  border-radius: 8px;
  font-size: 1.1rem;
  padding: 0.4rem 1.1rem;
  cursor: pointer;
  font-weight: 600;
  color: #232526;
  transition: background 0.2s, border 0.2s;
}
.question-count-row button:hover {
  background: #e6e6e6;
}
.question-count {
  font-size: 1.2rem;
  font-weight: 700;
  background: #f8f9fa;
  border-radius: 8px;
  padding: 0.4rem 1.2rem;
  border: 1.5px solid #e6e6e6;
}
.question-count-quick {
  display: flex;
  justify-content: center;
  gap: 0.7rem;
  margin-bottom: 0.7rem;
}
.question-count-quick button {
  background: #fff;
  border: 1.5px solid #e6e6e6;
  border-radius: 6px;
  font-size: 1rem;
  padding: 0.3rem 1.1rem;
  cursor: pointer;
  color: #232526;
  font-weight: 500;
  transition: background 0.2s, border 0.2s, color 0.2s;
}
.question-count-quick button.active,
.question-count-quick button:focus {
  background: #232526;
  color: #fff;
  border: 1.5px solid #232526;
}
.available-questions {
  color: #888;
  font-size: 0.98rem;
  margin-bottom: 1.2rem;
}
.quiz-info-box {
  background: #f8f9fa;
  border-radius: 8px;
  padding: 1.1rem 1rem 1rem 1.2rem;
  margin-bottom: 1.5rem;
  text-align: left;
  font-size: 1rem;
  color: #232526;
  border: 1px solid #e6e6e6;
}
.quiz-info-box ul {
  margin: 0.5rem 0 0 1.1rem;
  padding: 0;
}
.quiz-setup-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-top: 1.5rem;
}
.quiz-setup-actions select {
  background: #fff;
  color: #232526;
  border: 1px solid #e6e6e6;
  border-radius: 8px;
  padding: 0.6rem 1rem;
  font-size: 1rem;
  outline: none;
  transition: border 0.2s;
}
.quiz-setup-actions select:focus {
  border: 1.5px solid #424245;
}
.start-btn,
.quiz-controls button,
.q-actions button,
.submit-btn {
  background: linear-gradient(90deg, #6366f1 0%, #06b6d4 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 2rem;
  font-size: 1.08rem;
  font-weight: 600;
  transition: background 0.2s, box-shadow 0.2s;
  box-shadow: 0 2px 8px #6366f122;
}
.start-btn:hover,
.quiz-controls button:hover,
.q-actions button:hover:not(:disabled),
.submit-btn:hover {
  background: linear-gradient(90deg, #06b6d4 0%, #6366f1 100%);
  box-shadow: 0 4px 16px #6366f144;
}

.progress-bar {
  width: 100vw;
  background: #f8f9fa;
  height: 5px;
  margin-top: 60px;
}
.progress {
  background: linear-gradient(90deg, #6366f1 0%, #06b6d4 100%);
  height: 100%;
  border-radius: 3px;
  transition: width 0.5s cubic-bezier(.4,2.3,.3,1);
}

.main-card {
  max-width: 700px;
  margin: 90px auto 4.5rem auto;
  background: #18181b;
  border-radius: 18px;
  border: 1.5px solid #222;
  box-shadow: none;
  padding: 2.5rem 2rem 2rem 2rem;
  color: #fff;
  animation: fadeIn 0.7s cubic-bezier(.4,2.3,.3,1);
}
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(30px);}
  to { opacity: 1; transform: translateY(0);}
}

.question-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1.5rem;
  font-size: 1.2rem;
  font-weight: 500;
  color: #fff;
}
.q-title {
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  font-size: 1.3rem;
  font-weight: 700;
  color: #fff;
}
.q-subject {
  color: #22c55e;
  font-size: 1.1rem;
  font-weight: 700;
}
.q-timer {
  color: #fff;
  background: #18181b;
  border: 1.5px solid #22c55e;
  padding: 0.3rem 1rem;
  border-radius: 8px;
  font-weight: 700;
  letter-spacing: 1px;
}

.quiz-controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin: 2rem 0;
}
.quiz-controls select,
.quiz-controls input[type="number"] {
  background: #fff;
  color: #232526;
  border: 1px solid #bfc0c0;
  border-radius: 8px;
  padding: 0.6rem 1rem;
  font-size: 1rem;
  outline: none;
  transition: border 0.2s;
}
.quiz-controls select:focus,
.quiz-controls input[type="number"]:focus {
  border: 1.5px solid #424245;
}
.quiz-controls button {
  background: #424245;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 2rem;
  font-size: 1.05rem;
  font-weight: 600;
  transition: background 0.2s;
}
.quiz-controls button:hover {
  background: #232526;
}

.question-block {
  background: #111;
  color: #fff;
  border-radius: 14px;
  border: 1.5px solid #222;
  box-shadow: none;
  padding: 2rem 1.5rem;
  margin-bottom: 2rem;
}
.question-text {
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 1.5rem;
  color: #fff;
}
.options {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.option-box {
  display: flex;
  align-items: center;
  background: #18181b;
  border: 1.5px solid #222;
  border-radius: 8px;
  padding: 1rem 1.2rem;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: 500;
  transition: border 0.2s, background 0.2s, color 0.2s;
  user-select: none;
  color: #fff;
}
.option-box.selected,
.option-box:hover {
  border: 1.5px solid #22c55e;
  background: #22c55e22;
  color: #22c55e;
}
.option-box input[type="radio"] {
  margin-right: 1rem;
  accent-color: #22c55e;
}
.option-letter {
  font-weight: bold;
  margin-right: 0.7rem;
  color: #22c55e;
}

.q-actions {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 2rem;
}
.q-actions button,
.submit-btn {
  background: #22c55e;
  color: #111;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 1.7rem;
  font-size: 1.1rem;
  font-weight: 700;
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  transition: background 0.2s, color 0.2s, transform 0.13s;
}
.q-actions button:disabled {
  background: #222;
  color: #888;
  cursor: not-allowed;
}
.q-actions button:hover:not(:disabled),
.submit-btn:hover {
  background: #16a34a;
  color: #fff;
  transform: translateY(-2px) scale(1.03);
}

.review-answer {
  display: flex;
  gap: 2rem;
  margin: 1rem 0 0.5rem 0;
  font-size: 1rem;
}

.explanation {
  background-color: #f8f9fa;
  padding: 1rem;
  border-left: 4px solid #424245;
  margin-top: 1rem;
  border-radius: 8px;
  font-size: 0.98rem;
  color: #232526;
}

.message {
  text-align: center;
  font-size: 1.2rem;
  margin-bottom: 1.5rem;
  color: #232526;
}
.message.error {
  color: #ff4d4f;
}

.footer-bar {
  width: 100vw;
  <span style="color:#22c55e;">Your highlighted text</span>  background: #111;
  color: #888;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 1.1rem 2.5rem;
  font-size: 1rem;
  position: fixed;
  left: 0;
  bottom: 0;
  border-radius: 18px 18px 0 0;
  box-shadow: none;
  backdrop-filter: none;
  z-index: 100;
  text-align: center;
}
.footer-bar a {
  color: #22c55e;
  text-decoration: underline;
  margin: 0 0.5rem;
  transition: color 0.2s;
}
.footer-bar a:hover {
  color: #fff;
}

.info-landing {
  min-height: 100vh;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  background: #0f172a;
  padding-top: 80px;
  padding-bottom: 60px;
  flex-direction: column;
}
.info-card {
  background: #e6e6e6;
  border-radius: 14px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.06);
  padding: 2.5rem 2.5rem 2rem 2.5rem;
  max-width: 540px;
  margin: 0 auto;
  text-align: center;
  border: 1.5px solid #d1d1d1;
  animation: fadeIn 1s cubic-bezier(.4,2.3,.3,1);
}
.info-card h1 {
  font-size: 2.1rem;
  font-weight: 700;
  margin-bottom: 1.2rem;
  color: #232526;
}
.info-card ul {
  text-align: left;
  margin: 1.2rem 0 1.2rem 1.2rem;
  padding: 0;
  color: #232526;
  font-size: 1.08rem;
}
.info-card li {
  margin-bottom: 0.7rem;
}
.info-card p {
  color: #232526;
  font-size: 1.08rem;
  margin-bottom: 1.2rem;
}
.info-card .start-btn {
  background: #232526;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 2rem;
  font-size: 1.08rem;
  font-weight: 600;
  transition: background 0.2s;
  cursor: pointer;
  margin-top: 1.2rem;
}
.info-card .start-btn:hover {
  background: #424245;
}

.subjects-section {
  width: 100%;
  max-width: 700px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: none;
  box-shadow: none;
  border-radius: 0;
  padding: 0;
}

.subjects-title {
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  font-size: 3rem;
  font-weight: 700;
  color: #fff;
  margin-bottom: 2.5rem;
  text-align: center;
  letter-spacing: -0.01em;
}

.subjects-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1.5rem;
  justify-content: center;
  width: 100%;
}

.subject-card {
  min-width: 180px;
  min-height: 80px;
  background: #18181b;
  color: #fff;
  font-size: 1.25rem;
  font-weight: 600;
  border-radius: 14px;
  box-shadow: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: 
    background 0.18s,
    color 0.18s,
    transform 0.13s;
  border: 2px solid #222;
  padding: 0 2rem;
  user-select: none;
  letter-spacing: 0.01em;
}
.subject-card:hover,
.subject-card.selected {
  background: #22c55e;
  color: #111;
  border: 2px solid #22c55e;
  transform: translateY(-4px) scale(1.04);
}

.question-count-select {
  display: flex;
  gap: 1.5rem;
  justify-content: center;
  align-items: center;
  margin-top: 2rem;
}
.question-count-input {
  background: #18181b;
  color: #fff;
  border: 2px solid #222;
  border-radius: 10px;
  padding: 1rem 2rem;  <span style="color:#22c55e;">Your highlighted text</span>
  font-size: 1.3rem;
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  width: 130px;
  text-align: center;
  outline: none;
  transition: border 0.2s;
}
.question-count-input:focus {
  border: 2px solid #22c55e;
}
.start-btn {
  background: #22c55e;
  color: #111;
  border: none;
  border-radius: 10px;
  padding: 1rem 2.2rem;
  font-size: 1.15rem;
  font-weight: 700;
  font-family: 'Space Grotesk', 'Inter', sans-serif;
  transition: background 0.2s, box-shadow 0.2s, transform 0.13s;
  box-shadow: none;
  letter-spacing: 0.01em;
}
.start-btn:hover {
  background: #16a34a;
  color: #fff;
  transform: translateY(-2px) scale(1.03);
}

.info-card {
  background: #e6e6e6;
  border-radius: 14px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.06);
  padding: 2.5rem 2.5rem 2rem 2.5rem;
  max-width: 540px;
  margin: 0 auto;
  text-align: center;
  border: 1.5px solid #d1d1d1;
  animation: fadeIn 1s cubic-bezier(.4,2.3,.3,1);
}
.info-card h1 {
  font-size: 2.1rem;
  font-weight: 700;
  margin-bottom: 1.2rem;
  color: #232526;
}
.info-card ul {
  text-align: left;
  margin: 1.2rem 0 1.2rem 1.2rem;
  padding: 0;
  color: #232526;
  font-size: 1.08rem;
}
.info-card li {
  margin-bottom: 0.7rem;
}
.info-card p {
  color: #232526;
  font-size: 1.08rem;
  margin-bottom: 1.2rem;
}
.info-card .start-btn {
  background: #232526;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 2rem;
  font-size: 1.08rem;
  font-weight: 600;
  transition: background 0.2s;
  cursor: pointer;
  margin-top: 1.2rem;
}
.info-card .start-btn:hover {
  background: #424245;
}
</style>

<div class="info-card">
  <h1>Welcome to UPSC Master!</h1>
  <h2 style="margin-top: 0.5rem; font-weight: normal; color: #424245;">
    <span bind:this={typedElement}></span>
  </h2>
</div>