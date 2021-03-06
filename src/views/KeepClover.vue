<template lang="pug">
  article.green.fixed.z3.overlay.bg-white.flex
    .col-12.max-width-3.mx-auto.flex.flex-column.justify-between.outline
      header
        .h-header.border-bottom.flex.justify-between.items-center
          .col-3.pl2
            button.pointer(@click="close") Back
          //- h1.col-6.font-exp.center.nowrap Keep or Sell
          .col-9.pr2.right-align
            router-link.font-mono(:to="{name: 'Account/Trade'}") {{ prettyUserBalance }} ♣
      figure.flex-auto.relative
        //- clv.col-10.sm-col-6.mx-auto(:moveString="clover.movesString")
        .absolute.bg-contain.bg-no-repeat.bg-center(role="img", v-if="clover", :style="'background-image:url(' + cloverImage(clover) + ')'")
      footer(v-if="!submitted")
        .flex.border-top
          .col-6.p3.border-right.relative(@click="action = 'keep'")
            div(:class="{'opacity-25': action !== 'keep'}")
              small.block.lh1 Keep for ♣
              .font-exp.mt1.truncate {{tokenValue}}
              //- button.absolute.top-0.right-0.p2.pointer(@click="action = 'keep'")
                .icon-radio(:class="{'icon-radio--selected': action === 'keep'}")
          .col-6.p3.relative(@click="action = 'sell'")
            div(:class="{'opacity-25': action !== 'sell'}")
              small.block.lh1 Sell instantly for ♣
              .font-exp.mt1.truncate {{rewardValue}}
              //- button.absolute.top-0.right-0.p2.pointer(@click="action = 'sell'")
                .icon-radio(:class="{'icon-radio--selected': action === 'sell'}")
        //- keep btn
        .bg-green.white
          button.col-12.h-bttm-bar.font-exp.pointer(@click="btnClick", :class="{'pointer-events-none': submitting}")
            span.block.m-auto.capitalize(v-show="!submitting") {{action}}
            wavey-menu.m-auto(v-show="submitting", :isWhite="true")
          transition(name="fade")
            .p2.center.font-mono(v-if="submitting")
              p {{ infoText }}
      footer(v-else)
        .bg-green.white.col-12.h-bttm-bar.font-mono.flex.items-center
          p.center.m-auto.h3 {{ submitted }}
</template>

<script>
import WaveyMenu from '@/components/Icons/WaveyMenu'
import { mapGetters, mapActions } from 'vuex'
import { cloverImage } from '@/utils'
import { fromWei } from 'web3-utils'
import Reversi from 'clovers-reversi'
import BigNumber from 'bignumber.js'
const reversi = new Reversi()

export default {
  name: 'KeepClover',
  props: {
    clover: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      action: 'keep',
      value: null,
      reward: null,
      submitting: false,
      submitted: false
    }
  },
  computed: {
    rewardValue () {
      return this.reward ? fromWei(this.reward.toString()) : '...'
    },
    tokenValue () {
      return this.value ? fromWei(this.value.toString()) : '...'
    },
    sellValue () {
      return this.value
        ? fromWei(this.value.minus(this.$store.state.basePrice).toString())
        : '...'
    },
    infoText () {
      return this.action === 'keep'
        ? 'Your Clover is being submitted to the Contract. Once the Clover is verified by our Oracle, you will be confirmed as the owner.'
        : 'This reward is based on the rarity of the Clover. The Contract will buy this from you with Club Token (♣). Once the Oracle has verified the Clover you will receive the payout.'
    },

    ...mapGetters(['prettyUserBalance'])
  },
  methods: {
    cloverImage,

    close () {
      this.$emit('close')
      this.action = 'keep'
    },
    btnClick () {
      if (this.action === 'keep') this.keep()
      if (this.action === 'sell') this.sellToBank()
    },
    async keep () {
      this.submitting = true
      try {
        const tx = await this.buy(this.clover)
        console.log(tx)
        this.submitting = false
        this.handleSuccess(
          `Success! You kept ${this.clover.board}`,
          this.clover
        )
      } catch (error) {
        console.log(error)
        this.submitting = false
        // notification
        this.handleError(error)
      }
    },
    async sellToBank () {
      this.submitting = true
      try {
        const tx = await this.sell({ clover: this.clover })
        console.log(tx)
        this.submitting = false
        this.handleSuccess(
          `Success! You sold ${this.clover.board} to the bank`,
          this.clover
        )
      } catch (error) {
        console.log(error)
        this.submitting = false
        // notification
        this.handleError(error)
      }
    },
    getValue () {
      reversi.playGameMovesString(this.clover.movesString)
      const syms = reversi.returnSymmetriesAsBN()
      this.$store.dispatch('getReward', syms).then(wei => {
        wei = new BigNumber(wei)
        this.reward = wei
        this.value = wei.plus(this.$store.state.basePrice)
      })
    },
    handleError ({ message }) {
      this.selfDestructMsg({
        msg: message.replace('Error: ', ''),
        type: 'error'
      })
    },
    handleSuccess (msg, clover) {
      this.selfDestructMsg({
        msg,
        type: 'success'
      })
      this.submitted = msg
      this.$store.commit('REMOVE_SAVED_CLOVER', clover)
    },

    ...mapActions(['buy', 'sell', 'addMessage', 'selfDestructMsg'])
  },
  mounted () {
    this.getValue()
  },
  components: { WaveyMenu }
}
</script>

<style lang="css" scoped>
figure > div {
  width: calc(100% - 2rem);
  height: calc(100% - 4rem);
  top: 2rem;
  left: 1rem;
}
</style>
