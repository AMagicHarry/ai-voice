# AI-Voice

## Description
Inspired by Vercel's Language Model Specification, this is a proposal for introducing a Speech Model Specification to streamline the integration of various speech providers into our platform. This specification aims to provide a standardized interface for interacting with different speech models, eliminating the complexity of dealing with unique APIs and reducing the risk of vendor lock-in.

## Problem Statement
Currently, there are numerous speech providers available, each with its own distinct method for interfacing with their models. This lack of standardization complicates the process of switching providers and increases the likelihood of vendor lock-in. Developers are required to learn and implement different APIs for each provider, leading to increased development time and maintenance overhead.

Integration for following voice providers are available:

- OpenAI Provider (@bishwenduk029-ai-voice/openai)

- Elevenlabs Provider (@bishwenduk029-ai-voice/elevenlabs)

- Deepgram Provider (@bishwenduk029-ai-voice/deepgram)

- PlayHt Provider (@bishwenduk029-ai-voice/playht)

## Features

1. React Hook `useVoiceChat` for easy transcription and streaming voice chat in client side.
2. Consistent API for streaming speech response from various AI speech providers.
3. Works well with `streamText` from [Vercel AI SDK](https://sdk.vercel.ai/docs/ai-sdk-core/providers-and-models).

## Installation

```sh
pnpm install @bishwenduk029/ai-voice
```
![alt text](spec_v1.png)

## Usage

### Client Side React Hook
🎙️ Real-Time Speech Transcription React Hook
Seamlessly integrate real-time speech transcription into your React applications with this powerful and efficient hook! 🚀

✨ Hook Capabilities

- 🎤 Detect human speech end or silence using the robust @ricky0123/vad-react library
- ⏱️ Intelligently debounce speech input, ensuring continuous recording and transcription as long as the user speaks within a configurable time frame (e.g., 500ms)
- 🗣️ Gracefully handle speech interruptions, allowing users to pause and resume speaking naturally
- 🌐 Efficiently trigger REST calls for transcription, optimizing performance by waiting for the user to pause before sending requests
- 🔌 Easy to integrate into your existing React projects, with a simple and intuitive API
```ts
...
import { useVoiceChat } from '@bishwenduk029/ai-voice/ui'
...

export default function Chat({ id, initialMessages, className }: ChatProps) {
  const { speaking, listening, thinking, initialized, messages, setMessages } =
    useVoiceChat({
      api: '/api/chat/voice',
      initialMessages,
      transcribeAPI: '/api/transcribe',
      body: {
        id
      },
      speakerPause: 500,
      onSpeechCompletion: async () => {
        if (id) {
          const chat = await getChat(id)
          setMessages(chat?.messages || [])
        }
      }
    })

  return (
    <>
      <div className={cn('pb-[200px] pt-4 md:pt-10', className)}>
        {messages.length ? (
          <>
            <ChatList messages={messages} />
            <ChatScrollAnchor trackVisibility={initialized} />
          </>
        ) : (
          <EmptyScreen />
        )}
      </div>
      <ChatPanel
        id={id}
        initialized={initialized}
        speaking={speaking}
        listening={listening}
        thinking={thinking}
        messages={messages}
      />
    </>
  )
}
```

### Hook up Server APIs
`/api/chat/voice`
```ts
import 'server-only'
...
import { streamText } from 'ai'
import { ollama, createOllama } from 'ollama-ai-provider'
import { openaiSpeech, playhtSpeech, streamSpeech } from '@bishwenduk029/ai-voice/server'
...

export const runtime = 'edge'

const model = ollama('llama3:latest')

export async function POST(req: Request) {
  const cookieStore = cookies()
  const supabase = createRouteHandlerClient<Database>({
    cookies: () => cookieStore
  })
  const json = await req.json()
  const { messages } = json
  // const userId = (await auth({ cookieStore }))?.user.id

  // if (!userId) {
  //   return new Response('Unauthorized', {
  //     status: 401
  //   })
  // }

  const systemPrompt = `
  Your role is to act as a friendly human assistant by the user preferred name. Your given name is Nova.
  `

  const result = await streamText({
    model,
    messages: [
      {
        role: 'system',
        content: `${systemPrompt}`
      },
      ...messages.map((message: { role: any; content: any }) => ({
        role: message.role,
        content: message.content
      }))
    ]
  })

  // OpenAI - env:OPENAI_API_KEY
  const speechModel = openaiSpeech(
    'tts-1',   //openai_speech_model
    'nova'     //openai_voice_id
  )

  // ElevenLabsIO - env:ELEVENLABS_API_KEY
  // const speechModel = elevenlabsSpeech(
  //   'eleven_turbo_v2',   //elevenlabs_speech_model
  //   'DIBkDE5u33APYlfhjihh' //elevenlabs_voice_id
  // )

    // PlayHt - env:PLAYHT_API_KEY
  // const speechModel = playhtSpeech(
  //   'PlayHT2.0-turbo', //playht_speech_model
  //   '<your-playht-user-id>',  
  //   "s3://voice-cloning-zero-shot/1afba232-fae0-4b69-9675-7f1aac69349f/delilahsaad/manifest.json"   //playht_voice_id
  // )

  // Deepgram - env:DEEPGRAM_API_KEY
  // const speechModel = deepgramSpeech("aura-asteria-en")

  try {
    const speech = await streamSpeech(speechModel)(result.textStream)
    return new Response(speech, {
      headers: { 'Content-Type': 'audio/mpeg' }
    })
  } catch (error) {
    console.log(error)
    return new Response(null, {
      status: 500
    })
  }
}
```

## Limitations
Currently the speech streaming only works for english text streams. Multi-lingual support in future.

## Roadmap
- [x] Speech Model Specification Done
- [ ] Improve the implementation of sentence-boundary detection algorithm for the text stream to sentence stream conversion
- [ ] Add more tests
- [ ] Enhance the specification to also take case of WebSocket-based Speech Providers
