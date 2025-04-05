<template>
  <div class="chat-wrapper">
    <div class="chat-container">
      <!-- Mensagem fixa no topo -->
      <div class="system-message" v-if="systemMessage">
        <span class="user">{{ systemMessage.user }}:</span>
        <span class="text">{{ systemMessage.text }}</span>
      </div>
      <!-- Área de mensagens -->
      <div class="messages">
        <div v-for="(msg, index) in reversedMessages" :key="index" class="message">
          <span class="user">{{ msg.user }}:</span>
          <span class="text">{{ msg.text }}</span>
        </div>
      </div>
      <form @submit.prevent="sendMessage" class="input-group">
        <input
            v-model="newMessage"
            type="text"
            class="form-control"
            placeholder="Digite sua mensagem..."
            :disabled="loading"
        />
        <button type="submit" class="btn btn-primary" :disabled="loading">
          {{ loading ? 'Aguarde...' : 'Enviar' }}
        </button>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { HfInference } from "@huggingface/inference";

const HF_TOKEN = "seu_token"; // Substitua pelo seu token
const client = new HfInference(HF_TOKEN);

const messages = ref([{ user: 'Sistema AI Vue', text: 'Bem-vindo ao chat!' }]);
const newMessage = ref('');
const loading = ref(false);

// Computed para organizar mensagens
const systemMessage = computed(() => messages.value.length ? messages.value[0] : null);
const reversedMessages = computed(() => messages.value.slice(1).reverse());

async function sendMessage() {
  if (!newMessage.value.trim()) return;

  const userMsg = newMessage.value;
  messages.value.push({ user: 'Você', text: userMsg }); // Adiciona a mensagem do usuário ao histórico
  newMessage.value = '';
  loading.value = true;

  try {
    let botReply = "";

    // Montar o histórico de mensagens corretamente para enviar ao modelo
    const chatHistory = messages.value.slice(1).map(msg => ({
      role: msg.user === 'Você' ? 'user' : 'assistant',
      content: msg.text
    }));

    const stream = client.chatCompletionStream({
      model: "meta-llama/Meta-Llama-3-8B-Instruct", // Modelo válido
      messages: [...chatHistory, { role: "user", content: userMsg }], // Inclui histórico e a nova mensagem
      temperature: 0.5,
      max_tokens: 2048,
      top_p: 0.7
    });

    // Adiciona a resposta do bot em tempo real
    const botMessage = { user: 'Bot', text: '' };
    messages.value.push(botMessage);

    for await (const chunk of stream) {
      if (chunk.choices && chunk.choices.length > 0) {
        const newContent = chunk.choices[0].delta.content;
        botReply += newContent;
        botMessage.text = botReply; // Atualiza a mensagem do bot na interface
      }
    }
  } catch (error) {
    console.error("Erro na IA:", error);
    messages.value.push({ user: 'Bot', text: 'Erro ao gerar resposta.' });
  } finally {
    loading.value = false;
  }
}
</script>


<style scoped>
.chat-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #f8f9fa;
}

.chat-container {
  display: flex;
  flex-direction: column;
  width: 600px;
  height: 80vh;
  border: 1px solid #ccc;
  border-radius: 8px;
  background: #fff;
  overflow: hidden;
}

.system-message {
  padding: 1rem;
  border-bottom: 1px solid #ccc;
  background: #e9ecef;
}

.messages {
  flex: 1;
  display: flex;
  flex-direction: column-reverse;
  padding: 1rem;
  overflow-y: auto;
}

.message {
  margin-bottom: 0.5rem;
}

.user {
  font-weight: bold;
}

.input-group {
  padding: 0.5rem;
  border-top: 1px solid #ccc;
  background: #e9ecef;
}
</style>
