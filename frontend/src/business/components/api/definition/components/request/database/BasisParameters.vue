<template>
  <div v-loading="isReloadData">
    <el-row>
      <el-col :span="spanNum" style="padding-bottom: 20px">
        <div style="border:1px #DCDFE6 solid; height: 100%;border-radius: 4px ;width: 100% ;">
          <el-form :model="request" :rules="rules" ref="request" label-width="100px" :disabled="isReadOnly" style="margin: 10px">
            <el-row>
              <el-col :span="8">
                <el-form-item prop="environmentId" :label="$t('api_test.definition.request.run_env')">
                  <el-select v-model="request.environmentId" size="small" class="ms-htt-width"
                             :placeholder="$t('api_test.definition.request.run_env')"
                             @change="environmentChange" clearable>
                    <el-option v-for="(environment, index) in environments" :key="index"
                               :label="environment.name"
                               :value="environment.id"/>
                    <el-button class="environment-button" size="small" type="primary" @click="openEnvironmentConfig">
                      {{ $t('api_test.environment.environment_config') }}
                    </el-button>
                    <template v-slot:empty>
                      <div class="empty-environment">
                        <el-button class="environment-button" size="small" type="primary" @click="openEnvironmentConfig">
                          {{ $t('api_test.environment.environment_config') }}
                        </el-button>
                      </div>
                    </template>
                  </el-select>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('api_test.request.sql.dataSource')" prop="dataSourceId" style="margin-left: 10px">
                  <el-select v-model="request.dataSourceId" size="small" @change="reload">
                    <el-option v-for="(item, index) in databaseConfigsOptions" :key="index" :value="item.id" :label="item.name"/>
                  </el-select>
                </el-form-item>

              </el-col>
              <el-col :span="8">
                <el-form-item :label="$t('api_test.request.sql.timeout')" prop="queryTimeout" style="margin-left: 10px">
                  <el-input-number :disabled="isReadOnly" size="small" v-model="request.queryTimeout" :placeholder="$t('commons.millisecond')" :max="1000*10000000" :min="0"/>
                </el-form-item>
              </el-col>
            </el-row>


            <el-form-item :label="$t('api_test.request.sql.result_variable')" prop="resultVariable">
              <el-input v-model="request.resultVariable" maxlength="500" show-word-limit size="small"/>
            </el-form-item>

            <el-form-item :label="$t('api_test.request.sql.variable_names')" prop="variableNames">
              <el-input v-model="request.variableNames" maxlength="500" show-word-limit size="small"/>
            </el-form-item>

            <el-tabs v-model="activeName">
              <el-tab-pane :label="$t('api_test.scenario.variables')" name="variables">
                <ms-api-scenario-variables :is-read-only="isReadOnly" :items="request.variables"
                                           :description="$t('api_test.scenario.kv_description')"/>
              </el-tab-pane>
              <el-tab-pane :label="$t('api_test.request.sql.sql_script')" name="sql">
                <div class="sql-content">
                  <ms-code-edit mode="sql" :read-only="isReadOnly" :modes="['sql']" :data.sync="request.query" theme="eclipse" ref="codeEdit"/>
                </div>
              </el-tab-pane>
            </el-tabs>
          </el-form>
        </div>
      </el-col>
      <el-col :span="3" class="ms-left-cell" v-if="showScript">

        <el-button class="ms-left-buttion" size="small" style="color: #B8741A;background-color: #F9F1EA" @click="addPre">+{{$t('api_test.definition.request.pre_script')}}</el-button>
        <br/>
        <el-button class="ms-left-buttion" size="small" style="color: #783887;background-color: #F2ECF3" @click="addPost">+{{$t('api_test.definition.request.post_script')}}</el-button>
        <br/>
        <el-button class="ms-left-buttion" size="small" style="color: #FE6F71;background-color: #F2ECF3" @click="addPreSql">+{{$t('api_test.definition.request.pre_sql')}}</el-button>
        <br/>
        <el-button class="ms-left-buttion" size="small" style="color: #1483F6;background-color: #F2ECF3" @click="addPostSql">+{{$t('api_test.definition.request.post_sql')}}</el-button>
        <br/>
        <el-button class="ms-left-buttion" size="small" style="color: #A30014;background-color: #F7E6E9" @click="addAssertions">+{{$t('api_test.definition.request.assertions_rule')}}</el-button>
        <br/>
        <el-button class="ms-left-buttion" size="small" style="color: #015478;background-color: #E6EEF2" @click="addExtract">+{{$t('api_test.definition.request.extract_param')}}</el-button>
      </el-col>
    </el-row>

    <!-- 环境 -->
    <api-environment-config ref="environmentConfig" @close="environmentConfigClose"/>
  </div>
</template>

<script>
  import MsApiKeyValue from "../../ApiKeyValue";
  import MsApiAssertions from "../../assertion/ApiAssertions";
  import MsApiExtract from "../../extract/ApiExtract";
  import ApiRequestMethodSelect from "../../collapse/ApiRequestMethodSelect";
  import MsCodeEdit from "../../../../../common/components/MsCodeEdit";
  import MsApiScenarioVariables from "../../ApiScenarioVariables";
  import {createComponent} from "../../jmeter/components";
  import {Assertions, Extract} from "../../../model/ApiTestModel";
  import {parseEnvironment} from "../../../model/EnvironmentModel";
  import ApiEnvironmentConfig from "@/business/components/api/test/components/ApiEnvironmentConfig";
  import {getCurrentProjectID} from "@/common/js/utils";
  import {getUUID} from "@/common/js/utils";
  import MsJsr233Processor from "../../../../automation/scenario/component/Jsr233Processor";

  export default {
    name: "MsDatabaseConfig",
    components: {
      MsJsr233Processor,
      MsApiScenarioVariables,
      MsCodeEdit,
      ApiRequestMethodSelect, MsApiExtract, MsApiAssertions, MsApiKeyValue, ApiEnvironmentConfig
    },
    props: {
      request: {},
      basisData: {},
      moduleOptions: Array,
      showScript: {
        type: Boolean,
        default: true,
      },
      isReadOnly: {
        type: Boolean,
        default: false
      },
    },
    data() {
      return {
        spanNum: 21,
        environments: [],
        currentEnvironment: {},
        databaseConfigsOptions: [],
        isReloadData: false,
        activeName: "variables",
        rules: {},
      }
    },
    watch: {
      'request.dataSourceId'() {
        this.setDataSource();
      },
    },
    created() {
      if(this.showScript){
        this.spanNum = 21;
      }else {
        this.spanNum = 24;
      }
      this.getEnvironments();
    },
    computed: {
      projectId() {
        return getCurrentProjectID();
      },
    },
    methods: {
      addPre() {
        let jsr223PreProcessor = createComponent("JSR223PreProcessor");
        this.request.hashTree.push(jsr223PreProcessor);
        this.reload();
      },
      addPost() {
        let jsr223PostProcessor = createComponent("JSR223PostProcessor");
        this.request.hashTree.push(jsr223PostProcessor);
        this.reload();
      },
      addPreSql() {
        let jdbcPreProcessor = createComponent("JDBCPreProcessor");
        this.request.hashTree.push(jdbcPreProcessor);
      },
      addPostSql() {
        let jdbcPostProcessor = createComponent("JDBCPostProcessor");
        this.request.hashTree.push(jdbcPostProcessor);
      },
      addAssertions() {
        let assertions = new Assertions();
        this.request.hashTree.push(assertions);
        this.reload();
      },
      addExtract() {
        let jsonPostProcessor = new Extract();
        this.request.hashTree.push(jsonPostProcessor);
        this.reload();
      },
      remove(row) {
        let index = this.request.hashTree.indexOf(row);
        this.request.hashTree.splice(index, 1);
        this.reload();
      },
      copyRow(row) {
        let obj = JSON.parse(JSON.stringify(row));
        obj.id = getUUID();
        this.request.hashTree.push(obj);
        this.reload();
      },
      reload() {
        this.isReloadData = true
        this.$nextTick(() => {
          this.isReloadData = false
        })
      },
      validate() {
        this.$refs['request'].validate((valid) => {
          if (valid) {
            this.$emit('callback');
          }
        })
      },
      saveApi() {
        this.basisData.method = this.basisData.protocol;
        this.$emit('saveApi', this.basisData);
      },
      runTest() {

      },
      getEnvironments() {
        this.environments = [];
        let id = this.request.projectId ? this.request.projectId : this.projectId;
        this.$get('/api/environment/list/' + id, response => {
          this.environments = response.data;
          this.environments.forEach(environment => {
            parseEnvironment(environment);
          });
          if (!this.request.environmentId) {
            this.request.environmentId = this.$store.state.useEnvironment;
          }
          let hasEnvironment = false;
          for (let i in this.environments) {
            if (this.environments[i].id === this.request.environmentId) {
              hasEnvironment = true;
              break;
            }
          }
          if (!hasEnvironment) {
            this.request.environmentId = undefined;
          }
          if (!this.request.environmentId) {
            this.request.dataSourceId = undefined;
          }
          this.initDataSource();
        });
      },
      openEnvironmentConfig() {
        this.$refs.environmentConfig.open(getCurrentProjectID());
      },
      initDataSource() {
        let flag = false;
        for (let i in this.environments) {
          if (this.environments[i].id === this.request.environmentId) {
            this.databaseConfigsOptions = [];
            this.environments[i].config.databaseConfigs.forEach(item => {
              if (item.id === this.request.dataSourceId) {
                flag = true;
              }
              this.databaseConfigsOptions.push(item);
            });
            break;
          }
        }
        if (!flag) {
          this.request.dataSourceId = "";
        }
      },
      setDataSource() {
        this.initDataSource();

        for (let item of this.databaseConfigsOptions) {
          if (this.request.dataSourceId === item.id) {
            this.request.dataSource = item;
            break;
          }
        }
      },
      environmentChange(value) {
        this.request.dataSource = undefined;
        this.request.dataSourceId = "";
        for (let i in this.environments) {
          if (this.environments[i].id === value) {
            this.databaseConfigsOptions = [];
            this.environments[i].config.databaseConfigs.forEach(item => {
              this.databaseConfigsOptions.push(item);
            })
            break;
          }
        }
      },
      environmentConfigClose() {
        this.getEnvironments();
      },
    }

  }
</script>

<style scoped>
  .sql-content {
    height: calc(100vh - 570px);
  }

  .one-row .el-form-item {
    display: inline-block;
  }

  .one-row .el-form-item:nth-child(2) {
    margin-left: 60px;
  }

  .ms-left-cell {
    margin-top: 40px;
  }

  .ms-left-buttion {
    margin: 6px 0px 8px 30px;
  }

  /deep/ .el-form-item {
    margin-bottom: 15px;
  }
</style>
