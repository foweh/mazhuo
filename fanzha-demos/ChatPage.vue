<template>
  <div class="fz-page">

    <!-- ======= 顶栏 ======= -->
    <header class="fz-header">
      <div class="fz-h-left">
        <div class="fz-mascot"><img class="fz-mascot-img" :src="avatarSrc" alt="反诈小易" /></div>
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
        <div v-if="m.role === 'assistant'" class="fz-avatar fz-avatar-bot"><img class="fz-avatar-img" :src="avatarSrc" alt="反诈小易" /></div>

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

/* 头像图片：动态绑定相对路径，避免 SFC 编译器把静态资源改写为模块导入 */
const avatarSrc = './avatar.jpg'

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

/* ============ 演示自动播放（仅本地录屏用，不推送） ============ */
/* 为什么：录屏要稳定可复现，所以不走真实大模型，改为脚本化逐字打印 + 平滑滚动 */

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms))

/* ============ 演示脚本集（本地录屏用）============
 * 由 demo-*.html 里的 window.__DEMO__ 指定播放哪一段；
 * 正常打开 index.html（未指定）时不自动播放。 */
const DEMOS = {
  /* ① 海龟汤 */
  haiguitang: [
    { role: 'user', text: '来个海龟汤，要反诈主题的' },
    { role: 'assistant', text: '🕵️ **海龟汤 · 汤面**\n\n凌晨两点，室友发现小林还坐在床上对着手机傻笑。第二天，他旷了课，卡里的 3 万块只剩 21 块。可他逢人就说：「我交到了一个真心朋友。」\n\n到底发生了什么？尽管问，我只回答 **是 / 否 / 无关紧要**。' },
    { role: 'user', text: '小林的钱，是被骗走的吗？' },
    { role: 'assistant', text: '**是**。' },
    { role: 'user', text: '骗他的人，他见过面吗？' },
    { role: 'assistant', text: '**否**。' },
    { role: 'user', text: '对方是假装和他谈恋爱吗？' },
    { role: 'assistant', text: '**是**——你已经摸到关键了。' },
    { role: 'user', text: '钱是他自己主动转出去的吗？' },
    { role: 'assistant', text: '**是**。' },
    { role: 'user', text: '那个「真心朋友」是不是带他「投资赚钱」了？' },
    { role: 'assistant', text: '**是**。线索够了，试着把整个故事讲出来？' },
    { role: 'user', text: '我猜是杀猪盘：骗子装成网恋对象，先嘘寒问暖陪他聊天，再哄他去一个平台投钱，钱一到手就拉黑' },
    { role: 'assistant', text: '🎉 **恭喜破解！**\n\n**汤底**：这就是典型的「杀猪盘」。骗子盗用帅哥美女的照片，伪装成网友和你「网恋」，每天早安晚安、嘘寒问暖（先「养猪」）；等你放下戒备，就以「我有内部渠道、稳赚不赔」哄你在假平台投钱；你尝到小额甜头、不断加码，直到想提现时，对方早已拉黑消失（再「杀猪」）。\n\n**套路拆解**：\n① 只网聊，拒绝视频、拒绝见面\n② 主动晒收益截图，制造「真的能赚」的假象\n③ 平台数字随便改，提现总有「手续费 / 保证金」\n\n**防范要点**：网上可以谈感情，**一开口谈钱就拉黑**；凡是「稳赚不赔」的投资，都是诈骗。' }
  ],

  /* ② 剧本杀（单人版 + 多人版，一场都玩） */
  jubensha: [
    { role: 'user', text: '单人版剧本杀' },
    { role: 'assistant', text: '好嘞！单人版剧本杀开局～本次剧情基于真实案例改编。\n\n【剧情·第一幕：天降offer】\n开学第二周，你收到陌生短信：\n「同学你好，兼职打字员，日结150元，无需经验，加QQ：xxxx聊。」\n\n你最近正好想赚点生活费，犹豫了一下……\n你会怎么做？\n①加QQ看看，反正只是聊天不亏\n②直接删掉，来源不明的兼职不理\n③先问问对方要营业执照和公司信息\n\n回复编号，剧情继续～' },
    { role: 'user', text: '1' },
    { role: 'assistant', text: '【选择①·加QQ看看】\n你加上了对方。对方很热情，发来一张兼职群截图，群里全是「已到账」的收款图。\n他说：「先做个小任务试试，帮我给店铺点个赞，5元马上到账。」\n\n你点赞后，真的收到了5元红包。\n\n对方趁热打铁：「想赚更多吗？下载这个APP，做垫付单，一单返20%。」\n\n💡本节点骗术拆解：\n【先手给小钱】先小额返利建立信任，用小额返利建立你的信心。\n【群里晒截图】大多为托儿伪造返利截图。\n【一旦垫付】就是刷单返利骗局的开场！\n\n接下来你会？\n①下载APP试试，反正前面真给钱了\n②警惕起来，退群+拉黑+举报\n③先只做「不垫付」的任务，看看情况' },
    { role: 'user', text: '3' },
    { role: 'assistant', text: '【选择③·只做不垫付的任务】\n你回复：「我就做点赞这种，垫付的不做。」\n\n对方立刻换了话术：「那有个订单，先垫付50返65，做完立刻退，稳赚的！」你坚持拒绝。\n\n对方态度转冷：「不做拉倒，那你把刚才那5元退回来，不然我举报你账号。」还发来一张「你的信息已被记录」的截图威胁你。\n\n你有点慌，但没转账。\n\n💡本节点骗术拆解：\n拒绝垫付后，骗子会升级施压：威胁、恐吓。\n【退回5元】是借口，目的是让你主动转账。\n【信息已记录】截图多为PS伪造，制造恐惧。\n\n接下来你会？\n①把5块退回去，息事宁人\n②直接拉黑，不理会威胁\n③截图保存证据，报警/举报' },
    { role: 'user', text: '3' },
    { role: 'assistant', text: '【选择③·截图保存证据，报警/举报】\n你冷静截图了全部聊天记录、对方账号、那张伪造的威胁图，然后拉黑，并在「国家反诈中心APP」和平台举报。\n\n对方发现被拉黑，又换小号加你，你没理会。三天后，那个QQ号被封。\n\n你保住了钱包，还顺手帮平台清理了一个骗子账号。\n\n✅【结局：反诈达人】\n📌总复盘：\n骗局核心：刷单返利=100%诈骗，前期小利是诱饵。\n关键转折：你两次拒绝垫付，并及时保存证据，没有落入恐吓陷阱。\n✅标准操作：不转账+截图+拉黑+举报\n\n防范口诀：\n大钱，垫付都是骗局，小钱返利是诱饵；\n天上馅饼别慌张，截图举报最稳当。\n\n🎉太棒了！你成功识破了这个骗局，+30积分，继续上分✨\n还想玩吗？\n①换个剧本（冒充客服 / 校园贷 / 杀猪盘）\n②玩海龟汤\n③火眼金睛辨真假' },
    { role: 'user', text: '多人剧本杀' },
    { role: 'assistant', text: '好嘞～多人版剧本杀来啦！🐣\n\n【玩法·多人剧本杀】\n你们是同一个宿舍/班级的反诈小分队。\n剧情推进到关键节点时，每个人各自做出选择，我会汇总大家的决策，看看团队能不能一起避开骗局。\n\n先选个剧本主题（回复编号）：\n①【冒充客服】「你的快递丢了，高额赔付」——真实案例改编，考验信息核实\n②【校园贷】「无抵押、秒到账，学生专享」——揭秘套路贷陷阱\n③【杀猪盘】「网恋对象带你投资稳赚」——情感+金钱双重套路\n④【演唱会票务】「内部渠道，低价出票」——追星必防\n⑤【游戏交易】「高价收账号，走平台担保」——游戏党高频踩坑\n\n另外告诉我：\n你们一共有几个人一起玩？（2-6人都行）\n想让我怎么推进？（每人轮流发言/给出节点大家投票/分角色扮演）\n\n选好主题+人数+玩法，咱们就开演！🐻' },
    { role: 'user', text: '四个人，分角色，主题一' },
    { role: 'assistant', text: '好嘞！四人分队 + 分角色 +【冒充客服】主题，开演！\n\n【背景 · 基于真实案例改编】\n宿舍四人网购后，其中一人的快递突然显示「丢失」。不久，一个自称「XX快递客服」的电话打进来，说可以三倍赔付。\n\n你们四个角色：\n①阿泽 —— 刚下单新机，最关心赔付\n②小雨 —— 细心谨慎，爱查证\n③大鹏 —— 急性子，想赶紧拿到钱\n④小易 —— 宿舍长，负责协调\n\n【第一幕 · 来电】\n「客服」来电：「您的快递在运输中丢失，我们可三倍赔付共900元，请添加理赔专员微信、点击链接填写银行卡信息，我们马上打款。」\n\n现在请四位分别选择（各自回复编号）：\n①立刻加微信，按对方说的填银行卡\n②先挂电话，自己打官方客服电话核实\n③加微信，但不填信息，先看对方怎么说\n④直接在购物APP里查物流和官方客服宣传页\n\n每人回复一个编号，我汇总后推进剧情~' },
    { role: 'user', text: '4' },
    { role: 'assistant', text: '收到！小易（宿舍长）选了④：\n✅在购物APP里查物流和官方客服\n\n阿泽、小雨、大鹏三位还没选择！请选择！\n\n当前进度：\n①阿泽 —— 待选\n②小雨 —— 待选\n③大鹏 —— 待选\n④小易 —— ✔已在APP里查物流\n\n小易的操作结果：\n他在APP里看到物流显示【派送中，未丢失】，而且APP内的官方客服明确写着：\n【不会主动来电要求提供银行卡信息】。\n\n请阿泽、小雨、大鹏分别回复①②③④你的选择，咱们继续推进🐻' },
    { role: 'user', text: '①' },
    { role: 'assistant', text: '【阿泽选了①，点击微信，按对方说的填银行卡】\n阿泽这边已经点开链接，页面长得跟银行官网几乎一模一样，要求填：姓名、身份证号、银行卡号、短信验证码。\n他刚填到「验证码」这一步……\n\n💡本节点骗术拆解：\n「三倍赔付」是诱饵，先给甜头降低你的警惕。\n👉链接是钓鱼网站，仿银行界面。\n👉一旦填写验证码，卡里的钱会瞬间被转走！骗子不会真的赔付。\n\n现在宿舍其他人能不能把阿泽一把拦住！\n小雨、大鹏，做出反应（回复编号）\n①赶紧去抢下手机，让他别填\n②稳住语气提醒，不确定有诈\n③直接报警，96110，不对，现在\n④用手机录下全过程，留证\n\n等待小雨、大鹏的选择，以及阿泽是否填写验证码（可自选）🐻' },
    { role: 'user', text: '①' },
    { role: 'assistant', text: '【小雨选了①，冲过去抢下手机！】\n小雨一把按住阿泽的手：「验证码绝对不能填！赔付不会要这个！别填！」\n阿泽愣住，停在屏幕上。\n\n💡本节点拆解：\n验证码一旦泄露，钱就已经可以被转走。\n✅小雨的关键动作：物理阻断！\n\n✅一段小彩蛋（可选择）\n①海龟汤（AI出题 / 我要出题）\n②剧本杀（单人继续 / 多人继续）\n③查知识（可跳转反诈百科）' }
  ],

  /* ③ 火眼金睛（4 关连闯） */
  huoyanjinjing: [
    { role: 'user', text: '火眼金睛' },
    { role: 'assistant', text: '🔍火眼金睛 · 第1关\n请判断下面这条信息是真还是假：\n\n【校园通知】同学你好，因系统升级，你的奖学金需重新登记银行卡信息。请点击链接填写卡号、密码及短信验证码，逾期视为放弃。\n链接：http://scholar-edu.top/login\n\n请选：\n①真的，赶紧填\n②假的，是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '1' },
    { role: 'assistant', text: '别急，先给你个抱抱～答错很正常，这题套路确实深！😊\n正确答案是②：假的，是诈骗！\n\n破绽拆解：\n【破绽1】域名不对劲：正规学校通知一般走官网/官方APP，scholar-edu.top这种拼凑域名很可疑。\n【破绽2】索要敏感信息：奖学金登记绝不会要你的银行卡密码和验证码，这是铁律！\n【破绽3】制造紧迫感：「逾期视为放弃」是骗子常用的施压话术。\n\n防范口诀：奖学金走官方，密码验证码谁要都不给！\n\n再来一关试试？回复「继续」' },
    { role: 'user', text: '继续' },
    { role: 'assistant', text: '🔍火眼金睛 · 第2关\n请判断下面这条信息是真还是假：\n\n亲爱的用户，您有一笔快递理赔待领取。因包裹丢失，点击下方链接申请退款，填写银行卡号即可秒到账～\n客服电话：400-xxx-8888\n\n请选：\n①真的，赶紧申请\n②假的，是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '1' },
    { role: 'assistant', text: '没关系~这一关比第一关更会伪装，答错说明它在「装正规」！😊\n正确答案是②：假的，是诈骗！\n\n破绽拆解：\n【破绽1】主动理赔一反常：正规快递丢件，是你联系官方客服，而不是陌生人主动找你退款。\n【破绽2】索要银行卡号：理赔走官方平台原路退回，不用你填卡号，更不会「秒到账」。\n【破绽3】400电话可伪造：400号码骗子也能申请，别看到400就信。\n\n防范口诀：丢件理赔找官方，不明链接不点它！\n\n你已经连闯2关啦，回复「继续」挑战第3关🐣' },
    { role: 'user', text: '继续' },
    { role: 'assistant', text: '🔍火眼金睛 · 第3关\n请判断下面这条信息是真还是假：\n\n学长急用！我微信被限额了，你先借我2000转到我朋友的卡上，明天我就还你，别打电话，我这边不方便~\n\n请选：\n①真的，学长开口帮一把\n②假的，可能是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '1' },
    { role: 'assistant', text: '理解你～「学长」两个字确实容易让人放松警惕，这正是骗子高明的地方！😊\n正确答案是②：假的，可能是诈骗！\n\n破绽拆解：\n【破绽1】「别打电话」是最大红旗：真朋友借钱不会拒绝核实，这是怕你识破AI换声或盗号。\n【破绽2】转账到「朋友的卡」：借钱却转给第三方账户，是典型的洗钱/转移赃款套路。\n【破绽3】制造紧迫感：「急用、明天就还」逼你来不及多想。\n\n防范口诀：熟人借钱先核实，打个电话/视频最保险！\n\n🎉连闯3关！反诈雷达已上线！想换玩法回复「海龟汤」或者「剧本杀」' },
    { role: 'user', text: '继续' },
    { role: 'assistant', text: '🔍火眼金睛 · 第4关（进阶）\n请判断下面这条信息是真还是假：\n\n【教务处】由于你挂科名额，现开放【内部改分】通道。添加老师QQ：123456，缴纳800元手续费即可修改成绩，名额有限。\n\n请选：\n①真的，能改分值得\n②假的，是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '2' },
    { role: 'assistant', text: '🎉太棒了！你成功识破了这个骗局，+30积分，继续上分✨\n正确答案是②：假的，是诈骗！\n\n破绽拆解：\n【破绽1】「改分」本身违规：高校成绩管理有严格流程，不存在内部改分这种东西。\n【破绽2】加私人QQ：正规教务走官方系统，不会让你加陌生QQ私下交易。\n【破绽3】缴费+名额有限：收钱+紧迫感，双重套路。\n\n防范口诀：成绩只能靠努力，改分收费全是骗！\n\n✅连过4关啦，反应越来越快～继续挑战 **第5关** 吗？回复「继续」' },
    { role: 'user', text: '继续' },
    { role: 'assistant', text: '🔍火眼金睛 · 第5关\n请判断下面这条信息是真还是假：\n\n【校招内推】同学你好，恭喜你通过初筛！请添加「HR」微信，缴纳199元押金锁定内推名额，入职后全额退还。\n\n请选：\n①真的，先交钱锁名额\n②假的，是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '2' },
    { role: 'assistant', text: '🎉太棒了，又识破一个，+30积分✨\n正确答案是②：假的，是诈骗！\n\n破绽拆解：\n【破绽1】正规招聘不收费：任何以「押金 / 培训费 / 内推费」名义收钱的招聘，都是诈骗。\n【破绽2】「名额有限」是施压：制造紧迫感，让你来不及核实。\n【破绽3】加私人微信：正规企业走官方投递渠道，不会私下加微信收钱。\n\n防范口诀：找工作不交钱，交钱的都是骗！\n\n继续挑战 **第6关（终极）**？回复「继续」' },
    { role: 'user', text: '继续' },
    { role: 'assistant', text: '🔍火眼金睛 · 第6关（终极）\n请判断下面这条信息是真还是假：\n\n「我是你辅导员，学校有个助学金需要你配合做个『资金验证』，加我QQ开个屏幕共享，我指导你操作。」\n\n请选：\n①真的，老师让做就做\n②假的，是诈骗\n\n回复编号即可～' },
    { role: 'user', text: '2' },
    { role: 'assistant', text: '🎉满分！你已经是老反诈了，+30积分✨\n正确答案是②：假的，是诈骗！\n\n破绽拆解：\n【破绽1】「屏幕共享」= 把手机交给骗子：一旦共享，验证码、密码全被看光。\n【破绽2】「资金验证」是借口：助学金只会直接打到你卡里，绝不需要你「验证资金」。\n【破绽3】冒充老师：自称老师 / 辅导员却要你转账或开共享，大概率是冒充。\n\n防范口诀：谁要屏幕共享，谁就是骗子！\n\n🏆你已经通关全部 6 关，成功晋级「反诈守护神」！想挑战【PK晋级赛】还是听个【海龟汤】？🐣' }
  ],

  /* ④ PK 晋级赛 */
  pk: [
    { role: 'user', text: '开始 PK 晋级赛，出 5 道反诈题' },
    { role: 'assistant', text: '🏆 **PK 晋级赛 · 反诈知识闯关**\n\n答对一题 +20 分，答错也有解析。开始～\n\n**第 1 题**\n短信说「您的快递丢失，加微信理赔」，正确做法是？\n① 在购物 App 里找官方客服核实\n② 加那个微信，按对方指引操作\n③ 点短信里的链接填写信息\n\n回复序号～' },
    { role: 'user', text: '①' },
    { role: 'assistant', text: '✅ **答对！+20 分**\n\n**解析**：快递理赔一律走**官方 App 内客服**；加微信、点链接、填验证码，都是骗子的入口。\n\n**第 2 题**\n接到自称「公安局」的电话，说你涉嫌洗钱，要你把钱转到「安全账户」清查，应该？\n① 配合转钱证明清白\n② 挂断，打 96110 或去派出所核实\n③ 先加对方微信、发身份证\n\n回复序号～' },
    { role: 'user', text: '②' },
    { role: 'assistant', text: '✅ **答对！+20 分（共 40）**\n\n**解析**：公检法**绝不会**电话办案，也**没有**「安全账户」；凡是要你转钱自证清白的，都是诈骗。\n\n**第 3 题**\n网友聊了半个月，邀你一起玩「内部渠道」的投资平台，说稳赚不赔，应该？\n① 小投一点试试水\n② 先投 100 看能不能提现\n③ 立刻拒绝并拉黑\n\n回复序号～' },
    { role: 'user', text: '③' },
    { role: 'assistant', text: '✅ **答对！+20 分（共 60）**\n\n**解析**：这就是「杀猪盘」。**网恋 + 投资 = 100% 诈骗**；「能提现」是骗子在养猪，等你加大投入就收网。\n\n**第 4 题**\n「辅导员」的 QQ 消息说急用钱，让你先转 500 借他，应该？\n① 直接转，老师不会骗我\n② 打电话给辅导员本人核实\n③ 让他发条语音确认\n\n回复序号～' },
    { role: 'user', text: '②' },
    { role: 'assistant', text: '✅ **答对！+20 分（共 80）**\n\n**解析**：QQ / 微信里的「熟人」可能是**盗号冒充**，语音也能合成；**一定要打电话**核实本人。\n\n**第 5 题（决胜题）**\n找兼职时，对方要你「先垫付 500 做任务，完成后连本带利返还」，应该？\n① 垫钱冲一单\n② 先垫 50 试试\n③ 直接拒绝，要垫付的都是诈骗\n\n回复序号～' },
    { role: 'user', text: '③' },
    { role: 'assistant', text: '🎉 **答对！+20 分 —— 满分 100！**\n\n**解析**：**凡是要垫付的兼职，100% 是诈骗**。刷单返利用小额返现把你「养熟」，再骗你大额。\n\n🏆 **晋级成功：荣耀黄金 · 火眼金睛**\n\n**五题口诀**：\n① 快递理赔走官方 App\n② 公检法不会电话要钱\n③ 网恋 + 投资 = 杀猪盘\n④ 熟人借钱必电话核实\n⑤ 要垫付的兼职都是诈骗\n\n下一关「永恒钻石」等你来战～' }
  ]
}

/* 选择演示：由 demo-*.html 的 window.__DEMO__ 或 URL 上的 ?demo= 指定 */
const demoKey = (typeof window !== 'undefined' && (window.__DEMO__ || new URLSearchParams(location.search).get('demo'))) || ''
const DEMO_SCRIPT = DEMOS[demoKey] || []

let scrollRAF = null

/* 平滑跟随滚动：每帧把滚动条缓缓逼近底部，形成「慢慢往下滑」的效果 */
function demoSmoothScroll() {
  const el = bodyRef.value
  if (el) {
    const target = el.scrollHeight - el.clientHeight
    const diff = target - el.scrollTop
    if (diff > 0.3) el.scrollTop += Math.max(0.4, diff * 0.06)
  }
  scrollRAF = requestAnimationFrame(demoSmoothScroll)
}

/* 逐字打印一条消息；通过数组下标写入，保证 Vue 响应式生效 */
async function demoTypeMessage(role, text, perChar) {
  messages.value.push({ role, content: '', loading: false })
  const idx = messages.value.length - 1
  for (let i = 0; i < text.length; i++) {
    messages.value[idx].content += text[i]
    await sleep(perChar)
  }
}

/* 播放整段演示：用户消息稍快、机器人消息稍慢，句间留停顿 */
async function playDemo() {
  demoSmoothScroll()
  await sleep(800)
  for (const step of DEMO_SCRIPT) {
    await demoTypeMessage(step.role, step.text, step.role === 'user' ? 60 : 48)
    if (step.role === 'user') guardCount.value++
    else score.value += 10
    await sleep(900)
  }
}

onMounted(() => {
  resetChat()
  if (DEMO_SCRIPT.length) playDemo()
})
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
