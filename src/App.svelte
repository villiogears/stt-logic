
<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	let transcript = '';
	let recognizing = false;
	let recognition: SpeechRecognition | null = null;

	// 型定義
	type SpeechRecognitionType = typeof window & {
		SpeechRecognition?: any;
		webkitSpeechRecognition?: any;
	};


	// 3秒以上空いたら次の音声で上書きするためのタイマー
	let lastResultTime = 0;
	let overwriteNext = false;
	let silenceTimeout: ReturnType<typeof setTimeout> | null = null;

	function resetSilenceTimer() {
		if (silenceTimeout) clearTimeout(silenceTimeout);
		silenceTimeout = setTimeout(() => {
			overwriteNext = true;
			transcript = '';
			displayText = '';
		}, 3000);
	}


	onMount(() => {
		const SpeechRecognitionCtor =
			(window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
		if (!SpeechRecognitionCtor) {
			transcript = 'Web Speech APIはこのブラウザでサポートされていません。';
			return;
		}
		recognition = new SpeechRecognitionCtor();
		recognition.lang = 'ja-JP';
		recognition.continuous = true;
		recognition.interimResults = true;

		recognition.onresult = (event: any) => {
			let interim = '';
			const now = Date.now();
			// 3秒以上空いていたら上書き
			if (overwriteNext) {
				transcript = '';
				overwriteNext = false;
			}
			resetSilenceTimer();
			lastResultTime = now;
			for (let i = event.resultIndex; i < event.results.length; ++i) {
				const result = event.results[i];
				if (result.isFinal) {
					transcript += result[0].transcript;
				} else {
					interim += result[0].transcript;
				}
			}
			displayText = transcript + interim;
		};

		recognition.onstart = () => {
			recognizing = true;
			resetSilenceTimer();
		};
		recognition.onend = () => {
			recognizing = false;
			if (silenceTimeout) clearTimeout(silenceTimeout);
		};
	});

	onDestroy(() => {
		if (recognition && recognizing) {
			recognition.stop();
		}
		if (silenceTimeout) clearTimeout(silenceTimeout);
	});

	let displayText = '';

	function startRecognition() {
		if (recognition && !recognizing) {
			transcript = '';
			displayText = '';
			recognition.start();
		}
	}
	function stopRecognition() {
		if (recognition && recognizing) {
			recognition.stop();
		}
	}
</script>

<main>
	<h1>音声認識デモ（Web Speech API）</h1>
	<button on:click={startRecognition} disabled={recognizing}>開始</button>
	<button on:click={stopRecognition} disabled={!recognizing}>停止</button>
	<div style="margin-top:1em; padding:1em; border:1px solid #ccc; min-height:3em; background:#fafafa;">
		{#if displayText}
			<span style="color: #000;">{displayText}</span>
		{:else}
			<span style="color:#888;">ここに音声が文字起こしされます</span>
		{/if}
	</div>
</main>
