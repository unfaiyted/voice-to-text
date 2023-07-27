<script>
	import { onDestroy } from 'svelte';

	let mediaRecorder;
	let audioQueue = [];
	let sendIntervalId = null;
	let isRecording = false;
	let requestInProgress = false;
	const QUEUE_MAX_TIME = 20000;  // 30 seconds
	let transcript = '';

	async function startRecording() {
		if (!isRecording) {
			console.log("Start recording...");
			const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
			mediaRecorder = new MediaRecorder(stream);
			mediaRecorder.addEventListener("dataavailable", event => {
				audioQueue.push({data: event.data, timestamp: Date.now()});

				// Remove chunks from the queue that are older than QUEUE_MAX_TIME
				// Cut oldest audio data from queue if it exceeds QUEUE_MAX_TIME

				 console.log('audioQueue', audioQueue);

				while ((audioQueue[audioQueue.length - 1].timestamp - audioQueue[0].timestamp) > QUEUE_MAX_TIME) {
					console.log("Trimming audio queue...");

					console.log('trimmed-audioQueue', audioQueue);
					// Instead of removing a fixed number of chunks, we remove chunks until we're below QUEUE_MAX_TIME again
					// This ensures we don't accidentally cut an audio chunk in half
					audioQueue.shift();
					if (audioQueue.length === 0) {
						break;
					}
				}

			});

			mediaRecorder.start(1000);

			sendIntervalId = setInterval(async () => {
				if(requestInProgress) {
					return;
				}

				// Only send data if there's at least 2 seconds of audio (2000ms)
				if ((audioQueue[audioQueue.length - 1].timestamp - audioQueue[0].timestamp) >= 1000) {
					console.log("Sending data to server...");
					requestInProgress = true;

					// Construct form data
					let formData = new FormData();
					let audioBlob = new Blob(audioQueue.map(chunk => chunk.data));

					// Log the size of the blob
					console.log(`Sending Blob with size: ${audioBlob.size} bytes`);

					formData.append("audio_file", audioBlob);

					// Make the HTTP POST request to the ASR endpoint
					const response = await fetch('http://192.168.0.120:9000/asr', {
						method: 'POST',
						body: formData,
					});

					if (response.ok) {
						const data = await response.text();
						console.log("Received response from server:", data);
						transcript = data; // replace the old transcript with the new one
					} else {
						console.error('Error:', response.status, response.statusText);
					}

					requestInProgress = false;
				}
			}, 4000);

			isRecording = true;
		}
	}

	function stopRecording() {
		if (isRecording) {
			console.log("Stop recording...");
			// Stop the mediaRecorder and the sending interval
			mediaRecorder.stop();
			clearInterval(sendIntervalId);
			sendIntervalId = null;
			isRecording = false;
		}
	}

	onDestroy(() => {
		stopRecording();
	});
</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Record to text Live" />
</svelte:head>

<section>

	<div>
		<button on:click={startRecording}>Start Recording</button>
		<button on:click={stopRecording}>Stop Recording</button>
	</div>

	<div>
		<p>{transcript}</p>
	</div>


</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 0.6;
	}

	h1 {
		width: 100%;
	}

	.welcome {
		display: block;
		position: relative;
		width: 100%;
		height: 0;
		padding: 0 0 calc(100% * 495 / 2048) 0;
	}

	.welcome img {
		position: absolute;
		width: 100%;
		height: 100%;
		top: 0;
		display: block;
	}
</style>
