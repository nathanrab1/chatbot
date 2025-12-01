<script setup lang="ts">
/**
 * PAINEL DE PREVIEW DO CHATBOT
 *
 * Este componente simula a execução do chatbot, permitindo testar o fluxo
 * antes de publicá-lo. Ele percorre os blocos seguindo as conexões e
 * gerencia o estado da conversa.
 *
 * Funcionalidades:
 * - Exibe mensagens do bot e do usuário em bolhas estilizadas
 * - Processa perguntas abertas e múltipla escolha
 * - Avalia condições lógicas
 * - Interpola variáveis no texto (ex: {{nome}})
 * - Trata erros de fluxo (blocos não encontrados)
 */

import { ref, nextTick } from 'vue';
import type { Block, Variable, ChatMessage } from '../types/chatbot';
import { interpolateText, evaluateCondition } from '../utils/interpolation';

const props = defineProps<{
  blocks: Block[];
  variables: Record<string, Variable>;
}>();

// Estado da conversa
const messages = ref<ChatMessage[]>([]);
const currentBlockId = ref<string | null>(null);
const userInput = ref('');
const isWaitingForInput = ref(false);
const currentChoices = ref<{ id: string; label: string; nextBlockId?: string }[]>([]);
const sessionVariables = ref<Record<string, Variable>>({});
const chatEndRef = ref<HTMLDivElement | null>(null);
const isRunning = ref(false);

// NOVO: controla se o preview está em tela cheia
const isFullscreen = ref(false);

// Inicia uma nova sessão do chatbot
function startChat() {
  messages.value = [];
  currentBlockId.value = null;
  userInput.value = '';
  isWaitingForInput.value = false;
  currentChoices.value = [];
  isRunning.value = true;

  // Copia as variáveis para a sessão atual
  sessionVariables.value = {};
  Object.keys(props.variables).forEach(key => {
    sessionVariables.value[key] = { ...props.variables[key] };
  });

  // Busca o primeiro bloco para iniciar o fluxo
  if (props.blocks.length > 0) {
    // Tenta encontrar o primeiro bloco do tipo 'message' ou usa o primeiro disponível
    const startBlock = props.blocks.find(b => b.type === 'message') || props.blocks[0];
    currentBlockId.value = startBlock.id;
    processBlock(startBlock);
  } else {
    addErrorMessage('Nenhum bloco encontrado. Crie blocos no canvas para começar.');
  }
}

// Processa um bloco baseado no seu tipo
function processBlock(block: Block) {
  if (!block) {
    addErrorMessage('(Erro de fluxo: bloco não encontrado)');
    endChat();
    return;
  }

  switch (block.type) {
    case 'message':
      // Exibe uma mensagem e continua para o próximo bloco
      addBotMessage(block.content);
      setTimeout(() => {
        if (block.nextBlockId) {
          const nextBlock = props.blocks.find(b => b.id === block.nextBlockId);
          if (nextBlock) {
            currentBlockId.value = block.nextBlockId;
            processBlock(nextBlock);
          } else {
            console.error(`Bloco de destino não encontrado: ${block.nextBlockId}`);
            addErrorMessage('(Erro de fluxo: bloco de destino não encontrado)');
            endChat();
          }
        } else {
          endChat();
        }
      }, 500);
      break;

    case 'openQuestion':
      // Aguarda uma resposta de texto do usuário
      addBotMessage(block.content);
      isWaitingForInput.value = true;
      currentChoices.value = [];
      break;

    case 'choiceQuestion':
      // Exibe botões de múltipla escolha
      addBotMessage(block.content);
      if (block.choices && block.choices.length > 0) {
        currentChoices.value = block.choices;
        isWaitingForInput.value = false;
      } else {
        addErrorMessage('(Erro: pergunta sem opções de escolha)');
        endChat();
      }
      break;

    case 'condition':
      // Avalia condições e segue para o bloco correspondente
      let nextBlockId: string | undefined;

      if (block.conditions) {
        for (const condition of block.conditions) {
          if (
            evaluateCondition(
              condition.variableName,
              condition.operator,
              condition.value,
              sessionVariables.value
            )
          ) {
            nextBlockId = condition.nextBlockId;
            break;
          }
        }
      }

      if (nextBlockId) {
        const nextBlock = props.blocks.find(b => b.id === nextBlockId);
        if (nextBlock) {
          currentBlockId.value = nextBlockId;
          processBlock(nextBlock);
        } else {
          console.error(`Bloco de destino não encontrado: ${nextBlockId}`);
          addErrorMessage('(Erro de fluxo: bloco de destino não encontrado)');
          endChat();
        }
      } else {
        // Nenhuma condição foi satisfeita
        console.warn('Nenhuma condição foi satisfeita no bloco condicional');
        addErrorMessage('(Nenhuma condição foi satisfeita. Verifique o bloco condicional.)');
        endChat();
      }
      break;

    case 'end':
      // Bloco final
      if (block.content) {
        addBotMessage(block.content);
      }
      endChat();
      break;

    default:
      console.warn(`Tipo de bloco não suportado: ${block.type}`);
      addErrorMessage(`(Tipo de bloco não suportado: ${block.type})`);
      endChat();
      break;
  }
}

// Adiciona uma mensagem do bot ao chat
function addBotMessage(content: string) {
  const interpolated = interpolateText(content, sessionVariables.value);
  messages.value.push({
    id: `msg_${Date.now()}`,
    type: 'bot',
    content: interpolated
  });
  scrollToBottom();
}

// Adiciona uma mensagem do usuário ao chat
function addUserMessage(content: string) {
  messages.value.push({
    id: `msg_${Date.now()}`,
    type: 'user',
    content
  });
  scrollToBottom();
}

// Adiciona uma mensagem de erro ao chat
function addErrorMessage(content: string) {
  messages.value.push({
    id: `msg_${Date.now()}`,
    type: 'bot',
    content: `⚠️ ${content}`
  });
  scrollToBottom();
}

// Processa o envio de uma resposta de texto livre
function handleSendMessage() {
  if (!userInput.value.trim() || !isWaitingForInput.value) return;

  const input = userInput.value.trim();
  addUserMessage(input);

  // Salva a resposta na variável correspondente
  const currentBlock = props.blocks.find(b => b.id === currentBlockId.value);
  if (currentBlock && currentBlock.type === 'openQuestion' && currentBlock.variableName) {
    const variable = sessionVariables.value[currentBlock.variableName];
    if (variable) {
      if (variable.type === 'number') {
        sessionVariables.value[currentBlock.variableName] = {
          ...variable,
          value: Number(input) || 0
        };
      } else {
        sessionVariables.value[currentBlock.variableName] = {
          ...variable,
          value: input
        };
      }
    }
  }

  userInput.value = '';
  isWaitingForInput.value = false;

  // Segue para o próximo bloco
  if (currentBlock && currentBlock.nextBlockId) {
    const nextBlock = props.blocks.find(b => b.id === currentBlock.nextBlockId);
    if (nextBlock) {
      currentBlockId.value = currentBlock.nextBlockId;
      setTimeout(() => processBlock(nextBlock), 300);
    } else {
      console.error(`Bloco de destino não encontrado: ${currentBlock.nextBlockId}`);
      addErrorMessage('(Erro de fluxo: bloco de destino não encontrado)');
      endChat();
    }
  } else {
    endChat();
  }
}

// Processa a escolha de uma opção de múltipla escolha
function handleChoiceClick(choice: { id: string; label: string; nextBlockId?: string }) {
  addUserMessage(choice.label);
  currentChoices.value = [];

  if (choice.nextBlockId) {
    const nextBlock = props.blocks.find(b => b.id === choice.nextBlockId);
    if (nextBlock) {
      currentBlockId.value = choice.nextBlockId;
      setTimeout(() => processBlock(nextBlock), 300);
    } else {
      console.error(`Bloco de destino não encontrado: ${choice.nextBlockId}`);
      addErrorMessage('(Erro de fluxo: bloco de destino não encontrado)');
      endChat();
    }
  } else {
    console.warn('Escolha sem bloco de destino');
    addErrorMessage('(Erro: escolha sem destino definido)');
    endChat();
  }
}

// Finaliza a conversa
function endChat() {
  isWaitingForInput.value = false;
  currentChoices.value = [];
  isRunning.value = false;
}

// Rola o chat para o final automaticamente
function scrollToBottom() {
  nextTick(() => {
    if (chatEndRef.value) {
      chatEndRef.value.scrollIntoView({ behavior: 'smooth' });
    }
  });
}
</script>

<template>
  <!-- quando isFullscreen = true, adiciona classe modificadora -->
  <div :class="['preview-panel', { fullscreen: isFullscreen }]">
    <!-- Botão de tela cheia / voltar -->
    <button
      class="btn-fullscreen"
      @click="isFullscreen = !isFullscreen"
      :title="isFullscreen ? 'Fechar tela cheia' : 'Ver em tela cheia'"
    >
      <span v-if="!isFullscreen">⤢</span>
      <span v-else>✕</span>
    </button>

    <!-- Tela inicial (antes de rodar o chatbot) -->
    <div v-if="!isRunning && messages.length === 0" class="start-screen">
      <div class="start-icon">🤖</div>
      <h3>Preview do chatbot</h3>
      <p>
        Clique no botão abaixo para testar o fluxo do seu chatbot como se fosse um aluno
        conversando com ele.
      </p>
      <button class="btn-start" @click="startChat">
        ▶️ Iniciar conversa
      </button>
    </div>

    <!-- Chat em execução -->
    <div v-else class="chat-container">
      <div class="chat-header">
        <div class="avatar">🤖</div>
        <div class="header-text">
          <h3>Chatbot pedagógico</h3>
          <p>Simulação de conversa com o fluxo criado no canvas</p>
        </div>
      </div>

      <div class="chat-messages">
        <div
          v-for="msg in messages"
          :key="msg.id"
          :class="['chat-message', msg.type === 'bot' ? 'bot' : 'user']"
        >
          <div class="bubble">
            <p>{{ msg.content }}</p>
          </div>
        </div>

        <!-- Botões de múltipla escolha -->
        <div v-if="currentChoices.length > 0" class="choices-container">
          <button
            v-for="choice in currentChoices"
            :key="choice.id"
            class="choice-button"
            @click="handleChoiceClick(choice)"
          >
            {{ choice.label }}
          </button>
        </div>

        <div ref="chatEndRef"></div>
      </div>

      <!-- Campo de entrada para perguntas abertas -->
      <div class="chat-input" v-if="isWaitingForInput">
        <input
          v-model="userInput"
          type="text"
          class="input"
          placeholder="Digite sua resposta..."
          @keyup.enter="handleSendMessage"
        />
        <button class="btn-send" @click="handleSendMessage">
          Enviar
        </button>
      </div>

      <!-- Mensagem de conversa encerrada + botão de reiniciar -->
      <div v-if="!isRunning && messages.length > 0" class="chat-footer">
        <p class="end-text">
          🔚 A conversa chegou ao fim. Você pode ajustar o fluxo no canvas e testar novamente.
        </p>
        <button class="btn-restart" @click="startChat">
          🔁 Reiniciar conversa
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.preview-panel {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: #f9fafb;
  position: relative; /* para o botão absoluto funcionar */
}

/* MODO TELA CHEIA */
.preview-panel.fullscreen {
  position: fixed;
  inset: 0;
  z-index: 50;
  background: #f9fafb;
  padding: 16px;
}

.preview-panel.fullscreen .chat-container {
  max-width: 900px;
  margin: 0 auto;
}

.preview-panel.fullscreen .start-screen {
  max-width: 600px;
  margin: 0 auto;
}

/* Tela inicial */
.start-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  gap: 16px;
  padding: 32px;
  text-align: center;
}

.start-icon {
  font-size: 64px;
  margin-bottom: 8px;
}

.start-screen h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 700;
  color: #111827;
}

/* Botão de tela cheia */
.btn-fullscreen {
  position: absolute;
  top: 12px;
  right: 12px;
  z-index: 100;
  width: 36px;
  height: 36px;
  padding: 0;
  background: white;
  color: #6b7280;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.btn-fullscreen:hover {
  background: #f9fafb;
  color: #3b82f6;
  border-color: #3b82f6;
  transform: scale(1.1);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.2);
}

.btn-fullscreen:active {
  transform: scale(0.96);
  box-shadow: 0 1px 4px rgba(15, 23, 42, 0.2);
}

.start-screen p {
  margin: 0;
  max-width: 420px;
  font-size: 14px;
  color: #4b5563;
}

.btn-start {
  margin-top: 8px;
  padding: 10px 20px;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 8px;
}

.btn-start:hover {
  background: #059669;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.3);
}

/* Container principal do chat */
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
}

/* Cabeçalho do chat */
.chat-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid #e5e7eb;
  background: #ffffff;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 999px;
  background: #eff6ff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
}

.header-text h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: #111827;
}

.header-text p {
  margin: 0;
  font-size: 12px;
  color: #6b7280;
}

/* Área de mensagens */
.chat-messages {
  flex: 1;
  padding: 12px 16px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* Mensagens */
.chat-message {
  display: flex;
  margin-bottom: 4px;
}

.chat-message.bot {
  justify-content: flex-start;
}

.chat-message.user {
  justify-content: flex-end;
}

.bubble {
  max-width: 80%;
  padding: 10px 12px;
  border-radius: 12px;
  font-size: 14px;
  line-height: 1.4;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.08);
}

.chat-message.bot .bubble {
  background: #ffffff;
  border-bottom-left-radius: 4px;
  color: #111827;
}

.chat-message.user .bubble {
  background: #2563eb;
  border-bottom-right-radius: 4px;
  color: white;
}

/* Botões de múltipla escolha */
.choices-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 4px;
}

.choice-button {
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid #d1d5db;
  background: #ffffff;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.15s;
}

.choice-button:hover {
  background: #eff6ff;
  border-color: #3b82f6;
  color: #1d4ed8;
}

/* Campo de entrada */
.chat-input {
  display: flex;
  gap: 8px;
  padding: 10px 16px;
  border-top: 1px solid #e5e7eb;
  background: #f9fafb;
}

.input {
  flex: 1;
  padding: 8px 10px;
  border-radius: 999px;
  border: 1px solid #d1d5db;
  font-size: 14px;
  outline: none;
}

.input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 1px rgba(59, 130, 246, 0.3);
}

.btn-send {
  padding: 8px 14px;
  border-radius: 999px;
  border: none;
  background: #3b82f6;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s, transform 0.1s;
}

.btn-send:hover {
  background: #2563eb;
  transform: translateY(-1px);
}

/* Rodapé quando a conversa acaba */
.chat-footer {
  padding: 10px 16px 14px;
  border-top: 1px solid #e5e7eb;
  background: #f9fafb;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.end-text {
  margin: 0;
  font-size: 13px;
  color: #4b5563;
}

.btn-restart {
  align-self: flex-start;
  padding: 6px 12px;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-restart:hover {
  background: #059669;
  transform: translateY(-1px);
}
</style>
