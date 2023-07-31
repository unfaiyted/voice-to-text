<script>
	import { onDestroy } from 'svelte';

	let mediaRecorders = [];
	let audioQueue = [];
	let currentAudioQueueId = 0;
	let sendIntervalId = null;
	let overlapIntervalId = null;
	let submitIntervalId = null;
	let isRecording = false;
	let requestInProgress = false;
	const CHUNK_DURATION = 30 * 1000;  // 30 seconds in ms
	const SERVER_SEND_RATE = 2000; // 2 seconds in ms
	const OVERLAP_DURATION = 10 * 1000; // 10 seconds overlap
	let transcript = '';
	let transcriptHistory = [];
	let mergedTranscript = '';
	function jaccardSimilarityAndMerge(string1, string2, overlapThreshold = 0.5) {
		let string1Words = string1.replace(/[^\w\s]|_/g, "").replace(/\n/g, ' ').trim().toLowerCase().split(' ');
		let string2Words = string2.replace(/[^\w\s]|_/g, "").replace(/\n/g, ' ').trim().toLowerCase().split(' ');

		let intersection;
		let similarityScore;
		let numberOfWords = Math.min(string1Words.length, string2Words.length, 15); // Consider up to 15 words
		let lastIntersectingWord;
		let indexOfLastIntersectingWordInString2;
		let indexOfLastIntersectingWordInString2InFullText;

		// Try matching from 15 words down to 1 word
		for (; numberOfWords > 2; numberOfWords--) {
			let slice1Words = string1Words.slice(-numberOfWords);
			let slice2Words = string2Words.slice(0, numberOfWords);

			console.log(`Last ${numberOfWords} words of String 1:`, slice1Words);
			console.log(`First ${numberOfWords} words of String 2:`, slice2Words);

			intersection = slice1Words.filter(word => slice2Words.includes(word));
			let union = [...new Set([...slice1Words, ...slice2Words])];

			similarityScore = intersection.length / union.length;

			console.log("Intersection:", intersection);
			console.log("Similarity Score:", similarityScore);

			if (similarityScore > overlapThreshold) {
				lastIntersectingWord = intersection[intersection.length - 1];
				indexOfLastIntersectingWordInString2 = slice2Words.lastIndexOf(lastIntersectingWord);
				indexOfLastIntersectingWordInString2InFullText = string2.indexOf(string2.split(' ').slice(0, indexOfLastIntersectingWordInString2 + 1).join(' '));

				console.log("Last Intersecting Word:", lastIntersectingWord);
				console.log("Index of Last Intersecting Word in Full Text:", indexOfLastIntersectingWordInString2InFullText);

				break; // Exit loop if a good enough match was found
			}
		}

		if (similarityScore > overlapThreshold) {
			return string1 + string2.slice(indexOfLastIntersectingWordInString2InFullText + lastIntersectingWord.length);
		} else {
			return string1 + ' ' + string2;
		}
	}

	async function startRecording() {
		if (!isRecording) {
			console.log("Start recording...");
			const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
			startNewChunk(stream);

			// Start the sending interval
			sendIntervalId = setInterval(async () => {
				let oldRecorder = mediaRecorders[currentAudioQueueId];
				mergedTranscript = jaccardSimilarityAndMerge(mergedTranscript, transcript);
				transcriptHistory.push(transcript);
				transcriptHistory = transcriptHistory;
				console.log('Transcript history:', transcriptHistory);
				oldRecorder.stop();
				console.log(`Stopping old ${currentAudioQueueId} recorder...`);
				currentAudioQueueId++;

			}, CHUNK_DURATION);

			// Start the submitting interval
			submitIntervalId = setInterval(async () => {

				if(requestInProgress ) {
					console.log('Request in progress, skipping submit...');
					return;
				}

				// over 2 seconds
				if ((audioQueue[currentAudioQueueId][audioQueue[currentAudioQueueId].length - 1].timestamp - audioQueue[currentAudioQueueId][0].timestamp) >= 2000) {

					console.log('Submitting chunk...');
					let audioBlob = new Blob(audioQueue[currentAudioQueueId].map(chunk => chunk.data));
					// Log the size of the blob
					console.log(`Sending Blob with size: ${audioBlob.size} bytes`);

					// Construct form data
					let formData = new FormData();
					formData.append("audio_file", audioBlob);

					// Make the HTTP POST request to the ASR endpoint
					try {
						requestInProgress = true;
						const response = await fetch('http://localhost:9000/asr', {
							method: 'POST',
							body: formData,
						});
						requestInProgress = false;
							if (response.ok) {
								const data = await response.text();
								console.log("Received response from server:", data);
								transcript = data; // replace the old transcript with the new one
							} else {
								console.error('Error:', response.status, response.statusText);
							}
					} catch(e) {
						requestInProgress = false;
						console.log('Error:', e);
					}



				}
			}, SERVER_SEND_RATE);

			overlapIntervalId = setInterval(async () => {
				console.log('Starting overlap...');
				startNewChunk(stream, (currentAudioQueueId+1));
			}, CHUNK_DURATION - OVERLAP_DURATION);

			isRecording = true;
		}
	}

	function stopRecording() {
		if (isRecording) {
			console.log("Stop recording...");
			// Stop the mediaRecorders and the sending interval
			mediaRecorders.forEach(recorder => recorder.stop());
			mediaRecorders = [];
			clearInterval(sendIntervalId);
			clearInterval(overlapIntervalId);
			clearInterval(submitIntervalId);
			currentAudioQueueId = 0;
			sendIntervalId = null;
			overlapIntervalId = null;
			isRecording = false;
		}
	}

	function startNewChunk(stream, queueId=0) {
		console.log('Starting new chunk...');
		let newRecorder = new MediaRecorder(stream);
		mediaRecorders.push(newRecorder);

		newRecorder.start(1000);

		audioQueue.push([])
		newRecorder.addEventListener("dataavailable", event => {
			console.log('Recorder data available...');
			audioQueue[queueId].push({data: event.data, timestamp: Date.now()});
		})

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
		<ul>
		{#each transcriptHistory as script, index (index)}
			<li>{index + 1}: {script}</li>
		{/each}
		</ul>
		<p>T: {transcript}</p>
		<p>Merged: {mergedTranscript}</p>
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
