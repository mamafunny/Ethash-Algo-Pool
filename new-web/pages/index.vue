<template>
  <v-row justify="center" align="center" no-gutters class="pa-0">
    <v-col cols="12" class="pa-0">
      <v-card flat tile class="mb-0">
        <v-img
          height="200"
          :src="require('~/static/' + config.banner)"
          gradient="to top right, rgba(0,0,0,.9), rgba(255,255,201,.33)"
          class="white--text align-end"
        >
          <v-card-title>
            <v-list style="background-color: rgba(0, 0, 0, 0)">
              <v-list-item style="background-color: rgba(0, 0, 0, 0)">
                <v-list-item-avatar>
                  <img :src="require('~/static/' + config.logo)" />
                </v-list-item-avatar>
                <v-list-item-content>
                  <v-list-item-title class="white--text">{{ config.title }}</v-list-item-title>
                  <v-list-item-subtitle class="white--text">{{ config.description }}</v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
            </v-list>
          </v-card-title>
        </v-img>
        <v-alert
          v-if="network.testnet"
          outlined
          text
          dismissible
          tile
          type="warning"
          class="w-100 mb-0"
        >
          {{
            $tc('pages.home.testnetAlert', 0, {
              title: config.network.title,
              symbol: config.network.symbol,
            })
          }}
        </v-alert>
        <v-card-text class="py-1">
          <v-list style="background-color: rgba(0, 0, 0, 0)">
            <v-list-item style="background-color: rgba(0, 0, 0, 0)">
              <v-list-item-avatar>
                <img :src="require('~/static/' + config.network.icon)" />
              </v-list-item-avatar>
              <v-list-item-content>
                <v-list-item-title>{{ config.network.title }}</v-list-item-title>
                <v-list-item-subtitle>{{ config.network.algo }}</v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
          </v-list>
          <ul>
            <li>
              {{
                $tc('pages.home.minimumPayout', 0, {
                  threshold: config.payoutThreshold,
                  symbol: config.network.symbol,
                })
              }}
            </li>
            <li>{{ $tc('pages.home.mode', 0, { mode: config.mode }) }}</li>
            <li>{{ $t('pages.home.protocols') }}</li>
          </ul>
        </v-card-text>
      </v-card>
      <v-card flat tile>
        <v-card-title>
          <v-text-field
            v-model="search"
            append-icon="mdi-magnify"
            :label="$t('pages.home.search')"
            single-line
            outlined
            hide-details
          ></v-text-field>
        </v-card-title>
        <v-data-table
          dense
          flat
          :headers="headers"
          :items="miners"
          :search="search"
          :footer-props="{
            itemsPerPageText: $t('pages.home.minersPerPage'),
            itemsPerPageOptions: [25, 50, 100],
          }"
          :options="{ itemsPerPage: 25 }"
          :items-per-page="-1"
          :no-data-text="$t('pages.home.noMiners')"
        >
          <template #item.account="{ item }">
            <nuxt-link :to="'/account/' + item.account">{{ formatAccountHash(item.account) }}</nuxt-link>
          </template>
          <template #[`item.hashrate`]="{ item }">
            {{ formatHashrate(item.hashrate, true) }}
          </template>
          <template #item.lastBeat="{ item }">
            {{ formatLastBeat(item.lastBeat) }}
          </template>
        </v-data-table>
      </v-card>
    </v-col>
    <v-col cols="12" class="pa-0">
      <v-row no-gutters align="center" justify="space-between">
        <v-col cols="6">
          <div class="chart-title">Pool Hashrate</div>
          <div ref="poolChart"></div>
        </v-col>
        <v-col cols="6">
          <div class="chart-title">Network Difficulty</div>
          <div ref="netChart"></div>
        </v-col>
      </v-row>
    </v-col> 
  </v-row>  
</template>

<script>
import { formatDistance } from 'date-fns'
import ApexCharts from 'apexcharts'
import axios from 'axios'

export default {
  data() {
    return {
      search: '',
    }
  },
  computed: {
    headers() {
      return [
        { text: this.$t('pages.home.account'), align: 'start', value: 'account' },
        { text: this.$t('pages.home.hashrate'), value: 'hashrate' },
        { text: this.$t('pages.home.lastBeat'), align: 'right', value: 'lastBeat' },
      ]
    },
    miners() {
      const obj = this.$store.state.miners
      const arr = []
      for (const miner in obj) {
        arr.push({
          account: miner,
          hashrate: obj[miner].hr,
          lastBeat: obj[miner].lastBeat * 1000,
          offline: obj[miner.offline],
        })
      }
      return arr
    },
    config() {
      return this.$store.state.env
    },
    network() {
      return this.$store.state.env.network
    },
    now() {
      return this.$store.state.now
    },
  },
  mounted() {
    this.fetchPoolChartData()
  },
  methods: {
    formatHashrate(bytes, showHash) {
      const sizes = ['', 'K', 'M', 'G', 'T']
      if (bytes === 0) {
        return 'n/a'
      }
      const i = parseInt(Math.floor(Math.log(bytes) / Math.log(1000)))
      if (i === 0) {
        return bytes + ' ' + sizes[i]
      }
      let unit = ' ' + sizes[i]
      if (showHash) {
        unit = ' ' + sizes[i] + 'H'
      }
      return (bytes / 1000 ** i).toFixed(3) + unit
    },
    formatAccountHash(account) {
      const start = account.substring(0, 10)
      const end = account.substring(account.length - 10)
      return start + '...' + end
    },
    formatLastBeat(time) {
      return formatDistance(new Date(time), this.now, {
        addSuffix: true,
        includeSeconds: true,
      })
    },
    fetchPoolChartData() {
      axios
        .get(this.config.api + '/stats')
        .then(response => {
          const netChartData = response.data.netCharts.map(data => ({
            x: new Date(data.x * 1000).toLocaleString(navigator.language, {
              hour12: false,
              timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
              day: '2-digit',
              month: '2-digit',
              year: '2-digit',
              hour: '2-digit',
              minute: '2-digit'
            }),
            y: data.y
          }));
          const poolChartData = response.data.poolCharts.map(data => ({
            x: new Date(data.x * 1000).toLocaleString(navigator.language, {
              hour12: false,
              timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
              day: '2-digit',
              month: '2-digit',
              year: '2-digit',
              hour: '2-digit',
              minute: '2-digit'
            }),
            y: data.y
          }));
          this.renderChart(netChartData, 'Network Difficulty', '#FF0000', this.$refs.netChart);
          this.renderChart(poolChartData, 'Pool Hashrate', '#00FF00', this.$refs.poolChart);
        })
        .catch(error => {
          console.error('Error fetching pool data:', error);
        });
    },
    renderChart(chartData, name, color, element) {
      const options = {
        chart: {
          type: 'area',
          height: 350,
          toolbar: {
            show: false
          },
          zoom: false
        },
        series: [{
          name: name,
          data: chartData.map(data => ({
            x: data.x,
            y: data.y
          }))
        }],
        colors: [color],
        xaxis: {
          categories: chartData.map(data => data.x),
          labels: {
            style: {
              colors: ['#00FF00']
            }
          }
        },
        yaxis: {
          labels: {
            style: {
              colors: ['#000000']
            },
            formatter: function (value) {
              const sizes = ['H/s', 'KH/s', 'MH/s', 'GH/s', 'TH/s', 'PH/s']
              if (value === 0) {
                return 'n/a'
              }
              const i = Math.floor(Math.log(value) / Math.log(1000))
              if (i === 0) {
                return value + ' ' + sizes[i]
              }
              return (value / 1000 ** i).toFixed(3) + ' ' + sizes[i]
            }
          }
        },
        plotOptions: {
          area: {
            fillColor: '#FFFF00',
            lineColor: '#00FFFF',
          }
        },
        tooltip: {
          enabled: true,
          style: {
            colors: ['#FFFFFF']
          },
          theme: 'dark'
        },
        dataLabels: {
          enabled: false
        }
      };

      const chart = new ApexCharts(element, options);
      chart.render();
    }
  }
}
</script>

<style>
/* Your custom styles here */
</style>
