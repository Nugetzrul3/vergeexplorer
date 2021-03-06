<template>
  <div>
    <Heading :heading="heading" />

    <Alert v-if="isError" :error="error" />

    <template v-else>
      <ProgressCircular v-if="isRichlistLoading"></ProgressCircular>

      <template v-else>
        <v-layout align-center justify-center>
          <p>Wealth distribution of the richest addresses in the blockchain by balance</p>
        </v-layout>

        <v-data-table
          :headers="headers"
          :items="richlist"
          :pagination.sync="pagination"
          hide-actions
          class="elevation-3 hidden-sm-and-down"
        >
          <template v-slot:items="props">
            <td class="body-2">#{{ props.index + 1 }}</td>
            <td class="body-2 monospace">
              <router-link
                :to="{ name: 'address', params: { address: props.item.address }}"
              >{{ props.item.address }}</router-link>
              <v-progress-circular class="ml-4" :value="props.item.percentage" width="10"></v-progress-circular>
            </td>
            <td
              class="body-2 text-xs-right"
              :inner-html.prop="props.item.balance | formatAmount | formatMuted"
            ></td>
            <td class="body-2 text-xs-right warning--text">
              <v-progress-circular dark indeterminate v-if="isPriceLoading"></v-progress-circular>
              {{ props.item.usd | formatUSD }}
            </td>
            <td class="body-2 text-xs-right accent--text">
              <v-progress-circular dark indeterminate v-if="isInfoLoading"></v-progress-circular>
              {{ props.item.percentage }}
            </td>
          </template>
        </v-data-table>

        <v-data-iterator
          :items="richlist"
          hide-actions
          content-tag="v-layout"
          row
          wrap
          hidden-md-and-up
        >
          <template v-slot:item="props">
            <v-flex xs12 class="py-1">
              <v-card>
                <v-list-tile>
                  <v-list-tile-sub-title class="text-xs-center accent--text subheading">
                    #{{ props.index + 1 }}
                    <v-progress-circular :value="props.item.percentage" width="10" class="mx-4"></v-progress-circular>
                    <v-progress-circular dark indeterminate v-if="isInfoLoading"></v-progress-circular>
                    {{ props.item.percentage }} %
                  </v-list-tile-sub-title>
                </v-list-tile>
                <v-divider></v-divider>
                <v-list dense>
                  <v-card-title>
                    <v-tooltip top open-delay="0" close-delay="0">
                      <template v-slot:activator="{ on }">
                        <div v-on="on">
                          <router-link
                            :to="{ name: 'address', params: { address: props.item.address }}"
                            class="monospace break-all"
                          >{{ props.item.address }}</router-link>
                        </div>
                      </template>
                      <span>Address</span>
                    </v-tooltip>
                  </v-card-title>

                  <v-divider></v-divider>

                  <v-card-title class="py-2">
                    <v-layout justify-space-between row wrap>
                      <div class="subheading grey--text">Balance</div>
                      <div
                        class="subheading break-all pl-3"
                      >{{ props.item.balance | formatAmount }} {{ $CURRENCY }}</div>
                    </v-layout>
                  </v-card-title>

                  <v-card-title class="py-2">
                    <v-layout justify-space-between row wrap>
                      <div class="subheading mr-5 grey--text">Estimated Worth</div>
                      <div class="subheading break-all warning--text pl-3">
                        <v-progress-circular dark indeterminate v-if="isPriceLoading"></v-progress-circular>
                        {{ props.item.usd | formatUSD }}
                      </div>
                    </v-layout>
                  </v-card-title>
                </v-list>
              </v-card>
            </v-flex>
          </template>
        </v-data-iterator>
      </template>
    </template>
  </div>
</template>

<script>
import Heading from "../components/Heading.vue";
import ProgressCircular from "../components/ProgressCircular.vue";
import Alert from "../components/Alert.vue";

import { getMarketData, getInfo } from "../mixins.js";

export default {
  mixins: [getMarketData, getInfo],
  components: {
    Heading,
    ProgressCircular,
    Alert
  },
  data: () => ({
    heading: {
      title: "Richlist",
      icon: "fas fa-chart-pie"
    },
    headers: [
      { text: "#", value: "index", sortable: false },
      { text: "Address", value: "address", sortable: false },
      {
        text: "Balance",
        value: "balance",
        sortable: false,
        align: "right"
      },
      {
        text: "Estimated Worth",
        value: "usd",
        sortable: false,
        align: "right"
      },
      { text: "%", value: "percentage", sortable: false, align: "right" }
    ],
    pagination: {
      sortBy: "balance",
      descending: true,
      rowsPerPage: -1
    },
    richlist: [],
    error: "There was an error.",
    isError: false,
    isRichlistLoading: true,
    isInfoLoading: true,
    isPriceLoading: true
  }),
  async created() {
    try {
      const balanceHeader = this.headers.find(item => item.value === "balance");
      balanceHeader.text += ` (${this.$CURRENCY})`;

      this.richlist = await this.getRichlist();

      this.isRichlistLoading = false;

      const [info, marketData] = await Promise.all([
        this.getInfo(),
        this.getMarketData()
      ]);

      this.richlist = this.richlist.map(item => ({
        ...item,
        percentage: ((item.balance / info.moneysupply) * 100).toFixed(2),
        usd: (item.balance * marketData.usd).toFixed(2)
      }));

      this.isInfoLoading = false;
      this.isPriceLoading = false;
    } catch (error) {
      this.isError = true;
    } finally {
      this.isRichlistLoading = false;
      this.isInfoLoading = false;
      this.isPriceLoading = false;
    }
  },
  methods: {
    async getRichlist() {
      const {
        data: { data: richlist }
      } = await this.$http.get("/api/richlist");

      return richlist;
    }
  }
};
</script>

<style>
td {
  padding: 0 1em !important;
  height: 3em !important;
}

tbody tr:nth-of-type(even) {
  background-color: rgba(0, 0, 0, 0.1) !important;
}

tbody tr:nth-of-type(odd) {
  background-color: rgba(0, 0, 0, 0.3) !important;
}
</style>
