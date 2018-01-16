<template>
  <div class="followers">

    <h1>Top followers</h1>

    <p>
      Powered by <a href="https://sql.steemhelpers.com" target="_blank">SteemSQL Wrapper</a>.
      Built by <a href="https://steemit.com/@emptyname" target="_blank">@emptyname</a>
    </p>

    <el-input placeholder="@username" v-model="input">
      <el-button v-on:click="submitData" slot="append" icon="el-icon-search"></el-button>
    </el-input>

    <el-card class="table-wrapper">
      
      <el-table :data="tableData" v-loading="isLoading">
        <el-table-column prop="follower" label="Name"></el-table-column>
        <el-table-column prop="total_followers" label="Followers" width="105"></el-table-column>
        <el-table-column prop="total_steem_power" label="Steem Power" width="110"></el-table-column>

        <!--
          <el-table-column prop="steem_power" label="steem_power"></el-table-column>
          <el-table-column prop="delegated_steem_power" label="delegated_steem_power"></el-table-column>
          <el-table-column prop="received_steem_power" label="received_steem_power"></el-table-column>
          <el-table-column prop="total_steem_power_raw" label="total_steem_power_raw"></el-table-column>
        -->
      </el-table>

    </el-card>

  </div> <!-- /.followers -->

</template>

<script>

import Vue from 'vue';
import { Input, Button, Table, TableColumn, Card, Loading } from 'element-ui';
import steem from 'steem';

Vue.use(Loading.directive);

export default {
  name: 'Homepage',
  components: {
    'el-input': Input,
    'el-button': Button,
    'el-table': Table,
    'el-table-column': TableColumn,
    'el-card': Card,
  },
  data() {
    return {
      input: '',
      tableData: [],
      isLoading: false,
    };
  },
  methods: {

    submitData() {
      this.isLoading = true;

      const sql = `
        SELECT TOP 50
          followersTable.follower,
          followersTable.following,
          accountsTable.name,
          accountsTable.vesting_shares,
          accountsTable.delegated_vesting_shares,
          accountsTable.received_vesting_shares,
          ff.number_followers
        FROM
          Followers as followersTable
            LEFT JOIN Accounts as accountsTable ON followersTable.follower = accountsTable.name
            LEFT JOIN (
              SELECT COUNT(*) as number_followers, following
              FROM Followers
              GROUP BY following
            ) as ff
            ON accountsTable.name = ff.following
        WHERE followersTable.following='{username}'
        ORDER BY ff.number_followers DESC
      `;
      /*
      ORDER BY
        (accountsTable.vesting_shares +
          accountsTable.received_vesting_shares -
          accountsTable.delegated_vesting_shares)
      DESC
      */

      const ajaxData = {
        query: sql.replace('{username}', this.input.replace('@', '')),
      };

      fetch('https://sql.steemhelpers.com/api', {
        method: 'post',
        headers: {
          'Content-type': 'application/json; charset=utf-8',
        },
        body: JSON.stringify(ajaxData),
        mode: 'cors',
        dataType: 'json',
      })
        .then((response) => {
          if (response.status >= 400 && response.status < 600) {
            // return Promise.reject(new Error(response.statusText));
            throw new Error(response.statusText);
          }
          return response.json();
        })
        .then((data) => {
          this.tableData = data.rows;

          steem.api.setOptions({ url: 'https://api.steemit.com' });
          steem.api.getDynamicGlobalProperties((err, result) => {
            const totalVestingFundSteem = parseFloat(result.total_vesting_fund_steem.split(' ')[0]);
            const totalVestingShares = parseFloat(result.total_vesting_shares.split(' ')[0]);
            this.calculateSteemPower(totalVestingFundSteem, totalVestingShares);
            console.log(err, result);
          });
        })
        .catch((error) => {
          this.isLoading = false;
          console.log('Request failed', error);
        });
    }, // submitData

    formatter(valueInt) {
      let valueStr = valueInt.toString();
      if (valueStr.length > 6) {
        valueStr = `${Number((valueInt / 1000000).toFixed(2))}M`;
      } else if (valueStr.length > 3) {
        valueStr = `${Number((valueInt / 1000).toFixed(2))}k`;
      }
      return valueStr;
    },

    calculateSteemPower(totalVestingFundSteem, totalVestingShares) {
      const newTableData = [];

      this.tableData.forEach((row) => {
        const vests = parseFloat(row.vesting_shares.split(' ')[0]);
        const delegatedVests = parseFloat(row.delegated_vesting_shares.split(' ')[0]);
        const receivedVests = parseFloat(row.received_vesting_shares.split(' ')[0]);

        const steemPower = totalVestingFundSteem * (vests / totalVestingShares);
        const delegatedSteemPower = totalVestingFundSteem * (delegatedVests / totalVestingShares);
        const receivedSteemPower = totalVestingFundSteem * (receivedVests / totalVestingShares);

        let totalSteemPowerRaw = steemPower + (receivedSteemPower - delegatedSteemPower);
        totalSteemPowerRaw = parseInt(totalSteemPowerRaw);
        const totalSteemPower = this.formatter(totalSteemPowerRaw);

        const newRow = Object.assign({}, row, {
          steem_power: steemPower,
          delegated_steem_power: delegatedSteemPower,
          received_steem_power: receivedSteemPower,
          total_steem_power_raw: totalSteemPowerRaw,
          total_steem_power: totalSteemPower,
          total_followers: this.formatter(row.number_followers),
        });

        newTableData.push(newRow);
      });

      /*
      newTableData = newTableData.sort((aItem, bItem) =>
        bItem.total_steem_power_raw - aItem.total_steem_power_raw,
      );
      */

      this.tableData = newTableData;
      this.isLoading = false;
    },

  }, // methods
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>

  .followers{
    text-align: center;
    margin-top: 60px;
  }

  .followers .el-input{
    max-width: 300px;
    margin-top: 50px;
  }

  .followers .el-input div.el-input-group__append{
    transition: 0.1s;
  }
  .followers .el-input div.el-input-group__append:hover{
    background: rgba(227,234,255,.7);
  }

  .followers .table-wrapper{
    max-width: 400px;
    margin: 50px auto;
  }

  .followers .el-table th:nth-child(2),
  .followers .el-table td:nth-child(2){
    text-align: center;
  }

  .followers .el-table th:nth-child(3),
  .followers .el-table td:nth-child(3){
    text-align: right;
  }

  .followers .el-table td:nth-child(1){
    text-align: left;
  }

</style>
