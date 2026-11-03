<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const STORAGE_KEYS = {
  cart: 'warm-sun-cart-v1',
  orders: 'warm-sun-orders-v1',
  customer: 'warm-sun-customer-v1'
}

const categories = ['全部餐點', '人氣主餐', '飯麵料理', '輕食點心', '飲品甜點']

const menuItems = [
  { id: 1, name: '日式照燒雞腿飯', category: '人氣主餐', price: 150, tag: '人氣推薦', emoji: '🍱', color: 'peach', description: '香嫩雞腿搭配特製照燒醬，附季節時蔬與白飯。', available: true },
  { id: 2, name: '奶油蘑菇義大利麵', category: '飯麵料理', price: 180, tag: '濃郁奶香', emoji: '🍝', color: 'cream', description: '奶油白醬與新鮮蘑菇交織，口感滑順又滿足。', available: true },
  { id: 3, name: '香煎豬排咖哩飯', category: '人氣主餐', price: 160, tag: '店長推薦', emoji: '🍛', color: 'gold', description: '香煎豬排配上溫潤日式咖哩，暖心又暖胃。', available: true },
  { id: 4, name: '田園鮮蔬烤雞沙拉', category: '輕食點心', price: 135, tag: '清爽輕食', emoji: '🥗', color: 'green', description: '新鮮生菜、烤雞與多種蔬菜，搭配特製醬汁。', available: true },
  { id: 5, name: '酥脆黃金薯條', category: '輕食點心', price: 70, tag: '分享點心', emoji: '🍟', color: 'yellow', description: '外酥內鬆的金黃薯條，適合搭配任何主餐。', available: true },
  { id: 6, name: '手作起司蛋糕', category: '飲品甜點', price: 95, tag: '每日甜點', emoji: '🍰', color: 'pink', description: '綿密起司香氣，甜度恰到好處。', available: true },
  { id: 7, name: '經典冰拿鐵', category: '飲品甜點', price: 90, tag: '咖啡人氣款', emoji: '🧋', color: 'brown', description: '濃縮咖啡與香醇鮮奶，冰涼順口。', available: true },
  { id: 8, name: '蜂蜜檸檬氣泡飲', category: '飲品甜點', price: 75, tag: '清新氣泡', emoji: '🍋', color: 'lemon', description: '檸檬酸香加上淡淡蜂蜜，清爽解膩。', available: true },
  { id: 9, name: '香煎鮭魚餐盒', category: '飯麵料理', price: 195, tag: '均衡搭配', emoji: '🍣', color: 'salmon', description: '香煎鮭魚搭配時蔬、玉子燒與熱騰騰白飯。', available: true },
  { id: 10, name: '香蒜雞肉燉飯', category: '飯麵料理', price: 175, tag: '蒜香濃郁', emoji: '🍚', color: 'cream', description: '雞肉與米飯吸收香蒜高湯，濃郁但不膩口。', available: true },
  { id: 11, name: '焦糖布丁', category: '飲品甜點', price: 65, tag: '療癒甜點', emoji: '🍮', color: 'gold', description: '滑嫩布丁搭配微苦焦糖，飯後剛剛好。', available: true },
  { id: 12, name: '熱美式咖啡', category: '飲品甜點', price: 70, tag: '經典咖啡', emoji: '☕', color: 'brown', description: '清爽平衡的咖啡風味，適合午後時光。', available: true }
]

const activePage = ref('menu')
const selectedCategory = ref('全部餐點')
const searchText = ref('')
const sortBy = ref('recommended')
const toastMessage = ref('')
const toastVisible = ref(false)
const showCheckout = ref(false)
const showOrderDetail = ref(false)
const selectedOrder = ref(null)
const mobileMenuOpen = ref(false)

const cart = ref([])
const orders = ref([])
const customer = ref({
  name: '',
  phone: '',
  dining: '外帶',
  address: '',
  payment: '到店付款',
  note: ''
})

const money = (value) => new Intl.NumberFormat('zh-TW', {
  style: 'currency',
  currency: 'TWD',
  maximumFractionDigits: 0
}).format(value)

const filteredMenu = computed(() => {
  let result = [...menuItems]
  if (selectedCategory.value !== '全部餐點') {
    result = result.filter(item => item.category === selectedCategory.value)
  }
  const keyword = searchText.value.trim().toLowerCase()
  if (keyword) {
    result = result.filter(item =>
      item.name.toLowerCase().includes(keyword) ||
      item.description.toLowerCase().includes(keyword) ||
      item.category.toLowerCase().includes(keyword)
    )
  }
  if (sortBy.value === 'price-low') result.sort((a, b) => a.price - b.price)
  if (sortBy.value === 'price-high') result.sort((a, b) => b.price - a.price)
  if (sortBy.value === 'name') result.sort((a, b) => a.name.localeCompare(b.name, 'zh-Hant'))
  return result
})

const cartCount = computed(() => cart.value.reduce((sum, item) => sum + item.quantity, 0))
const subtotal = computed(() => cart.value.reduce((sum, item) => sum + item.price * item.quantity, 0))
const discount = computed(() => subtotal.value >= 500 ? 50 : 0)
const deliveryFee = computed(() => customer.value.dining === '外送' && cart.value.length ? 40 : 0)
const total = computed(() => Math.max(0, subtotal.value - discount.value + deliveryFee.value))
const remainingForDiscount = computed(() => Math.max(0, 500 - subtotal.value))
const isCartEmpty = computed(() => cart.value.length === 0)

function safeParse(key, fallback) {
  try {
    const raw = localStorage.getItem(key)
    return raw ? JSON.parse(raw) : fallback
  } catch {
    return fallback
  }
}

onMounted(() => {
  cart.value = safeParse(STORAGE_KEYS.cart, [])
  orders.value = safeParse(STORAGE_KEYS.orders, [])
  const savedCustomer = safeParse(STORAGE_KEYS.customer, null)
  if (savedCustomer) customer.value = { ...customer.value, ...savedCustomer }
})

watch(cart, value => localStorage.setItem(STORAGE_KEYS.cart, JSON.stringify(value)), { deep: true })
watch(orders, value => localStorage.setItem(STORAGE_KEYS.orders, JSON.stringify(value)), { deep: true })
watch(customer, value => localStorage.setItem(STORAGE_KEYS.customer, JSON.stringify(value)), { deep: true })

function notify(message) {
  toastMessage.value = message
  toastVisible.value = true
  window.clearTimeout(notify.timer)
  notify.timer = window.setTimeout(() => { toastVisible.value = false }, 2600)
}

function scrollToMenu() {
  document.getElementById('menu-section')?.scrollIntoView({ behavior: 'smooth' })
}

function addToCart(item) {
  const existing = cart.value.find(cartItem => cartItem.id === item.id)
  if (existing) existing.quantity += 1
  else cart.value.push({ id: item.id, name: item.name, price: item.price, emoji: item.emoji, quantity: 1 })
  notify(`已將「${item.name}」加入購物車`)
}

function changeQuantity(itemId, amount) {
  const item = cart.value.find(cartItem => cartItem.id === itemId)
  if (!item) return
  item.quantity += amount
  if (item.quantity <= 0) cart.value = cart.value.filter(cartItem => cartItem.id !== itemId)
}

function removeFromCart(itemId) {
  cart.value = cart.value.filter(item => item.id !== itemId)
  notify('已移除餐點')
}

function clearCart() {
  cart.value = []
}

function goTo(page) {
  activePage.value = page
  showCheckout.value = false
  mobileMenuOpen.value = false
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function openCheckout() {
  if (isCartEmpty.value) {
    notify('請先選擇想吃的餐點')
    return
  }
  showCheckout.value = true
  activePage.value = 'cart'
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function submitOrder() {
  if (!customer.value.name.trim()) return notify('請填寫訂購人姓名')
  if (!customer.value.phone.trim()) return notify('請填寫聯絡電話')
  if (!/^09\d{8}$/.test(customer.value.phone.trim())) return notify('請輸入正確的手機號碼（例如 0912345678）')
  if (customer.value.dining === '外送' && !customer.value.address.trim()) return notify('外送訂單請填寫送達地址')
  if (!cart.value.length) return notify('購物車目前沒有餐點')

  const order = {
    id: `WS${Date.now().toString().slice(-8)}`,
    createdAt: new Date().toISOString(),
    status: '已成立',
    customer: { ...customer.value },
    items: cart.value.map(item => ({ ...item })),
    subtotal: subtotal.value,
    discount: discount.value,
    deliveryFee: deliveryFee.value,
    total: total.value
  }
  orders.value.unshift(order)
  selectedOrder.value = order
  clearCart()
  showCheckout.value = false
  showOrderDetail.value = true
  activePage.value = 'orders'
  customer.value.note = ''
  notify('訂單已成立！感謝您的訂購')
}

function viewOrder(order) {
  selectedOrder.value = order
  showOrderDetail.value = true
}

function cancelOrder(order) {
  if (order.status === '已取消') return
  if (!window.confirm(`確定要取消訂單 ${order.id} 嗎？`)) return
  order.status = '已取消'
  notify('訂單已取消')
}

function closeOrderDetail() {
  showOrderDetail.value = false
  selectedOrder.value = null
}

function resetDemoData() {
  if (!window.confirm('確定要清除這個瀏覽器中的購物車、訂單與顧客資料嗎？')) return
  cart.value = []
  orders.value = []
  customer.value = { name: '', phone: '', dining: '外帶', address: '', payment: '到店付款', note: '' }
  localStorage.removeItem(STORAGE_KEYS.cart)
  localStorage.removeItem(STORAGE_KEYS.orders)
  localStorage.removeItem(STORAGE_KEYS.customer)
  notify('示範資料已清除')
}
</script>

<template>
  <div class="app-shell">
    <div class="announcement">
      <span>☀</span> 暖心好食光，滿 $500 現折 $50 <span class="announcement-dot">✦</span> 外送服務費 $40
    </div>

    <header class="site-header">
      <a class="brand" href="#" @click.prevent="goTo('menu')">
        <span class="brand-mark">暖</span>
        <span class="brand-copy">
          <strong>暖日食堂</strong>
          <small>WARM SUN KITCHEN</small>
        </span>
      </a>

      <button class="mobile-toggle" type="button" aria-label="開啟導覽選單" @click="mobileMenuOpen = !mobileMenuOpen">
        {{ mobileMenuOpen ? '✕' : '☰' }}
      </button>

      <nav class="main-nav" :class="{ 'nav-open': mobileMenuOpen }">
        <button :class="{ active: activePage === 'menu' }" @click="goTo('menu')">線上菜單</button>
        <button :class="{ active: activePage === 'cart' }" @click="goTo('cart')">購物車 <span v-if="cartCount" class="nav-count">{{ cartCount }}</span></button>
        <button :class="{ active: activePage === 'orders' }" @click="goTo('orders')">我的訂單</button>
      </nav>

      <button class="header-cart" @click="goTo('cart')">
        <span class="cart-icon">🛍</span>
        <span>購物車</span>
        <span class="cart-badge">{{ cartCount }}</span>
      </button>
    </header>

    <main>
      <template v-if="activePage === 'menu'">
        <section class="hero">
          <div class="hero-content">
            <span class="eyebrow"><span class="eyebrow-line"></span> MADE WITH LOVE, SERVED WITH WARMTH</span>
            <h1>每一餐，都是<br /><em>好好生活</em>的開始。</h1>
            <p>用新鮮食材與溫暖心意，為平凡的日子<br class="desktop-break" />準備一份不平凡的美味。</p>
            <button class="primary-button hero-button" @click="scrollToMenu">
              探索今日菜單 <span>↘</span>
            </button>
            <div class="hero-footnote"><span class="tiny-sun">☀</span> 每日新鮮製作・安心美味</div>
          </div>
          <div class="hero-art" aria-label="餐點插畫">
            <div class="hero-sun"></div>
            <div class="hero-circle circle-one"></div>
            <div class="hero-circle circle-two"></div>
            <div class="hero-plate">
              <div class="plate-rim">
                <div class="plate-food food-rice">🍚</div>
                <div class="plate-food food-chicken">🍗</div>
                <div class="plate-food food-veggie">🥦</div>
                <div class="plate-food food-tomato">🍅</div>
                <div class="plate-food food-egg">🥚</div>
              </div>
            </div>
            <div class="floating-note note-top"><span>✦</span> 今日新鮮現做</div>
            <div class="floating-note note-bottom"><span>♡</span> 用心準備每一餐</div>
            <div class="hero-sparkle sparkle-one">✦</div>
            <div class="hero-sparkle sparkle-two">✧</div>
          </div>
          <div class="hero-stamp"><span>GOOD</span><strong>FOOD</strong><span>GOOD MOOD</span></div>
        </section>

        <section class="benefits">
          <div class="benefit"><span class="benefit-icon">🌿</span><div><strong>新鮮食材</strong><small>每日嚴選好食材</small></div></div>
          <div class="benefit"><span class="benefit-icon">♡</span><div><strong>用心製作</strong><small>每份餐點都用心</small></div></div>
          <div class="benefit"><span class="benefit-icon">♧</span><div><strong>簡單點餐</strong><small>輕鬆完成每一餐</small></div></div>
          <div class="benefit"><span class="benefit-icon">☼</span><div><strong>暖心服務</strong><small>讓日常多一點美好</small></div></div>
        </section>

        <section id="menu-section" class="menu-section">
          <div class="section-heading">
            <div>
              <span class="section-kicker">OUR MENU</span>
              <h2>今天的美味提案<span class="heading-dot">.</span></h2>
              <p>從主餐到甜點，找到今天最想吃的那一味。</p>
            </div>
            <div class="menu-tools">
              <label class="search-box">
                <span>⌕</span>
                <input v-model="searchText" type="search" placeholder="搜尋餐點..." aria-label="搜尋餐點" />
                <button v-if="searchText" class="clear-search" @click="searchText = ''" aria-label="清除搜尋">×</button>
              </label>
              <select v-model="sortBy" aria-label="排序餐點">
                <option value="recommended">推薦排序</option>
                <option value="price-low">價格由低到高</option>
                <option value="price-high">價格由高到低</option>
                <option value="name">依名稱排序</option>
              </select>
            </div>
          </div>

          <div class="category-tabs">
            <button v-for="category in categories" :key="category" :class="{ selected: selectedCategory === category }" @click="selectedCategory = category">
              {{ category }} <span v-if="category === '全部餐點'" class="tab-sparkle">✦</span>
            </button>
          </div>

          <div v-if="filteredMenu.length" class="menu-grid">
            <article v-for="item in filteredMenu" :key="item.id" class="menu-card">
              <div class="food-art" :class="`food-${item.color}`">
                <span class="food-tag">{{ item.tag }}</span>
                <span class="food-emoji" aria-hidden="true">{{ item.emoji }}</span>
                <span class="food-deco deco-a">✦</span>
                <span class="food-deco deco-b">✧</span>
                <span class="food-art-label">WARM SUN</span>
              </div>
              <div class="menu-card-content">
                <div class="food-meta"><span>{{ item.category }}</span><span class="fresh-dot"></span><span>新鮮現做</span></div>
                <h3>{{ item.name }}</h3>
                <p>{{ item.description }}</p>
                <div class="menu-card-bottom">
                  <div class="price">{{ money(item.price) }}<small> / 份</small></div>
                  <button class="add-button" :aria-label="`加入${item.name}`" @click="addToCart(item)"><span>＋</span> 加入</button>
                </div>
              </div>
            </article>
          </div>
          <div v-else class="empty-state menu-empty">
            <span class="empty-illustration">🔎</span>
            <h3>找不到符合的餐點</h3>
            <p>試試其他關鍵字或切換餐點分類吧。</p>
            <button class="outline-button" @click="searchText = ''; selectedCategory = '全部餐點'">清除搜尋條件</button>
          </div>

          <div class="menu-bottom-note"><span>☀</span> 好好吃飯，是一天裡很重要的小事。</div>
        </section>
      </template>

      <section v-else-if="activePage === 'cart'" class="page-section">
        <div class="page-title">
          <span class="section-kicker">{{ showCheckout ? 'CHECKOUT' : 'YOUR CART' }}</span>
          <h1>{{ showCheckout ? '確認訂購資訊' : '我的購物車' }}<span class="heading-dot">.</span></h1>
          <p>{{ showCheckout ? '確認餐點與聯絡資料，準備享用美味。' : '把喜歡的餐點放進購物車，讓美味離你更近一步。' }}</p>
        </div>

        <div v-if="!showCheckout && isCartEmpty" class="empty-state cart-empty">
          <span class="empty-illustration">🛍️</span>
          <h2>購物車還是空的</h2>
          <p>美味正在等你，去菜單挑選今天想吃的餐點吧！</p>
          <button class="primary-button" @click="goTo('menu')">前往線上菜單 <span>→</span></button>
        </div>

        <template v-else>
          <div class="cart-layout">
            <div class="cart-main">
              <div v-if="!showCheckout" class="cart-panel">
                <div class="panel-heading"><h2>已選餐點 <span class="muted-count">({{ cartCount }} 件)</span></h2><button class="text-button" @click="clearCart">清空購物車</button></div>
                <article v-for="item in cart" :key="item.id" class="cart-item">
                  <div class="cart-item-art">{{ item.emoji }}</div>
                  <div class="cart-item-info"><h3>{{ item.name }}</h3><p>單價 {{ money(item.price) }}</p><button class="remove-button" @click="removeFromCart(item.id)">移除</button></div>
                  <div class="cart-item-end"><strong>{{ money(item.price * item.quantity) }}</strong><div class="quantity-control"><button @click="changeQuantity(item.id, -1)" :aria-label="`減少${item.name}數量`">−</button><span>{{ item.quantity }}</span><button @click="changeQuantity(item.id, 1)" :aria-label="`增加${item.name}數量`">＋</button></div></div>
                </article>
                <button class="continue-shopping" @click="goTo('menu')">← 繼續挑選餐點</button>
              </div>

              <div v-else class="cart-panel checkout-panel">
                <div class="panel-heading"><h2>訂購人資料</h2><button class="text-button" @click="showCheckout = false">返回購物車</button></div>
                <form @submit.prevent="submitOrder">
                  <div class="form-grid">
                    <label class="form-field"><span>訂購人姓名 <b>*</b></span><input v-model="customer.name" required maxlength="40" placeholder="請輸入姓名" /></label>
                    <label class="form-field"><span>聯絡手機 <b>*</b></span><input v-model="customer.phone" required inputmode="numeric" maxlength="10" placeholder="例如：0912345678" /></label>
                  </div>
                  <div class="form-field"><span>取餐方式 <b>*</b></span><div class="choice-grid">
                    <button v-for="method in [{name:'內用',icon:'🍽️',desc:'到店享用'},{name:'外帶',icon:'🥡',desc:'自取餐點'},{name:'外送',icon:'🛵',desc:'送到指定地址'}]" :key="method.name" type="button" class="choice-card" :class="{ chosen: customer.dining === method.name }" @click="customer.dining = method.name">
                      <span class="choice-icon">{{ method.icon }}</span><strong>{{ method.name }}</strong><small>{{ method.desc }}</small><span class="choice-check">{{ customer.dining === method.name ? '●' : '○' }}</span>
                    </button>
                  </div></div>
                  <label v-if="customer.dining === '外送'" class="form-field"><span>送達地址 <b>*</b></span><input v-model="customer.address" required maxlength="160" placeholder="請輸入完整地址" /></label>
                  <div class="form-field"><span>付款方式</span><div class="payment-options"><label><input v-model="customer.payment" type="radio" value="到店付款" /> 到店付款</label><label><input v-model="customer.payment" type="radio" value="取餐時付款" /> 取餐時付款</label></div><small class="field-hint">此為課堂展示系統，不會進行真實付款。</small></div>
                  <label class="form-field"><span>餐點備註</span><textarea v-model="customer.note" rows="3" maxlength="250" placeholder="例如：不要香菜、醬料另外放（選填）"></textarea></label>
                  <div class="form-actions"><button class="outline-button" type="button" @click="showCheckout = false">返回修改餐點</button><button class="primary-button" type="submit">確認送出訂單 <span>→</span></button></div>
                </form>
              </div>
            </div>

            <aside class="summary-card">
              <div class="summary-title"><span>✦</span><h2>訂單摘要</h2></div>
              <div class="summary-line"><span>商品小計</span><strong>{{ money(subtotal) }}</strong></div>
              <div class="summary-line"><span>優惠折扣</span><strong class="discount-value">− {{ money(discount) }}</strong></div>
              <div class="summary-line"><span>外送服務費</span><strong>{{ money(deliveryFee) }}</strong></div>
              <div v-if="subtotal < 500" class="discount-hint">再消費 <strong>{{ money(remainingForDiscount) }}</strong>，即可折抵 $50！</div>
              <div v-else class="discount-hint achieved">✦ 已達滿額折抵門檻</div>
              <div class="summary-total"><span>應付總金額</span><strong>{{ money(total) }}</strong></div>
              <button v-if="!showCheckout" class="primary-button full-button" :disabled="isCartEmpty" @click="openCheckout">前往結帳 <span>→</span></button>
              <div class="secure-note"><span>♡</span> 輕鬆點餐，安心享用</div>
            </aside>
          </div>
        </template>
      </section>

      <section v-else class="page-section">
        <div class="page-title">
          <span class="section-kicker">ORDER HISTORY</span>
          <h1>我的訂單紀錄<span class="heading-dot">.</span></h1>
          <p>每一份訂單，都是一段美味的小回憶。</p>
        </div>
        <div v-if="!orders.length" class="empty-state orders-empty">
          <span class="empty-illustration">🧾</span>
          <h2>目前還沒有訂單</h2>
          <p>完成第一次點餐後，訂單就會顯示在這裡。</p>
          <button class="primary-button" @click="goTo('menu')">去挑選美味餐點 <span>→</span></button>
        </div>
        <div v-else class="orders-list">
          <article v-for="order in orders" :key="order.id" class="order-card">
            <div class="order-card-top">
              <div><span class="order-label">訂單編號</span><strong class="order-id">{{ order.id }}</strong></div>
              <span class="status-pill" :class="{ cancelled: order.status === '已取消' }">{{ order.status }}</span>
            </div>
            <div class="order-card-meta"><span>◷ {{ new Date(order.createdAt).toLocaleString('zh-TW', { hour12: false }) }}</span><span>♧ {{ order.customer.dining }}</span><span>餐點 {{ order.items.reduce((sum, item) => sum + item.quantity, 0) }} 件</span></div>
            <div class="order-card-bottom"><div><span>訂單金額</span><strong>{{ money(order.total) }}</strong></div><div class="order-actions"><button class="outline-button small-button" @click="viewOrder(order)">查看明細</button><button v-if="order.status !== '已取消'" class="text-button cancel-order-button" @click="cancelOrder(order)">取消訂單</button></div></div>
          </article>
          <p class="local-note">訂單資料只儲存在目前瀏覽器中，清除瀏覽器資料後可能會消失。</p>
        </div>
      </section>
    </main>

    <footer class="site-footer">
      <div class="footer-main">
        <div class="footer-brand">
          <a class="brand footer-logo" href="#" @click.prevent="goTo('menu')"><span class="brand-mark">暖</span><span class="brand-copy"><strong>暖日食堂</strong><small>WARM SUN KITCHEN</small></span></a>
          <p>用一份好好準備的餐點，<br />讓日常多一點溫暖。</p>
        </div>
        <div class="footer-column"><strong>快速連結</strong><button @click="goTo('menu')">線上菜單</button><button @click="goTo('cart')">購物車</button><button @click="goTo('orders')">我的訂單</button></div>
        <div class="footer-column"><strong>營業資訊</strong><span>週一至週日</span><span>11:00 – 20:30</span><span>☎ 02-1234-5678（示意）</span></div>
        <div class="footer-message"><span>“</span><p>Eat well,<br /><em>live warmly.</em></p><small>好好吃飯，好好生活。</small></div>
      </div>
      <div class="footer-bottom"><span>© 2026 暖日食堂 Warm Sun Kitchen</span><span>以溫暖為配方，以美味為日常。 <b>☀</b></span><button @click="resetDemoData">清除本機示範資料</button></div>
    </footer>

    <div v-if="toastVisible" class="toast-message" role="status"><span>✓</span>{{ toastMessage }}</div>

    <div v-if="showOrderDetail && selectedOrder" class="modal-backdrop" @click.self="closeOrderDetail">
      <section class="order-modal" role="dialog" aria-modal="true" aria-labelledby="order-modal-title">
        <button class="modal-close" aria-label="關閉" @click="closeOrderDetail">×</button>
        <div class="modal-success">{{ selectedOrder.status === '已取消' ? '↺' : '✓' }}</div>
        <span class="section-kicker">ORDER DETAILS</span>
        <h2 id="order-modal-title">{{ selectedOrder.status === '已取消' ? '訂單已取消' : '訂單成立！' }}</h2>
        <p class="modal-subtitle">{{ selectedOrder.status === '已取消' ? '這筆訂單已標記為取消。' : '感謝你的訂購，期待為你帶來美味。' }}</p>
        <div class="modal-order-id"><span>訂單編號</span><strong>{{ selectedOrder.id }}</strong></div>
        <div class="modal-items">
          <div v-for="item in selectedOrder.items" :key="item.id" class="modal-item"><span>{{ item.emoji }} {{ item.name }} × {{ item.quantity }}</span><strong>{{ money(item.price * item.quantity) }}</strong></div>
        </div>
        <div class="modal-totals"><div><span>商品小計</span><span>{{ money(selectedOrder.subtotal) }}</span></div><div><span>優惠折扣</span><span>− {{ money(selectedOrder.discount) }}</span></div><div><span>外送服務費</span><span>{{ money(selectedOrder.deliveryFee) }}</span></div><div class="modal-grand-total"><strong>總計</strong><strong>{{ money(selectedOrder.total) }}</strong></div></div>
        <div class="modal-customer"><p><span>訂購人</span>{{ selectedOrder.customer.name }}</p><p><span>聯絡電話</span>{{ selectedOrder.customer.phone }}</p><p><span>取餐方式</span>{{ selectedOrder.customer.dining }}</p><p v-if="selectedOrder.customer.address"><span>送達地址</span>{{ selectedOrder.customer.address }}</p><p><span>付款方式</span>{{ selectedOrder.customer.payment }}</p><p v-if="selectedOrder.customer.note"><span>備註</span>{{ selectedOrder.customer.note }}</p></div>
        <button class="primary-button full-button" @click="closeOrderDetail">完成 <span>✓</span></button>
      </section>
    </div>
  </div>
</template>
