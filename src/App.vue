<template>
	<main class="app">
		<h1>The Quiz</h1>

		<!-- Mute/Unmute Button -->
		<button v-if="!showWelcome" class="mute-btn" @click="toggleSound">
			<span v-if="isMuted">🔇</span>
			<span v-else>🔊</span>
		</button>

		<!-- Landing Page -->
		<section v-if="showWelcome" class="fade-in welcome">
			<p>Welcome to the Quiz App!</p>
			<div class="dropdowns">
				<label>
					Difficulty:
					<select v-model="selectedDifficulty">
						<option value="easy">Easy</option>
						<option value="medium">Medium</option>
						<option value="hard">Hard</option>
					</select>
				</label>
				<label>
					Category:
					<select v-model="selectedCategory">
						<option value="9">General Knowledge</option>
						<option value="18">Science: Computers</option>
						<option value="23">History</option>
						<option value="22">Geography</option>
					</select>
				</label>
				<label>
					No. of Questions:
					<select v-model="numQuestions">
						<option :value="5">5</option>
						<option :value="10">10</option>
						<option :value="20">20</option>
					</select>
				</label>
			</div>
			<br />
			<button @click="startQuiz">Start Quiz</button>
		</section>

		<!-- Loading Screen -->
		<section v-else-if="loading" class="fade-in">
			<p>Loading questions...</p>
		</section>

		<!-- Quiz Section -->
		<section class="quiz fade-in" v-else-if="!quizCompleted && questions.length > 0">
			<div class="quiz-info">
				<span class="question" v-html="decodeEntities(getCurrentQuestion.question)"></span>
				<span class="score">Score {{ score }}/{{ questions.length }}</span>
				<span class="q-no">Q{{ currentQuestion + 1 }}/{{ questions.length }}</span>
			</div>

			<div class="progress-bar">
				<div class="progress" :style="{ width: ((currentQuestion + 1) / questions.length * 100) + '%' }"></div>
				<span class="progress-label">Question {{ currentQuestion + 1 }} of {{ questions.length }}</span>
			</div>

			<div v-if="timeLeft > 0" class="timer">Time Left: {{ timeLeft }}s</div>
			<div v-else class="timer">Time's up!</div>

			<div class="options">
				<label
					v-for="(option, index) in getCurrentQuestion.options"
					:for="'option' + index"
					:class="`option ${getCurrentQuestion.selected == index
						? index == getCurrentQuestion.correct_answer
							? 'correct'
							: 'wrong'
						: ''} ${getCurrentQuestion.selected != null && index != getCurrentQuestion.selected ? 'disabled' : ''}`">
					<input
						type="radio"
						:id="'option' + index"
						:name="getCurrentQuestion.index"
						:value="index"
						v-model="getCurrentQuestion.selected"
						:disabled="getCurrentQuestion.selected"
						@change="SetAnswer"
					/>
					<span>{{ decodeEntities(option) }}</span>
				</label>
			</div>

			<button class="next-btn" @click="NextQuestion" :disabled="!getCurrentQuestion.selected && timeLeft > 0">
				{{
					getCurrentQuestion.index === questions.length - 1
						? 'Finish'
						: getCurrentQuestion.selected === null
							? 'Select an option'
							: 'Next question'
				}}
			</button>
		</section>

		<!-- Completion Section -->
		<section v-else-if="quizCompleted" class="fade-in">
			<h2>You have finished the quiz!</h2>
			<p>Your score is {{ score }}/{{ questions.length }}</p>
			<ul>
				<li v-for="q in questions" :key="q.index">
					<strong v-html="decodeEntities(q.question)"></strong><br />
					Your answer:
					<span :class="{ correct: q.selected == q.correct_answer, wrong: q.selected != q.correct_answer }">
						{{ decodeEntities(q.options[q.selected]) }}
					</span><br />
					Correct answer: {{ decodeEntities(q.options[q.correct_answer]) }}
				</li>
			</ul>
			<button class="restart-btn" @click="restartQuiz">Restart</button>
		</section>
	</main>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import confetti from 'canvas-confetti';

const loading = ref(false);
const showWelcome = ref(true);
const questions = ref([]);
const quizCompleted = ref(false);
const currentQuestion = ref(0);
const selectedDifficulty = ref('easy');
const selectedCategory = ref('9');
const numQuestions = ref(10);
const timeLeft = ref(15);
let timerInterval = null;
const isMuted = ref(false);
const bgMusic = new Audio('https://www.bensound.com/bensound-music/bensound-funnysong.mp3');
bgMusic.loop = true;

const toggleSound = () => {
	isMuted.value = !isMuted.value;
	bgMusic.muted = isMuted.value;
};

const startQuiz = () => {
	showWelcome.value = false;
	loading.value = true;
	bgMusic.play().catch(() => {});
	fetchData();
};

const fetchData = async () => {
	try {
		const response = await axios.get(`https://opentdb.com/api.php?amount=${numQuestions.value}&difficulty=${selectedDifficulty.value}&category=${selectedCategory.value}`);
		const data = response.data;
		questions.value = data.results.map((question, index) => {
			const options = [...question.incorrect_answers, question.correct_answer].sort(() => Math.random() - 0.5);
			return {
				...question,
				options,
				selected: null,
				correct_answer: options.indexOf(question.correct_answer),
				index,
			};
		});
		loading.value = false;
		startTimer();
	} catch (error) {
		console.error('Error fetching questions:', error);
		loading.value = false;
	}
};

const startTimer = () => {
	clearInterval(timerInterval);
	timeLeft.value = 15;
	timerInterval = setInterval(() => {
		if (timeLeft.value > 0) {
			timeLeft.value--;
		} else {
			clearInterval(timerInterval);
			NextQuestion();
		}
	}, 1000);
};

const decodeEntities = (html) => {
	const txt = document.createElement('textarea');
	txt.innerHTML = html;
	return txt.value;
};

const score = computed(() => {
	let value = 0;
	questions.value.forEach((q) => {
		if (q.selected !== null && String(q.correct_answer) === String(q.selected)) {
			value++;
		}
	});
	return value;
});

const getCurrentQuestion = computed(() => {
	return questions.value[currentQuestion.value];
});

const SetAnswer = (e) => {
	questions.value[currentQuestion.value].selected = e.target.value;
	clearInterval(timerInterval);
};

const fireConfetti = () => {
	const duration = 3 * 1000;
	const end = Date.now() + duration;

	const interval = setInterval(() => {
		confetti({
			particleCount: 50,
			startVelocity: 30,
			spread: 360,
			ticks: 60,
			origin: {
				x: Math.random(),
				y: Math.random() - 0.2,
			},
			colors: ['#2cce7d', '#ff5a5f', '#fff', '#f9c80e'],
		});

		if (Date.now() > end) clearInterval(interval);
	}, 250);
};

const NextQuestion = () => {
	if (currentQuestion.value < questions.value.length - 1) {
		currentQuestion.value++;
		startTimer();
		return;
	}
	quizCompleted.value = true;
	clearInterval(timerInterval);
	fireConfetti();
};

const restartQuiz = () => {
	showWelcome.value = true;
	questions.value = [];
	quizCompleted.value = false;
	currentQuestion.value = 0;
	clearInterval(timerInterval);

	// ✅ Stop the background music
	bgMusic.pause();
	bgMusic.currentTime = 0;
};
</script>

<style src="./style.css"></style>