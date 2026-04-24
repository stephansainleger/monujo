<template>
  <div id="payment-requests">
    <FoldableSectionCard
      id="payment-requests-list"
      v-if="paymentRequestList.length"
      :isFolded="isFolded"
    >
      <template #title>
        {{ $gettext("My unpaid payment requests") }}
      </template>
      <p class="top-up-info">
        {{
          $gettext("The following payment requests needs to be paid.")
        }}
      </p>
      <TransactionItem
        v-for="paymentRequest in recentPaymentRequestList"
        :key="paymentRequest"
        class="payment-request-item"
        :transaction="paymentRequest"
        @click="openModal(paymentRequest)"
      />
      <div v-if="recentPaymentRequestList.length" class="has-text-centered mt-5">
        <button
          @click="openListModal"
          class="button custom-button custom-inverted"
        >
          {{ $gettext("See more") }}
        </button>
      </div>
    </FoldableSectionCard>
  </div>
</template>

<script lang="ts">
  import { mapGetters } from "vuex"
  import { Options, Vue } from "vue-class-component"

  import { mapModuleState } from "@/utils/vuex"
  import TransactionItem from "./TransactionItem.vue"
  import FoldableSectionCard from "./FoldableSectionCard.vue"
  import { UIError } from "../exception"
  import { showSpinnerMethod, replaceWithLoader } from "@/utils/showSpinner"
  import applyDecorators from "@/utils/applyDecorators"

  @Options({
    name: "PaymentRequests",
    components: {
      TransactionItem,
      FoldableSectionCard,
    },
    props: {
      refreshToggle: Boolean,
      account: Object,
      isFolded: {
        type: Boolean,
        default: false,
      },
    },
    data(this: any) {
      return {
        paymentRequestList: [],
        hasFinishedFirstLoading: false,
      }
    },
    async mounted() {
      await this.fetchPaymentRequestList()
    },
    computed: {
      recentPaymentRequestList() {
        return this.paymentRequestList.slice(0, 5)
      },
      ...mapModuleState("lokapi", ["userProfile"]),
      ...mapGetters(["numericFormat", "relativeDateFormat", "dateFormat"]),
    },

    methods: {
      fetchPaymentRequestList: applyDecorators(
        [
          showSpinnerMethod(function (this: any, isLoading: boolean) {
            if (this.hasFinishedFirstLoading)
              this.$emit("triggerTransactionRefresh", isLoading, this)
            if (!this.hasFinishedFirstLoading && !isLoading) {
              this.hasFinishedFirstLoading = true
            }
          }),
          showSpinnerMethod(function (this: any, isLoading: boolean) {
            if (!this.hasFinishedFirstLoading) {
              return replaceWithLoader.apply(this, ["#payment-requests"])
            }
          }),
        ],
        async function (this: any): Promise<void> {
          try {
            this.paymentRequestList = await this.account._obj.getPaymentRequests(["open", "refused"])
          } catch (err) {
            throw new UIError(
              this.$gettext(
                "An unexpected server error occured while fetching payment requests list"
              ),
              err
            )
          }
        }
      ),
      async openModal(paymentRequest: any) {
        await this.$modal.open("PaymentRequestModal", {
          paymentRequest: paymentRequest,
          account: this.account,
          refreshTransaction: this.refreshTransaction,
          refreshAccounts: this.refreshAccounts,
        })
      },
      async openListModal() {
        await this.$modal.open("PaymentRequestListModal", {
          account: this.account,
          refreshPaymentRequests: this.fetchPaymentRequestList,
          refreshTransaction: this.refreshTransaction,
          refreshAccounts: this.refreshAccounts,
        })
      },
      refreshTransaction() {
        this.$emit("refreshTransaction")
      },
      refreshAccounts() {
        this.$emit("refreshAccounts")
      },
    },
    watch: {
      refreshToggle: async function () {
        this.fetchPaymentRequestList()
      },
    },
  })
  export default class PaymentRequests extends Vue {}
</script>
<style lang="scss" scoped>
  .top-up-info {
    font-style: italic;
  }
</style>
