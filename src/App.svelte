<script>
  import { onMount } from 'svelte';
  
  // State variables
  let messages = [];
  let inputText = '';
  let isListening = false;
  let isSpeaking = false;
  let selectedVoice = null;
  let voices = [];
  let isDarkTheme = false;
  let silenceThreshold = 0.5;
  let pitch = 1.0;
  let rate = 1.0;
  let volume = 1.0;
  
  // Speech recognition setup
  let recognition;
  let synthesis;
  
  onMount(() => {
    // Check if browser supports speech recognition
    if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
      console.error('Speech recognition not supported in this browser');
      return;
    }

    // Fix speech recognition initialization
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    recognition = new SpeechRecognition();
    
    // Configure recognition settings
    recognition.continuous = true;
    recognition.interimResults = true;
    recognition.lang = 'en-US'; // Add language setting

    // Error handling for recognition
    recognition.onerror = (event) => {
      console.error('Speech recognition error:', event.error);
      isListening = false;
    };

    synthesis = window.speechSynthesis;
    voices = synthesis.getVoices();
    selectedVoice = voices[0];
    
    synthesis.onvoiceschanged = () => {
      voices = synthesis.getVoices();
      selectedVoice = voices[0];
    };
    
    // Speech recognition event handlers
    recognition.onresult = (event) => {
      let final_transcript = '';
      for (let i = event.resultIndex; i < event.results.length; ++i) {
        if (event.results[i].isFinal) {
          final_transcript += event.results[i][0].transcript;
        }
      }
      if (final_transcript) {
        inputText = final_transcript;
        handleUserInput(final_transcript);
      }
    };

    recognition.onend = () => {
      if (isListening) {
        recognition.start();
      }
    };
  });

  async function handleUserInput(input) {
    if (!input.trim()) return;
    
    messages = [...messages, { role: 'user', content: input }];
    inputText = '';
    
    try {
      const endpoint = 'https://n101d0uod9.execute-api.us-east-1.amazonaws.com/dev/test';
      console.log('Attempting to fetch from:', endpoint);
      
      const requestBody = {
        message: input,
        history: messages.map(m => ({ role: m.role, content: m.content }))
      };
      
      console.log('Request body:', requestBody);

      const response = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'Origin': window.location.origin
        },
        body: JSON.stringify(requestBody)
      });
      
      if (!response.ok) {
        const errorText = await response.text();
        console.error('Response status:', response.status);
        console.error('Response text:', errorText);
        throw new Error(`HTTP error! status: ${response.status}, message: ${errorText}`);
      }
      
      const data = await response.json();
      console.log('API Response:', data);
      
      if (!data || !data.message) {
        throw new Error('Invalid response format from API');
      }
      
      const assistantMessage = data.message;
      messages = [...messages, { role: 'assistant', content: assistantMessage }];
      
      scrollToBottom();
      
      if (isListening) {
        stopListening();
      }
      speak(assistantMessage);
      
    } catch (error) {
      console.error('Detailed error:', error);
      messages = [...messages, { 
        role: 'assistant', 
        content: `Error: ${error.message}` 
      }];
      scrollToBottom();
    }
  }

  function speak(text) {
    isSpeaking = true;
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.voice = selectedVoice;
    utterance.pitch = pitch;
    utterance.rate = rate;
    utterance.volume = volume;
    
    utterance.onend = () => {
      isSpeaking = false;
      if (!isListening) {
        startListening();
      }
    };
    
    synthesis.speak(utterance);
  }

  function startListening() {
    try {
      isListening = true;
      recognition.start();
    } catch (error) {
      console.error('Error starting speech recognition:', error);
      isListening = false;
    }
  }

  function stopListening() {
    isListening = false;
    recognition.stop();
  }

  function stopSpeaking() {
    isSpeaking = false;
    synthesis.cancel();
  }

  function scrollToBottom() {
    setTimeout(() => {
      const chatContainer = document.querySelector('.chat-container');
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }, 0);
  }

  function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file) {
      // Handle file upload logic here
      console.log('File selected:', file.name);
    }
  }

  function toggleTheme() {
    isDarkTheme = !isDarkTheme;
    document.body.classList.toggle('dark-theme', isDarkTheme);
  }
</script>

<main class="container" class:dark-theme={isDarkTheme}>
  <!-- Theme Toggle -->
  <button class="theme-toggle" on:click={toggleTheme}>
    {#if isDarkTheme}
      <span>🌙</span>
    {:else}
      <span>☀️</span>
    {/if}
  </button>

  <!-- Chat messages -->
  <div class="chat-container">
    {#each messages as message}
      <div class="message {message.role}">
        {message.content}
      </div>
    {/each}
  </div>
  
  <!-- Input area -->
  <div class="input-container">
    <input 
      type="text" 
      bind:value={inputText} 
      placeholder="Type your message..."
      on:keydown={(e) => e.key === 'Enter' && handleUserInput(inputText)}
    />
    <button 
      class="voice-btn {isListening ? 'listening' : ''}" 
      on:click={() => isListening ? stopListening() : startListening()}
    >
      {isListening ? 'Stop' : 'Start'} Listening
    </button>
    {#if isSpeaking}
      <button 
        class="stop-btn" 
        on:click={stopSpeaking}
      >
        Stop Speaking
      </button>
    {/if}
  </div>
  
  <!-- Settings panel -->
  <div class="settings-panel">
    <h3>Settings</h3>
    
    <!-- Silence threshold -->
    <div class="setting-item">
      <label for="silenceThreshold">Silence Threshold (seconds)</label>
      <input 
        id="silenceThreshold"
        type="range" 
        min="0.2" 
        max="2" 
        step="0.1" 
        bind:value={silenceThreshold}
      />
      <span>{silenceThreshold}s</span>
    </div>
    
    <!-- SOP Upload -->
    <div class="setting-item">
      <label>Upload SOP</label>
      <input 
        type="file" 
        accept=".pdf,.doc,.docx" 
        on:change={handleFileUpload}
      />
    </div>
    
    <!-- Voice Settings -->
    <div class="setting-item">
      <label>Voice</label>
      <select bind:value={selectedVoice}>
        {#each voices as voice}
          <option value={voice}>{voice.name}</option>
        {/each}
      </select>
    </div>
    
    <div class="setting-item">
      <label>Pitch ({pitch})</label>
      <input type="range" min="0.5" max="2" step="0.1" bind:value={pitch} />
    </div>
    
    <div class="setting-item">
      <label>Rate ({rate})</label>
      <input type="range" min="0.5" max="2" step="0.1" bind:value={rate} />
    </div>
    
    <div class="setting-item">
      <label>Volume ({volume})</label>
      <input type="range" min="0" max="1" step="0.1" bind:value={volume} />
    </div>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    transition: background-color 0.3s ease;
  }

  :global(body.dark-theme) {
    background-color: #1a1a1a;
    color: #ffffff;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    display: grid;
    grid-template-columns: minmax(300px, 1fr) 300px;
    gap: 20px;
    position: relative;
    height: 100vh;
  }

  .theme-toggle {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 12px;
    border-radius: 50%;
    border: none;
    background: var(--bg-secondary);
    font-size: 1.5rem;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    z-index: 1000;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .theme-toggle:hover {
    transform: scale(1.1);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  }

  .chat-container {
    height: calc(100vh - 100px);
    overflow-y: auto;
    border-radius: 12px;
    padding: 16px;
    background: var(--chat-bg);
    display: flex;
    flex-direction: column;
    margin-bottom: 70px;
    scrollbar-width: thin;
    scrollbar-color: var(--border-color) transparent;
  }

  .chat-container::-webkit-scrollbar {
    width: 6px;
  }

  .chat-container::-webkit-scrollbar-thumb {
    background-color: var(--border-color);
    border-radius: 3px;
  }

  .message {
    margin: 4px 0;
    padding: 10px 14px;
    border-radius: 12px;
    max-width: 70%;
    word-wrap: break-word;
    clear: both;
  }

  .user {
    background: #2196f3;
    color: white;
    align-self: flex-end;
    margin-left: auto;
    border-bottom-right-radius: 4px;
  }

  .assistant {
    background: var(--message-bg);
    color: var(--text-primary);
    align-self: flex-start;
    margin-right: auto;
    border-bottom-left-radius: 4px;
  }

  .input-container {
    position: fixed;
    bottom: 20px;
    left: 20px;
    right: 340px;
    display: flex;
    gap: 10px;
    padding: 10px;
    background: var(--bg-primary);
    border-radius: 12px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    z-index: 100;
  }

  .input-container input {
    flex: 1;
    padding: 12px 16px;
    border-radius: 8px;
    border: 1px solid var(--border-color);
    background: var(--input-bg);
    color: var(--input-text);
    font-size: 0.95rem;
  }

  .voice-btn {
    padding: 12px 24px;
    border-radius: 8px;
    border: none;
    background: #2196f3;
    color: white;
    cursor: pointer;
    transition: background-color 0.2s ease;
    font-weight: 500;
    white-space: nowrap;
  }

  .voice-btn.listening {
    background: #f44336;
  }

  .settings-panel {
    position: fixed;
    right: 20px;
    top: 20px;
    bottom: 20px;
    width: 300px;
    padding: 20px;
    border-radius: 12px;
    background: var(--bg-secondary);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    border: 1px solid var(--border-color);
    overflow-y: auto;
  }

  .setting-item {
    margin: 16px 0;
  }

  .setting-item label {
    display: block;
    margin-bottom: 6px;
    font-size: 0.9rem;
    color: var(--text-primary);
  }

  /* Theme variables */
  :root {
    --bg-primary: #ffffff;
    --bg-secondary: #f8f9fa;
    --text-primary: #333333;
    --border-color: #e0e0e0;
    --message-bg: #f5f5f5;
    --input-bg: #ffffff;
    --input-text: #333333;
    --chat-bg: #ffffff;
  }

  :global(.dark-theme) {
    --bg-primary: #1a1a1a;
    --bg-secondary: #2d2d2d;
    --text-primary: #ffffff;
    --border-color: #404040;
    --message-bg: #363636;
    --input-bg: #2d2d2d;
    --input-text: #ffffff;
    --chat-bg: #1a1a1a;
  }

  /* Responsive design */
  @media (max-width: 768px) {
    .container {
      grid-template-columns: 1fr;
    }

    .settings-panel {
      position: fixed;
      right: -300px;
      transition: right 0.3s ease;
    }

    .settings-panel.open {
      right: 0;
    }

    .input-container {
      right: 20px;
    }
  }

  .stop-btn {
    padding: 12px 24px;
    border-radius: 8px;
    border: none;
    background: #f44336;
    color: white;
    cursor: pointer;
    transition: background-color 0.2s ease;
    font-weight: 500;
    white-space: nowrap;
  }

  .stop-btn:hover {
    background: #d32f2f;
  }
</style>