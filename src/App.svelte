<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	let transcript = '';
	let recognizing = false;

	type BrowserSpeechRecognition = {
		lang: string;
		continuous: boolean;
		interimResults: boolean;
		onresult: ((event: any) => void) | null;
		onstart: (() => void) | null;
		onend: (() => void) | null;
		start: () => void;
		stop: () => void;
	};

	let recognition: BrowserSpeechRecognition | null = null;
	let appContainer: HTMLElement | null = null;
	let isFullscreenMode = false;

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

	function handleFullscreenChange() {
		isFullscreenMode = document.fullscreenElement === appContainer;
	}

	async function toggleFullscreenMode() {
		if (!appContainer) return;
		try {
			if (document.fullscreenElement === appContainer) {
				await document.exitFullscreen();
				return;
			}
			if (!document.fullscreenElement) {
				await appContainer.requestFullscreen();
			}
		} catch (error) {
			console.error('全画面切り替えに失敗しました', error);
		}
	}


	onMount(() => {
		document.addEventListener('fullscreenchange', handleFullscreenChange);

		const SpeechRecognitionCtor =
			(window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
		if (!SpeechRecognitionCtor) {
			transcript = 'Web Speech APIはこのブラウザでサポートされていません。';
			return;
		}
		const recognitionInstance: BrowserSpeechRecognition = new SpeechRecognitionCtor();
		recognition = recognitionInstance;
		recognitionInstance.lang = 'ja-JP';
		recognitionInstance.continuous = true;
		recognitionInstance.interimResults = true;

		recognitionInstance.onresult = (event: any) => {
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

		recognitionInstance.onstart = () => {
			recognizing = true;
			resetSilenceTimer();
		};
		recognitionInstance.onend = () => {
			recognizing = false;
			if (silenceTimeout) clearTimeout(silenceTimeout);
		};
	});

	onDestroy(() => {
		if (recognition && recognizing) {
			recognition.stop();
		}
		if (silenceTimeout) clearTimeout(silenceTimeout);
		document.removeEventListener('fullscreenchange', handleFullscreenChange);
		if (document.fullscreenElement === appContainer) {
			document.exitFullscreen();
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

<div bind:this={appContainer}>
	{#if !isFullscreenMode}
		<main style="background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%); color: #e0e0e0; min-height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;">
			<div style="max-width: 600px; width: 90%; padding: 2rem;">
				<h1 style="font-size: 2.5rem; margin-bottom: 1.5rem; font-weight: 700; text-align: center; color: #00d4ff;">音声認識</h1>
				<p style="text-align: center; color: #b0b0b0; margin-bottom: 2rem; font-size: 0.95rem;">Web Speech API</p>

				<div style="display: flex; gap: 1rem; margin-bottom: 2rem; justify-content: center; flex-wrap: wrap;">
					<button
						on:click={startRecognition}
						disabled={recognizing}
						style="padding: 0.75rem 1.5rem; background: linear-gradient(135deg, #00d4ff, #0099ff); color: #000; border: none; border-radius: 0.5rem; font-weight: 600; cursor: pointer; transition: all 0.3s ease; font-size: 1rem;"
					>
						開始
					</button>
					<button
						on:click={stopRecognition}
						disabled={!recognizing}
						style="padding: 0.75rem 1.5rem; background: linear-gradient(135deg, #ff6b6b, #ee5a52); color: #fff; border: none; border-radius: 0.5rem; font-weight: 600; cursor: pointer; transition: all 0.3s ease; font-size: 1rem;"
					>
						停止
					</button>
					<button
						on:click={toggleFullscreenMode}
						style="padding: 0.75rem 1.5rem; background: linear-gradient(135deg, #b388ff, #7c4dff); color: #fff; border: none; border-radius: 0.5rem; font-weight: 600; cursor: pointer; transition: all 0.3s ease; font-size: 1rem;"
					>
						全画面表示
					</button>
				</div>

				<div style="margin-top: 1.5rem; padding: 1.5rem; border: 2px solid #00d4ff; border-radius: 0.75rem; min-height: 8rem; background: rgba(0, 212, 255, 0.05);">
					{#if displayText}
						<span style="color: #00d4ff; font-size: 1.1rem; line-height: 1.8; word-wrap: break-word;">{displayText}</span>
					{:else}
						<span style="color: #5a5a7a; font-style: italic;">ここに音声が文字起こしされます</span>
					{/if}
				</div>

				{#if recognizing}
					<div style="margin-top: 1.5rem; text-align: center;">
						<div style="display: inline-block; width: 12px; height: 12px; background: #00d4ff; border-radius: 50%;"></div>
						<p style="color: #00d4ff; margin-top: 0.5rem; font-size: 0.9rem;">リッスン中...</p>
					</div>
				{/if}
			</div>
		</main>
	{:else}
		<main style="background: #0a0a14; color: #e0e0e0; min-height: 100vh; width: 100%; display: flex; align-items: center; justify-content: center; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; position: relative; padding: 2rem; box-sizing: border-box;">
			<button
				on:click={toggleFullscreenMode}
				style="position: absolute; top: 1.5rem; right: 1.5rem; padding: 0.65rem 1.1rem; background: linear-gradient(135deg, #ff6b6b, #ee5a52); color: #fff; border: none; border-radius: 0.5rem; font-weight: 600; cursor: pointer; font-size: 0.95rem;"
			>
				全画面終了
			</button>

			<div style="max-width: 92vw; text-align: center;">
				{#if displayText}
					<span style="color: #00d4ff; font-size: clamp(2rem, 6vw, 5rem); line-height: 1.5; word-wrap: break-word;">
						{displayText}
					</span>
				{:else}
					<span style="color: #5a5a7a; font-style: italic; font-size: clamp(1.3rem, 3vw, 2rem);">
						ここに音声が文字起こしされます
					</span>
				{/if}
			</div>
		</main>
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		background: #0f0f1e;
	}

	button:disabled {
		opacity: 0.5;
		cursor: not-allowed !important;
	}

button:not(:disabled):hover {
	transform: translate(0, -2px);
}

</style>	