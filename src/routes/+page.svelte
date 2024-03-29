<script lang="ts">
  import { onMount } from 'svelte';
  import ChatMessage from '$lib/components/ChatMessage.svelte'
  import type { ChatCompletionRequestMessage } from 'openai'
  import { SSE } from 'sse.js'
  import { speakText } from '$lib/text-to-speech'

  function saveChatMessages() {
    if (typeof window !== 'undefined') {
      localStorage.setItem('chatMessages', JSON.stringify(chatMessages));
    }
  }
  
  function loadChatMessages() {
    if (typeof window !== 'undefined') {
      const loadedMessages = localStorage.getItem('chatMessages');
      return loadedMessages ? JSON.parse(loadedMessages) : [];
    } else {
      return [];
    }
  }
  
  function clearChat() {
    chatMessages = [];
    saveChatMessages();
  }

  let query: string = '';
  let answer: string = '';
  let loading: boolean = false;
  export let chatMessages: ChatCompletionRequestMessage[] = [];

  let scrollToDiv: HTMLDivElement;

  function scrollToBottom() {
    setTimeout(function () {
      scrollToDiv.scrollIntoView({ behavior: 'smooth', block: 'end', inline: 'nearest' });
    }, 100);
  }

  const readOutLoud = async () => {
    const lastAssistantMessage = chatMessages
      .filter(message => message.role === 'assistant')
      .map(message => message.content)
      .pop(); // Get the last assistant message

    if (lastAssistantMessage) {
      await speakText(lastAssistantMessage);
    }
  };

  const shareChat = () => {
    // Add your logic for sharing the chat
    console.log('Sharing chat...');
  };

  const handleSubmit = async () => {
    loading = true;
    chatMessages = [...chatMessages, { role: 'user', content: query }];
    saveChatMessages();

    const eventSource = new SSE('/api/chat', {
      headers: {
        'Content-Type': 'application/json'
      },
      payload: JSON.stringify({ messages: chatMessages })
    });

    query = '';

    eventSource.addEventListener('error', handleError);

    eventSource.addEventListener('message', (e) => {
      scrollToBottom();
      try {
        loading = false;
        if (e.data === '[DONE]') {
          chatMessages = [...chatMessages, { role: 'assistant', content: answer }];
          answer = '';
          saveChatMessages(); // Save messages after the assistant's response
          return;
        }

        const completionResponse = JSON.parse(e.data);
        const [{ delta }] = completionResponse.choices;

        if (delta.content) {
          answer = (answer ?? '') + delta.content;
        }
      } catch (err) {
        handleError(err);
      }
    });
    eventSource.stream();
    scrollToBottom();
  };

  function handleError<T>(err: T) {
    loading = false;
    query = '';
    answer = '';
    console.error(err);
  }

  onMount(() => {
    chatMessages = loadChatMessages(); // load chat messages on mount
  });
</script>

<style>
  /* Global styles */
  html, body {
    @apply h-full;
    background-color: #f8f8f8; /* Set the background color */
  }

  body {
    @apply text-gray-800; /* Set the default text color to a dark gray */
  }

  /* Chat bubble styles */
  .chat-bubble {
    @apply p-4 rounded-lg shadow-md;
    max-width: 70%;
    margin-bottom: 10px;
    word-wrap: break-word;
  }

  .chat-bubble-primary {
    @apply bg-blue-500 text-white self-end;
  }

  .chat-bubble-secondary {
    @apply bg-red-500 text-white self-start;
  }

  .chat-bubble-system {
    @apply bg-green-500 text-white text-center italic;
    font-size: 0.9em;
  }

  /* User input area styles */
  .input-area {
    @apply bg-white p-4 rounded-md; /* Change the background color to white */
  }

  .input {
    @apply w-full p-2 bg-white border border-gray-300 rounded-md text-gray-800; /* Set the text color to a dark gray */
  }

  .btn-accent {
    @apply p-2 bg-blue-500 text-white rounded-md cursor-pointer;
  }

  /* Component-specific styles */
  .parent-container {
    height: 100vh;
    background-color: transparent !important;
  }

  .chat-container {
    height: calc(99vh - 3.5rem); /* Subtract the height of the input area */
    padding-bottom: env(safe-area-inset-bottom); /* Add padding for the virtual keyboard on mobile devices */
  }

  .btn {
    padding: 5px;
  }

  .btn img {
    width: 24px;
    height: 24px;
  }

  /* New styles for the static header */
  .header {
    @apply bg-white p-4 rounded-md shadow-md;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 1000;
    display: flex;
    justify-content: space-between;
  }

  .header-buttons {
    display: flex;
    gap: 8px;
  }

  .header-title {
    @apply font-semibold text-lg;
  }

  /* Adjust margin for the main chat area */
  .main-chat-area {
    @apply mt-16; /* Adjust the margin to make space for the header */
  }
</style>

<div class="flex flex-col w-full px-0 items-center h-full parent-container">
  <!-- Static header with clear, read out loud, and share buttons -->
  <div class="header">
    <div class="header-buttons">
      <button type="button" class="btn btn-accent" on:click={clearChat}>
        <img src="./clear.png" alt="Clear Chat" />
      </button>
      <div class="header-title">The Great George Carlin</div>
      <button type="button" class="btn btn-accent" on:click={readOutLoud}>
        <img src="./audio.png" alt="Read Out Loud" />
      </button>
      <button type="button" class="btn btn-accent" on:click={shareChat}>
        <img src="./share.png" alt="Share" />
      </button>
    </div>
  </div>

  <!-- Main chat area -->
  <div class="main-chat-area">
    <div class="chat-container w-full transparent rounded-md p-4 overflow-y-auto flex flex-col gap-4">
      <div class="flex flex-col gap-2">
        <ChatMessage type="assistant" message="What do you want!" />
        {#each chatMessages as message}
          <ChatMessage type={message.role} message={message.content} />
        {/each}
        {#if answer}
          <ChatMessage type="assistant" message={answer} />
        {/if}
        {#if loading}
          <ChatMessage type="assistant" message="Loading.." />
        {/if}
      </div>
      <div class="" bind:this={scrollToDiv} />
    </div>

    <!-- Input area -->
    <form class="input-area flex w-full rounded-md gap-4 bg-white p-4" on:submit|preventDefault={() => handleSubmit()}>
      <div class="relative flex-grow">
        <input type="text" class="input input-bordered w-full pr-10" bind:value={query} />
        <button type="submit" class="btn btn-accent absolute right-1 top-1/2 transform -translate-y-1/2">
          <img src="./send.png" alt="Send" />
        </button>
      </div>
    </form>
  </div>
</div>
