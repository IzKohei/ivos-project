<template>
<div class="container">
  <el-menu :default-active="activeIndex" class="el-menu-demo" mode="horizontal" @select="handleSelect">
    <el-menu-item index="1">
      <nuxt-link to="/">Top</nuxt-link>
    </el-menu-item>
    <el-menu-item index="2">
      <nuxt-link to="/my_item_list">MyPage</nuxt-link>
    </el-menu-item>
    <el-submenu index="3">
      <template slot="title">アイドルはこちら</template>
        <el-menu-item index="3-1">
          <nuxt-link to="/idol-account">アイドルアカウント登録</nuxt-link>
        </el-menu-item>
        <el-menu-item index="3-2">
          <nuxt-link to="/voice">音声登録</nuxt-link>
        </el-menu-item>
    </el-submenu>
    <el-menu-item index="4" disabled>
      <el-tag type="info" style="backgroundColor: black; color: white">Rinkeby Support</el-tag>
    </el-menu-item>
  </el-menu>
  <el-card class="box-card" style="margin: 20px 0">
    <div class="text item" center>
      ADDRESS :  {{ address }}
    </div>
  </el-card>
  <el-row justify="center" >
    <el-col class="card" :span="4">
      <el-card :body-style="{ padding: '0px' }">
        <img :src="idol.image" class="image">
        <div style="padding: 14px;">
          <span>{{ idol.name }}</span>
        </div>
      </el-card>
    </el-col>
  </el-row>
  <el-card class="box-card" style="margin: 20px 0">
    <el-table
      :data="tableData"
      style="width: 100%">
      <el-table-column
        label="ID"
        prop="voiceId">
      </el-table-column>
      <el-table-column
        label="名前"
        prop="name">
      </el-table-column>
      <el-table-column
        label="最大発行量"
        prop="totalSupply">
      </el-table-column>
      <el-table-column
        label="発行済み"
        prop="issuedNum">
      </el-table-column>
      <el-table-column
        label="金額"
        prop="price">
      </el-table-column>
      <el-table-column
        align="right">
        <template slot-scope="scope">
          <el-button type="primary" icon="el-icon-lollipop" value="" @click.native.prevent="onSubmit(scope.$index)">購入</el-button>
        </template>
      </el-table-column>
    </el-table>
  </el-card>
</div>
</template>

<script>
import Web3 from 'web3'
import abi from '~/plugins/abi'
import voiceVue from '../voice.vue'
import axios from "axios"
var web3

// init client web3 js
if (process.browser) {
  console.log('givenProvider=')
  console.log(Web3.givenProvider)
  web3 = new Web3(Web3.givenProvider)
}

// const contract = new web3.eth.Contract(abi, '0x7da9e677e4b2269dc209e35b7f17a8e7de828433')
let a = process.env.contract_addr
const contract = new web3.eth.Contract(abi, process.env.contract_addr)

  export default {
    data() {
      return {
        address: '',
        myAddress: '',
        idol: {
          id: '',
          name: '',
          image: '',
          address: ''
        },
        tableData: []
      }
    },
    async asyncData({app, params}) {
      console.log("aa" + params.id)
      const myAddresses = await web3.eth.getAccounts()
      const myAddress = myAddresses[0]

      const address = params.id
      const response = await axios.get(`idol_token/idol?address=${address}`, {
        baseURL: 'https://ivos.herokuapp.com/',
        headers: { "Content-Type": "application/json", 'X-Requested-With': 'XMLHttpRequest' },
        responseType: 'json',
        data: {}
      })
      console.log(response.data)
      return { 
        address: address,
        myAddress: myAddress,
        idol: response.data
      }
    },
    async created() {
      contract.methods.balanceOfOwnedVoices(
        this.$route.params.id
      ).call()
      .then((result) => {
        const num = web3.utils.hexToNumber(result)
        for(var i=0; i < num; i++) {
          contract.methods.ownedVoices(
            this.$route.params.id,
            i
          ).call()
          .then((result) => {
            const voiceId = web3.utils.hexToNumber(result)
            contract.methods.voices(
              voiceId
            ).call()
            .then(async (result) => {
              console.log(result.price)
              console.log(web3.utils.hexToNumber(result.price))
              const totalSupply = web3.utils.hexToNumber(result.totalSupply)
              const issuedNum = web3.utils.hexToNumber(result.issuedNum)
              const price = web3.utils.hexToNumber(result.price)
              console.log(price)
              const response = await axios.get(`idol_token/item/${voiceId}/`, {
                baseURL: 'https://ivos.herokuapp.com/',
                headers: { "Content-Type": "application/json", 'X-Requested-With': 'XMLHttpRequest' },
                responseType: 'json',
                data: {}
              })
              this.tableData.push({
                name: response.data.title,
                totalSupply: totalSupply,
                issuedNum: issuedNum,
                price: price,
                voiceId: voiceId
              })
            })
          })
        }
      })
    },
  methods: {
    handleChange(event) {
      console.log(event)
    },
    onSubmit(index) {
      const voiceId = this.tableData[index].voiceId
      const price = this.tableData[index].price
      console.log(voiceId)
      contract.methods.mintVoice(
        voiceId
      )
      .send({
          from: this.myAddress,
          gas: 3000000,
          gasPrice: web3.eth.gasPrice,
          gasLimit: web3.eth.getBlock("latest").gasLimit,
          value: price
      })
      .on('transactionHash', (hash) => {
        this.transaction = hash
      })
      .on('confirmation', (confirmationNumber, receipt) => {
        if(receipt.status) {
          this.$router.push({ path: `/idol/${receipt.from}` })
        }
      })
    }
  }
  }
</script>
