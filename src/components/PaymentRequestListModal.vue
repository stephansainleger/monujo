<template>
  <div
    class="modal is-active"
    v-if="$modal.modal.value == $options.name"
    ref="paymentRequests"
  >
    <div class="modal-background"></div>
    <div class="modal-card">
      <header class="modal-card-head">
        <p class="modal-card-title is-title-shrink">
          <span class="ml-2">{{ $gettext("All unpaid payment requests") }}</span>
        </p>
        <button
          class="delete"
          aria-label="close"
          @click="$modal.back()"
        ></button>
      </header>
      <div class="filter-area">
        <div class="ml-2 mt-2">
          <div
            class="
              is-flex-direction-column
              is-align-items-center
              is-justify-content-space-between
              mb-2
            "
          >
            <div class="mb-1">
              <strong>{{ $gettext("Select timespan:") }}</strong>
            </div>
            <div class="datepicker-export">
              <date-picker
                v-model:value="filterDate"
                :open="datePickerShow ? true : null"
                range
                prefix-class="xmx"
                :editable="false"
                :placeholder="$gettext('All payment requests')"
                @clear="
                  () => {
                    selectedTimeSpanType = ''
                    datePickerShow = false
                  }
                "
                @change="datePickerShow = selectedTimeSpanType ? true : false"
                @pick="selectedTimeSpanType = ''"
                :disabled-date="disabledDates"
              >
                <template #header="{ emit }">
                  <div>
                    <div
                      v-for="selector in selectorsOrder"
                      :key="selector"
                      :class="{
                        selected: selector == selectedTimeSpanType,
                      }"
                      class="timespan"
                    >
                      <button
                        class="xmx-btn xmx-btn-text"
                        @click="
                          () => {
                            selectedTimeSpanOffset =
                              selectedTimeSpanType != selector
                                ? -1
                                : selectedTimeSpanOffset - 1
                            selectedTimeSpanType = selector
                            emit(selectedTimeSpan)
                          }
                        "
                      >
                        <i class="xmx-icon-left"></i>
                      </button>
                      <button
                        class="xmx-btn xmx-btn-text"
                        @click="
                          () => {
                            selectedTimeSpanType = selector
                            selectedTimeSpanOffset = 0
                            emit(selectedTimeSpan)
                          }
                        "
                      >
                        {{ selectorLabels[selector] }}
                      </button>
                      <button
                        class="xmx-btn xmx-btn-text"
                        @click="
                          ;[selectedTimeSpanOffset++, emit(selectedTimeSpan)]
                        "
                        :class="{
                          hide:
                            selectedTimeSpanType != selector ||
                            isSelectionCurrent,
                        }"
                      >
                        <i class="xmx-icon-right"></i>
                      </button>
                      <button
                        class="xmx-btn xmx-btn-text confirm"
                        @click="datePickerShow = false"
                        :class="{ hide: selectedTimeSpanType != selector }"
                      >
                        {{ $gettext("confirm") }}
                      </button>
                    </div>
                  </div>
                </template>
              </date-picker>
            </div>
            <div class="mb-1 mt-3">
              <strong>{{ $gettext("Select recipient:") }}</strong>
            </div>
            <div class="recipient-filter is-flex is-flex-direction-row">
              <div class="recipient-filter-input">
                <model-list-select
                  :list="
                    recipientBatchLoader.elements.map((r, idx) => ({
                      name: r.name,
                      idx,
                    }))
                  "
                  option-value="idx"
                  option-text="name"
                  v-model="selectedRecipientIdx"
                  :placeholder="$gettext('All recipients')"
                  @searchchange="onRecipientSearch"
                  id="recipientSelector"
                >
                </model-list-select>
              </div>
              <div>
                <button
                  class="recipient-filter-reset"
                  :class="{ disable: selectedRecipientIdx === null }"
                  @click="selectedRecipientIdx = null"
                >
                  <fa-icon
                    class="refreshing"
                    v-if="recipientBatchLoader.isNewBatchLoading"
                    icon="sync"
                  ></fa-icon>
                  <fa-icon
                    v-else-if="selectedRecipientIdx !== null"
                    icon="fa-xmark"
                  >
                  </fa-icon>
                  <fa-icon v-else icon="fa-user"></fa-icon>
                </button>
              </div>
            </div>
          </div>
        </div>
        <div class="container is-fluid custom-heavy-line-separator"></div>
      </div>
      <section
        class="modal-card-body"
        ref="paymentRequestsContainer"
        @scroll="onScroll"
      >
        <div
          class="
            custom-card
            is-flex-direction-column
            is-align-items-center
            is-justify-content-space-between
            mb-4
          "
        >
          <TransactionItem
            v-for="paymentRequest in filteredPaymentRequests"
            :key="paymentRequest"
            :transaction="paymentRequest"
            @click="openPaymentRequestModal(paymentRequest)"
          />
          <div
            v-if="filteredPaymentRequests.length === 0"
            class="is-flex is-align-items-center is-justify-content-center"
          >
            {{ $gettext("No payment request found") }}
          </div>
        </div>
      </section>
      <footer
        class="modal-card-foot custom-modal-card-foot is-justify-content-end"
      >
        <button
          class="button custom-button-modal has-text-weight-medium"
          @click="$modal.back()"
        >
          {{ $gettext("Close") }}
        </button>
      </footer>
    </div>
  </div>
</template>

<script lang="ts">
  import { Options, Vue } from "vue-class-component"
  import DatePicker from "vue-datepicker-next"
  import { ModelListSelect } from "vue-search-select"
  import moment from "moment"

  import TransactionItem from "./TransactionItem.vue"
  import UseBatchLoading from "@/services/UseBatchLoading"
  import { showSpinnerMethod } from "@/utils/showSpinner"
  import applyDecorators from "@/utils/applyDecorators"

  import "vue-datepicker-next/index.css"
  import "vue-search-select/dist/VueSearchSelect.css"
  import "@/assets/datepicker.scss"

  @Options({
    name: "PaymentRequestListModal",
    components: {
      DatePicker,
      ModelListSelect,
      TransactionItem,
    },
    data(this: any) {
      return {
        filterDate: ["", ""],
        datePickerShow: false,
        selectorLabels: {
          day: this.$gettext("day"),
          week: this.$gettext("week"),
          month: this.$gettext("month"),
          year: this.$gettext("year"),
        },
        selectorsOrder: ["day", "week", "month", "year"],
        selectedTimeSpanType: "",
        selectedTimeSpanOffset: 0,
        selectedRecipientIdx: null,
        recipientBatchLoader: null,
        paymentRequestList: [],
      }
    },
    created() {
      const [opts] = this.$modal.args.value
      this.modalArgs = opts
      this.account = opts.account
      const backend = this.account._obj.parent

      this.recipientBatchLoader = UseBatchLoading({
        genFactory: backend.searchRecipients.bind(backend),
        needMorePredicate: () =>
          this.$recipients.scrollHeight -
            (this.$recipients.scrollTop + this.$recipients.offsetHeight) <=
          50,
        onError: (e) => {
          this.$msg.error(
            this.$gettext(
              "An unexpected issue occured while downloading recipient list"
            )
          )
          throw e
        },
      })
    },
    mounted() {
      this.$refs.paymentRequests.focus()
      const $recipients = this.$el.querySelector(".menu")

      $recipients.addEventListener(
        "scroll",
        this.recipientBatchLoader.getNextElements.bind(
          this.recipientBatchLoader
        )
      )
      this.$recipients = $recipients
      this.recipientBatchLoader.newGen("")
      this.fetchPaymentRequestList()
    },
    computed: {
      isSelectionCurrent(): boolean {
        return moment().isBetween(this.filterDate[0], this.filterDate[1])
      },
      selectedTimeSpan() {
        const now = moment().toDate()
        const timeSpanType = this.selectedTimeSpanType
        const offset = this.selectedTimeSpanOffset
        const dateSelected = moment(now)
          .subtract(-offset, timeSpanType)
          .toDate()
        const [begin, end] = [
          moment(dateSelected).startOf(timeSpanType),
          moment(dateSelected).endOf(timeSpanType),
        ].map((m) => m.toDate())

        return [begin, now < end ? now : end]
      },
      filteredPaymentRequests(): any[] {
        const [dateBegin, dateEnd] = this.filterDate
        const selectedRecipientName =
          this.recipientBatchLoader.elements[this.selectedRecipientIdx]?.name

        return this.paymentRequestList.filter((paymentRequest: any) => {
          if (dateBegin && paymentRequest.date < dateBegin) return false
          if (dateEnd && paymentRequest.date > dateEnd) return false
          if (
            selectedRecipientName &&
            selectedRecipientName !== paymentRequest.related
          )
            return false
          return true
        })
      },
    },
    methods: {
      fetchPaymentRequestList: applyDecorators(
        [showSpinnerMethod(".modal-card-body")],
        async function (this: any): Promise<void> {
          try {
            this.paymentRequestList = (
              await this.account._obj.getPaymentRequests(["open", "refused"])
            ).map((paymentRequest: any) => {
              paymentRequest.currency = this.account.curr
              return paymentRequest
            })
          } catch (err) {
            this.$msg.error(
              this.$gettext(
                "An unexpected server error occured while fetching payment requests list"
              )
            )
            throw err
          }
        }
      ),
      async refreshPaymentRequests() {
        await this.fetchPaymentRequestList()
        if (this.modalArgs.refreshPaymentRequests) {
          await this.modalArgs.refreshPaymentRequests()
        }
        if (this.modalArgs.refreshTransaction) {
          this.modalArgs.refreshTransaction()
        }
      },
      refreshAccounts() {
        if (this.modalArgs.refreshAccounts) {
          this.modalArgs.refreshAccounts()
        }
      },
      async openPaymentRequestModal(paymentRequest: any) {
        await this.$modal.open("PaymentRequestModal", {
          paymentRequest,
          account: this.account,
          refreshTransaction: this.refreshPaymentRequests,
          refreshAccounts: this.refreshAccounts,
        })
      },
      onScroll() {},
      disabledDates(date: Date) {
        return date > moment().endOf("day").toDate()
      },
      async onRecipientSearch(recipientsSearchString: any) {
        if (
          this.selectedRecipientIdx !== null &&
          recipientsSearchString === ""
        ) {
          return
        }
        if (
          recipientsSearchString.length > 2 ||
          recipientsSearchString.length === 0
        ) {
          this.recipientBatchLoader.newGen(recipientsSearchString)
        }
      },
    },
    watch: {
      selectedRecipientIdx: async function (): Promise<void> {
        this.onRecipientSearch("")
      },
      filterDate: async function (newFilterDate): Promise<void> {
        const [newBegin, newEnd] = newFilterDate
        const [normBegin, normEnd] = [
          newBegin ? moment(newBegin).startOf("day").toDate() : null,
          newEnd ? moment(newEnd).endOf("day").toDate() : null,
        ]
        if (
          normBegin &&
          normEnd &&
          (+newBegin != +normBegin || +newEnd != +normEnd)
        ) {
          this.filterDate = [normBegin, normEnd]
        }
      },
    },
  })
  export default class PaymentRequestListModal extends Vue {}
</script>

<style lang="scss">
  @import "@/assets/custom-variables";

  section.modal-card-body {
    padding: 0.5em;
  }

  div.selected {
    background-color: $color-1;
  }

  div.timespan {
    padding: 0;
    margin: 0;
    border-radius: 2em;
    width: 15em;
    display: grid;
    grid-template-columns: 2em 5em 2em 6em;

    button.xmx-btn {
      text-align: center;
      border-radius: 2em;

      &.confirm {
        margin-left: 1em;
        &,
        &:hover {
          background-color: $color-2;
          color: $color-1;
        }
      }
    }
  }

  .datepicker-export {
    .xmx-datepicker-range {
      width: auto !important;
    }
  }

  div.xmx-datepicker-content {
    user-select: none;
  }

  .filter-area {
    background: #f0faf9;
  }

  .recipient-filter {
    align-items: stretch;
    padding: 0.1em 2em 0.5em 0.5em;
  }

  .recipient-filter-reset {
    height: 100%;
    padding: 0 1em;
    border: 0;
    cursor: pointer;

    &.disable {
      cursor: initial;
    }
  }

  .recipient-filter-input {
    flex-grow: 1;
  }
</style>
