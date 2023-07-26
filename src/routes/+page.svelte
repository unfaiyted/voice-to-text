<script>
	import Counter from './Counter.svelte';

		let transcript = '';
		let mediaRecorder = null;
		let audioChunks = [];

		const startRecording = async () => {
		const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
		mediaRecorder = new MediaRecorder(stream);
		mediaRecorder.start();

		mediaRecorder.addEventListener("dataavailable", event => {
			audioChunks.push(event.data);
		});

		mediaRecorder.addEventListener("stop", async () => {
			const audioBlob = new Blob(audioChunks);
			audioChunks = []; // clear previous audio data

			// Construct form data
			let formData = new FormData();
			formData.append("audio_file", audioBlob);

			// Make the HTTP POST request to the ASR endpoint
			const response = await fetch('http://192.168.0.120:9000/asr', {
			method: 'POST',
			body: formData,
	});

		if (response.ok) {
			console.log('Success!', response)
		const data = await response.text();
		transcript = data; // this depends on the structure of the API response
	} else {
		console.error('Error:', response.status, response.statusText);
	}
	});
	};

		const stopRecording = () => {
		mediaRecorder.stop();
	};

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


	<Counter />
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
