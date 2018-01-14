<template>
  <div class="followers">

    <h1>Top followers</h1>

    <p>
      Powered by <a href="https://sql.steemhelpers.com" target="_blank">SteemSQL Wrapper</a>.
      Built by <a href="https://steemit.com/@emptyname" target="_blank">@emptyname</a>
    </p>

    <el-input placeholder="@username" v-model="input" style="max-width: 300px; margin-top: 50px;">
      <el-button v-on:click="submitData" slot="append" icon="el-icon-search"></el-button>
    </el-input>

    <div style="max-width: 400px; margin: 50px auto;">
      
      <el-table :data="tableData" v-loading="isLoading">
        <el-table-column prop="follower" label="Name"></el-table-column>
        <el-table-column prop="number_followers" label="Followers"></el-table-column>

        <!--
        <el-table-column
          v-for="item in tableHeaders"  
          v-bind:prop="item"
          v-bind:label="item"
          v-bind:key="item">
        </el-table-column>
      -->
      </el-table>

    </div>

  </div>

</template>

<script>

import Vue from 'vue';
import { Input, Button, Table, TableColumn, Loading } from 'element-ui';

Vue.use(Loading.directive);

export default {
  name: 'Homepage',
  components: {
    'el-input': Input,
    'el-button': Button,
    'el-table': Table,
    'el-table-column': TableColumn,
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
          this.isLoading = false;

          if (response.status >= 400 && response.status < 600) {
            // return Promise.reject(new Error(response.statusText));
            throw new Error(response.statusText);
          }
          return response.json();
        })
        .then((data) => {
          this.tableData = data.rows;
        })
        .catch((error) => {
          console.log('Request failed', error);
        });
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>

  .followers{
    text-align: center;
    margin-top: 60px;
  }

  .followers .el-input div.el-input-group__append{
    transition: 0.1s;
  }
  .followers .el-input div.el-input-group__append:hover{
    background: rgba(227,234,255,.7);
  }

  .followers .el-table th:nth-child(2),
  .followers .el-table td:nth-child(2){
    text-align: right;
  }

  .followers .el-table td:nth-child(1){
    text-align: left;
  }

</style>
