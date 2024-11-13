<script>
  import { onMount } from 'svelte';
  
  // State variables
  let messages = [];
  let inputText = '';
  let isListening = false;
  let openAIKey = '';
  let silenceThreshold = 0.5;
  let selectedVoice = null;
  let voices = [];
  let pitch = 1;
  let rate = 1;
  let volume = 1;
  
  // Speech recognition setup
  let recognition;
  let synthesis;
  
  // Add theme state
  let isDarkTheme = false;
  
  function toggleTheme() {
    isDarkTheme = !isDarkTheme;
    document.body.classList.toggle('dark-theme');
  }
  
  onMount(() => {
    // Initialize Web Speech API
    recognition = new (window.SpeechRecognition || window.webkitSpeechRecognition)();
    recognition.continuous = true;
    recognition.interimResults = true;
    
    // Get available voices
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
        handleUserInput(final_transcript);
      }
    };
    
    let silenceTimer;
    recognition.onaudioend = () => {
      silenceTimer = setTimeout(() => {
        if (isListening) {
          stopListening();
          processInput();
        }
      }, silenceThreshold * 1000);
    };
    
    // Initial scroll to bottom
    setTimeout(() => {
      const chatContainer = document.querySelector('.chat-container');
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }, 0);
  });
  
  // Handle user input (both voice and text)
  async function handleUserInput(input) {
    if (!input.trim()) return;
    
    messages = [...messages, { role: 'user', content: input }];
    inputText = '';
    
    try {
      const response = await fetch('YOUR_EC2_ENDPOINT', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${openAIKey}`
        },
        body: JSON.stringify({ message: input })
      });
      
      const data = await response.json();
      messages = [...messages, { role: 'assistant', content: data.response }];
      
      // Scroll to bottom after new message
      setTimeout(() => {
        const chatContainer = document.querySelector('.chat-container');
        chatContainer.scrollTop = chatContainer.scrollHeight;
      }, 0);
      
      speak(data.response);
    } catch (error) {
      console.error('Error:', error);
      messages = [...messages, { 
        role: 'assistant', 
        content: 'Sorry, there was an error processing your request.' 
      }];
      
      // Scroll to bottom after error message
      setTimeout(() => {
        const chatContainer = document.querySelector('.chat-container');
        chatContainer.scrollTop = chatContainer.scrollHeight;
      }, 0);
    }
  }
  
  // Speech synthesis
  function speak(text) {
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.voice = selectedVoice;
    utterance.pitch = pitch;
    utterance.rate = rate;
    utterance.volume = volume;
    
    utterance.onend = () => {
      // Resume listening after speaking
      startListening();
    };
    
    synthesis.speak(utterance);
  }
  
  // Voice control functions
  function startListening() {
    isListening = true;
    recognition.start();
  }
  
  function stopListening() {
    isListening = false;
    recognition.stop();
  }
  
  // File upload handler
  async function handleFileUpload(event) {
    const file = event.target.files[0];
    const formData = new FormData();
    formData.append('sop', file);
    
    try {
      await fetch('YOUR_EC2_ENDPOINT/upload', {
        method: 'POST',
        body: formData
      });
    } catch (error) {
      console.error('Error uploading file:', error);
    }
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
  </div>
  
  <!-- Settings panel -->
  <div class="settings-panel">
    <h3>Settings</h3>
    
    <!-- Silence threshold -->
    <div class="setting-item">
      <label>Silence Threshold (seconds)</label>
      <input 
        type="range" 
        min="0.2" 
        max="2" 
        step="0.1" 
        bind:value={silenceThreshold}
      />
      <span>{silenceThreshold}s</span>
    </div>
    
    <!-- OpenAI Key -->
    <div class="setting-item">
      <label>OpenAI API Key</label>
      <input 
        type="password" 
        bind:value={openAIKey} 
        placeholder="Enter your OpenAI API key"
      />
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
</style>