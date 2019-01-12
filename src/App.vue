<template>
    <div class="container">
        <div class="col-6 offset-3">
            <div class="card bg-light">
                <div class="card-header">Payment Information</div>
                <div class="card-body">
                    <div class="alert alert-danger" v-if="error">
                        {{ error }}
                    </div>
                    <div class="alert alert-success" v-if="nonce">
                        Nonce created successfully!
                    </div>
                    <form>
                        <div class="form-group">
                            <label for="amount">Amount</label>
                            <div class="input-group">
                                <div class="input-group-prepend"><span class="input-group-text">$</span></div>
                                <input type="number" v-model="amount" id="amount" class="form-control" placeholder="Enter Amount">
                            </div>
                        </div>
                        <hr />
                        <div class="form-group">
                            <label>Credit Card Number</label>
                            <div id="creditCardNumber" class="form-control"></div>
                        </div>
                        <div class="form-group">
                            <div class="row">
                                <div class="col-6">
                                    <label>Expire Date</label>
                                    <div id="expireDate" class="form-control"></div>
                                </div>
                                <div class="col-6">
                                    <label>CVV</label>
                                    <div id="cvv" class="form-control"></div>
                                </div>
                            </div>
                        </div>
                        <div class="text-center">
                            <button class="btn btn-primary btn-block" @click.prevent="pay" :disabled="preventPaying">Pay with Credit Card</button>
                        </div>
                        <hr />
                        <div class="form-group text-center">
                            <div id="paypalButton"></div>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</template>

<script>

import { client, hostedFields, paypalCheckout } from 'braintree-web';
import paypal from 'paypal-checkout';

const today = new Date();
const currentMonth = today.getMonth()+1 < 10 ? "0"+(today.getMonth()+1) : today.getMonth()+1;
const currentYear = today.getFullYear();

const AUTH_KEY = "REPLACE WITH YOUR AUTHORIZATION TOKEN";

export default {
    data() {
        return {
            amount: 100,
            nonce: "",
            hostedFieldsInstance: false,
            paypalInstance: false,
            error: ""
        }
    },
    computed: {
        preventPaying() {
            return !this.hostedFieldsInstance || isNaN(this.amount) || parseFloat(this.amount) <= 0 || !this.amount;
        }
    },
    methods: {
        pay() {
            if(!this.preventPaying)
            {
                this.error = "";
                this.nonce = "";

                let amount = parseFloat(this.amount);
                this.hostedFieldsInstance.tokenize()
                .then(payload => {
                    console.log(payload);
                    this.nonce = payload.nonce;
                })
                .catch(err => {
                    console.error(err);
                    if(typeof err.message !== 'undefined')
                    {
                        this.error = err.message;
                    }
                    else
                    {
                        this.error = "An error occurred while processing the payment.";
                    }
                })
            }
        },
        initBraintree() {
            client.create({
                authorization: AUTH_KEY
            })
            .then(clientInstance => {
                let options = {
                    client: clientInstance,
                    styles: {
                        input: {
                            'font-size': '14px',
                            'font-family': 'Open Sans'
                        }
                    },
                    fields: {
                        number: {
                            selector: '#creditCardNumber',
                            placeholder: 'Enter Credit Card'
                        },
                        cvv: {
                            selector: '#cvv',
                            placeholder: 'Enter CVV'
                        },
                        expirationDate: {
                            selector: '#expireDate',
                            placeholder: currentMonth + ' / ' + currentYear
                        }
                    }
                }


                let promises = [];
                promises.push(hostedFields.create(options));
                promises.push(paypalCheckout.create({ client: clientInstance }));

                return Promise.all(promises);
            })
            .then(instances => {
                this.hostedFieldsInstance = instances[0];
                const paypalInstance       = instances[1];

                return paypal.Button.render({
                    env: 'sandbox',
                    style: {
                        label: 'paypal',
                        size: 'responsive',
                        shape: 'rect'
                    },
                    payment: () => {
                        return paypalInstance.createPayment({
                                flow: 'checkout',
                                intent: 'sale',
                                amount: this.amount ? this.amount : 10,
                                displayName: 'Braintree Testing',
                                currency: 'USD'
                        })
                    },
                    onAuthorize: (data, options) => {
                        return paypalInstance.tokenizePayment(options).then(payload => {
                            console.log(payload);
                            this.error = "";
                            this.nonce = payload.nonce;
                        })
                    },
                    onCancel: (data) => {
                        console.log(data);
                        console.log("Payment Cancelled");
                    },
                    onError: (err) => {
                        console.error(err);
                        this.error = "An error occurred while processing the paypal payment.";
                    }
                }, '#paypalButton')

            })
            .catch(err => {
                console.error(err);
                this.error = "An error occurred while creating the payment form.";
            })
        }
    },
    mounted() {
        this.initBraintree();
    }
}
</script>

<style>
    body {
        padding: 20px;
    }
</style>
