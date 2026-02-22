<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	let transcript = '';
	let recognizing = false;

	// キーバインド設定用変数
	let startKey = 's'; // デフォルト: sキー
	let stopKey = 'd';  // デフォルト: dキー
	let keyBindInputMode: 'none' | 'start' | 'stop' = 'none';
	let tempKey = '';

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
	let sentencePauseMs = 3000;

	// 型定義
	type SpeechRecognitionType = typeof window & {
		SpeechRecognition?: any;
		webkitSpeechRecognition?: any;
	};

	function handleKeyBindInput(e: KeyboardEvent) {
		// キーバインド設定モード時
		if (keyBindInputMode !== 'none') {
			// 英数字・記号・ファンクションキーのみpreventDefault（Tab, Enter, Escなどは除外）
			// ただし、設定用キーは全て受け付ける
			tempKey = e.key;
			// 例: Tab, Enter, Shift, Control, Alt, Escape などはUI操作に影響するので除外
			const ignoreKeys = ['Tab', 'Enter', 'Escape', 'Shift', 'Control', 'Alt', 'Meta', 'ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'];
			if (!ignoreKeys.includes(tempKey)) {
				e.preventDefault();
			}
			if (keyBindInputMode === 'start') {
				startKey = tempKey;
			} else if (keyBindInputMode === 'stop') {
				stopKey = tempKey;
			}
			keyBindInputMode = 'none';
			return;
		}
		// 通常時: キーバインドで開始・停止
		if (e.target instanceof HTMLElement && ['INPUT', 'TEXTAREA'].includes(e.target.tagName)) return;
		if (e.key === startKey && !recognizing) {
			startRecognition();
		} else if (e.key === stopKey && recognizing) {
			stopRecognition();
		}
	}


	// 3秒以上空いたら次の音声で上書きするためのタイマー
	let overwriteNext = false;
	let silenceTimeout: ReturnType<typeof setTimeout> | null = null;
	let previousInterim = '';

	function mergeLongSound(finalText: string, interimText: string) {
		const finalTrimmed = finalText.trim();
		const interimTrimmed = interimText.trim();
		if (!finalTrimmed || !interimTrimmed) return finalText;
		if (!interimTrimmed.startsWith(finalTrimmed)) return finalText;
		const suffix = interimTrimmed.slice(finalTrimmed.length);
		if (!/^ー+$/.test(suffix)) return finalText;
		return `${finalText}${suffix}`;
	}

	function resetSilenceTimer() {
		if (silenceTimeout) clearTimeout(silenceTimeout);
		silenceTimeout = setTimeout(() => {
			overwriteNext = true;
			transcript = '';
			displayText = '';
		}, sentencePauseMs);
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
		// キーバインド用イベントリスナー
		window.addEventListener('keydown', handleKeyBindInput);

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
			// 3秒以上空いていたら上書き
			if (overwriteNext) {
				transcript = '';
				overwriteNext = false;
			}
			resetSilenceTimer();
			for (let i = event.resultIndex; i < event.results.length; ++i) {
				const result = event.results[i];
				if (result.isFinal) {
					const finalText = mergeLongSound(result[0].transcript, previousInterim);
					transcript += finalText;
					previousInterim = '';
				} else {
					interim += result[0].transcript;
				}
			}
			if (interim) {
				previousInterim = interim;
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

			// Restart only if not manually stopped
			if (recognition && !manualStop) {
				recognition.start();
			}
		};
	});

	onDestroy(() => {
		if (recognition && recognizing) {
			recognition.stop();
		}
		if (silenceTimeout) clearTimeout(silenceTimeout);
		document.removeEventListener('fullscreenchange', handleFullscreenChange);
		window.removeEventListener('keydown', handleKeyBindInput);
		if (document.fullscreenElement === appContainer) {
			document.exitFullscreen();
		}
	});

	let displayText = '';

	$: textLength = displayText.length;
	$: normalTextFontSize =
		textLength > 180 ? '0.85rem' : textLength > 120 ? '0.95rem' : textLength > 70 ? '1rem' : '1.1rem';
	$: fullscreenTextFontSize =
		textLength > 220
			? 'clamp(1.2rem, 2.6vw, 2rem)'
			: textLength > 150
				? 'clamp(1.4rem, 3.2vw, 2.8rem)'
				: textLength > 90
					? 'clamp(1.8rem, 4.2vw, 3.8rem)'
					: 'clamp(2rem, 6vw, 5rem)';

	let manualStop = false; // Flag to prevent auto-restart

	function startRecognition() {
		if (recognition && !recognizing) {
			manualStop = false; // 停止後の再開を許可
			transcript = '';
			displayText = '';
			previousInterim = '';
			recognition.start();
		}
	}
	function stopRecognition() {
		if (recognition && recognizing) {
			manualStop = true; // Set flag to indicate manual stop
			recognition.stop();
			recognizing = false;
			if (silenceTimeout) clearTimeout(silenceTimeout);
		}
	}
</script>

<div bind:this={appContainer}>
	{#if !isFullscreenMode}
		<main style="background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%); color: #e0e0e0; min-height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;">
			<div style="max-width: 600px; width: 90%; padding: 2rem;">
				<h1 style="font-size: 2.5rem; margin-bottom: 1.5rem; font-weight: 700; text-align: center; color: #00d4ff;">音声認識</h1>
				<p style="text-align: center; color: #b0b0b0; margin-bottom: 2rem; font-size: 0.95rem;">Web Speech API</p>

				<!-- キーバインド設定UI -->
				<div style="margin-bottom: 1.2rem; text-align: center;">
					<label style="color: #b0b0b0; font-size: 0.95rem; margin-right: 1rem;">
						開始キー: <span style="color: #00d4ff; font-weight: 600;">{startKey}</span>
						<button style="margin-left: 0.5rem; padding: 0.3rem 0.7rem; font-size: 0.9rem; border-radius: 0.3rem; border: none; background: #222; color: #fff; cursor: pointer;" on:click={() => { keyBindInputMode = 'start'; tempKey = ''; }}>変更</button>
					</label>
					<label style="color: #b0b0b0; font-size: 0.95rem; margin-left: 1rem;">
						停止キー: <span style="color: #ff6b6b; font-weight: 600;">{stopKey}</span>
						<button style="margin-left: 0.5rem; padding: 0.3rem 0.7rem; font-size: 0.9rem; border-radius: 0.3rem; border: none; background: #222; color: #fff; cursor: pointer;" on:click={() => { keyBindInputMode = 'stop'; tempKey = ''; }}>変更</button>
					</label>
					{#if keyBindInputMode !== 'none'}
						<div style="margin-top: 0.7rem; color: #b0b0b0; font-size: 0.95rem;">
							新しいキーを押してください...
						</div>
					{/if}
				</div>

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

				<div style="margin-bottom: 1.5rem;">
					<label for="sentence-delay" style="display: block; color: #b0b0b0; font-size: 0.95rem; margin-bottom: 0.5rem; text-align: center;">
						次の文章に移るまで: <span style="color: #00d4ff; font-weight: 600;">{(sentencePauseMs / 1000).toFixed(1)}秒</span>
					</label>
					<input
						id="sentence-delay"
						type="range"
						min="1000"
						max="10000"
						step="500"
						bind:value={sentencePauseMs}
						style="width: 100%; accent-color: #00d4ff;"
					/>
				</div>

				<div style="margin-top: 1.5rem; padding: 1.5rem; border: 2px solid #00d4ff; border-radius: 0.75rem; min-height: 8rem; background: rgba(0, 212, 255, 0.05);">
					{#if displayText}
						<span style={`color: #00d4ff; font-size: ${normalTextFontSize}; line-height: 1.8; word-wrap: break-word;`}>
							{displayText}
						</span>
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
					<span style={`color: #00d4ff; font-size: ${fullscreenTextFontSize}; line-height: 1.5; word-wrap: break-word;`}>
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