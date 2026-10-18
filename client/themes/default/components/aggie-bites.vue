<template>
  <v-card outlined class="aggie-bites mb-5" :class="{ 'bites-dark': $vuetify.theme.dark }" aria-labelledby="bites-heading">
    <v-card-title class="pb-1"><span id="bites-heading">{{ zh ? '今日食堂' : 'Today’s dining' }}</span><v-spacer /><span class="caption">Aggie Bites</span></v-card-title>
    <v-card-text>
      <div class="bites-controls">
        <v-select v-model="hall" :items="hallOptions" :label="zh ? '食堂' : 'Dining hall'" dense hide-details outlined />
        <v-select v-model="meal" :items="mealOptions" :label="zh ? '餐次' : 'Meal'" dense hide-details outlined />
      </div>
      <div aria-live="polite" :aria-busy="loading ? 'true' : 'false'">
        <p v-if="loading" class="mt-4">{{ zh ? '正在加载菜单…' : 'Loading menu…' }}</p>
        <p v-else-if="error" class="mt-4">{{ zh ? '菜单暂时无法加载' : 'The menu is temporarily unavailable' }} · <a :href="menuLink">{{ zh ? '前往菜单站' : 'Visit the menu site' }}</a></p>
        <template v-else-if="selected">
          <p class="caption mt-4">{{ payload.date }} · {{ mealLabel }}</p>
          <p v-if="selected.fetch_status !== 'ok'">{{ zh ? '最新菜单暂未更新，以下为最近记录。' : 'The latest fetch is unavailable. Showing the last recorded menu.' }}</p>
          <p v-if="selected.status !== 'ok'">{{ statusLabel }}</p>
          <template v-else>
            <p v-if="aiBlurb" class="body-1 font-weight-medium">{{ aiBlurb }}</p>
            <ul class="bites-list"><li v-for="dish in selected.highlights" :key="dish.id" class="bites-dish" :class="zoneClass(dish.zone)">
              <a :href="menuLink + 'dish/' + dish.id">{{ zh ? (dish.name_zh || dish.name_en) : dish.name_en }}</a>
              <span v-if="zh && dish.name_zh" class="caption d-block">{{ dish.name_en }}<span v-if="dish.translation_status === 'ai'"> · 自动翻译</span></span>
              <span v-if="zoneLabel(dish.zone)" class="bites-zone caption text--secondary"><span class="bites-zone-swatch" aria-hidden="true" />{{ zoneLabel(dish.zone) }}</span>
            </li></ul>
          </template>
          <a :href="menuLink + selected.hall_id + '/' + payload.date">{{ zh ? '查看完整菜单' : 'View the full menu' }} →</a>
        </template>
      </div>
    </v-card-text>
  </v-card>
</template>

<script>
export default {
  name: 'AggieBites',
  props: { locale: { type: String, default: 'en' } },
  data () { return { hall: 'cuarto', meal: '', payload: null, loading: false, error: false, controller: null, generation: 0 } },
  computed: {
    zh () { return this.locale.toLowerCase().startsWith('zh') },
    menuLink () { return `https://menu.ucdavis.top/${this.zh ? 'zh' : 'en'}/` },
    hallOptions () { return this.payload ? this.payload.halls.map(h => ({ text: this.zh ? h.name_zh : h.name_en, value: h.hall_id })) : ['Cuarto', 'Segundo', 'Tercero', 'Latitude'].map(h => ({ text: h, value: h.toLowerCase() })) },
    mealOptions () { return [{ text: this.zh ? '当前／下一餐' : 'Current / next meal', value: '' }, ...['breakfast', 'lunch', 'dinner'].map((m, i) => ({ text: this.zh ? ['早餐', '午餐', '晚餐'][i] : ['Breakfast', 'Lunch', 'Dinner'][i], value: m }))] },
    selected () { return this.payload && this.payload.halls.find(h => h.hall_id === this.hall) },
    aiBlurb () { return this.selected && this.selected.summary_source === 'ai' ? (this.zh ? this.selected.blurb_zh : this.selected.blurb_en) : '' },
    mealLabel () { const m = this.payload && this.payload.meal; return this.zh ? ({ breakfast: '早餐', lunch: '午餐', dinner: '晚餐' }[m] || m) : m },
    statusLabel () { return this.zh ? ({ closed_expected: '本餐不营业', no_menu: '菜单尚未公布', unavailable: '菜单暂未更新' }[this.selected.status] || '菜单暂未更新') : this.selected.status.replace(/_/g, ' ') }
  },
  watch: { meal () { this.load() }, locale () { this.load() } },
  mounted () { this.load() },
  beforeDestroy () { this.generation++; if (this.controller) this.controller.abort() },
  methods: {
    zoneClass (zone) {
      const key = typeof zone === 'string' ? zone.trim().toLowerCase() : ''
      return ['red', 'yellow', 'blue', 'green', 'purple', 'pink'].includes(key) ? `bites-zone-${key}` : ''
    },
    zoneLabel (zone) {
      if (typeof zone !== 'string' || !zone.trim()) return ''
      const name = zone.trim()
      const labels = { red: '红区', yellow: '黄区', blue: '蓝区', green: '绿区', purple: '紫区', pink: '粉区' }
      return this.zh ? (labels[name.toLowerCase()] || name) : `${name} zone`
    },
    async load () {
      const generation = ++this.generation
      if (this.controller) this.controller.abort()
      const controller = new AbortController()
      this.controller = controller
      this.loading = true
      this.error = false
      const timeout = setTimeout(() => controller.abort(), 5000)
      try {
        const response = await fetch('/bites-api/today' + (this.meal ? '?meal=' + encodeURIComponent(this.meal) : ''), { signal: controller.signal, credentials: 'omit' })
        if (!response.ok) throw new Error('Menu API unavailable')
        const data = await response.json()
        if (data.schema_version !== '1' || !Array.isArray(data.halls) || !data.halls.length || data.halls.some(h => !Array.isArray(h.highlights))) throw new Error('Invalid API response')
        if (generation !== this.generation) return
        this.payload = data
        if (!data.halls.some(h => h.hall_id === this.hall)) this.hall = data.halls[0].hall_id
      } catch (error) {
        if (generation === this.generation) this.error = true
      } finally {
        clearTimeout(timeout)
        if (generation === this.generation) this.loading = false
      }
    }
  }
}
</script>

<style scoped>
.aggie-bites { border-top: 3px solid var(--v-primary-base, #c9a227); }
.bites-controls { display: flex; gap: 12px; margin-top: 12px; }
.bites-controls > * { min-width: 0; flex: 1; }
/* Only location accents are fixed; surfaces, typography and text inherit Wiki's theme. */
.aggie-bites {
  --bites-red: #d04437;
  --bites-yellow: #d99a10;
  --bites-blue: #2c6cb0;
  --bites-green: #3d8c4e;
  --bites-purple: #7650ae;
  --bites-pink: #d2558a;
}
.aggie-bites.bites-dark {
  --bites-red: #ec6a5d;
  --bites-yellow: #f0c043;
  --bites-blue: #6aa5e6;
  --bites-green: #6fbe7e;
  --bites-purple: #a98ae0;
  --bites-pink: #ef86b2;
}
.bites-zone-red { --bites-zone: var(--bites-red); }
.bites-zone-yellow { --bites-zone: var(--bites-yellow); }
.bites-zone-blue { --bites-zone: var(--bites-blue); }
.bites-zone-green { --bites-zone: var(--bites-green); }
.bites-zone-purple { --bites-zone: var(--bites-purple); }
.bites-zone-pink { --bites-zone: var(--bites-pink); }
.bites-list { list-style: none; padding-left: 0; margin: 12px 0 20px; }
.bites-dish { margin-bottom: 14px; padding-left: 12px; border-left: 3px solid var(--bites-zone, transparent); }
.bites-zone { display: flex; align-items: center; gap: 6px; margin-top: 4px; }
.bites-zone-swatch { width: 8px; height: 8px; border-radius: 2px; background: var(--bites-zone, currentColor); flex: none; }
</style>
