<template lang="pug">
  .ga-services
    .ga-services__wrapper
      ui-debio-error-dialog(
        :show="!!error"
        :title="error ? error.title : ''"
        :message="error ? error.message : ''"
        @close="error = null"
      )
      
      WarningDialog(
        :show="showActiveOrder"
        title="Unfinished Order"
        btnMessage="Go to Dashboard"
        message="You still have active orders to complete. Resume any pending orders before continuing with this process."
        @close="showActiveOrder = false"
        @submit="goToDashboard"
      )

      ui-debio-modal.ga-services__modal-card(
        :show="showModal"
        :show-title="false"
        :show-cta="false"
        @onClose="showModal = false"
      )

        .ga-services__modal-title Delete Service
        ui-debio-icon(:icon="alertTriangleIcon" stroke size="88")
        .ga-services__modal-desc By delete this Service, you might not be able to retrieve it back
        
        .ga-services__tx-weight
          .ga-services__tx-weight-text Estimated transaction weight   
            v-tooltip.visible(right)
              template(v-slot:activator="{ on, attrs }")
                v-icon.dialog-confirmation__trans-weight-icon(
                  style="font-size: 12px;"
                  color="primary"
                  dark
                  v-bind="attrs"
                  v-on="on"
                ) mdi-alert-circle-outline
              span(style="font-size: 10px;") Total fee paid in DBIO to execute this transaction.

          span( style="font-size: 12px;" ) {{ txWeight }}

        .ga-services__modal-button
          ui-debio-button(
            block
            :loading="isLoading"
            color="secondary"
            @click="onDelete"
          ) Delete


      ui-debio-banner.ga-services__banner(
        title="Genetic Analyst"
        subtitle="Provide service for customer who have been tested previously can get interpretation that leads to improved maintenance."
        gradient-color="tertiary"
        with-decoration
      )
        template(slot="illustration")
          ui-debio-icon(
            :icon="geneticAnalystIllustration"
            :size="cardBlock ? 250 : 120"
            view-box="0 0 120 120"
            fill
          )

        template(slot="cta")
          ui-debio-card(
            :to="{ name: 'ga-add-services'}"
            title="Add Services"
            sub-title="Add Genetic Services"
            tiny-card 
            with-icon
            width="250"
            :block="cardBlock"
          )
            ui-debio-icon(:icon="plusCircle" slot="icon" size="34" color="#C400A5" fill)


      .ga-services__table
        ui-debio-data-table(:headers="headers" :items="items" :loading="isLoading")
          template(slot="prepend")
            .ga-services__text
              h2.ga-services__table-title My Services
              p.ga-services__table-subtitle.mb-0 List of all Services

          template(v-slot:[`item.serviceName`]="{ item }")
            .d-flex.flex-column.ga-services__service-name
              span {{ item.serviceName }}

          template(v-slot:[`item.description`]="{ item }")
            .d-flex.flex-column.ga-services__description
              span {{ item.description }}

          template(v-slot:[`item.price`]="{ item }")
            .d-flex.flex-column.ga-services__price
              span {{ item.price }} {{ formatUSDTE(item.currency) }}

          template(v-slot:[`item.duration`]="{ item }")
            .d-flex.flex-column.ga-services__duration
              span {{ item.duration }}

          template(v-slot:[`item.action`]="{ item }")
            .ga-services__action
              router-link(:to="{ name: 'ga-edit-service', params: { id: item.id, item } }")
                ui-debio-icon(:icon="pencilIcon" size="16" role="button" stroke)
              ui-debio-icon(:icon="trashIcon" size="16" role="button" stroke @click="showDialog(item.id)")



</template>

<script>
import {mapState} from "vuex"

import {
  geneticAnalystIllustration,
  plusCircle,
  pencilIcon,
  trashIcon,
  alertTriangleIcon
} from "@debionetwork/ui-icons"

import {errorHandler} from "@/common/lib/error-handler"
import {queryGeneticAnalystByAccountId} from "@debionetwork/polkadot-provider"
import {queryGeneticAnalystServicesByHashId} from "@debionetwork/polkadot-provider"
import {
  deleteGeneticAnalystService,
  deleteGeneticAnalystServiceFee
} from "@debionetwork/polkadot-provider"
import {GAGetOrders} from "@/common/lib/api"
import { formatUSDTE } from "@/common/lib/balance-format.js"

import WarningDialog from "@/common/components/Dialog/WarningDialog"

export default {
  name: "GAServices",

  components: {WarningDialog},

  data: () => ({
    geneticAnalystIllustration,
    plusCircle,
    pencilIcon,
    trashIcon,
    alertTriangleIcon,
    formatUSDTE,
    error: "",
    txWeight: "",
    serviceId: "",

    showModal: false,
    showActiveOrder: false,
    cardBlock: false,
    isLoading: false,
    serviceList: [],
    items: [],
    orders: [],
    headers: [
      {
        text: "Service Name",
        value: "serviceName",
        sortable: true
      },
      {
        text: "Description",
        value: "description",
        sortable: true
      },
      {
        text: "Total Price",
        value: "price",
        sortable: true
      },
      {
        text: "Service Duration",
        value: "duration",
        sortable: true
      },
      {
        text: "Action",
        value: "action",
        align: "center",
        sortable: false
      }
    ]
  }),

  computed: {
    ...mapState({
      lastEventData: (state) => state.substrate.lastEventData,
      api: (state) => state.substrate.api,
      wallet: (state) => state.substrate.wallet,
      web3: (state) => state.web3Store.web3
    })
  },

  watch: {
    lastEventData(e) {
      if (e !== null) {
        const dataEvent = JSON.parse(e.data.toString())
        if (dataEvent[1] === this.wallet.address) {
          const toRefetch = [
            "GeneticAnalystServiceDeleted",
            "GeneticAnalystServiceUpdated",
            "GeneticAnalystServiceCreated"
          ]

          if (toRefetch.includes(e.method)) {
            this.getServiceList()
            this.showModal = false
            this.isLoading = false
            this.serviceId = ""
          }
        }
      }
    }
  },

  async mounted() {
    await this.getGAOrders()
    await this.getServiceList()
  },

  methods: {
    async getDeleteServiceFee() {
      this.txWeight = "Calculating..."
      const txWeight = await deleteGeneticAnalystServiceFee(
        this.api,
        this.wallet,
        this.serviceId
      )
      this.txWeight = `${Number(
        this.web3.utils.fromWei(String(txWeight.partialFee), "ether")
      ).toFixed(4)} DBIO`
    },

    async getServiceList() {
      this.items = []
      const analystDetail = await queryGeneticAnalystByAccountId(
        this.api,
        this.wallet.address
      )
      this.serviceList = analystDetail.services

      for (const serviceId of this.serviceList) {
        try {
          const serviceDetail = await queryGeneticAnalystServicesByHashId(
            this.api,
            serviceId
          )
          const {
            id,
            info: {description, name: serviceName, testResultSample}
          } = serviceDetail

          const currency = serviceDetail.info.pricesByCurrency[0].currency
          const price = this.formatPrice(
            serviceDetail.info.pricesByCurrency[0].totalPrice, currency
          )
          const duration = `${serviceDetail.info.expectedDuration.duration} ${serviceDetail.info.expectedDuration.durationType}`

          const service = {
            id,
            description,
            serviceName,
            testResultSample,
            price,
            duration,
            currency
          }
          this.items.push(service)
        }
        catch(err) {
          console.error(err)
        }
      }
    },

    async getGAOrders() {
      const orders = await GAGetOrders()

      this.orders = orders
    },

    formatPrice(price, currency) {
      let unit
      let formatedBalance = 0
      currency === "USDT" || currency === "USDT.e" || currency === "USDTE" ? unit = "mwei" : unit = "ether"
      formatedBalance = this.web3.utils.fromWei(String(price.replaceAll(",", "")), unit)
      return formatedBalance
    },

    async showDialog(id) {
      this.serviceId = id
      this.showModal = true
      await this.getDeleteServiceFee()
    },

    onDelete() {
      const isActiveOrder = this.orders?.data?.some(
        (order) =>
          order._source.service_id === this.serviceId &&
          order._source.status === "Paid"
      )
      
      if (isActiveOrder) this.showActiveOrder = true
      else this.deleteService() 
    },

    async deleteService() {
      try {
        this.isLoading = true
        await deleteGeneticAnalystService(this.api, this.wallet, this.serviceId)
      } catch (e) {
        this.error = await errorHandler(e.message)
        this.showModal = false
        this.isLoading = false
      }
    },

    goToDashboard() {
      this.$router.push({name: "dashboard"})
    }
  }
}
</script>

<style lang="sass" scoped>
@import "@/common/styles/functions.sass"
@import "@/common/styles/mixins.sass"

.ga-services
  &__wrapper
    display: flex
    flex-direction: column
    gap: toRem(35px)

  &__modal-title
    margin-top: toRem(34px)
    margin-bottom: toRem(36px)
    @include h6

  &__modal-desc
    display: flex
    align-items: center
    letter-spacing: -0.0044em
    margin-top: toRem(21px)
    padding-left: toRem(24px)
    padding-right: toRem(24px)
    @include new-body-text-2-medium

  &__modal-button
    padding: toRem(24px)
    width: 100%

  &__tx-weight
    padding-left: toRem(24px)
    padding-right: toRem(24px)
    width: 100%
    display: flex
    justify-content: space-between

  &__tx-weight-text
    letter-spacing: -0.004em
    @include body-text-3-opensans


  &__table-title
    @include body-text-medium-1

  &__table-subtitle
    @include body-text-3

  &__service-name
    width: toRem(120px)

  &__description
    width: toRem(328px)

  &__price
    width: toRem(78px)

  &__duration
    width: toRem(96px)

  &__action
    width: toREm(45px)
    display: flex
    align-items: center
    justify-content: center
    gap: toRem(20px)



  &::v-deep
    .v-data-table > .v-data-table__wrapper > table > tbody > tr > td
      @include body-text-3

    .banner__content-description
      margin-left: 10.25rem

    .banner__subtitle
      max-width: 32.2rem !important

    .ui-debio-modal__card
      width: toRem(400px)

.verification-status
  display: flex
  align-items: center
  background: #FFFFFF
  border-radius: toRem(8px)
  transition: all cubic-bezier(.7,-0.04,.61,1.14) .3s
  padding: toRem(20px)

  &:hover
    box-shadow: 0 0.125rem 0.625rem rgba(0, 0, 0, 0.1)
    border-bottom: toRem(5px) solid rgba(0, 0, 0, 0.15)
    transform: translateY(toRem(-5px))

  &__icon
    margin-right: toRem(18px)

  &__text
    pointer-events: none
    @include body-text-medium-1

.transition-slide-x
  &-enter-active,
  &-leave-active
    transition: all cubic-bezier(.7, -0.04, .61, 1.14) .3s

  &-enter
    opacity: 0
    transform: translateX(1.563rem)

  &-leave-to
    opacity: 0
    transform: translateX(-12.813rem)
</style>
