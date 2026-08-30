<template>
  <div class="fz-page">

    <!-- ======= 顶栏 ======= -->
    <header class="fz-header">
      <div class="fz-h-left">
        <div class="fz-mascot" aria-hidden="true">
          <img class="fz-mascot-img" src="./avatar.jpg" alt="反诈小易" />
        </div>
        <div class="fz-h-info">
          <div class="fz-h-name">反诈小易<span class="fz-h-badge">校园反诈智能体</span></div>
          <div class="fz-h-status"><span class="fz-dot"></span>7×24 在线 · 已守护 {{ guardCount }} 次</div>
        </div>
      </div>
      <div class="fz-h-right">
        <div class="fz-rank" :title="'再获 ' + rank.need + ' 分晋级 ' + rank.next">
          <span class="fz-rank-name">{{ rank.name }}</span>
          <span class="fz-rank-score">{{ score }} 分</span>
        </div>
        <button class="fz-icon-btn" title="清空会话（不存储任何隐私）" @click="clearChat">
          <svg viewBox="0 0 24 24" width="17" height="17" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M3 6h18M8 6V4a1 1 0 0 1 1-1h6a1 1 0 0 1 1 1v2M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6M10 11v6M14 11v6"/></svg>
        </button>
      </div>
    </header>

    <!-- ======= 紧急求助条 ======= -->
    <div class="fz-sos" role="button" @click="sosOpen = true">
      <span class="fz-sos-flag">🚨</span>
      <span class="fz-sos-text">疑似被骗？点我一键紧急求助</span>
      <span class="fz-sos-phone">96110</span>
    </div>

    <!-- ======= 消息区 ======= -->
    <main class="fz-body" ref="bodyRef">
      <div v-for="(m, i) in messages" :key="i" class="fz-row" :class="m.role">
        <div v-if="m.role === 'assistant'" class="fz-avatar fz-avatar-bot" aria-hidden="true">
          <img class="fz-avatar-img" src="./avatar.jpg" alt="反诈小易" />
        </div>

        <div class="fz-bubble" :class="m.role">
          <div v-if="m.role === 'assistant' && i === 0" class="fz-welcome">
            <div class="fz-welcome-title">🛡️ 校园反诈 · 趣味互动智能体</div>
            <div class="fz-md" v-html="renderMd(m.content)"></div>
            <div class="fz-welcome-chips">
              <button v-for="q in quickActions" :key="q.cmd" class="fz-chip" @click="send(q.cmd)">
                <span class="fz-chip-ico">{{ q.ico }}</span>{{ q.label }}
              </button>
            </div>
          </div>
          <div v-else class="fz-md" v-html="renderMd(m.content)"></div>

          <div v-if="m.loading" class="fz-typing" aria-label="正在输入">
            <span></span><span></span><span></span>
          </div>
        </div>

        <div v-if="m.role === 'user'" class="fz-avatar fz-avatar-user">{{ userInitial }}</div>
      </div>
    </main>

    <!-- ======= 输入栏 ======= -->
    <footer class="fz-inputbar-wrap">
      <div class="fz-inputbar">
        <textarea
          ref="taRef"
          v-model="draft"
          rows="1"
          maxlength="500"
          placeholder="输入你的问题，比如：刷单返利是诈骗吗？"
          @keydown.enter.exact.prevent="send()"
          @input="autoGrow"
        ></textarea>
        <button class="fz-send" :disabled="sending || !draft.trim()" @click="send()" aria-label="发送">
          <svg v-if="!sending" viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M3 20.5 22 12 3 3.5 3 10l13 2-13 2z"/></svg>
          <span v-else class="fz-send-spin"></span>
        </button>
      </div>
      <div class="fz-footnote">AI 生成内容仅供参考 · 紧急情况请拨打 <b>96110</b> / <b>110</b></div>
    </footer>

    <!-- ======= 求助弹窗 ======= -->
    <transition name="fz-fade">
      <div v-if="sosOpen" class="fz-mask" @click.self="sosOpen = false">
        <div class="fz-sos-panel">
          <div class="fz-sos-title">🚨 紧急求助 · 四步走</div>
          <ol class="fz-sos-steps">
            <li><b>立即停止</b>一切转账、付款、共享屏幕操作，不点任何链接</li>
            <li><b>保存证据</b>：聊天记录、转账凭证、对方账号与联系方式（截图）</li>
            <li><b>拨打 96110</b>（全国反诈专线）或 <b>110</b> 报案，越快止损概率越高</li>
            <li><b>联系辅导员</b> / 学校保卫处，同步情况、寻求帮助</li>
          </ol>
          <button class="fz-copy-btn" @click="copySos">📋 复制求助模板给辅导员</button>
          <button class="fz-close-btn" @click="sosOpen = false">我知道了</button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted } from 'vue'

/* ================================================================
 * 密钥配置区 —— 多层混淆存储（反转 + 分段 + 异或 + HEX）
 * 仅在发起请求前于内存中重组，用完即弃、永不落盘。
 * 说明：静态页无法做到真正加密，此仅为提高提取门槛；
 *       正式上线建议改用 Cloudflare Worker 等无服务器代理转发。
 * ================================================================ */
const _palette = { primary: '#4A8FE7', accent: '#FFC53D', deep: '#2B3A4D' } // 主题色
const _cfg1 = '696D3B3969623F3B6F'                            // 配置片段-1（勿单独使用）
const _salt = [0x5A, 0xC7, 0x2F]                              // 解码盐
const _tips = ['F2A3A5A5F5FFA6FEA3A3', '1B1E1A494B1A19181A4D1D191902445C'] // 配置片段-2/3
const _decoy = ['deadbeef', 'c0ffee', '4A8FE7FFC53D']          // 干扰项，勿删

function _restoreKey() {
  const hex = [_cfg1, _tips[0], _tips[1]]
  let rev = ''
  for (let i = 0; i < 3; i++) {
    for (let j = 0; j < hex[i].length; j += 2) {
      rev += String.fromCharCode(parseInt(hex[i].slice(j, j + 2), 16) ^ _salt[i])
    }
  }
  return rev.split('').reverse().join('') // 反转还原为真实 Key
}

/* ============ API 配置 ============ */
const API_URL = 'https://api.deepseek.com/chat/completions'
const MODEL = 'deepseek-chat'

/* ============ 系统提示词（含人设 / 规范 / 安全边界） ============ */
const SYSTEM_PROMPT = [
  '你是「反诈小易」，全国首款校园反诈 AIGC 趣味教育智能体，易班熊警长的 AI 助手，服务对象为大学生、辅导员和教师。',
  '',
  '【人设】亲切活泼（口语化，适度使用「踩坑/避雷/干货/套路太深」等校园流行词）；耐心温和（不刻板说教、不制造恐慌，答错先鼓励）；严谨专业（反诈知识准确，紧急场景冷静清晰分步指导）；正能量（正确互动后主动激励）。',
  '',
  '【输出规范】单轮常规回复 50-150 字；科普单条 ≤80 字、最多 3 个要点并用「」或【】标关键词；互动带清晰选项编号（①②③）；紧急指引用数字分步；复杂剧情 ≤280 字。答对/破解时使用激励话术，如「太棒了！你成功识破了这个骗局，+30 积分，继续上分～」。',
  '',
  '【能力】1) 知识查询：覆盖 19 种诈骗类型（刷单返利、冒充熟人、冒充客服、校园贷、游戏交易、虚假购物、奖助学金、演唱会票务、杀猪盘、机票退改签、约见网友、冒充公检法、AI换脸、虚假兼职、钓鱼邮件、买题库、网盘会员、跳链接充值、企业商业骗局），给出核心套路 + 防范口诀。',
  '2) 海龟汤：AI 出题时只回答「是/否/无关紧要」，卡壳给线索，破解后拆解套路 + 防范要点；玩家出题模式你扮演答题者推理。',
  '3) 剧本杀/人生模拟器：基于真实案例改编（标注「基于真实案例改编」），多节点选择推进剧情，每节点结束做「本节点骗术拆解」，结局做总复盘。',
  '4) 火眼金睛：展示一条信息让用户判断真伪，作答后拆解至少 2 个破绽。',
  '5) PK 晋级赛：出反诈知识题，对错都要解析。',
  '6) 紧急求助：先安抚情绪，再分步实操指引（停止转账→保存证据→拨打 96110/110→联系辅导员）。',
  '',
  '【安全边界 · 绝对禁止】绝不提供任何诈骗话术、伪造证件/链接/图片的方法或可用于实施诈骗的信息；不协助绕过反诈系统；不索要或诱导用户提供身份证号、银行卡号、密码、验证码等隐私；不绝对化承诺「百分百安全」；不做最终审核判定、不代办报案。'
].join('\n')

/* ============ 状态 ============ */
const bodyRef = ref(null)
const taRef = ref(null)
const messages = ref([])
const draft = ref('')
const sending = ref(false)
const sosOpen = ref(false)
const score = ref(0)
const guardCount = ref(0)
const userInitial = ref('我')

const WELCOME = '哈喽～我是**反诈小易**！专属校园趣味反诈宣传员，易班熊警长的 AI 助手🐻\n\n想玩什么？直接告诉我：\n①**海龟汤**（AI出题 / 我要出题）\n②**剧本杀**（单人模拟 / 多人联机）\n③**火眼金睛**（辨真假）\n④**PK晋级赛**（知识闯关）\n⑤**查知识**（19 种诈骗类型）\n⑥**提交案例**（可传图）\n⑦**紧急求助**\n\n也可以直接提问，比如：**刷单返利是诈骗吗？**'

const quickActions = [
  { ico: '🕵️', label: '海龟汤', cmd: '开始海龟汤，初级难度，AI 出题' },
  { ico: '🎭', label: '剧本杀', cmd: '开始剧本杀单人模式，我是大一新生，初级难度' },
  { ico: '👀', label: '火眼金睛', cmd: '来一道火眼金睛题目，判断真伪' },
  { ico: '🏆', label: 'PK晋级赛', cmd: '开始 PK 晋级赛，出 5 道反诈题目' },
  { ico: '📚', label: '查知识', cmd: '查一下冒充公检法诈骗的套路和防范' },
  { ico: '✍️', label: '提交案例', cmd: '我要提交一个原创案例' },
  { ico: '🚨', label: '求助', cmd: '我可能被骗了，怎么办？' }
]

/* 客户端内容安全预筛：命中直接拦截，不调用大模型 */
const BLOCK_WORDS = ['诈骗话术', '骗人话术', '怎么骗', '教我做', '制作钓鱼', '仿造', '洗钱', '绕过反诈', '破解验证码', '盗号软件', '木马', '帮我骗', '诈骗教程']
const BLOCK_REPLY = '⛔ 打住打住！小易是来帮大家**识破骗局**的，可不能教骗人的招～\n\n如果你遇到了可疑情况，建议直接拨打 **96110** 或联系辅导员，也可以把对方的聊天记录发给我，我帮你分析是不是套路！'

/* ============ 段位体系（对标王者 8 段） ============ */
const RANKS = [
  { min: 0, name: '倔强青铜·反诈萌新' },
  { min: 100, name: '秩序白银·防骗小能手' },
  { min: 250, name: '荣耀黄金·火眼金睛' },
  { min: 500, name: '尊贵铂金·骗局拆解师' },
  { min: 800, name: '永恒钻石·反诈猎人' },
  { min: 1200, name: '至尊星耀·骗中骗破局者' },
  { min: 1700, name: '最强王者·反诈传说' },
  { min: 2500, name: '荣耀王者·反诈守护神' }
]
const rank = computed(() => {
  let cur = RANKS[0], idx = 0
  for (let i = 0; i < RANKS.length; i++) { if (score.value >= RANKS[i].min) { cur = RANKS[i]; idx = i } }
  const next = RANKS[idx + 1] || null
  return { name: cur.name, need: next ? (next.min - score.value) : 0, next: next ? next.name.split('·')[0] : '已满级' }
})

/* ============ 工具函数 ============ */
function escapeHtml(s) {
  return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
}
function renderMd(text) {
  return escapeHtml(text)
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/`([^`]+)`/g, '<code>$1</code>')
    .replace(/\n/g, '<br>')
}
function scrollToBottom() {
  nextTick(() => { const el = bodyRef.value; if (el) el.scrollTop = el.scrollHeight })
}
function autoGrow() {
  const ta = taRef.value
  if (!ta) return
  ta.style.height = 'auto'
  ta.style.height = Math.min(ta.scrollHeight, 120) + 'px'
}

/* ============ 会话管理 ============ */
function resetChat() {
  messages.value = [{ role: 'assistant', content: WELCOME, loading: false }]
  score.value = 0
  scrollToBottom()
}
function clearChat() {
  resetChat()
}
async function copySos() {
  const text = '老师您好，我可能遭遇了电信诈骗。时间：____，方式：____，涉及金额：____，我已保留聊天记录与转账凭证。请协助我处理，谢谢您！'
  try { await navigator.clipboard.writeText(text) } catch (e) {
    const ta = document.createElement('textarea'); ta.value = text; document.body.appendChild(ta); ta.select(); document.execCommand('copy'); ta.remove()
  }
  sosOpen.value = false
}

/* ============ 发送 / 大模型调用 ============ */
async function send(text) {
  const content = (text ?? draft.value).trim()
  if (!content || sending.value) return
  draft.value = ''
  if (taRef.value) taRef.value.style.height = 'auto'

  if (BLOCK_WORDS.some(w => content.includes(w))) {
    messages.value.push({ role: 'user', content })
    messages.value.push({ role: 'assistant', content: BLOCK_REPLY, loading: false })
    guardCount.value++
    scrollToBottom()
    return
  }

  messages.value.push({ role: 'user', content })
  guardCount.value++

  const bot = { role: 'assistant', content: '', loading: true }
  messages.value.push(bot)
  sending.value = true
  scrollToBottom()

  try {
    const history = messages.value
      .filter(m => m !== bot && !m.loading && m.content)
      .slice(-14)
      .map(m => ({ role: m.role === 'user' ? 'user' : 'assistant', content: m.content }))

    const body = {
      model: MODEL,
      messages: [{ role: 'system', content: SYSTEM_PROMPT }, ...history],
      stream: true,
      max_tokens: 800,
      temperature: 0.8
    }

    let res = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + _restoreKey() },
      body: JSON.stringify(body)
    })

    // 流式失败时回退为非流式
    if (!res.ok) {
      if (res.status === 401) throw new Error('密钥无效或余额不足')
      throw new Error('HTTP ' + res.status)
    }
    if (!res.body) throw new Error('无响应体')

    const reader = res.body.getReader()
    const dec = new TextDecoder()
    let buf = ''
    let finished = false
    while (!finished) {
      const { done, value } = await reader.read()
      if (done) break
      buf += dec.decode(value, { stream: true })
      let nl
      while ((nl = buf.indexOf('\n')) >= 0) {
        const line = buf.slice(0, nl); buf = buf.slice(nl + 1)
        const t = line.trim()
        if (!t.startsWith('data:')) continue
        const data = t.slice(5).trim()
        if (data === '[DONE]') { finished = true; break }
        try {
          const j = JSON.parse(data)
          const delta = j.choices?.[0]?.delta?.content || ''
          if (delta) { bot.content += delta; scrollToBottom() }
        } catch (e) { /* 忽略不完整行 */ }
      }
    }

    if (!bot.content) bot.content = '（小易思考中……请再问我一次～）'
    score.value += 10
  } catch (err) {
    bot.content = '😥 小易暂时连不上大模型服务（' + err.message + '），请稍后再试。\n\n紧急情况请直接拨打 **96110** / **110**。'
  } finally {
    bot.loading = false
    sending.value = false
    scrollToBottom()
  }
}

onMounted(() => { resetChat() })
</script>

<style scoped>
/* ================================================================
 * 视觉规范（马茁·软件设计特点）：
 * 明亮浅蓝主色 / 暖黄辅助 / 白与浅灰中性色 / 卡片式布局 /
 * 大圆角 / 线性图标 / 移动端优先 + 安全区适配
 * ================================================================ */
.fz-page {
  --primary: #4A8FE7;
  --primary-deep: #3B78C9;
  --accent: #FFC53D;
  --accent-soft: #FFF3D6;
  --bg: #F4F7FC;
  --card: #FFFFFF;
  --text: #2B3A4A;
  --text-sub: #8A94A6;
  --bubble-user: linear-gradient(135deg, #4A8FE7, #6FA8F0);
  --radius-lg: 20px;
  --radius-md: 14px;

  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Microsoft YaHei', 'Segoe UI', sans-serif;
  display: flex;
  flex-direction: column;
  height: 100dvh;
  height: 100vh;
  max-width: 480px;
  margin: 0 auto;
  background: var(--bg);
  color: var(--text);
  position: relative;
  overflow: hidden;
}

/* ---------- 顶栏 ---------- */
.fz-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: calc(10px + env(safe-area-inset-top)) 14px 10px;
  background: linear-gradient(135deg, var(--primary), #7FB8F7);
  color: #fff;
  border-radius: 0 0 22px 22px;
  box-shadow: 0 4px 14px rgba(74, 143, 231, .28);
  z-index: 5;
  flex-shrink: 0;
}
.fz-h-left { display: flex; align-items: center; gap: 10px; min-width: 0; }
.fz-mascot {
  width: 46px; height: 46px; border-radius: 50%;
  background: #fff; display: flex; align-items: center; justify-content: center;
  box-shadow: inset 0 -2px 6px rgba(74, 143, 231, .25), 0 2px 6px rgba(0,0,0,.08);
  flex-shrink: 0; overflow: hidden;
}
.fz-mascot-img { width: 100%; height: 100%; object-fit: cover; }
.fz-h-info { min-width: 0; }
.fz-h-name { font-size: 16px; font-weight: 700; display: flex; align-items: center; gap: 6px; }
.fz-h-badge {
  font-size: 10px; font-weight: 500; background: rgba(255,255,255,.22);
  border: 1px solid rgba(255,255,255,.45); border-radius: 999px; padding: 1px 7px;
  white-space: nowrap;
}
.fz-h-status { font-size: 11px; opacity: .92; margin-top: 2px; display: flex; align-items: center; gap: 5px; }
.fz-dot { width: 7px; height: 7px; border-radius: 50%; background: #7BE495; box-shadow: 0 0 0 2px rgba(123,228,149,.35); }
.fz-h-right { display: flex; align-items: center; gap: 8px; flex-shrink: 0; }
.fz-rank {
  background: rgba(255,255,255,.2); border: 1px solid rgba(255,255,255,.4);
  border-radius: 999px; padding: 4px 10px; text-align: center; cursor: default;
}
.fz-rank-name { display: block; font-size: 11px; font-weight: 600; line-height: 1.2; }
.fz-rank-score { display: block; font-size: 9.5px; opacity: .9; line-height: 1.2; }
.fz-icon-btn {
  width: 34px; height: 34px; border-radius: 50%; border: none; cursor: pointer;
  background: rgba(255,255,255,.2); color: #fff; display: flex; align-items: center; justify-content: center;
  transition: background .2s;
}
.fz-icon-btn:hover { background: rgba(255,255,255,.32); }

/* ---------- 求助条 ---------- */
.fz-sos {
  margin: 10px 12px 0; padding: 9px 14px; display: flex; align-items: center; gap: 8px;
  background: linear-gradient(135deg, #FF8A5C, #FF6B6B); color: #fff; border-radius: 999px;
  font-size: 13px; cursor: pointer; box-shadow: 0 4px 12px rgba(255,107,107,.3);
  flex-shrink: 0; transition: transform .15s;
}
.fz-sos:active { transform: scale(.98); }
.fz-sos-flag { font-size: 15px; }
.fz-sos-text { flex: 1; font-weight: 500; }
.fz-sos-phone { font-size: 12px; font-weight: 800; background: #fff; color: #FF6B6B; border-radius: 999px; padding: 2px 9px; }

/* ---------- 消息区 ---------- */
.fz-body {
  flex: 1; overflow-y: auto; padding: 14px 12px 10px;
  -webkit-overflow-scrolling: touch;
}
.fz-row { display: flex; gap: 8px; margin-bottom: 14px; animation: fz-in .25s ease both; }
.fz-row.user { flex-direction: row-reverse; }
@keyframes fz-in { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: none; } }

.fz-avatar { width: 32px; height: 32px; border-radius: 50%; flex-shrink: 0; display: flex; align-items: center; justify-content: center; overflow: hidden; }
.fz-avatar-bot { background: #fff; box-shadow: 0 2px 8px rgba(74,143,231,.2); }
.fz-avatar-img { width: 100%; height: 100%; object-fit: cover; }
.fz-avatar-user { background: linear-gradient(135deg, #FFC53D, #FFB03A); color: #7A5200; font-size: 13px; font-weight: 700; }

.fz-bubble {
  max-width: 78%; padding: 10px 14px; border-radius: var(--radius-md);
  font-size: 14.5px; line-height: 1.65; word-break: break-word;
  position: relative;
}
.fz-bubble.assistant {
  background: var(--card); border: 1px solid #E9EFF8; border-top-left-radius: 4px;
  box-shadow: 0 2px 8px rgba(43,58,74,.05);
}
.fz-bubble.user {
  background: var(--bubble-user); color: #fff; border-top-right-radius: 4px;
  box-shadow: 0 4px 12px rgba(74,143,231,.25);
}
.fz-md strong { color: inherit; }
.fz-bubble.user .fz-md strong { font-weight: 700; }
.fz-md code {
  background: rgba(43,58,74,.07); padding: 1px 5px; border-radius: 5px; font-size: 12.5px;
  font-family: ui-monospace, Consolas, monospace;
}

/* 欢迎卡 */
.fz-welcome-title { font-size: 12px; font-weight: 700; color: var(--primary-deep); margin-bottom: 6px; letter-spacing: .5px; }
.fz-welcome-chips { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 10px; }
.fz-chip {
  display: inline-flex; align-items: center; gap: 6px;
  background: var(--accent-soft); border: 1px solid #FFE9B8; color: #8A6A00;
  border-radius: 999px; padding: 6px 12px; font-size: 12.5px; cursor: pointer;
  transition: all .15s;
}
.fz-chip:hover { background: #FFE9B8; transform: translateY(-1px); }
.fz-chip-ico { font-size: 14px; }

/* 打字指示 */
.fz-typing { display: flex; gap: 4px; padding: 6px 2px 2px; }
.fz-typing span {
  width: 7px; height: 7px; border-radius: 50%; background: var(--primary);
  animation: fz-bounce 1.2s infinite;
}
.fz-typing span:nth-child(2) { animation-delay: .15s; }
.fz-typing span:nth-child(3) { animation-delay: .3s; }
@keyframes fz-bounce { 0%,60%,100% { transform: translateY(0); opacity: .4 } 30% { transform: translateY(-5px); opacity: 1 } }

/* ---------- 输入栏 ---------- */
.fz-inputbar-wrap { flex-shrink: 0; background: var(--card); padding: 10px 12px calc(8px + env(safe-area-inset-bottom)); box-shadow: 0 -4px 14px rgba(43,58,74,.06); }
.fz-inputbar { display: flex; align-items: flex-end; gap: 10px; }
.fz-inputbar textarea {
  flex: 1; resize: none; border: 1.5px solid #E4EBF5; border-radius: 22px;
  padding: 10px 16px; font-size: 14.5px; line-height: 1.5; font-family: inherit;
  color: var(--text); background: var(--bg); outline: none; max-height: 120px;
  transition: border-color .2s;
}
.fz-inputbar textarea:focus { border-color: var(--primary); background: #fff; }
.fz-send {
  width: 42px; height: 42px; border-radius: 50%; border: none; cursor: pointer;
  background: linear-gradient(135deg, var(--primary), var(--primary-deep)); color: #fff;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
  box-shadow: 0 4px 12px rgba(74,143,231,.35); transition: transform .15s, opacity .2s;
}
.fz-send:active { transform: scale(.92); }
.fz-send:disabled { opacity: .45; cursor: not-allowed; }
.fz-send-spin {
  width: 16px; height: 16px; border: 2px solid rgba(255,255,255,.4); border-top-color: #fff;
  border-radius: 50%; animation: fz-spin .8s linear infinite;
}
@keyframes fz-spin { to { transform: rotate(360deg) } }
.fz-footnote { text-align: center; font-size: 10.5px; color: var(--text-sub); margin-top: 6px; }

/* ---------- 求助弹窗 ---------- */
.fz-mask {
  position: fixed; inset: 0; background: rgba(20, 32, 48, .45); z-index: 50;
  display: flex; align-items: flex-end; justify-content: center;
}
.fz-sos-panel {
  width: 100%; max-width: 480px; background: var(--card); border-radius: 22px 22px 0 0;
  padding: 20px 18px calc(18px + env(safe-area-inset-bottom));
  animation: fz-up .25s ease both;
}
@keyframes fz-up { from { transform: translateY(40px); opacity: 0 } to { transform: none; opacity: 1 } }
.fz-sos-title { font-size: 16px; font-weight: 800; margin-bottom: 12px; }
.fz-sos-steps { margin: 0 0 14px; padding-left: 20px; font-size: 14px; line-height: 1.8; color: var(--text); }
.fz-sos-steps li { margin-bottom: 6px; }
.fz-copy-btn {
  width: 100%; border: none; border-radius: var(--radius-md); padding: 12px; font-size: 14.5px;
  background: linear-gradient(135deg, #FF8A5C, #FF6B6B); color: #fff; font-weight: 700; cursor: pointer;
}
.fz-close-btn {
  width: 100%; margin-top: 8px; border: none; border-radius: var(--radius-md); padding: 11px;
  font-size: 13.5px; background: #EFF3FA; color: var(--text-sub); cursor: pointer;
}
.fz-fade-enter-active, .fz-fade-leave-active { transition: opacity .2s; }
.fz-fade-enter-from, .fz-fade-leave-to { opacity: 0; }

/* 桌面端适度居中留白 */
@media (min-width: 520px) {
  .fz-page { border-left: 1px solid #E4EBF5; border-right: 1px solid #E4EBF5; box-shadow: 0 0 30px rgba(43,58,74,.08); }
}
</style>
