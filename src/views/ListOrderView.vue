<template>
  <main style="padding: 2rem">
    <div style="display:grid; grid-template-columns: 2fr 1fr 1fr">
      <v-text-field v-model="search" label="Search"></v-text-field>
      <v-text-field v-model="startDate" label="Start Date" type="date" style="margin-left: 1rem"></v-text-field>
      <v-text-field v-model="endDate" label="End Date" type="date" style="margin-left: 1rem"></v-text-field>
    </div>
    <div style="display: flex; margin-bottom: 2rem;">
      <v-btn @click="filterClicked">Filter</v-btn>
      <v-btn @click="resetClicked" style="margin-left: 1rem">Reset</v-btn>
    </div>
    <v-data-table-server v-model:page="page" v-model:items-per-page="itemsPerPage"
      :items-per-page-options="itemPerPageOptions" v-model:sort-by="sortBy" :search="searchID" :headers="headers"
      :items-length="totalItems" :items="orders" :loading="loading" item-value="name" @update:options="updateOptions"
      show-current-page>
    </v-data-table-server>
  </main>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { watch } from 'vue';
import * as config from '../config'
import moment from 'moment-timezone';

const headers: any[] = [
  { title: 'Order name', key: 'orderName', align: 'start', sortable: false },
  { title: 'Customer Company', key: 'customerCompanyName', align: 'end', sortable: false },
  { title: 'Customer Name', key: 'customerName', align: 'end', sortable: false },
  { title: 'Order Date', key: 'orderDate', align: 'end' },
  { title: 'Delivered Amount (A$)', key: 'deliveredAmount', align: 'end', sortable: false },
  { title: 'Total Amount (A$)', key: 'totalAmount', align: 'end', sortable: false },
];

type SortItem = {
  key: string;
  order?: boolean | 'asc' | 'desc';
};

type ItemPerPageOption = {
  title: string;
  value: number;
}

var page = defineModel<number>("page", { default: 1 })
var itemsPerPage = defineModel<number>("itemPerPage", { default: 5 })
var itemPerPageOptions = ref<ItemPerPageOption[]>([{ title: "5", value: 5 }, { title: "10", value: 10 }, { title: "25", value: 25 }])
var sortBy = defineModel<SortItem[]>("sort")

var loading = ref<boolean>()
var orders = ref<Order[]>([])
var totalItems = defineModel<number>("totalItems", { default: 0 })

var search = defineModel<string>("name", { default: "" })
var startDate = defineModel<string>("startDate", { default: "" })
var endDate = defineModel<string>("endDate", { default: "" })

var searchID = ref<string>("")
var prevCursor = ref<string>("")
var currCursor = ref<string>("")
var currFilter = ref<{ search: string, startDate: string, endDate: string }>({ search: "", startDate: "", endDate: "" })
var hasNext = ref<boolean>(false)

interface Order {
  orderName: string
  customerCompanyName: string
  customerName: string
  orderDate: string
  deliveredAmount: string
  totalAmount: string
}

watch(sortBy, () => {
  currCursor.value = ""
  page.value = 1
})

watch(itemsPerPage, () => {
  currCursor.value = ""
  page.value = 1
})

watch(hasNext, (value) => {
  if (page.value == undefined || itemsPerPage.value == undefined) {
    return
  }

  totalItems.value = page.value * itemsPerPage.value + (value == true ? 1 : 0)
})

const filterClicked = () => {
  currCursor.value = ""
  currFilter.value = {
    search: search.value,
    startDate: startDate.value,
    endDate: endDate.value
  }

  searchID.value = String(Date.now())
}

const resetClicked = () => {
  search.value = ""
  startDate.value = ""
  endDate.value = ""
  page.value = 1
  itemsPerPage.value = 5

  currCursor.value = ""
  currFilter.value = { search: "", startDate: "", endDate: "" }
  sortBy.value = []

  searchID.value = String(Date.now())
}

const fetchOrders: (() => Promise<{ orders: Order[], nextCursor: string }>) = async () => {
  var startDate = undefined
  if (currFilter.value.startDate != "") {
    startDate = moment(currFilter.value.startDate).tz(config.TIME_ZONE, true).toISOString()
  }

  var endDate = undefined
  if (currFilter.value.endDate != "") {
    endDate = moment(currFilter.value.endDate).hour(23).minute(59).second(59).tz(config.TIME_ZONE, true).toISOString()
  }

  const request: RequestInit = {
    method: "POST",
    body: JSON.stringify({
      filters: {
        search: currFilter.value.search,
        startDate,
        endDate
      },
      pagination: {
        cursor: currCursor.value,
        size: itemsPerPage.value,
      },
      order_type: sortBy.value == undefined || sortBy.value[0]?.order != "desc" ? 1 : 2
    }),
    headers: {
      'Content-Type': 'application/json',
    },
  }

  try {
    const res = await fetch(config.BASE_URL + "/orders:list", request)
    const body: { orders: any[] | undefined, pagination: { nextCursor: string } | undefined } = await res.json()

    if (body.orders == undefined || body.orders.length == 0) {
      return { orders: [], nextCursor: "" }
    }

    const orders = body.orders.map<Order>(o => {
      var orderName = o.orderName
      if (o.products != undefined && o.products.length != 0) {
        orderName += " - " + o.products.join(", ")
      }

      return {
        orderName: orderName,
        customerCompanyName: o.customerCompanyName,
        customerName: o.customerName,
        orderDate: moment(o.orderDate).tz(config.TIME_ZONE).toString(),
        deliveredAmount: (+(o.deliveredAmount * config.SMALLEST_MONEY_UNIT).toFixed(4)).toString(),
        totalAmount: (+(o.totalAmount * config.SMALLEST_MONEY_UNIT).toFixed(4)).toString(),
      }
    })

    const nextCursor = body.pagination == undefined ? "" : body.pagination.nextCursor
    return { orders, nextCursor }
  } catch (e) {
    console.log(e)
  }

  return { orders: [], nextCursor: "" }
}

const updateOptions = async () => {
  if (loading.value == true) {
    return
  }

  loading.value = true
  const { orders: serverOrders, nextCursor } = await fetchOrders()

  orders.value = serverOrders
  prevCursor.value = currCursor.value
  currCursor.value = nextCursor
  hasNext.value = nextCursor == "" ? false : true
  loading.value = false
}
</script>

<style>
.v-data-table-footer__info {
  display: none!important
}
</style>