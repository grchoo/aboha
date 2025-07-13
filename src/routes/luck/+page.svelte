<script>
	import { onMount } from 'svelte';
	import { fade } from 'svelte/transition';
	import { getRandomNumber, isSameDate } from '$utils';
	import cookie from '$lib/images/cookie1.png';
	import crackedCookie from '$lib/images/cookie2.png';
	import Background from './Background.svelte';
	import html2canvas from 'html2canvas';

	export let data;

	let randomFortuneData;
	let cracked = false;
	let captureAreaElement;

	const STORAGE_KEY = 'fortuneData';

	const currentDate = new Date().toLocaleString('en-US', { timeZone: 'Asia/Seoul' });

	const generateNewData = () => {
		const { database } = data;
		const dataCount = database.length;
		randomFortuneData = database[getRandomNumber(0, dataCount)];

		const newData = { data: randomFortuneData, date: currentDate };
		localStorage.setItem(STORAGE_KEY, JSON.stringify(newData));
	};

	onMount(() => {
		const storedData = localStorage.getItem(STORAGE_KEY);

		if (storedData) {
			const { data: savedData, date: savedDate } = JSON.parse(storedData);
			if (isSameDate(new Date(savedDate), new Date())) {
				randomFortuneData = savedData;
			} else {
				generateNewData();
			}
		} else {
			generateNewData();
		}
	});

	const crackCookie = () => {
		cracked = true;
	};

	const fallbackImageCopy = (canvas) => {
		const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
		const link = document.createElement('a');
		link.download = `운세_${new Date().toISOString().split('T')[0]}.png`;
		link.href = canvas.toDataURL();
		link.click();

		if (isIOS) {
			alert(
				'클립보드 복사가 지원되지 않아 이미지를 다운로드했습니다.\n\n사진 앱에 저장하려면:\n1. 파일 앱 열기\n2. \'다운로드\' 폴더에서 이미지 찾기\n3. 이미지를 길게 눌러 [공유] > [이미지 저장] 선택'
			);
		} else {
			alert('클립보드 복사에 실패하여 이미지를 다운로드했습니다.');
		}
	};

	const shareFortune = async () => {
		if (!captureAreaElement) return;

		// 사용자 동작(클릭) 상태가 만료되지 않도록 지연 시간을 최소화합니다.
		setTimeout(async () => {
			const luckText = captureAreaElement.querySelector('.luck-text');
			const shareButton = captureAreaElement.querySelector('.share-button');
			let originalTransition = '';

			try {
				// 캡처 직전에 애니메이션을 비활성화하고 텍스트를 보이게 강제합니다.
				if (luckText) {
					originalTransition = luckText.style.transition;
					luckText.style.transition = 'none';
					luckText.style.opacity = '1';
				}

				const captureOptions = {
					backgroundColor: null,
					scale: 1.5, // 이미지 크기 최적화
					useCORS: true,
					allowTaint: true,
					width: captureAreaElement.offsetWidth,
					height: captureAreaElement.offsetHeight
				};

				if (shareButton) {
					const captureAreaRect = captureAreaElement.getBoundingClientRect();
					const shareButtonRect = shareButton.getBoundingClientRect();
					captureOptions.height = shareButtonRect.bottom - captureAreaRect.top + 50;
				}

				const canvas = await html2canvas(captureAreaElement, captureOptions);

				const blob = await new Promise((resolve) => canvas.toBlob(resolve, 'image/png'));
				if (!blob) {
					alert('이미지 생성에 실패했습니다.');
					return;
				}

				const file = new File([blob], '운세.png', { type: 'image/png' });
				const shareData = {
					files: [file]
				};

				if (navigator.share && navigator.canShare(shareData)) {
					try {
						await navigator.share(shareData);
					} catch (error) {
						if (error.name !== 'AbortError') {
							console.error('Share API 에러:', error);
							alert(
								'이미지를 공유하는 데 실패했습니다.\n공유하기 버튼을 눌러 다시 시도해주세요.'
							);
						}
					}
				} else {
					fallbackImageCopy(canvas);
				}
			} catch (captureError) {
				console.error('이미지 캡처 실패: ', captureError);
				alert('이미지 캡처에 실패했습니다.');
			} finally {
				// 캡처 후에는 원래 스타일로 복구합니다.
				if (luckText) {
					luckText.style.transition = originalTransition;
				}
			}
		}, 100); // 지연 시간 최소화
	};
</script>

<div bind:this={captureAreaElement} class="capture-area">
	<Background />
	<header>
		<h1>아 운세 보고 싶다</h1>
		<span class="date">{new Date().getFullYear()}년 {new Date().getMonth() + 1}월 {new Date().getDate()}일</span>
	</header>
	<div class="container">
		<div
			class="cookie-container"
			role="button"
			aria-pressed={cracked}
			on:click={crackCookie}
			on:keydown={(e) => {
				if (e.key === 'Enter' || e.key === ' ') {
					e.preventDefault();
					crackCookie();
				}
			}}
			tabindex="0"
		>
			<img src={cracked ? crackedCookie : cookie} alt="fortune cookie" class:cracked />
			<!-- 쿠키 위에 겹치는 운세 텍스트 -->
			{#if cracked}
				<div class="luck-text" transition:fade={{ duration: 1000 }}>{randomFortuneData}</div>
			{/if}
		</div>
		{#if cracked}
			<button class="share-button" on:click={shareFortune} transition:fade={{ duration: 1000 }}>공유하기</button>
		{/if}
	</div>
</div>

<style>
.capture-area {
	position: relative;
	width: 100vw;
	min-height: 100vh;
	overflow-x: hidden;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: flex-start;
	background: transparent;
}
header {
	text-align: center;
	color: white;
}
h1 {
	font-size: 1.8rem;
	font-weight: 700;
	margin: 2rem 0 1rem 0;
}
.date {
	color: #eeeeee;
	font-size: 0.8rem;
}
.container {
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	margin-top: 20px;
	position: relative;
	z-index: 0;
}
.cookie-container {
	cursor: pointer;
	width: 80%;
	max-width: 400px;
	position: relative;
}
.cookie-container img {
	width: 100%;
	height: auto;
	object-fit: contain;
	transition: transform 0.5s ease, opacity 0.5s ease;
}
.cookie-container img.cracked {
	transform: scale(1.1);
	opacity: 0.8;
}
.luck-text {
	position: absolute;
	top: 50%;
	left: 50%;
	transform: translate(-50%, -50%);
	min-width: 70%;
	max-width: 85%;
	font-family: 'Pretendard', 'Noto Sans KR', 'IBM Plex Sans KR', -apple-system, BlinkMacSystemFont,
		'Segoe UI', Roboto, sans-serif;
	font-size: 1.2rem;
	font-weight: 600;
	color: #2c3e50;
	background: rgba(255, 255, 255, 0.95);
	backdrop-filter: blur(10px);
	border-radius: 16px;
	padding: 16px 24px;
	text-align: center;
	white-space: pre-wrap;
	word-wrap: break-word;
	line-height: 1.6;
	letter-spacing: -0.02em;
	box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12), 0 4px 16px rgba(0, 0, 0, 0.08),
		inset 0 1px 0 rgba(255, 255, 255, 0.8);
	border: 1px solid rgba(255, 255, 255, 0.3);
	z-index: 10;
}
.share-button {
	margin-top: 32px;
	padding: 14px 32px;
	background: linear-gradient(135deg, #4a90e2, #357abd);
	color: white;
	border: none;
	border-radius: 25px;
	font-size: 1.1rem;
	font-weight: 600;
	cursor: pointer;
	box-shadow: 0 4px 15px rgba(74, 144, 226, 0.3);
	transition: all 0.3s ease;
}
.share-button:hover {
	transform: translateY(-2px);
	box-shadow: 0 6px 20px rgba(74, 144, 226, 0.4);
	background: linear-gradient(135deg, #357abd, #2c5aa0);
}
.share-button:active {
	transform: translateY(0);
	box-shadow: 0 2px 10px rgba(74, 144, 226, 0.3);
}
@media (max-width: 768px) {
	.luck-text {
		font-size: 1rem;
		padding: 12px 20px;
		min-width: 80%;
		max-width: 90%;
		border-radius: 14px;
	}
	.share-button {
		padding: 10px 20px;
		font-size: 0.95rem;
	}
}
</style>
