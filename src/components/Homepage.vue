<template>
  <div class="followers">

    <h1>Top followers</h1>

    <p>
      Powered by <a href="https://sql.steemhelpers.com" target="_blank">SteemSQL Wrapper</a>.
      Built by <a href="https://steemit.com/@emptyname" target="_blank">@emptyname</a>
    </p>

    <div class="followers__search">
      <el-input
        placeholder="@username"
        v-model="input"
        v-on:keyup.enter.native="submitData"
        :autofocus="true">
          <el-button v-on:click="submitData" slot="append" icon="el-icon-search"></el-button>
      </el-input>
    </div>

    <div class="followers__main-content">

      <div class="followers__filters">

        <el-select
          v-model="orderType"
          placeholder="Sort by"
          @change="changeOrder"
          class="followers__order">
            <el-option
              v-for="item in orderTypeOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value">
            </el-option>
        </el-select>

        <el-select
          v-model="massFollowersFilter"
          placeholder="Show/hide mass-followers"
          clearable
          @change="changeMassFollowersFilter">
            <el-option
              v-for="item in massFollowersOptions"
              :key="item.value"
              :label="item.label"
              :value="item.value">
            </el-option>
        </el-select>
      </div>

      <el-card class="table-wrapper">
        <el-table :data="tableData" v-loading="isLoading">
          <el-table-column prop="follower" label="Name">
            <template slot-scope="scope">
              <a :href="`https://steemit.com/@${scope.row.follower}`" target="_blank">
                {{scope.row.follower}}
              </a>
            </template>
          </el-table-column>
          <el-table-column prop="number_followers" label="Followers" width="105"></el-table-column>
          <el-table-column prop="number_following" label="Following" width="105"></el-table-column>

          <!-- <el-table-column prop="ratio" label="Ratio" width="60"></el-table-column> -->
          <!-- <el-table-column prop="reverse_ratio" label="rev ratio" width="60"></el-table-column> -->

          <el-table-column prop="total_steem_power" label="Steem Power" width="120"></el-table-column>

          <!--
            <el-table-column prop="steem_power" label="steem_power"></el-table-column>
            <el-table-column prop="delegated_steem_power" label="delegated_steem_power"></el-table-column>
            <el-table-column prop="received_steem_power" label="received_steem_power"></el-table-column>
            <el-table-column prop="total_steem_power_raw" label="total_steem_power_raw"></el-table-column>
          -->
        </el-table>
      </el-card>

    </div>

  </div> <!-- /.followers -->

</template>

<script>

import Vue from 'vue';
import { Input, Button, Table, TableColumn, Card, Loading, Select, Option } from 'element-ui';
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
    'el-select': Select,
    'el-option': Option,
  },
  data() {
    return {
      input: '',
      tableData: [],
      isLoading: false,
      orderType: 'steemPower',
      orderTypeOptions: [
        {
          value: 'steemPower',
          label: 'Sort: Steem Power',
        },
        {
          value: 'followers',
          label: 'Sort: Followers',
        },
        {
          value: 'following',
          label: 'Sort: Following',
        },
      ],

      massFollowersFilter: '',
      massFollowersOptions: [
        {
          value: 'show',
          label: 'Show only mass-followers',
        },
        {
          value: 'hide',
          label: 'Hide mass-followers',
        },
      ],
    };
  },
  methods: {

    changeOrder() {
      if (!this.tableData.length) {
        return;
      }

      this.submitData();
    },

    changeMassFollowersFilter() {
      if (!this.tableData.length) {
        return;
      }

      this.submitData();
    },

    submitData() {
      if (!this.input) {
        return;
      }

      this.isLoading = true;

      const orderTypeList = {
        steemPower: '(steemPowerTable.my_vesting_shares + steemPowerTable.my_received_vesting_shares - steemPowerTable.my_delegated_vesting_shares)',
        followers: 'numberFollowersTable.number_followers_raw',
        following: 'numberFollowingTable.number_following_raw',
      };

      const massFollowersFilterList = {
        show: 'AND (numberFollowingTable.number_following_raw / numberFollowersTable.number_followers_raw) >= 1 AND numberFollowersTable.number_followers_raw > 200',
        hide: 'AND (numberFollowingTable.number_following_raw / numberFollowersTable.number_followers_raw) < 1',
      };

      let sql = `
        SELECT TOP 50
          followersTable.follower,
          followersTable.following,
          accountsTable.name,

          numberFollowersTable.number_followers_raw,
          numberFollowingTable.number_following_raw,
          (numberFollowersTable.number_followers_raw / numberFollowingTable.number_following_raw) as ratio,
          (numberFollowingTable.number_following_raw / numberFollowersTable.number_followers_raw) as reverse_ratio,

          steemPowerTable.my_vesting_shares,
          steemPowerTable.my_delegated_vesting_shares,
          steemPowerTable.my_received_vesting_shares
        FROM
          Followers as followersTable
            LEFT JOIN Accounts as accountsTable ON followersTable.follower = accountsTable.name
            LEFT JOIN (
              SELECT COUNT(*) as number_followers_raw, following
              FROM Followers
              GROUP BY following
            ) as numberFollowersTable
            ON accountsTable.name = numberFollowersTable.following
            LEFT JOIN (
              SELECT COUNT(*) as number_following_raw, follower
              FROM Followers
              GROUP BY follower
            ) as numberFollowingTable
            ON accountsTable.name = numberFollowingTable.follower
            LEFT JOIN (
              SELECT
                name,
                (SELECT TOP 1 convert(float,value)
                  FROM STRING_SPLIT(vesting_shares, ' ')
                ) as my_vesting_shares,
                (SELECT TOP 1 convert(float,value)
                  FROM STRING_SPLIT(delegated_vesting_shares, ' ')
                ) as my_delegated_vesting_shares,
                (SELECT TOP 1 convert(float,value)
                  FROM STRING_SPLIT(received_vesting_shares, ' ')
                ) as my_received_vesting_shares
                FROM Accounts
            ) as steemPowerTable
            ON accountsTable.name = steemPowerTable.name
        WHERE followersTable.following='{username}' {massFollowersFilter}
        ORDER BY {orderType} DESC
      `;

      const currentUsername = this.input.replace('@', '');
      sql = sql.replace('{username}', currentUsername);
      const currentOrderType = orderTypeList[this.orderType];
      sql = sql.replace('{orderType}', currentOrderType);

      let currentMassFollowersFilter = '';
      if (this.massFollowersFilter) {
        currentMassFollowersFilter = massFollowersFilterList[this.massFollowersFilter];
      }
      sql = sql.replace('{massFollowersFilter}', currentMassFollowersFilter);

      const ajaxData = {
        query: sql,
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
        const vests = parseFloat(row.my_vesting_shares);
        const delegatedVests = parseFloat(row.my_delegated_vesting_shares);
        const receivedVests = parseFloat(row.my_received_vesting_shares);

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
          number_followers: this.formatter(row.number_followers_raw),
          number_following: this.formatter(row.number_following_raw),
          ratio: row.ratio,
          reverse_ratio: row.reverse_ratio,
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

  a {
    color: #039be5;
    text-decoration: none;
    -webkit-tap-highlight-color: transparent;
  }

  .followers{
    text-align: center;
    margin-top: 60px;
  }

  .followers .followers__search .el-input{
    max-width: 300px;
    margin-top: 50px;
  }

  .followers .followers__search .el-input div.el-input-group__append{
    transition: 0.1s;
  }
  .followers .followers__search .el-input div.el-input-group__append:hover{
    background: rgba(227,234,255,.7);
  }

  .followers__main-content{
    max-width: 505px;
    /*max-width: 705px;*/
    margin: 50px auto;
  }

  .followers__filters{
    text-align: left;
  }

  .followers__filters .followers__order.el-select .el-input__inner{
    width: 170px;
  }

  .followers .table-wrapper{
    margin-top:  20px;
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
