<template>
  <div>
    <Appbar :group-name="groupName"/>
    <div class="mdui-container">
      <LoadingSpinner class="mdui-center mdui-m-t-3" v-if="loadingHistoryMessage"/>
      <div class="mdui-row" v-if="!loadingHistoryMessage">
        <Message v-for="i in historyMessage"
          v-if="i.group === groupName"
          :key="i.id"
          :username='i.username'
          :content='i.content'
          :email="i.email"
          :timestamp="i.timestamp"
          :self="i.username === username">
        </Message>
      </div>
      <div class="mdui-row">
        <div class="mdui-divider mdui-m-t-1 mdui-m-b-1" v-if="!loadingHistoryMessage"></div>
        <Message v-for="i in messages"
          v-if="i.group === groupName"
          :key="i.id"
          :username='i.username'
          :content='i.content'
          :email="i.email"
          :timestamp="i.timestamp"
          :self="i.username === username">
        </Message>
      </div>
      <div class="mdui-row" id="inputArea" :class="{ nightMode: nightMode, dayMode: !nightMode}">
        <div class="mdui-m-t-2">
          <div v-if="!logined">
            <div class="mdui-col-xs-6">
              <div class="mdui-textfield mdui-textfield-floating-label">
                <label class="mdui-textfield-label">Username</label>
                <input class="mdui-textfield-input" type="text" v-model="username" required/>
                <div class="mdui-textfield-error">用户名不能为空</div>
              </div>
            </div>
            <div class="mdui-col-xs-6">
              <div class="mdui-textfield mdui-textfield-floating-label">
                <label class="mdui-textfield-label">Email</label>
                <input class="mdui-textfield-input" type="email" v-model="email" required/>
                <div class="mdui-textfield-error">邮箱格式错误</div>
              </div>
            </div>
          </div>
          <div class="mdui-col-xs-12">
            <div class="mdui-textfield mdui-textfield-floating-label">
              <label class="mdui-textfield-label">Message</label>
              <div class="mdui-textfield-helper mdui-typo">
                <a href="https://segmentfault.com/markdown" target="_blank">MarkDown</a> is Supported
              </div>
            <textarea class="mdui-textfield-input" @keyup.ctrl.enter="sendMessage" v-model="inputMessage"></textarea>
            </div>
          </div>
        </div>
      </div>
      <button @click="sendMessage" class="mdui-fab mdui-fab-fixed mdui-color-theme-accent mdui-ripple">
        <i class="mdui-icon material-icons">send</i>
      </button>
    </div>
  </div>
</template>

<script>
import Message from './SingleMessage'
import LoadingSpinner from './LoadingSpinner'
import Appbar from './Appbar.vue'
import mdui from 'mdui'
import { renderAllMath } from '../utilities/MathRenderer.js'
import gavatarLink from '../utilities/GravatarLink'
import delay from 'lodash/delay'
export default {
  name: 'MessageArea',
  components: {Message, LoadingSpinner, Appbar},
  props: ['nightMode', 'resetUsername', 'groupName'],
  sockets: {
    connect: function () {
      mdui.snackbar('连接到服务器')
      console.log('socket connected')
    },
    disconnect: function () {
      mdui.snackbar('看起来您掉线了..')
      this.newlyAdded = false
      console.log('socket disconnected')
    },
    newMessage: function (val) {
      this.addMessage(val)
    },
    login: function (data) {
      mdui.snackbar(`成功连接服务器，现有${data.numUsers}人在线`)
    },
    getHistoryMessage: function (data) {
      this.historyMessage = data
      this.loadingHistoryMessage = false
      delay(renderAllMath, 300)
    }
  },
  mounted () {
    mdui.mutation()
    if (localStorage.getItem('username') !== null) {
      this.username = localStorage.getItem('username')
      this.logined = true
    }
    if (localStorage.getItem('email') !== null) {
      this.email = localStorage.getItem('email')
      this.logined = true
    }
    this.$socket.emit('fetchHistory')
  },
  updated () {
    document.getElementById('end').scrollIntoView()
    // var scroll = new SmoothScroll()
    // var anchor = document.querySelector('#end')
    // scroll.animateScroll(anchor)
    // document.getElementById('inputText').focus()
  },
  methods: {
    addMessage: function (data) {
      this.messages.push(data)
      if (data.username !== this.username && data.group === this.groupName) {
        this.$notification.show(`${data.username} said:`, {
          body: data.content,
          icon: gavatarLink('', data.email)
        }, {})
      }
    },
    sendMessage: function () {
      if (this.inputMessage === '') return
      var data = {
        username: this.username,
        content: this.inputMessage,
        email: this.email,
        timestamp: (new Date()).valueOf(),
        group: this.groupName
      }
      localStorage.setItem('email', this.email)
      localStorage.setItem('username', this.username)
      this.logined = true
      if (!this.newlyAdded) {
        this.$socket.emit('addUser', this.username, this.email)
        this.newlyAdded = true
      }
      this.addMessage(data)
      this.inputMessage = ''
      this.$socket.emit('newMessage', data)
    }
  },
  data () {
    return {
      username: '',
      email: '',
      messages: [],
      inputMessage: '',
      newlyAdded: false,
      logined: false,
      historyMessage: [],
      loadingHistoryMessage: true
    }
  },
  watch: {
    resetUsername: function () {
      this.logined = false
    }
  }
}
</script>

<style scoped>
textarea {
    font-family: 'Fira Code', Consolas, 'Sarasa Gothic SC',
    PingFang SC ,
    Microsoft YaHei,
    Source Han Sans SC,
    Noto Sans CJK SC,
    WenQuanYi Micro Hei,
    monospace;
}
#inputArea {
  position: sticky;
  bottom: 0px;
}
.nightMode {
  background-color: rgba(48, 48, 48, 1);
}
.dayMode {
  background-color: white;
}
</style>
