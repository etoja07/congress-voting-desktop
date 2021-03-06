<template>
  <v-dialog v-model="dialog" fullscreen persistent hide-overlay content-class="FreezingDialogOverlay">
    <v-stepper v-model="stepNo" class="FreezingDialog">
      <v-stepper-items>
        <v-stepper-content step="1" class="step1">
          <h3>{{$t('requesting freezing')}}</h3>
          <span>{{$t('freezing should be 10,000 bos at least')}}</span>
          <v-text-field
            v-model="amountDisplayed"
            ref="amountText"
            :rules="amountRules"
            autofocus
            required
          >
          </v-text-field>
          <div class="unit" :style="{ marginLeft: `${unitMargin * 15}px` }">{{unit}}<abbr>BOS</abbr></div>
          <div class="currentAmount">{{$t('available amount')}} {{prettify(availableBalance)}} BOS</div>
          <button class="button" :disabled="!isDoneStep1" @click="moveStep2">{{$t('freezing')}}</button>
        </v-stepper-content>

        <v-stepper-content step="2" class="step2">
          <h3>{{$t('requesting freezing')}}</h3>
          <span class="desc">{{$t('you can get rewards by frozen amount')}}</span>
          <div class="preview">
            <div class="row">
              <span class="label">{{$t('total withdrawal')}}</span>
              <span class="value">{{prettify(amount)}}<abbr>BOS</abbr></span>
            </div>
          </div>
          <span class="desc2">{{$t('freezing is not cancelable, you should unfreezing it')}}</span>
          <button class="button" @click="stepNo=3">{{$t('entering passphrase')}}</button>
        </v-stepper-content>

        <v-stepper-content step="3" class="step3">
          <h3>{{$t('confirm passphrase')}}</h3>
          <span>{{$t('freezing is not cancelable, you should unfreezing it')}}</span>
          <v-text-field v-model="passphrase" ref="passphraseText" type="password"
                        :rules="passphraseRules" :label="$t('enter your passphrase')" required/>
          <button class="button" @click="submit">{{$t('done')}}</button>
        </v-stepper-content>
      </v-stepper-items>

      <div class="stepIndicator">
        <span :class="{step: true, on: (stepNo === 1)}"></span>
        <span :class="{step: true, on: (stepNo === 2)}"></span>
        <span :class="{step: true, on: (stepNo === 3)}"></span>
      </div>

      <img :src="closeIcon" @click="reset" class="FreezingDialogClose" alt="close"/>
    </v-stepper>
    <bos-toast v-model="showMessage" :timeout="3000" pullRight>{{message}}</bos-toast>
  </v-dialog>
</template>

<script>
  import Unit from '@/lib/unit';
  import Helper from '@/lib/helper';

  import closeIcon from '../../assets/svg/close.svg';

  export default {
    name: 'freezing-dialog',
    props: ['wallet', 'callback'],
    data() {
      return this.init();
    },
    computed: {
      unit() {
        return (10000).toLocaleString().substr(1);
      },
    },
    methods: {
      init() {
        return {
          dialog: false,
          availableBalance: '0',
          fee: 0.001,
          stepNo: 1,
          isDoneStep1: false,
          unitMargin: 0,
          amount: null,
          amountDisplayed: null,
          amountRules: [
            (v) => {
              let amount = v && v.match(/\d/g) ? v.match(/\d/g).join('') : '';
              if (parseInt(amount, 10) * 10000 > parseInt(this.availableBalance, 10)) {
                amount = this.availableBalance.substr(0, this.availableBalance.length - 4);
              }

              // add commas
              const temp = parseInt(amount * 10, 10).toLocaleString();
              const commaCount = temp.length - amount.length - 1;
              const numerCount = amount.length;
              amount = temp.substr(0, temp.length - 1);

              // move unit guide
              this.unitMargin = numerCount + (commaCount * 0.5);

              this.amountDisplayed = amount;
              this.isDoneStep1 = !!amount;
              return true;
            },
          ],
          passphrase: null,
          passphraseRules: [
            v => !!v || this.$t('enter your passphrase'),
          ],
          showMessage: false,
          message: '',
          closeIcon,
        };
      },
      open() {
        this.dialog = true;
      },
      prettify(b) {
        if (!b) { return b; }
        return parseFloat(b, 10).toLocaleString();
      },
      reset() {
        this.dialog = false;
        Object.assign(this.$data, this.init());
        this.setBalance();
      },
      moveStep2() {
        this.amount = parseFloat(this.amountDisplayed.replace(',', '')) * 10000;
        this.stepNo = 2;
      },
      async submit() {
        try {
          await this.$store.dispatch('freeze', {
            address: this.wallet.address,
            amount: this.amount,
            passphrase: this.passphrase,
          });
        } catch (err) {
          if (err.message.match(/bad decrypt/)) {
            this.message = this.$t('passphrase is wrong');
            this.showMessage = true;
            return;
          }

          throw err;
        }
        if (this.callback) {
          this.callback(this.passphrase);
        }
        this.reset();
      },
      getAccount() {
        if (this.wallet.address != null) {
          return this.$store.getters.account(this.wallet.address);
        }
        return null;
      },
      async setBalance() {
        const account = await this.getAccount();
        const available = Unit.convert(account.balance, 'gon', 'bos');
        this.availableBalance = Helper.calcMinimumFreezeAmount(available);
      },
    },
    async mounted() {
      this.$store.state.App.ga.send('screenview', { cd: 'freezing-dialog' });
      await this.setBalance();
    },
  };
</script>

<style>
  .FreezingDialogOverlay {
    background-color: rgba(216, 223, 232, 0.7);
    height: 100%;
    width: 714px;
    margin-left: 158px;
    display: flex;
    align-items: center;
    justify-items: center;
  }

  .FreezingDialog.theme--light.v-stepper {
    padding: 56px 36px 31px;
    position: relative;
    width: 634px;
    height: 550px;
    border-radius: 2px;
    box-shadow: 0 6px 20px 0 rgba(178, 199, 220, 0.8);
    border: solid 1px #ececec;
    background-color: #ffffff;
    color: #333333;
    margin-left: 40px;
  }

  .FreezingDialogClose {
    position: absolute;
    top: 30px;
    right: 30px;
    cursor: pointer;
  }

  .FreezingDialog h3 {
    text-align: center;
    font-size: 30px;
    font-weight: bold;
  }

  .FreezingDialog span {
    font-size: 14px;
    color: #333333;
    display: block;
    margin: 5px auto 0;
    text-align: center;
  }

  .FreezingDialog .button {
    margin: 0 auto;
  }

  .FreezingDialog .stepIndicator {
    text-align: center;
    height: 20px;
    margin-top: -7px;
  }

  .FreezingDialog .v-stepper__content {
    padding-bottom: 0;
  }

  .FreezingDialog .v-stepper__wrapper {
    padding-bottom: 15px;
  }

  .FreezingDialog .step {
    display: inline-block;
    width: 6px;
    height: 6px;
    border-radius: 6px;
    background-color: #bdc2c7;
  }

  .FreezingDialog .step.on {
    background-color: #3c92e4;
  }

  .FreezingDialog .step1 span {
    margin-bottom: 80px;
  }

  .FreezingDialog .step1 .unit {
    font-size: 25px;
    font-weight: bold;
    color: #333333;
    margin-top: -55px;
  }

  .FreezingDialog .step1 .unit abbr {
    font-size: 10px;
    color: #909090;
  }

  .FreezingDialog .step1 .theme--light.v-label.primary--text {
    color: #99a4b0 !important;
  }

  .FreezingDialog .step1 .v-text-field > .v-input__control > .v-input__slot:before {
    border-color: #1993f1;
  }

  .FreezingDialog .step1 .v-text-field > .v-input__control > .v-input__slot {
    font-size: 25px;
    font-weight: bold;
    color: #333333;
    position: relative;
  }

  .FreezingDialog .step1 .currentAmount {
    font-size: 12px;
    color: #1993f1;
    margin-top: 3px;
    margin-bottom: 113px;
  }

  .FreezingDialog .step2 .desc {
    margin-bottom: 42px;
  }

  .FreezingDialog .step2 .preview {
    width: 450px;
    margin: 0 auto 10px;
    border-top: dashed 1px #b3bec8;
    border-bottom: dashed 1px #b3bec8;
  }

  .FreezingDialog .step2 .preview .row {
    display: flex;
    align-items: center;
    margin: 31px auto;
    width: 349px;
    height: 37px;
  }

  .FreezingDialog .step2 .preview .row .label {
    width: 150px;
    text-align: left;
  }

  .FreezingDialog .step2 .preview .row .value {
    width: 199px;
    font-size: 25px;
    font-weight: bold;
    text-align: right;
    color: #333333;
    position: relative;
  }

  .FreezingDialog .step2 .preview .row .value abbr {
    font-size: 10px;
    color: #909090;
    position: absolute;
    bottom: 6px;
    right: -24px;
  }

  .FreezingDialog .step2 .desc2 {
    font-size: 12px;
    color: #728395;
    margin-bottom: 90px;
  }

  .FreezingDialog .step3 .v-input {
    width: 400px;
    margin: 72px auto 125px;
  }
</style>
