<template>
  <ion-page>
    <ion-header>
      <ion-toolbar class="app-toolbar">
        <ion-title class="app-title">
          <span class="logo-text">CRYPTO</span>
          <span class="logo-tag">APP</span>
        </ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <div class="container">
        <p class="status-text" v-if="loading && !coins.length">Memuat data...</p>

        <div class="pill-row" v-if="coins.length">
          <button
            v-for="c in pillCoins"
            :key="c.id"
            :class="['pill', { aktif: c.id === selectedId }]"
            @click="selectedId = c.id"
          >
            {{ c.symbol }}
          </button>
        </div>

        <div class="card card-blue" v-if="selected">
          <div class="price-top">
            <div class="price-left">
              <span class="price-label">{{ selected.name }} price</span>
              <span class="price-big">$ {{ selected.price_usd }}</span>
              <span :class="['price-change', kelasPersen(selected.percent_change_24h)]">
                {{ formatPersen(selected.percent_change_24h) }} (24 jam)
              </span>
            </div>
            <span class="coin-badge">{{ selected.symbol }}</span>
          </div>
        </div>

        <div class="card" v-if="selected">
          <div class="period-row">
            <button
              v-for="p in periodeList"
              :key="p.kode"
              :class="['chip', { aktif: p.kode === periode }]"
              @click="periode = p.kode"
            >
              {{ p.label }}
            </button>
          </div>

          <div class="bubble-row">
            <span :class="['change-bubble', kelasPersen(persenPeriode)]">
              {{ labelPeriode }}: {{ formatPersen(persenPeriode) }}
            </span>
          </div>

          <div v-if="chartSvg">
            <svg :viewBox="'0 0 ' + chartSvg.W + ' ' + chartSvg.H" class="chart-svg">
              <defs>
                <linearGradient id="areaGrad" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stop-color="#2f6df6" stop-opacity="0.25" />
                  <stop offset="100%" stop-color="#2f6df6" stop-opacity="0" />
                </linearGradient>
              </defs>
              <line
                v-for="n in 4"
                :key="n"
                x1="0"
                :x2="chartSvg.W"
                :y1="(n * chartSvg.H) / 5"
                :y2="(n * chartSvg.H) / 5"
                stroke="#dbe5f5"
                stroke-width="1"
                stroke-dasharray="3 4"
              />
              <polygon :points="chartSvg.area" fill="url(#areaGrad)" />
              <polyline
                :points="chartSvg.garis"
                fill="none"
                stroke="#2f6df6"
                stroke-width="3"
                stroke-linecap="round"
                stroke-linejoin="round"
              />
              <circle
                v-for="(k, i) in chartSvg.titik"
                :key="i"
                :cx="k.x"
                :cy="k.y"
                r="5"
                fill="#ffffff"
                stroke="#2f6df6"
                stroke-width="3"
              />
            </svg>
            <div class="chart-labels">
              <div class="chart-label" v-for="(k, i) in chartSvg.titik" :key="i">
                <span class="cl-time">{{ k.label }}</span>
                <span class="cl-val">{{ formatHarga(k.value) }}</span>
              </div>
            </div>
            <p class="chart-note">
              Titik grafik dihitung dari persentase perubahan harga (1 jam, 24 jam, 7 hari) yang diberikan API.
            </p>
          </div>
        </div>

        <div class="card" v-if="selected">
          <h2 class="section-title">Detail {{ selected.name }}</h2>
          <div class="info-row">
            <span class="info-label">Rank</span>
            <span class="info-value">{{ selected.rank }}</span>
          </div>
          <div class="info-row">
            <span class="info-label">Symbol</span>
            <span class="info-value">{{ selected.symbol }}</span>
          </div>
          <div class="info-row">
            <span class="info-label">Volume 24 Jam</span>
            <span class="info-value">{{ formatRingkas(Number(selected.volume24)) }}</span>
          </div>
          <div class="info-row">
            <span class="info-label">Market Cap</span>
            <span class="info-value">{{ formatRingkas(Number(selected.market_cap_usd)) }}</span>
          </div>
        </div>

        <ion-button class="btn-refresh" expand="block" @click="ambilData" :disabled="loading">
          {{ loading ? 'Memuat...' : 'Refresh' }}
        </ion-button>

        <p class="error-text" v-if="error">{{ error }}</p>

        <div class="card" v-if="coins.length">
          <h2 class="section-title">Daftar Cryptocurrency</h2>
          <div class="coin-row" v-for="coin in coins" :key="coin.id">
            <span class="rank-badge">{{ coin.rank }}</span>
            <div class="coin-info">
              <span class="coin-symbol">{{ coin.symbol }}</span>
              <span class="coin-name">{{ coin.name }}</span>
            </div>
            <span class="coin-price">$ {{ coin.price_usd }}</span>
          </div>
        </div>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton } from '@ionic/vue';
import { defineComponent } from 'vue';
import EndPointAccess from '@/services/EndPointAccess';

interface Coin {
  id: string;
  rank: number;
  name: string;
  symbol: string;
  price_usd: string;
  percent_change_1h: string;
  percent_change_24h: string;
  percent_change_7d: string;
  volume24: number;
  market_cap_usd: string;
}

const JUMLAH_PILL = 4;

export default defineComponent({
  name: 'HomePage',
  data() {
    return {
      coins: [] as Coin[],
      selectedId: '',
      periode: '7d',
      periodeList: [
        { kode: '1h', label: '1 Jam' },
        { kode: '24h', label: '24 Jam' },
        { kode: '7d', label: '7 Hari' }
      ],
      loading: false,
      error: ''
    };
  },
  computed: {
    pillCoins(): Coin[] {
      return this.coins.slice(0, JUMLAH_PILL);
    },
    selected(): Coin | null {
      return this.coins.find((c) => c.id === this.selectedId) ?? null;
    },
    persenPeriode(): string {
      const c = this.selected;
      if (!c) return '0';
      if (this.periode === '1h') return c.percent_change_1h;
      if (this.periode === '24h') return c.percent_change_24h;
      return c.percent_change_7d;
    },
    labelPeriode(): string {
      const p = this.periodeList.find((x) => x.kode === this.periode);
      return p ? p.label : '';
    },
    chartPoints(): { label: string; value: number }[] {
      const c = this.selected;
      if (!c) return [];
      const sekarang = Number(c.price_usd);
      const hargaLalu = (persen: string) => sekarang / (1 + Number(persen) / 100);
      const now = { label: 'Sekarang', value: sekarang };
      const h1 = { label: '1 jam lalu', value: hargaLalu(c.percent_change_1h) };
      const h24 = { label: '24 jam lalu', value: hargaLalu(c.percent_change_24h) };
      const d7 = { label: '7 hari lalu', value: hargaLalu(c.percent_change_7d) };
      if (this.periode === '1h') return [h1, now];
      if (this.periode === '24h') return [h24, h1, now];
      return [d7, h24, h1, now];
    },
    chartSvg() {
      const pts = this.chartPoints;
      if (pts.length < 2) return null;
      const W = 300;
      const H = 120;
      const PX = 20;
      const PY = 14;
      const nilai = pts.map((p) => p.value);
      const min = Math.min(...nilai);
      const max = Math.max(...nilai);
      const range = max - min;
      const titik = pts.map((p, i) => ({
        x: PX + (i * (W - 2 * PX)) / (pts.length - 1),
        y: range === 0 ? H / 2 : PY + (1 - (p.value - min) / range) * (H - 2 * PY),
        label: p.label,
        value: p.value
      }));
      const garis = titik.map((k) => k.x + ',' + k.y).join(' ');
      const awal = titik[0];
      const akhir = titik[titik.length - 1];
      return {
        W,
        H,
        titik,
        garis,
        area: awal.x + ',' + H + ' ' + garis + ' ' + akhir.x + ',' + H
      };
    }
  },
  mounted() {
    this.ambilData();
  },
  methods: {
    kelasPersen(persen: string): string {
      return Number(persen) >= 0 ? 'naik' : 'turun';
    },
    formatPersen(persen: string): string {
      return (Number(persen) >= 0 ? '+' : '') + persen + '%';
    },
    formatHarga(n: number): string {
      return n >= 1 ? n.toFixed(2) : n.toFixed(6);
    },
    formatRingkas(n: number): string {
      return new Intl.NumberFormat('en-US', {
        notation: 'compact',
        maximumFractionDigits: 2
      }).format(n);
    },
    async ambilData() {
      this.loading = true;
      this.error = '';
      try {
        const api = new EndPointAccess('https://api.coinlore.net/api/tickers/');
        const response = await api.getRes();
        this.coins = response.data.data.map((c: any) => ({
          id: c.id,
          rank: c.rank,
          name: c.name,
          symbol: c.symbol,
          price_usd: c.price_usd,
          percent_change_1h: c.percent_change_1h,
          percent_change_24h: c.percent_change_24h,
          percent_change_7d: c.percent_change_7d,
          volume24: c.volume24,
          market_cap_usd: c.market_cap_usd
        }));
        if (!this.selectedId && this.coins.length) {
          this.selectedId = this.coins[0].id;
        }
      } catch (e) {
        this.error = 'Gagal mengambil data. Periksa koneksi internet.';
      } finally {
        this.loading = false;
      }
    }
  },
  components: { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton }
});
</script>

<style scoped>
ion-header {
  box-shadow: none;
}
ion-header::after {
  display: none;
}
.app-toolbar {
  --background: #bcd9ff;
  --color: #1b2a49;
  --border-width: 0;
}
.app-title {
  text-align: center;
  font-weight: 800;
}
.logo-text {
  color: #2f6df6;
  font-size: 22px;
  margin-right: 6px;
}
.logo-tag {
  display: inline-block;
  background: linear-gradient(135deg, #2f6df6, #00b8ff);
  color: #ffffff;
  font-size: 20px;
  font-weight: 600;
  padding: 0 10px;
  border-radius: 8px;
}

ion-content {
  --background: linear-gradient(180deg, #bcd9ff 0%, #e6eeff 45%, #f5f8ff 100%);
}
.container {
  max-width: 480px;
  margin: 0 auto;
  padding: 16px;
}
.status-text {
  color: #7a8aa8;
  text-align: center;
}

.naik {
  color: #1f9d6b;
}
.turun {
  color: #e5383b;
}

.card {
  background: #ffffff;
  border: 1px solid #e3ebf8;
  border-radius: 24px;
  padding: 16px;
  margin-bottom: 14px;
  box-shadow: 0 6px 18px rgba(60, 110, 200, 0.12);
}
.card-blue {
  background: linear-gradient(135deg, #2f6df6, #00b8ff);
  border: none;
  box-shadow: 0 10px 22px rgba(47, 109, 246, 0.35);
}

.pill-row {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 10px;
  margin: 4px 0 16px;
}
.pill {
  border: none;
  background: #ffffff;
  color: #2f6df6;
  font-size: 15px;
  font-weight: 700;
  padding: 9px 20px;
  border-radius: 999px;
  box-shadow: 0 4px 10px rgba(60, 110, 200, 0.15);
}
.pill.aktif {
  background: linear-gradient(135deg, #2f6df6, #00b8ff);
  color: #ffffff;
}

.price-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}
.price-left {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.price-label {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.85);
}
.price-big {
  font-size: clamp(24px, 8vw, 34px);
  font-weight: 800;
  color: #ffffff;
}
.price-change {
  align-self: flex-start;
  background: #ffffff;
  font-size: 13px;
  font-weight: 700;
  padding: 3px 12px;
  border-radius: 999px;
}
.coin-badge {
  min-width: 56px;
  height: 56px;
  display: grid;
  place-items: center;
  background: rgba(255, 255, 255, 0.2);
  border: 2px solid rgba(255, 255, 255, 0.7);
  border-radius: 50%;
  color: #ffffff;
  font-size: 13px;
  font-weight: 800;
  padding: 0 4px;
}

.period-row {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 12px;
}
.chip {
  border: none;
  background: #eef2fa;
  color: #4b5b78;
  font-size: 14px;
  font-weight: 700;
  padding: 8px 18px;
  border-radius: 999px;
}
.chip.aktif {
  background: #2f6df6;
  color: #ffffff;
}
.bubble-row {
  text-align: right;
  margin-bottom: 4px;
}
.change-bubble {
  display: inline-block;
  background: #eef3ff;
  font-size: 13px;
  font-weight: 700;
  padding: 4px 12px;
  border-radius: 999px;
}
.chart-svg {
  width: 100%;
  height: auto;
}
.chart-labels {
  display: flex;
  justify-content: space-between;
  padding: 0 4px;
}
.chart-label {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
}
.cl-time {
  font-size: 11px;
  color: #7a8aa8;
}
.cl-val {
  font-size: 12px;
  font-weight: 700;
  color: #1b2a49;
}
.chart-note {
  font-size: 11px;
  color: #7a8aa8;
  margin: 8px 0 0;
  text-align: center;
}

.section-title {
  font-size: 16px;
  font-weight: 800;
  color: #1b2a49;
  margin: 0 0 8px;
}
.info-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 11px 0;
  border-bottom: 1px solid #eef2fa;
}
.info-row:last-child {
  border-bottom: none;
}
.info-label {
  font-size: 14px;
  color: #7a8aa8;
}
.info-value {
  font-size: 15px;
  font-weight: 800;
  color: #1b2a49;
}

.btn-refresh {
  --background: linear-gradient(135deg, #2f6df6, #00b8ff);
  --color: #ffffff;
  --border-radius: 16px;
  --box-shadow: 0 8px 18px rgba(47, 109, 246, 0.35);
  font-weight: 800;
  height: 52px;
  margin: 0 0 14px;
}
.error-text {
  color: #e5383b;
  font-size: 13px;
  text-align: center;
}

.coin-row {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 12px 0;
  border-bottom: 1px solid #eef2fa;
}
.coin-row:last-child {
  border-bottom: none;
}
.rank-badge {
  min-width: 38px;
  height: 38px;
  line-height: 38px;
  text-align: center;
  background: linear-gradient(135deg, #2f6df6, #00b8ff);
  color: #ffffff;
  font-weight: 800;
  border-radius: 12px;
}
.coin-info {
  display: flex;
  flex-direction: column;
  flex: 1;
}
.coin-symbol {
  font-size: 17px;
  font-weight: 800;
  color: #1b2a49;
}
.coin-name {
  font-size: 13px;
  color: #7a8aa8;
}
.coin-price {
  font-size: 16px;
  font-weight: 700;
  color: #1b2a49;
}
</style>