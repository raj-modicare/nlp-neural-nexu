<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { 
  Brain, 
  MessageSquare, 
  FileText, 
  Activity, 
  Key, 
  Send, 
  Sparkles, 
  Loader2,
  Languages
} from 'lucide-vue-next';
import OpenAI from 'openai';

// Types
type Mode = 'chat' | 'summary' | 'sentiment' | 'translate';
type Message = { role: 'user' | 'assistant'; content: string };

// State
const apiKey = ref('');
const showKeyInput = ref(false);
const mode = ref<Mode>('chat');
const isDemoMode = ref(true); // Default to Demo Mode so the user can see it working immediately

// Chat State
const chatHistory = ref<Message[]>([]);
const input = ref('');
const isLoading = ref(false);

// Other Modes State
const textInput = ref('');
const result = ref('');

// Load API Key
onMounted(() => {
  const storedKey = localStorage.getItem('openai_api_key');
  if (storedKey) {
    apiKey.value = storedKey;
    isDemoMode.value = false; // If key exists, try Live mode
  } else {
    showKeyInput.value = true;
  }
});

const saveApiKey = () => {
  localStorage.setItem('openai_api_key', apiKey.value);
  showKeyInput.value = false;
  isDemoMode.value = false;
};

const clearApiKey = () => {
  apiKey.value = '';
  localStorage.removeItem('openai_api_key');
  showKeyInput.value = true;
  isDemoMode.value = true;
};

const getOpenAIClient = () => {
  if (!apiKey.value) throw new Error("API Key missing");
  // Switching to Groq: It uses the same OpenAI library but a different URL
  return new OpenAI({ 
    apiKey: apiKey.value, 
    baseURL: "https://api.groq.com/openai/v1", 
    dangerouslyAllowBrowser: true 
  });
};

const handleDemoSubmit = async () => {
  await new Promise(resolve => setTimeout(resolve, 1000)); // Simulate lag
  
  if (mode.value === 'chat') {
    chatHistory.value.push({ role: 'user', content: input.value });
    input.value = '';
    const responses = [
      "That's a fascinating point! Tell me more.",
      "In Demo Mode, I'm just simulating a response, but your UI looks great!",
      "I agree completely. NLP is a powerful field of study.",
      "If this were a live API call, I'd give you a detailed breakdown."
    ];
    chatHistory.value.push({ role: 'assistant', content: responses[Math.floor(Math.random() * responses.length)] });
  } else if (mode.value === 'summary') {
    result.value = "[DEMO SUMMARY]\n" + textInput.value.split('.').slice(0, 2).join('.') + "...\n\nKey Concepts identified: NLP, Neural Networks, Intelligence.";
  } else if (mode.value === 'sentiment') {
    const text = textInput.value.toLowerCase();
    const positive = ['good', 'great', 'happy', 'love', 'amazing'].some(w => text.includes(w));
    const negative = ['bad', 'error', 'hate', 'sad', 'poor'].some(w => text.includes(w));
    
    result.value = positive ? "Positive (Simulated): The text contains optimistic language." : 
                   negative ? "Negative (Simulated): Critical or pessimistic tone detected." : 
                   "Neutral (Simulated): The tone appears balanced.";
  } else if (mode.value === 'translate') {
    result.value = "[DEMO TRANSLATION]\nThis is a simulated translation of your text. In Live Mode, this would be converted perfectly to English or Hindi!";
  }
};

const handleSubmit = async () => {
  if (!input.value && !textInput.value) return;
  isLoading.value = true;
  result.value = '';

  if (isDemoMode.value) {
    await handleDemoSubmit();
    isLoading.value = false;
    return;
  }

  try {
    const client = getOpenAIClient();
    // ... rest of the live logic ...
    
    if (mode.value === 'chat') {
      const userMsg: Message = { role: 'user', content: input.value };
      chatHistory.value.push(userMsg);
      input.value = '';

      const completion = await client.chat.completions.create({
        messages: [{ role: 'system', content: "You are a helpful, witty, and intelligent AI assistant." }, ...chatHistory.value],
        model: 'llama-3.3-70b-versatile',
      });

      const reply = completion.choices[0].message.content || "No response";
      chatHistory.value.push({ role: 'assistant', content: reply });
    } else if (mode.value === 'summary') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: "Summarize the following text concisely." },
          { role: 'user', content: textInput.value }
        ],
        model: 'llama-3.3-70b-versatile',
      });
      result.value = completion.choices[0].message.content || "Could not summarize.";
    } else if (mode.value === 'sentiment') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: "Analyze the sentiment of the following text. Respond with: 'Positive', 'Negative', or 'Neutral', followed by a brief explanation." },
          { role: 'user', content: textInput.value }
        ],
        model: 'llama-3.3-70b-versatile',
      });
      result.value = completion.choices[0].message.content || "Could not analyze.";
    } else if (mode.value === 'translate') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: "Translate the following text into English. If it is already in English, translate it into Hindi." },
          { role: 'user', content: textInput.value }
        ],
        model: 'llama-3.3-70b-versatile',
      });
      result.value = completion.choices[0].message.content || "Could not translate.";
    }
  } catch (error: any) {
    alert("Error: " + error.message);
    result.value = "Error occurred. Check your API Key.";
  } finally {
    isLoading.value = false;
  }
};
</script>

<template>
  <div class="app-container">
    <!-- Header -->
    <header class="header animate-fade-in">
      <div class="brand">
        <div class="logo-box">
          <Brain class="icon white" />
        </div>
        <h1 class="title">Neural Nexus</h1>
      </div>
      
      <div class="header-actions">
        <button 
          @click="isDemoMode = !isDemoMode"
          :class="['demo-toggle', { active: isDemoMode }]"
        >
          {{ isDemoMode ? 'Demo Mode: ON' : 'Live API: ON' }}
        </button>
        
        <button 
          @click="showKeyInput = !showKeyInput"
          class="key-btn glass-panel"
        >
          <Key class="icon-sm" />
          {{ apiKey ? 'API Key Configured' : 'Set API Key' }}
        </button>
      </div>
    </header>

    <!-- API Key Input -->
    <div v-if="showKeyInput" class="key-input-area glass-panel animate-fade-in">
      <Key class="icon-indigo" />
      <input 
        type="password" 
        placeholder="sk-..." 
        v-model="apiKey"
        class="input-field"
      />
      <div class="key-actions">
        <button @click="saveApiKey" class="btn-primary">Save Key</button>
        <button v-if="apiKey" @click="clearApiKey" class="btn-secondary">Reset</button>
      </div>
    </div>

    <!-- Navigation Cards -->
    <div class="nav-grid animate-fade-in-delayed-1">
      <button 
        @click="mode = 'chat'"
        :class="['nav-card', { active: mode === 'chat' }]"
      >
        <div class="nav-icon-box">
          <MessageSquare :size="24" />
        </div>
        <h3>AI Chat</h3>
        <p>Conversational Intelligence</p>
      </button>

      <button 
        @click="mode = 'summary'"
        :class="['nav-card', { active: mode === 'summary' }]"
      >
        <div class="nav-icon-box">
          <FileText :size="24" />
        </div>
        <h3>Summarizer</h3>
        <p>Condense long documents</p>
      </button>

      <button 
        @click="mode = 'sentiment'"
        :class="['nav-card', { active: mode === 'sentiment' }]"
      >
        <div class="nav-icon-box">
          <Activity :size="24" />
        </div>
        <h3>Sentiment</h3>
        <p>Analyze emotional tone</p>
      </button>

      <button 
        @click="mode = 'translate'"
        :class="['nav-card', { active: mode === 'translate' }]"
      >
        <div class="nav-icon-box">
          <Languages :size="24" />
        </div>
        <h3>Translator</h3>
        <p>English ↔ Hindi</p>
      </button>
    </div>

    <!-- Main Workspace -->
    <div class="workspace glass-panel animate-fade-in-delayed-2">
      
      <!-- Chat Mode -->
      <div v-if="mode === 'chat'" class="chat-container">
        <div class="chat-history">
          <div v-if="chatHistory.length === 0" class="empty-state">
            <Sparkles class="icon-large opacity-20" />
            <p>Start a conversation with the AI...</p>
          </div>
          
          <div 
            v-for="(msg, i) in chatHistory" 
            :key="i" 
            :class="['message-row', msg.role === 'user' ? 'user-row' : 'bot-row']"
          >
            <div :class="['message-bubble', msg.role === 'user' ? 'user-bubble' : 'bot-bubble']">
              {{ msg.content }}
            </div>
          </div>

          <div v-if="isLoading" class="loading-row">
            <div class="loading-bubble">
              <Loader2 class="icon-spin icon-indigo" /> Thinking...
            </div>
          </div>
        </div>
        
        <div class="chat-input-area">
          <input 
            v-model="input"
            @keydown.enter="handleSubmit"
            placeholder="Type your message..."
            class="input-field flex-1"
          />
          <button @click="handleSubmit" :disabled="isLoading" class="btn-primary">
            <Send class="icon-sm" /> Send
          </button>
        </div>
      </div>

      <!-- Text Modes -->
      <div v-else class="text-mode-container">
        <div class="input-section">
          <label>Input Text</label>
          <textarea 
            v-model="textInput"
            :placeholder="mode === 'summary' ? 'Paste text to summarize...' : 'Paste text to analyze...'"
            class="textarea-field"
          ></textarea>
        </div>

        <div class="action-bar">
           <button @click="handleSubmit" :disabled="isLoading" class="btn-primary">
             <component :is="isLoading ? Loader2 : Sparkles" :class="{'icon-spin': isLoading, 'icon-sm': true}" />
             {{ mode === 'summary' ? 'Generate Summary' : 'Analyze Sentiment' }}
           </button>
        </div>

        <div v-if="result" class="result-box animate-fade-in">
          <h3>Result</h3>
          <p>{{ result }}</p>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
/* Layout Containers */
.app-container {
  min-height: 100vh;
  text-align: left;
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

/* Header */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 2rem;
  padding: 1rem 0;
}

.brand {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.logo-box {
  padding: 0.75rem;
  border-radius: 0.75rem;
  background: linear-gradient(135deg, #6366f1, #9333ea);
  box-shadow: 0 4px 10px rgba(99, 102, 241, 0.3);
  display: flex;
  place-items: center;
}

.title {
  font-size: 1.875rem;
  font-weight: 700;
  background: linear-gradient(to right, #ffffff, #9ca3af);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  margin: 0;
}

.header-actions {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.demo-toggle {
  padding: 0.5rem 1rem;
  border-radius: 9999px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: rgba(0, 0, 0, 0.3);
  color: #9ca3af;
  font-size: 0.75rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.demo-toggle.active {
  background: rgba(245, 158, 11, 0.1);
  border-color: #f59e0b;
  color: #f59e0b;
  box-shadow: 0 0 10px rgba(245, 158, 11, 0.2);
}

.key-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  color: #e5e7eb;
  background: rgba(17, 24, 39, 0.6);
  cursor: pointer;
  font-size: 0.875rem;
}

.key-input-area {
  display: flex;
  gap: 1rem;
  align-items: center;
  margin-bottom: 2rem;
  padding: 1.5rem;
}

/* Nav Grid */
.nav-grid {
  display: grid;
  grid-template-columns: repeat(1, 1fr);
  gap: 1.5rem;
  margin-bottom: 2rem;
}

@media (min-width: 768px) {
  .nav-grid {
    grid-template-columns: repeat(4, 1fr);
  }
}

.nav-card {
  padding: 1.5rem;
  text-align: left;
  border-radius: 0.75rem;
  border: 1px solid transparent;
  background: rgba(17, 24, 39, 0.4);
  color: #d1d5db;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.nav-card:hover {
  background: rgba(31, 41, 55, 0.6);
}

.nav-card.active {
  background: rgba(79, 70, 229, 0.1);
  border-color: #6366f1;
  box-shadow: 0 0 20px rgba(99, 102, 241, 0.15);
}

.nav-card.active h3 {
  color: white;
}

.nav-card.active .nav-icon-box {
  background: #6366f1;
  color: white;
}

.nav-icon-box {
  padding: 0.75rem;
  border-radius: 0.5rem;
  background: #1f2937;
  color: #9ca3af;
  display: inline-block;
  width: fit-content;
  transition: all 0.3s;
}

.nav-card h3 {
  font-size: 1.125rem;
  font-weight: 700;
  margin: 0.5rem 0 0 0;
}

.nav-card p {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0;
}

/* Workspace */
.workspace {
  min-height: 500px;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
}

/* Chat Styles */
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  gap: 1rem;
  flex: 1;
}

.chat-history {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-height: 500px;
  padding-right: 0.5rem;
}

.empty-state {
  margin-top: 5rem;
  text-align: center;
  color: #6b7280;
}

.message-row {
  display: flex;
}

.user-row { justify-content: flex-end; }
.bot-row { justify-content: flex-start; }

.message-bubble {
  max-width: 80%;
  padding: 1rem;
  border-radius: 1rem;
  line-height: 1.5;
}

.user-bubble {
  background: #4f46e5;
  color: white;
  border-bottom-right-radius: 0;
}

.bot-bubble {
  background: #1f2937;
  color: #e5e7eb;
  border-bottom-left-radius: 0;
}

.loading-row {
  display: flex;
  justify-content: flex-start;
}

.loading-bubble {
  background: #1f2937;
  padding: 1rem;
  border-radius: 1rem;
  border-bottom-left-radius: 0;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #e5e7eb;
}

.chat-input-area {
  margin-top: auto;
  padding-top: 1rem;
  display: flex;
  gap: 1rem;
}

/* Text Mode Styles */
.text-mode-container {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  height: 100%;
  flex: 1;
}

.input-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  flex: 1;
}

.input-section label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.textarea-field {
  flex: 1;
  min-height: 200px;
  resize: none;
  font-family: inherit;
  background: rgba(0, 0, 0, 0.2);
  border: 1px solid var(--card-border);
  border-radius: 0.5rem;
  padding: 1rem;
  color: #f9fafb;
}

.action-bar {
  display: flex;
  justify-content: flex-end;
}

.result-box {
  background: rgba(0, 0, 0, 0.3);
  padding: 1.5rem;
  border-radius: 0.75rem;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.result-box h3 {
  color: #818cf8;
  font-size: 0.875rem;
  font-weight: 700;
  margin: 0 0 0.5rem 0;
  text-transform: uppercase;
}

.result-box p {
  white-space: pre-wrap;
  color: #e5e7eb;
  margin: 0;
}

/* Utilities */
.flex-1 { flex: 1; }
.icon { width: 32px; height: 32px; }
.icon-white { color: white; }
.icon-sm { width: 16px; height: 16px; }
.icon-large { width: 48px; height: 48px; }
.icon-indigo { color: #818cf8; }
.icon-spin { animation: spin 1s linear infinite; }

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.key-actions {
  display: flex;
  gap: 0.5rem;
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 0.6em 1.2em;
  font-size: 0.875rem;
  font-weight: 500;
  color: #9ca3af;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-secondary:hover {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border-color: rgba(255, 255, 255, 0.2);
}

.animate-fade-in-delayed-1 {
  animation: fadeIn 0.6s ease-out forwards;
  animation-delay: 0.1s;
  opacity: 0;
}

.animate-fade-in-delayed-2 {
  animation: fadeIn 0.6s ease-out forwards;
  animation-delay: 0.2s;
  opacity: 0;
}
</style>
