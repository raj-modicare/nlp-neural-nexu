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
  Languages,
  Mic,
  MicOff,
  Volume2,
  Paperclip,
  Image,
  Palette,
  Download
} from 'lucide-vue-next';
import OpenAI from 'openai';

// Access PDF.js from global scope (CDN)
const pdfjsLib = (window as any).pdfjsLib;
if (pdfjsLib) {
  pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
}

// Types
type Mode = 'chat' | 'summary' | 'sentiment' | 'translate' | 'image-gen';
type Message = { role: 'user' | 'assistant'; content: string };

// State
const apiKey = ref('');
const showKeyInput = ref(false);
const mode = ref<Mode>('chat');
const isDemoMode = ref(true);

// Models
const availableModels = [
  { id: 'llama-3.3-70b-versatile', name: 'Llama 3.3 (High Intelligence)', icon: '🧠' },
  { id: 'meta-llama/llama-4-scout-17b-16e-instruct', name: 'Llama 4 Scout (Vision)', icon: '👁️' },
  { id: 'llama-3.1-8b-instant', name: 'Llama 3.1 (Ultra Fast)', icon: '⚡' },
  { id: 'mixtral-8x7b-32768', name: 'Mixtral 8x7B (Large Context)', icon: '📚' },
  { id: 'gemma2-9b-it', name: 'Gemma 2 (Efficient)', icon: '💎' }
];
const selectedModel = ref('llama-3.3-70b-versatile');

// Language Translation State
const targetLanguage = ref('Hindi');
const indianLanguages = [
  'Hindi', 'Marathi', 'Bengali', 'Tamil', 'Telugu', 
  'Kannada', 'Gujarati', 'Punjabi', 'Malayalam', 'Odia', 
  'Assamese', 'Urdu', 'Sanskrit'
];

// Chat State
const chatHistory = ref<Message[]>([]);
const input = ref('');
const isLoading = ref(false);

// Image Generation State
const generatedImageUrl = ref('');
const isImageLoading = ref(false);

// Other Modes State
const textInput = ref('');
const result = ref('');

// Voice State
const isListening = ref(false);
const recognition = (window as any).webkitSpeechRecognition ? new ((window as any).webkitSpeechRecognition)() : null;

if (recognition) {
  recognition.continuous = false;
  recognition.interimResults = false;
  recognition.lang = 'en-US';
}

// Voice Functions
const startListening = (target: 'input' | 'textInput') => {
  if (!recognition) return alert("Speech recognition not supported in this browser.");
  
  isListening.value = true;
  recognition.start();

  recognition.onresult = (event: any) => {
    const transcript = event.results[0][0].transcript;
    if (target === 'input') input.value += transcript;
    else textInput.value += transcript;
  };

  recognition.onend = () => { isListening.value = false; };
  recognition.onerror = () => { isListening.value = false; };
};

const speak = (text: string) => {
  const utterance = new SpeechSynthesisUtterance(text);
  window.speechSynthesis.speak(utterance);
};

// PDF Processing
const fileInput = ref<HTMLInputElement | null>(null);
const isExtracting = ref(false);

const triggerFileUpload = () => {
  fileInput.value?.click();
};

const handlePdfUpload = async (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file || file.type !== 'application/pdf') return;

  const pdfLib = (window as any).pdfjsLib;
  if (!pdfLib) {
    alert("PDF library is still loading. Please try again in 2 seconds.");
    return;
  }

  isExtracting.value = true;
  const reader = new FileReader();
  
  reader.onload = async () => {
    try {
      const typedarray = new Uint8Array(reader.result as ArrayBuffer);
      const pdf = await pdfLib.getDocument(typedarray).promise;
      let fullText = '';

      for (let i = 1; i <= pdf.numPages; i++) {
        const page = await pdf.getPage(i);
        const textContent = await page.getTextContent();
        const pageText = textContent.items.map((item: any) => item.str).join(' ');
        fullText += pageText + '\n';
      }

      // Add as a special message or paste to input
      input.value = `[Document: ${file.name}]\n\nAttached Content:\n${fullText.slice(0, 2000)}${fullText.length > 2000 ? '...' : ''}\n\n(I have extracted the content from your PDF. How can I help you with this?)`;
      
      if (mode.value !== 'chat') mode.value = 'chat';
    } catch (error) {
      alert("Error reading PDF: " + error);
    } finally {
      isExtracting.value = false;
    }
  };
  
  reader.readAsArrayBuffer(file);
};

// Image Processing
const imageInput = ref<HTMLInputElement | null>(null);
const attachedImage = ref<string | null>(null);

const triggerImageUpload = () => {
  imageInput.value?.click();
};

const handleImageUpload = (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file || !file.type.startsWith('image/')) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new window.Image();
    img.onload = () => {
      // Create canvas for resizing
      const canvas = document.createElement('canvas');
      let width = img.width;
      let height = img.height;
      
      // Max dimensions for AI analysis (balanced for detail and speed)
      const MAX_SIZE = 800; 
      if (width > height) {
        if (width > MAX_SIZE) {
          height *= MAX_SIZE / width;
          width = MAX_SIZE;
        }
      } else {
        if (height > MAX_SIZE) {
          width *= MAX_SIZE / height;
          height = MAX_SIZE;
        }
      }
      
      canvas.width = width;
      canvas.height = height;
      const ctx = canvas.getContext('2d');
      ctx?.drawImage(img, 0, 0, width, height);
      
      // Convert to optimized JPEG string
      attachedImage.value = canvas.toDataURL('image/jpeg', 0.8);
      
      if (mode.value !== 'chat') mode.value = 'chat';
      selectedModel.value = 'meta-llama/llama-4-scout-17b-16e-instruct';
    };
    img.src = e.target?.result as string;
  };
  reader.readAsDataURL(file);
};

// Image Generation Logic
const handleImageGen = async () => {
  const promptText = textInput.value || input.value;
  if (!promptText) return;
  
  isLoading.value = true;
  isImageLoading.value = true;
  generatedImageUrl.value = '';
  
  try {
    const seed = Math.floor(Math.random() * 1000000);
    const encodedPrompt = encodeURIComponent(promptText);
    // Simplified URL for maximum compatibility
    const url = `https://pollinations.ai/p/${encodedPrompt}?width=1024&height=1024&seed=${seed}&nofeed=true&nologo=true`;
    
    generatedImageUrl.value = url;
    input.value = '';
    textInput.value = '';
    
    // Set URL - the @load event on the img tag will handle stopping the loader
    generatedImageUrl.value = url;
  } catch (error) {
    alert("Request failed. Please try a simpler prompt.");
    isLoading.value = false;
    isImageLoading.value = false;
  }
};

const downloadImage = async () => {
  if (!generatedImageUrl.value) return;
  try {
    // Improved download with CORS handling
    const response = await fetch(generatedImageUrl.value);
    const blob = await response.blob();
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `neural-nexus-art-${Date.now()}.jpg`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    window.URL.revokeObjectURL(url);
  } catch (error) {
    // If fetch fails, open in new tab as fallback
    window.open(generatedImageUrl.value, '_blank');
  }
};

const handleImageError = () => {
  isImageLoading.value = false;
  isLoading.value = false;
  // Clear the image URL so the error state v-if can trigger
  generatedImageUrl.value = '';
};

const openImageDirectly = () => {
  if (generatedImageUrl.value) {
    window.open(generatedImageUrl.value, '_blank');
  }
};
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
  } else if (mode.value === 'image-gen') {
    // Make AI Artist work even in Demo Mode!
    await handleImageGen();
  }

  if (attachedImage.value && mode.value === 'chat') {
    chatHistory.value.push({ role: 'user', content: "[Image Uploaded]" });
    chatHistory.value.push({ role: 'assistant', content: "[DEMO VISION]\nI can see your image! In Live Mode using Llama 3.2 Vision, I would now describe exactly what's in that picture (e.g., colors, objects, text)." });
    attachedImage.value = null;
  }
};

const handleSubmit = async () => {
  if (!input.value && !textInput.value && !attachedImage.value) return; // Added check for attachedImage
  isLoading.value = true;
  result.value = '';

  if (isDemoMode.value) {
    await handleDemoSubmit();
    isLoading.value = false;
    return;
  }

  try {
    const client = getOpenAIClient();
    
    // 2. CHAT MODE LOGIC
    if (mode.value === 'chat') {
      const userMsg: Message = { role: 'user', content: input.value };
      chatHistory.value.push(userMsg);
      
      const currentInput = input.value;
      const currentImage = attachedImage.value;
      
      input.value = '';
      attachedImage.value = null;

      let messages: any[] = [{ role: 'system', content: "You are a helpful, witty, and intelligent AI assistant." }];
      
      if (currentImage) {
        // Prepare multimodal message for vision
        messages.push({
          role: 'user',
          content: [
            { type: 'text', text: currentInput || "Analyze this image." },
            { type: 'image_url', image_url: { url: currentImage } }
          ]
        });
      } else {
        messages = [...messages, ...chatHistory.value];
      }

      const completion = await client.chat.completions.create({
        messages: messages,
        model: selectedModel.value,
      });

      const reply = completion.choices[0].message.content || "No response";
      chatHistory.value.push({ role: 'assistant', content: reply });
    } else if (mode.value === 'summary') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: "Summarize the following text concisely." },
          { role: 'user', content: textInput.value }
        ],
        model: selectedModel.value,
      });
      result.value = completion.choices[0].message.content || "Could not summarize.";
    } else if (mode.value === 'sentiment') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: "Analyze the sentiment of the following text. Respond with: 'Positive', 'Negative', or 'Neutral', followed by a brief explanation." },
          { role: 'user', content: textInput.value }
        ],
        model: selectedModel.value,
      });
      result.value = completion.choices[0].message.content || "Could not analyze.";
    } else if (mode.value === 'translate') {
      const completion = await client.chat.completions.create({
        messages: [
          { role: 'system', content: `You are a professional translator. Translate the following text into ${targetLanguage.value}. Ensure the translation is culturally accurate and maintains the original tone.` },
          { role: 'user', content: textInput.value }
        ],
        model: selectedModel.value,
      });
      result.value = completion.choices[0].message.content || "Could not translate.";
    } else if (mode.value === 'image-gen') {
      await handleImageGen();
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

    <!-- Model Selection Area -->
    <div class="model-selector-container animate-fade-in-delayed-1">
      <div v-for="m in availableModels" :key="m.id" 
        @click="selectedModel = m.id"
        :class="['model-pills', { active: selectedModel === m.id }]"
      >
        <span class="m-icon">{{ m.icon }}</span>
        <span class="m-name">{{ m.name }}</span>
      </div>
    </div>

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
        <p>Multilingual Indian Support</p>
      </button>

      <button 
        @click="mode = 'image-gen'"
        :class="['nav-card', { active: mode === 'image-gen' }]"
      >
        <div class="nav-icon-box">
          <Palette :size="24" />
        </div>
        <h3>AI Artist</h3>
        <p>Generate Art from Text</p>
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
            type="file" 
            ref="fileInput" 
            style="display: none" 
            accept=".pdf" 
            @change="handlePdfUpload" 
          />
          <input 
            type="file" 
            ref="imageInput" 
            style="display: none" 
            accept="image/*" 
            @change="handleImageUpload" 
          />
          
          <button 
            @click="triggerFileUpload"
            class="icon-btn"
            title="Upload PDF"
            :disabled="isExtracting"
          >
            <component :is="isExtracting ? Loader2 : Paperclip" :class="[{ 'icon-spin': isExtracting }]" :size="20" />
          </button>

          <button 
            @click="triggerImageUpload"
            class="icon-btn"
            title="Analyze Image"
          >
            <Image :size="20" />
          </button>

          <button 
            @click="startListening('input')" 
            :class="['icon-btn', { 'pulse-red': isListening }]"
            title="Speak instead of typing"
          >
            <component :is="isListening ? MicOff : Mic" :size="20" />
          </button>

          <div v-if="attachedImage" class="image-preview-mini animate-fade-in">
            <img :src="attachedImage" />
            <button @click="attachedImage = null" class="close-mini">×</button>
          </div>
          
          <input 
            v-model="input"
            @keydown.enter="handleSubmit"
            placeholder="Type or speak your message..."
            class="input-field flex-1"
          />
          <button @click="handleSubmit" :disabled="isLoading" class="btn-primary">
            <Send class="icon-sm" /> Send
          </button>
        </div>
      </div>

      <div v-else class="text-mode-container">
        <!-- Language Selector for Translator -->
        <div v-if="mode === 'translate'" class="language-selector animate-fade-in">
          <label>Translate to:</label>
          <div class="lang-grid">
            <button 
              v-for="lang in indianLanguages" 
              :key="lang"
              @click="targetLanguage = lang"
              :class="['lang-pill', { active: targetLanguage === lang }]"
            >
              {{ lang }}
            </button>
          </div>
        </div>

        <div class="input-section">
          <label>Input Text</label>
          <textarea 
            v-model="textInput"
            :placeholder="mode === 'summary' ? 'Paste text to summarize...' : mode === 'sentiment' ? 'Paste text to analyze...' : mode === 'translate' ? `Paste text to translate into ${targetLanguage}...` : 'Describe the image you want to create (e.g. A sunset over the Taj Mahal)...'"
            class="textarea-field"
          ></textarea>
        </div>

        <div class="action-bar">
           <button 
             @click="startListening('textInput')" 
             class="btn-secondary"
             style="margin-right: 1rem"
           >
             <Mic :size="18" /> {{ isListening ? 'Listening...' : 'Dictate' }}
           </button>
           
           <button @click="handleSubmit" :disabled="isLoading" class="btn-primary">
             <component :is="isLoading ? Loader2 : Sparkles" :class="{'icon-spin': isLoading, 'icon-sm': true}" />
             {{ mode === 'summary' ? 'Generate Summary' : mode === 'sentiment' ? 'Analyze Sentiment' : mode === 'translate' ? 'Translate Text' : 'Create Masterpiece' }}
           </button>
        </div>

        <div v-if="mode === 'image-gen' && (generatedImageUrl || isImageLoading)" class="art-result animate-fade-in">
          <div class="art-card">
            <div v-if="isImageLoading" class="art-loader">
               <Loader2 class="icon-spin" :size="48" />
               <p>Painting your vision...</p>
            </div>
            <div v-if="!isImageLoading && !generatedImageUrl && !isLoading" class="art-error-state">
               <p>The image is taking a while. You can try opening it directly:</p>
               <button @click="openImageDirectly" class="btn-secondary">🔗 Open Image in New Tab</button>
            </div>

            <img 
              v-show="!isImageLoading && generatedImageUrl" 
              :src="generatedImageUrl" 
              class="generated-img" 
              @load="isImageLoading = false; isLoading = false"
              @error="handleImageError"
            />
            <div v-if="!isImageLoading && generatedImageUrl" class="art-actions">
              <button @click="downloadImage" class="btn-download">
                <Download :size="18" /> Download High-Res
              </button>
              <button @click="openImageDirectly" class="btn-secondary" style="margin-left: 1rem">
                🔗 View Source
              </button>
            </div>
          </div>
        </div>

        <div v-else-if="result" class="result-box animate-fade-in">
          <div class="result-header">
            <h3>Result</h3>
            <button @click="speak(result)" class="icon-btn-sm" title="Read Aloud">
              <Volume2 :size="16" />
            </button>
          </div>
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
    grid-template-columns: repeat(5, 1fr);
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

/* Model Selector Styles */
.model-selector-container {
  display: flex;
  overflow-x: auto;
  gap: 1rem;
  margin-bottom: 2rem;
  padding-bottom: 0.5rem;
}

.model-selector-container::-webkit-scrollbar {
  display: none;
}

.model-pills {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: rgba(17, 24, 39, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 9999px;
  cursor: pointer;
  transition: all 0.3s;
  white-space: nowrap;
  font-size: 0.8125rem;
  color: #9ca3af;
}

.model-pills:hover {
  background: rgba(31, 41, 55, 0.6);
  border-color: rgba(255, 255, 255, 0.1);
}

.model-pills.active {
  background: rgba(99, 102, 241, 0.15);
  border-color: #6366f1;
  color: white;
  box-shadow: 0 0 10px rgba(99, 102, 241, 0.1);
}

.m-icon { font-size: 1rem; }

.m-icon { font-size: 1rem; }

/* Language Selector Styles */
.language-selector {
  margin-bottom: 2rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.language-selector label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.lang-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.lang-pill {
  padding: 0.4rem 1rem;
  background: rgba(17, 24, 39, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 9999px;
  cursor: pointer;
  transition: all 0.3s;
  font-size: 0.75rem;
  color: #9ca3af;
}

.lang-pill:hover {
  background: rgba(31, 41, 55, 0.6);
  border-color: rgba(255, 255, 255, 0.1);
}

.lang-pill.active {
  background: rgba(99, 102, 241, 0.2);
  border-color: #6366f1;
  color: white;
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

/* Voice UI */
.icon-btn {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 0.5rem;
  padding: 0.75rem;
  color: #9ca3af;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  color: white;
}

.icon-btn-sm {
  background: transparent;
  border: none;
  color: #6b7280;
  cursor: pointer;
  padding: 0.25rem;
  transition: color 0.2s;
}

.icon-btn-sm:hover { color: #818cf8; }

.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.pulse-red {
  background: rgba(239, 68, 68, 0.1) !important;
  color: #ef4444 !important;
  border-color: #ef4444 !important;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.4); }
  70% { transform: scale(1.05); box-shadow: 0 0 0 10px rgba(239, 68, 68, 0); }
  100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(239, 68, 68, 0); }
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

.image-preview-mini {
  position: relative;
  width: 42px;
  height: 42px;
  border-radius: 6px;
  overflow: hidden;
  border: 1px solid var(--accent-primary);
}

.image-preview-mini img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.close-mini {
  position: absolute;
  top: -2px;
  right: -2px;
  background: #ef4444;
  color: white;
  border: none;
  font-size: 10px;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  line-height: 1;
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
/* Art Result Styles */
.art-result {
  margin-top: 1rem;
}

.art-card {
  background: rgba(0, 0, 0, 0.4);
  border-radius: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.1);
  overflow: hidden;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.5);
}

.generated-img {
  width: 100%;
  height: auto;
  display: block;
  max-height: 600px;
  object-fit: contain;
}

.art-loader {
  height: 400px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1.5rem;
  color: #818cf8;
  background: rgba(255, 255, 255, 0.02);
}

.art-loader p {
  font-weight: 500;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  font-size: 0.875rem;
}

.art-error-state {
  height: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  padding: 2rem;
  text-align: center;
}

.art-error-state p {
  color: #9ca3af;
  margin-bottom: 1rem;
}

.art-actions {
  padding: 1rem;
  display: flex;
  justify-content: center;
  background: rgba(255, 255, 255, 0.03);
}

.btn-download {
  background: var(--accent-primary);
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 0.5rem;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-download:hover {
  transform: translateY(-2px);
  filter: brightness(1.1);
}

.btn-download:active {
  transform: translateY(0);
}
</style>
