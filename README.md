# Voice-to-Text

This project uses svelte and the web speech api to convert voice to text. It is a simple project that I made to learn svelte.
It also uses the whisper api to convert the text to speech. The api is ran locally in a docker container

Speech to Text: [Whispser ASR](https://github.com/ahmetoner/whisper-asr-webservice/)

## Developing
Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```bash
npm run dev
# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building
To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://kit.svelte.dev/docs/adapters) for your target environment.
