<template>
  <v-container>
    <v-row>
      <v-col cols="12" md="6">
        <v-card>
          <v-card-title>
            <v-icon>mdi-calendar-range</v-icon>
            <span class="text-h5">
              Expenses Calendar
            </span>
          </v-card-title>
          <v-card-text>
            <calendar-heatmap :dark-mode="theme.global.name === 'dark'" :values="calendarValues" :end-date="today" tooltip-unit="€" no-data-text="no purchases this day" @day-click="handleDayClick" />
            <span class="text-h5">
              Part of organic products
            </span>
            <calendar-heatmap :dark-mode="theme.global.name === 'dark'" :range-color="organicColorRange" :values="calendarOrganicValues" :end-date="today" tooltip-unit="% of organic products" no-data-text="no purchases this day" @day-click="handleDayClick" />
          </v-card-text>
        </v-card>
      </v-col>
      <v-col cols="12" md="6">
        <v-card>
          <v-card-title>
            <v-icon>mdi-chart-bar</v-icon>
            <span class="text-h5">
              Overall part of organic products
            </span>
          </v-card-title>
          <v-card-text>
            <v-progress-linear
              v-model="totalOrganicPercentage"
              :color="totalOrganicPercentage > 50 ? 'success' : 'info'"
              height="25"
            >
              <strong>{{ totalOrganicPercentage }}%</strong>
            </v-progress-linear>
            <strong>{{ totalOrganicPriceSum.toFixed(2) }} €</strong> of <strong>{{ totalPricesSum.toFixed(2) }} €</strong>
          </v-card-text>
        </v-card>
        <v-card>
          <v-card-title>
            <v-icon>mdi-cash-multiple</v-icon>
            <span class="text-h5">
              Total expenses in the last 31 days
            </span>
          </v-card-title>
          <v-card-text>
            <strong>{{ sumInMonth }} €</strong> ({{ diffSumPreviousMonthPercent }}% compared to the previous month)
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
    <v-row>
      <v-col v-for="price in priceList" :key="price" cols="12" sm="6" md="4" xl="3">
        <PriceCard :price="price" :product="price.product" elevation="1" height="100%" />
      </v-col>
    </v-row>
    <v-row v-if="loading">
      <v-col align="center">
        <v-progress-circular indeterminate :size="30" />
      </v-col>
    </v-row>
  </v-container>
</template>
<script>
import { defineAsyncComponent } from 'vue'
import { mapStores } from 'pinia'
import { useAppStore } from '../store'
import api from '../services/api'
import { CalendarHeatmap } from 'vue3-calendar-heatmap'
import 'vue3-calendar-heatmap/dist/style.css'
import { useTheme } from 'vuetify'

export default {
  components: {
    CalendarHeatmap,
    PriceCard: defineAsyncComponent(() => import('../components/PriceCard.vue'))
  },
  data() {
    return {
      calendarValues: [],
      calendarOrganicValues: [],
      today: new Date(),
      priceList: [],
      loading: false,
      totalOrganicPercentage: 0,
      sumInMonth: 0,
      diffSumPreviousMonthPercent: 0,
      totalOrganicPriceSum: 0,
      totalPricesSum: 0,
      theme: useTheme()
    }
  },
  computed: {
    ...mapStores(useAppStore),
    organicColorRange() {
      return this.theme.global.name === 'dark' ? 
        ['#1f1f22', '#21452a', '#236a30', '#259037', '#27b53d', '#29db44'] 
        : ['#ebedf0', '#c4e9ce', '#9de6ab', '#77e289', '#50df66', '#29db44']
    }
  },
  mounted() {
    this.getUserStats()
  },
  methods: {
    getUserStats() {
      api.getPricesGroupedStats({ group_by: "date", user: this.appStore.user.username })
        .then((data) => {
          let sumInPreviousMonth = 0
          let monthDate = new Date()
          monthDate.setDate(monthDate.getDate() - 31)
          let previousMonthDate = new Date()
          previousMonthDate.setDate(previousMonthDate.getDate() - 62)
          for (let i = 0; i < data.items.length; i++) {
            data.items[i].count = data.items[i].price__sum
            if (data.items[i].date > monthDate.toISOString()) {
              this.sumInMonth += data.items[i].price__sum
            } else if (data.items[i].date > previousMonthDate.toISOString()) {
              sumInPreviousMonth += data.items[i].price__sum
            }
          }
          const _diffSumPreviousMonthPercent = Math.round((this.sumInMonth - sumInPreviousMonth) * 100 / this.sumInMonth)
          if (_diffSumPreviousMonthPercent > 0) {
            this.diffSumPreviousMonthPercent = "+"
          }
          this.diffSumPreviousMonthPercent += _diffSumPreviousMonthPercent
          this.calendarValues = data.items
          this.getOrganicStats()
        })
    },
    async getOrganicStats() {
      const dailyOrganicProducts = await api.getPricesGroupedStats({ group_by: "date", product_labels_tags__contains: "organic", user: this.appStore.user.username })
      const dailyOrganicCategories = await api.getPricesGroupedStats({ group_by: "date", labels_tags__contains: "organic", user: this.appStore.user.username })

      this.calendarOrganicValues = this.calendarValues.map(item => {
        let organicProductsSum = dailyOrganicProducts.items.find(dailyitem => dailyitem.date === item.date)?.price__sum || 0
        let organicCategoriesSum = dailyOrganicCategories.items.find(dailyitem => dailyitem.date === item.date)?.price__sum || 0
        this.totalOrganicPriceSum += organicProductsSum + organicCategoriesSum
        this.totalPricesSum += item.price__sum
        return {
          ...item,
          organicProductsSum: organicProductsSum,
          organicCategoriesSum: organicCategoriesSum,
          count: Math.round((organicProductsSum + organicCategoriesSum) * 100 / item.price__sum)
        }
      })
      this.totalOrganicPercentage = Math.round(this.totalOrganicPriceSum * 100 / this.totalPricesSum)
      
    },
    handleDayClick(event) {
      this.priceList = []
      this.loading = true
      console.log(event.date)
      const offset = event.date.getTimezoneOffset()
      const date = new Date(event.date.getTime() - (offset*60*1000))
      api.getPrices({ date: date.toISOString().split('T')[0], user: this.appStore.user.username })
        .then((data) => {
          this.priceList = data.items
          this.loading = false
        })
    }
  }
}
</script>