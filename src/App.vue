<template>
  <div class="app">
    <!-- 標題 -->
    <header class="header">
      <h1>🍜 阿龍小吃店</h1>
      <span class="cart-icon" @click="showCart = !showCart">
        🛒 {{ totalQty }} 項
      </span>
    </header>

    <!-- 桌號輸入 -->
    <div class="table-input">
      <label>桌號：</label>
      <input v-model="tableNumber" placeholder="請輸入桌號，例：A3" />
    </div>

    <!-- 分類 Tabs -->
    <div class="tabs">
      <button
        v-for="cat in categories" :key="cat"
        :class="['tab', { active: activeCategory === cat }]"
        @click="activeCategory = cat"
      >{{ cat }}</button>
    </div>

    <!-- 菜單格 -->
    <div v-if="loading" class="loading">
      ⏳ 載入菜單中...
    </div>
    <div v-else class="menu-grid">
      <div
        v-for="item in filteredMenu" :key="item.id"
        class="menu-card"
        :class="{ 'in-cart': getQty(item.id) > 0 }"
      >
        <!-- 顯示來自試算表的 tag (招牌/熱門) -->
        <span v-if="item.tag" class="badge">{{ item.tag }}</span>
        <span class="emoji">{{ item.emoji }}</span>
        <p class="name">{{ item.name }}</p>
        <p class="desc">{{ item.desc }}</p>
        <div class="card-footer">
          <span class="price">${{ item.price }}</span>
          <!-- 未加入：顯示加入按鈕 -->
          <button v-if="getQty(item.id) === 0" class="btn-add" @click="add(item)">+ 加入</button>
          <!-- 已加入：顯示數量控制 -->
          <div v-else class="qty-control">
            <button @click="minus(item.id)">－</button>
            <span>{{ getQty(item.id) }}</span>
            <button @click="add(item)">＋</button>
          </div>
        </div>
      </div>
    </div>

    <!-- 購物車浮層 -->
    <div class="cart-panel" v-if="showCart && cart.length > 0">
      <div class="cart-header">
        <h3>訂單明細</h3>
        <button class="btn-clear" @click="clearCart">清空</button>
      </div>
      <div v-for="item in cart" :key="item.id" class="cart-row">
        <div class="cart-item-info">
          <span>{{ item.name }}</span>
          <span class="cart-item-price">${{ item.price * item.qty }}</span>
        </div>
        <div class="cart-item-actions">
          <div class="qty-control">
            <button @click="minus(item.id)">－</button>
            <span>{{ item.qty }}</span>
            <button @click="add(item)">＋</button>
          </div>
          <button class="btn-remove" @click="remove(item.id)">🗑️</button>
        </div>
      </div>
      <div class="cart-note">
        <label>備註：</label>
        <input v-model="note" placeholder="例：不加辣、少油" />
      </div>
      <div class="cart-total">合計：${{ totalAmount }}</div>
      <button class="btn-order" @click="submit" :disabled="!tableNumber">送出訂單</button>
      <p v-if="!tableNumber" style="color:red;font-size:12px">請先填入桌號</p>
    </div>

    <!-- 送出成功畫面 -->
    <div class="success" v-if="orderDone">
      <div class="success-card">
        <div class="icon">🎉</div>
        <h2>訂單已送出！</h2>
        <p v-if="lastOrder && lastOrder.id">訂單編號：{{ lastOrder.id }}</p>
        <p>桌號：{{ lastOrder.table }}</p>
        <ul>
          <li v-for="item in lastOrder.items" :key="item.id">
            {{ item.name }} × {{ item.qty }}　${{ item.price * item.qty }}
          </li>
        </ul>
        <p><strong>合計：${{ lastOrder.total }}</strong></p>
        <button @click="reset">繼續點餐</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

// 資料
const apiUrl       = ref('https://script.google.com/macros/s/AKfycbyWxzQn064nejZTWhE0LeaQQrTTTtaknWwJwFwncMhpYbts01H5tqV2-TCpZCgJLXR0NA/exec')
const loading      = ref(true)
const tableNumber  = ref('')
const note         = ref('')
const activeCategory = ref('全部')
const showCart     = ref(false)
const orderDone    = ref(false)
const lastOrder    = ref(null)
const cart         = ref([])
const menu         = ref([])

// 計算屬性
const categories   = computed(() => ['全部', ...new Set(menu.value.map(i => i.category))])
const filteredMenu = computed(() =>
  activeCategory.value === '全部' ? menu.value : menu.value.filter(i => i.category === activeCategory.value)
)
const totalQty     = computed(() => cart.value.reduce((s, i) => s + i.qty, 0))
const totalAmount  = computed(() => cart.value.reduce((s, i) => s + i.price * i.qty, 0))

// 方法
async function loadMenu() {
  try {
    const res  = await fetch(apiUrl.value + '?action=getMenu')
    const data = await res.json()
    if (data.success) {
      // 過濾掉無效品項（確保 ID 與 名稱皆存在，避免顯示試算表中的空白行）
      menu.value = data.data.filter(item => item.id && item.name)
    } else {
      console.error('API 錯誤:', data.error)
    }
  } catch (e) {
    console.error('載入菜單失敗', e)
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadMenu()
})

// 輔助函式：統一購物車品項的比對邏輯，確保 ID 類型一致性
const findCartItem = (id) => cart.value.find(i => String(i.id).trim() === String(id).trim())

function getQty(id) {
  return findCartItem(id)?.qty ?? 0
}

function add(item) {
  const found = findCartItem(item.id)
  if (found) {
    found.qty++
  } else {
    cart.value.push({ ...item, qty: 1 })
  }
  showCart.value = true
}

function remove(id) {
  cart.value = cart.value.filter(i => String(i.id).trim() !== String(id).trim())
}

function minus(id) {
  const found = findCartItem(id)
  if (found) {
    if (found.qty > 1) {
      found.qty--
    } else {
      // 當數量為 1 時點擊「－」，自動將該品項從清單中移除
      remove(id)
    }
  }
}
function clearCart() {
  if (confirm('確定要清空所有品項嗎？')) cart.value = []
}
async function submit() {
  try {
    const res = await fetch(apiUrl.value, {
      method: 'POST',
      headers: { 'Content-Type': 'text/plain' },
      body: JSON.stringify({
        action: 'submitOrder',
        tableNumber: tableNumber.value,
        items: cart.value,
        totalAmount: totalAmount.value,
        note: note.value,
      })
    })
    const text = await res.text() // 先轉成純文字，避免 JSON 解析失敗看不到報錯
    const data = JSON.parse(text)
    if (data.success) {
      lastOrder.value = {
        id:    data.data.orderId,
        table: tableNumber.value,
        items: [...cart.value],
        total: totalAmount.value,
        note:  note.value,
      }
      orderDone.value = true
      cart.value = []
      showCart.value = false
    } else {
      alert('送出失敗：' + (data.error || data.message || '未知錯誤'))
    }
  } catch (e) {
    console.error('送出訂單失敗', e)
    alert('網路錯誤，請稍後再試')
  }
}
function reset() {
  orderDone.value  = false
  tableNumber.value = ''
  note.value = ''
}
</script>

<style scoped>
.app  { max-width: 900px; margin: 0 auto; padding: 1rem; font-family: sans-serif; }
.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
h1 { color: #c84b2f; margin: 0; }
.cart-icon { cursor: pointer; font-size: 18px; background: #2a1f14; color: white; padding: 6px 14px; border-radius: 99px; }
.table-input { margin-bottom: 1rem; }
.table-input input { padding: 6px 10px; border: 1px solid #ccc; border-radius: 6px; width: 200px; }
.tabs { display: flex; gap: 8px; margin-bottom: 1rem; flex-wrap: wrap; }
.tab { padding: 5px 14px; border: 1px solid #ccc; border-radius: 99px; background: white; cursor: pointer; }
.tab.active { background: #2a1f14; color: white; border-color: #2a1f14; }
.menu-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(160px, 1fr)); gap: 12px; }
.menu-card { background: white; border: 1px solid #ddd; border-radius: 12px; padding: 12px; position: relative; }
.menu-card.in-cart { border-color: #c9962a; background: #fef9ec; }
.emoji { font-size: 28px; }
.name  { font-weight: bold; margin: 4px 0; }
.desc  { font-size: 12px; color: #888; margin-bottom: 8px; }
.card-footer { display: flex; justify-content: space-between; align-items: center; }
.price { color: #c84b2f; font-weight: bold; }
.btn-add { padding: 4px 10px; background: #c84b2f; color: white; border: none; border-radius: 6px; cursor: pointer; }
.qty-control { display: flex; gap: 6px; align-items: center; }
.qty-control button { width: 24px; height: 24px; border-radius: 50%; background: #2a1f14; color: white; border: none; cursor: pointer; }
.badge { position: absolute; top: 10px; right: 10px; background: #ffec3d; color: #cf1322; font-size: 10px; padding: 2px 6px; border-radius: 4px; font-weight: bold; }
.cart-panel { position: fixed; right: 20px; bottom: 20px; width: 300px; background: white; border: 1px solid #ddd; border-radius: 12px; padding: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.15); }
.cart-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
.cart-header h3 { margin: 0; }
.cart-row   { display: flex; flex-direction: column; gap: 8px; font-size: 14px; padding: 10px 0; border-bottom: 1px solid #eee; }
.cart-item-info { display: flex; justify-content: space-between; font-weight: bold; }
.cart-item-actions { display: flex; justify-content: space-between; align-items: center; }
.cart-item-price { color: #c84b2f; }
.btn-remove { background: none; border: none; cursor: pointer; font-size: 16px; opacity: 0.6; }
.btn-remove:hover { opacity: 1; }
.btn-clear { background: #f0f0f0; border: none; padding: 4px 8px; border-radius: 4px; font-size: 12px; cursor: pointer; color: #666; }
.cart-note  { margin: 10px 0; }
.cart-note input { width: 100%; padding: 6px; border: 1px solid #ccc; border-radius: 6px; }
.cart-total { font-size: 18px; font-weight: bold; color: #c84b2f; margin: 8px 0; }
.btn-order  { width: 100%; padding: 10px; background: #2a1f14; color: white; border: none; border-radius: 6px; cursor: pointer; }
.btn-order:disabled { opacity: 0.4; cursor: not-allowed; }
.success    { position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; }
.success-card { background: white; border-radius: 16px; padding: 2rem; text-align: center; max-width: 360px; width: 90%; }
.icon { font-size: 48px; }
.loading { text-align: center; padding: 40px; color: #888; width: 100%; }
</style>
