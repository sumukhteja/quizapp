<script lang="ts">
  import { Clerk } from "@clerk/clerk-js";
  import { onMount } from "svelte";

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
    "Previous Year Questions"
  ];

async function fetchQuestions() {
  loading = true;
  submitted = false;
  error = "";
  try {
    const res = await fetch(
      `https://backend.sunny-vanamala4.workers.dev/api/questions?subject=${encodeURIComponent(subject)}&count=${count}`
    );
    if (!res.ok) throw new Error("Failed to fetch questions");
    questions = await res.json();
    answers = {};
    quizStarted = true;     // 🔥 Start quiz UI
    currentIndex = 0;
    startTimer();
  } catch (err: any) {
    error = err.message || "Unknown error";
    questions = [];
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


</script>

<!-- Header Bar: Centered page name and auth buttons -->
<nav class="main-nav">
  <div class="nav-content">
    <div class="nav-title">🧑‍🎓 UPSC Master</div>
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

{#if !quizStarted && !submitted && !showMockTest}
  <div class="info-landing">
    <div class="info-card">
      <h1>Welcome to UPSC Master!</h1>
      <p>
        <b>UPSC Master</b> is your one-stop platform for practicing high-quality, exam-oriented mock tests for the UPSC Civil Services Preliminary Exam.
      </p>
      <ul>
        <li>📝 Practice subject-wise and full-length mock tests.</li>
        <li>📊 Get instant feedback and explanations for every answer.</li>
        <li>⏱️ Simulate real exam conditions with a timer and progress tracking.</li>
        <li>🏆 Track your performance and compare with others on the leaderboard.</li>
        <li>🔒 Sign in to save your progress and access advanced features.</li>
      </ul>
      <p>
        <b>Why use UPSC Master?</b><br>
        Practicing with realistic questions and explanations is the best way to boost your confidence and improve your score. Whether you are a beginner or a seasoned aspirant, our platform adapts to your needs.
      </p>
      <button class="start-btn" on:click={() => showMockTest = true}>Start Practicing</button>
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
      <div class="question-header">
        <span class="q-title">Question {currentIndex + 1} of {questions.length}</span>
        <span class="q-subject">{subject}</span>
        <span class="q-timer">⏰ {Math.floor(timeLeft / 60)}:{String(timeLeft % 60).padStart(2, '0')}</span>
      </div>
      <div class="question-block">
        <div class="question-text">{questions[currentIndex].question}</div>
        <div class="options">
          {#each questions[currentIndex].options as opt, idx}
            <label class="option-box {answers[currentIndex] === opt ? 'selected' : ''}">
              <input type="radio" name={"q" + currentIndex} value={opt} bind:group={answers[currentIndex]} />
              <span class="option-letter">{String.fromCharCode(65 + idx)}.</span>
              <span>{opt}</span>
            </label>
          {/each}
        </div>
        <div class="q-actions">
          <button on:click={() => currentIndex--} disabled={currentIndex === 0}>Previous</button>
          {#if currentIndex < questions.length - 1}
            <button on:click={() => currentIndex++}>Next Question</button>
          {:else}
            <button class="submit-btn" on:click={submitAnswers}>Submit Quiz</button>
          {/if}
        </div>
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

<style>
  :global(body) {
  font-family: 'Inter', 'Segoe UI', Roboto, Arial, sans-serif;
  background: #f8f9fa;
  color: #232526;
  margin: 0;
  padding: 0;
}

.main-nav {
  width: 100vw;
  background: #fff;
  border-bottom: 1.5px solid #e6e6e6;
  box-shadow: 0 2px 8px #0001;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 100;
}
.nav-content {
  max-width: 900px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 60px;
  padding: 0 2rem;
}
.nav-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #424245;
  letter-spacing: 1px;
  text-align: center;
  flex: 1;
}
.nav-auth {
  display: flex;
  align-items: center;
  gap: 1rem;
}
.nav-auth button {
  background: #424245;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.4rem 1.1rem;
  font-weight: bold;
  font-size: 1rem;
  transition: background 0.2s;
}
.nav-auth button:hover {
  background: #232526;
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
.start-btn {
  background: #232526;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.7rem 2rem;
  font-size: 1.05rem;
  font-weight: 600;
  transition: background 0.2s;
  cursor: pointer;
}
.start-btn:hover {
  background: #424245;
}

.progress-bar {
  width: 100vw;
  background: #f8f9fa;
  height: 5px;
  margin-top: 60px;
}
.progress {
  background: #424245;
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s;
}

.main-card {
  max-width: 900px;
  margin: 90px auto 4.5rem auto;
  background: #fff;
  border-radius: 12px;
  border: 1.5px solid #e6e6e6;
  box-shadow: 0 4px 24px rgba(0,0,0,0.06);
  padding: 2.5rem 2rem 2rem 2rem;
}

.question-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1.5rem;
  font-size: 1.1rem;
  font-weight: 500;
  color: #232526;
}
.q-title { font-weight: 600; }
.q-subject { color: #888; font-size: 1rem; }
.q-timer { color: #424245; font-weight: 600; }

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
  background: #fff;
  color: #232526;
  border-radius: 12px;
  border: 1.5px solid #e6e6e6;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  padding: 2rem 1.5rem;
  margin-bottom: 2rem;
}
.question-text {
  font-size: 1.18rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
}
.options {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.option-box {
  display: flex;
  align-items: center;
  background: #f8f9fa;
  border: 1.5px solid #e6e6e6;
  border-radius: 8px;
  padding: 0.8rem 1rem;
  cursor: pointer;
  font-size: 1.05rem;
  font-weight: 500;
  transition: border 0.2s, background 0.2s;
  user-select: none;
}
.option-box.selected,
.option-box:hover {
  border: 1.5px solid #424245;
  background: #f1f2f6;
}
.option-box input[type="radio"] {
  margin-right: 1rem;
  accent-color: #424245;
}
.option-letter {
  font-weight: bold;
  margin-right: 0.7rem;
  color: #424245;
}

.q-actions {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 2rem;
}
.q-actions button,
.submit-btn {
  background: #424245;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.6rem 1.5rem;
  font-size: 1rem;
  font-weight: 600;
  transition: background 0.2s;
}
.q-actions button:disabled {
  background: #e6e6e6;
  color: #aaa;
  cursor: not-allowed;
}
.q-actions button:hover:not(:disabled),
.submit-btn:hover {
  background: #232526;
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
  background: #fff;
  color: #232526;
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
  box-shadow: 0 -2px 12px rgba(0,0,0,0.08);
  z-index: 100;
  text-align: center;
}
.footer-bar > div {
  margin: 0.2rem 0;
}
.footer-bar a {
  color: #424245;
  text-decoration: underline;
  margin: 0 0.5rem;
  transition: color 0.2s;
}
.footer-bar a:hover {
  color: #232526;
}

.info-landing {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #f8f9fa;
    padding-top: 80px;
    padding-bottom: 60px;
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