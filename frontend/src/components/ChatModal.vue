<template>
  <!-- 聊天模态框 -->
  <transition name="slide">
    <div
      class="chat-modal"
      v-if="isVisible"
    >
      <div class="modal-content">
        <!-- 模态框头部 -->
        <div class="modal-header">
          <h5 class="modal-title">
            {{ mode }}
          </h5>
          <div class="basic-button">
            <button type="button" class="btn btn-sm me-1 add" @click="startNewConversation">
            <i class="fas fa-edit"></i>
          </button>
          <button type="button" class="close btn p-0" @click="closeModal" aria-label="关闭">
              <span aria-hidden="true">&times;</span>
            </button>
          </div>
        </div>
        
        <!-- 模态框主体 -->
        <div class="chat-container">
          <div id="chatContent" ref="chatContent">
            <!-- 聊天内容 -->
            <div
              v-for="(msg, index) in messages"
              :key="index"
              :class="['chat-message', getMessageClass(msg.sender)]"
            >
              <!-- 头像 -->
              <img
                :src="getAvatar(msg.sender)"
                alt="avatar"
                class="avatar"
                @error="handleImageError"
              />
              <!-- 消息内容 -->
              <div class="message-text" v-html="formatMessage(msg.text)">
              </div>
            </div>
          </div>
        </div>
        <!-- 预设选项按钮区域 -->
        <div class="preset-options " v-if="showPresetOptions">
          <button 
            v-for="(option, index) in presetOptions" 
            :key="index"
            class="preset-option-btn rounded-pill"
            @click="handlePresetOption(option)"
          >
            <i :class="option.icon"></i>
            {{ option.text }}
          </button>
        </div>
        <!-- 预设问题按钮区域 -->
        <div class="preset-questions">
          <button 
            v-for="(question, index) in question_list" 
            :key="index" 
            class="preset-button"
            @click="handlePresetQuestion(question)"
          >
            {{ question }}
          </button>
        </div>
        <!-- 模态框底部 -->
        <div class="modal-footer">
          <input
            type="text"
            class="form-control"
            v-model="message"
            @keyup.enter="sendMessage"
            placeholder="输入消息..."
          />
          <button type="button" class="btn send-button" @click="sendMessage">
            send
          </button>
        </div>
        <!-- 归属信息 -->
        <div class="attribution">
          Icon by <a href="https://www.flaticon.com/free-icon/ai_2814666?term=robot&page=1&position=9&origin=search&related_id=2814666" target="_blank">Flaticon</a>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
import axios from "axios";

export default {
  name: 'ChatModal',
  props: {
    visible: {
      type: Boolean,
      default: false,
    },
    mode: {
      type: String,
      default: 'Chat'
    },
    claim: {
      type: String
    },
    userAvatar: { // 用户头像
      type: String,
      default: 'https://via.placeholder.com/40?text=U' // 默认用户头像
    },
    botAvatar: { // 机器人头像
      type: String,
      default: 'https://www.flaticon.com/svg/static/icons/svg/2814/2814666.svg' // Flaticon 机器人头像URL
    }
  },
  data() {
    return {
      message: '',
      messages: [],
      user_cookie: '',
      question_list: [],
      presetOptions: [
        { icon: 'fas fa-plus', text: 'Critical exercises' },
        { icon: 'fas fa-search', text: 'Explore Claims' },
      ],
      showPresetOptions: true,
    };
  },
  computed: {
    isVisible() {
      return this.visible;
    },
  },
  watch:{
    claim(newVal){
      console.log("newVal",newVal)
      if(newVal.trim() !== ''){
        // 在开启新话题的时候清空聊天窗口
        this.messages = [{ text: `正在加载关于支持观点" ${newVal} "的理由...`, sender: 'bot' }]
        this.getGptReply()
        this.getQuestionList()
      }
    }
  },
  methods: {
    closeModal() {
      this.$emit('close');
    },
    sendMessage() {
      if (this.message.trim() !== '') {
        const userMessage = this.message;
        this.messages.push({ text: userMessage, sender: 'user' });
      
        axios.get(`http://localhost:5000/roleplay?claim=${this.claim}&message=${this.message}`,{withCredentials: true})
        .then(response => {
          this.messages.push({text: response.data, sender: 'bot'})
          this.scrollToBottom();
        })
        .catch(error => {
          console.error("return reply error: ", error)
        })
        this.message = '';
        // // 模拟回复
        // setTimeout(() => {
        //   this.messages.push({ text: '这是自动回复的消息', sender: 'bot' });
        //   this.scrollToBottom();
        // }, 1000);
      }
    },
    startNewConversation(){
      // 清空当前对话
      this.messages = [];
      // 重新显示预设选项
      this.showPresetOptions = true;
      // 可能需要重置其他相关状态
      // 例如：重置 claim
      this.$emit('reset-claim');
    },
    handlePresetQuestion(question) {
      if (question.trim() !== '') {
        const userMessage = question;
        this.messages.push({ text: userMessage, sender: 'user' });

        axios.get(`http://localhost:5000/roleplay?claim=${this.claim}&message=${userMessage}`, { withCredentials: true })
          .then(response => {
            this.messages.push({ text: response.data, sender: 'bot' });
            this.scrollToBottom();
          })
          .catch(error => {
            console.error("返回回复错误: ", error)
          });
      }
    },
    handlePresetOption(option) {
      console.log('Selected option:', option);
      // 这里可以根据选项执行相应的操作，比如发送特定的消息
      this.message = option.text;
      // 完成特定的操作
      this.messages.push({ text: this.message, sender: 'user' });
      // this.sendMessage();
      this.showPresetOptions = false; // 隐藏预设选项
      this.message = '';
    },
    getMessageClass(sender) {
      return sender === 'user' ? 'user-message' : 'bot-message';
    },
    getAvatar(sender) {
      return sender === 'user' ? this.userAvatar : this.botAvatar;
    },
    getGptReply(){
      axios.get(`http://localhost:5000/roleplay?claim=${this.claim}`)
      .then(response => {
        this.messages.push({text: response.data, sender: 'bot'})
        this.scrollToBottom();
        const userId = this.getCookie('user_id')
        if(userId){
          this.userId = userId
          console.log('获取到的 user_id:', this.userId);
        }
      })
      .catch(error => {
        console.error('roleplay first reply error', error)
      })
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const chatContent = this.$refs.chatContent;
        if (chatContent) {
          chatContent.scrollTop = chatContent.scrollHeight;
        }
      });
    },
    formatMessage(text) {
      // 转换有序列表
      const listPattern = /^(\d+\.\s.+(\n|$))+/
      if (listPattern.test(text)) {
        const listItems = text.split('\n').filter(item => item.trim() !== '');
        const formattedList = listItems.map(item => `<li>${item.replace(/^\d+\.\s/, '')}</li>`).join('');
        return `<ol>${formattedList}</ol>`;
      }
      // 保留换行
      return text.replace(/\n/g, '<br>');
    },
    handleImageError(event) {
      // 头像加载失败时显示默认占位图
      event.target.src = 'https://via.placeholder.com/40?text=?';
    },
    getCookie(name){
      const value = `; ${document.cookie}`;
      const parts = value.split(`; ${name}=`);
      if (parts.length === 2) return parts.pop().split(';').shift();
      return null;
    },
    getQuestionList(){
      axios.get(`http://localhost:5000/roleplay/prompt?claim=${this.claim}`, {withCredentials: true})
      .then(response => {
        try {
          // 假设 response.data 是一个字符串，需要解析为 JSON
          console.log("response.data",response.data)
          this.question_list = response.data.question_list
        } catch (e) {
          console.error("解析问题列表时出错:", e);
        }
      })
      .catch(error => {
        console.error('roleplay question list error', error)
      })
    }
  },
};
</script>

<style scoped>
/* 自定义样式 */

/* 过渡效果 */
.slide-enter-active, .slide-leave-active {
  transition: transform 0.4s ease, opacity 0.4s ease;
}

.slide-enter-from {
  opacity: 0;
}

.slide-enter-to {
  opacity: 1;
}

.slide-leave-from {
  opacity: 1;
}

.slide-leave-to {
  opacity: 0;
}

/* 基础模态框样式 */
.chat-modal {
  background: rgba(255, 255, 255, 0.9); /* 浅色背景 */
  position: fixed; /* 固定位置 */
  bottom: 50px; /* 距离底部20px */
  right: 20px; /* 距离右侧20px */
  width: 450px; /* 设置固定宽度 */
  max-width: 90%; /* 响应式宽度 */
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  border-radius: 10px;
  overflow: hidden;
  z-index: 10000; /* 确保在其他内容之上 */
}

/* 内容区域样式 */
.modal-content {
  background: #f9f9f9; /* 浅色背景 */
  border: none;
  border-radius: 10px;
  overflow: hidden;
}

/* 头部样式 */
.modal-header {
  background: #ffffff; /* 统一的浅色调 */
  color: #333;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 10px;
  border-bottom: 1px solid #ddd;
}

.modal-header .modal-title {
  font-family: 'Roboto', sans-serif;
  font-weight: 500;
}


.modal-header .add {
  font-size: 15px;
  color: #bbb;
  transition: color 0.3s;
  justify-content: flex-end;
}

.modal-header .add:hover {
  color: #333;
}

.modal-header .close {
  font-size: 24px;
  color: #bbb;
  transition: color 0.3s;
  justify-content: flex-end;
}

.modal-header .close:hover {
  color: #333;
}



/* 聊天区域样式 */
.chat-container {
  background: #ffffff;
  padding: 20px;
  height: 250px; /* 调整高度 */
  overflow-y: auto;
}

#chatContent {
  display: flex;
  flex-direction: column;
}

/* 聊天气泡样式 */
.chat-message {
  display: flex;
  align-items: flex-start;
  margin: 10px 0;
}

/* 头像样式 */
.chat-message .avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin: 0 10px;
}

/* 消息内容样式 */
.message-text {
  max-width: 70%;
  padding: 5px 5px;
  border-radius: 15px;
  background: #e0e0e0;
  color: #333;
  font-family: 'Roboto', sans-serif;
  word-wrap: break-word;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  position: relative;
  white-space: pre-wrap; /* 保留换行符 */
  text-align: left;
}

/* 用户消息样式 */
.user-message {
  align-self: flex-end;
  flex-direction: row-reverse; /* 将头像放在右侧 */
}

.user-message .avatar {
  margin: 0 0 0 10px; /* 调整头像间距 */
}

.user-message .message-text {
  background: #198754; /* 统一的主色调 */
  color: #fff;
  border-bottom-right-radius: 0;
}

/* 机器人消息样式 */
.bot-message {
  align-self: flex-start;
  flex-direction: row; /* 将头像放在左侧 */
}

.bot-message .avatar {
  margin: 0 10px 0 0; /* 调整头像间距 */
}

.bot-message .message-text {
  background: #e0e0e0;
  color: #333;
  border-bottom-left-radius: 0;
}

/* 底部样式 */
.modal-footer {
  display: flex;
  padding: 15px 20px;
  background: #ffffff; /* 统一的浅色调 */
  border-top: 1px solid #ddd;
}

.modal-footer input {
  flex: 1;
  padding: 10px 15px;
  border: 1px solid #ccc;
  border-radius: 20px;
  background: #f5f5f5;
  color: #333;
  font-size: 14px;
  outline: none;
  transition: background 0.3s, border-color 0.3s;
}

.modal-footer input::placeholder {
  color: #aaa;
}

.modal-footer input:focus {
  background: #ffffff;
  border-color: #198754;
}

.send-button {
  margin-left: 10px;
  padding: 10px 20px;
  border: none;
  border-radius: 20px;
  background: #198754; /* 统一的主色调 */
  color: #fff;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.3s, transform 0.2s;
}

.send-button:hover {
  background: #15602b; /* 深化颜色以显示悬停效果 */
  color:#ddd;
  transform: translateY(-2px);
}

/* 滚动条样式 */
.chat-container::-webkit-scrollbar {
  width: 8px;
}

.chat-container::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.chat-container::-webkit-scrollbar-thumb {
  background-color: #c1c1c1;
  border-radius: 4px;
}

/* 有序列表样式 */
.message-text ol {
  padding-left: 20px;
}

.message-text li {
  margin-bottom: 5px;
}

/* 预设问题按钮样式 */
.preset-questions {
  background: rgba(255, 255, 255, 0.8);
  padding: 10px 20px;
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.preset-button {
  padding: 8px 16px;
  border: none;
  border-radius: 20px;
  background: #20c997; /* 蓝绿色背景 */
  color: #fff;
  cursor: pointer;
  transition: background 0.3s, transform 0.2s, box-shadow 0.3s;
  font-size: 14px;
}

.preset-button:hover {
  background: #17a2b8; /* 悬停时的蓝绿色 */
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

/* 响应式调整 */
@media (max-width: 800px) {
  .chat-modal {
    width: 90%; /* 调整宽度 */
    right: 5%; /* 调整右边距 */
  }

  .modal-footer input {
    width: 100%;
  }

  .send-button {
    width: 100%;
    margin-left: 0;
    margin-top: 10px;
  }

  .chat-message {
    flex-direction: column;
    align-items: flex-start;
  }

  .user-message {
    align-self: flex-end;
  }

  .user-message .avatar {
    margin: 10px 0 0 0;
  }

  .message-text {
    max-width: 100%;
  }
}

/* 归属信息样式 */
.attribution {
  text-align: center;
  font-size: 10px;
  color: #aaa;
  padding: 5px 0;
}

.attribution a {
  color: #aaa;
  text-decoration: none;
}

.attribution a:hover {
  color: #4a90e2;
}


/* 预设选项按钮样式 */
.preset-options {
  display: flex;
  flex-direction: column;
  padding: 10px;
  background-color: white;
}

.preset-option-btn {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  margin-bottom: 5px;
  border: none;
  background-color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
  text-align: left;
}

.preset-option-btn:hover {
  background-color: #e9e9e9;
}

.preset-option-btn i {
  margin-right: 10px;
  font-size: 16px;
  width: 20px;
  text-align: center;
}
</style>