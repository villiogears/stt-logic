
<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	let transcript = '';
	let recognizing = false;
	let recognition: SpeechRecognition | null = null;

	// 型定義
	type SpeechRecognition = typeof window.SpeechRecognition;
	type SpeechRecognitionInstance = InstanceType<SpeechRecognition>;

	onMount(() => {
		const SpeechRecognition =
			(window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
		if (!SpeechRecognition) {
			transcript = 'Web Speech APIはこのブラウザでサポートされていません。';
			return;
		}
		recognition = new SpeechRecognition();
		recognition.lang = 'ja-JP';
		recognition.continuous = true;
		recognition.interimResults = true;

		recognition.onresult = (event: SpeechRecognitionEvent) => {
			let interim = '';
			for (let i = event.resultIndex; i < event.results.length; ++i) {
				const result = event.results[i];
				if (result.isFinal) {
					transcript += result[0].transcript;
				} else {
					interim += result[0].transcript;
				}
			}
			// 一時結果も表示
			$: displayText = transcript + interim;
		};

		recognition.onstart = () => {
			recognizing = true;
		};
		recognition.onend = () => {
			recognizing = false;
		};
	});

	onDestroy(() => {
		if (recognition && recognizing) {
			recognition.stop();
		}
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
