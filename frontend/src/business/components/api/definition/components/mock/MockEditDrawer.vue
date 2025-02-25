<template>
  <ms-drawer :size="60" @close="close" :visible="showDrawer" direction="bottom" ref="mockDrawer">
    <template v-slot:header>
      <mock-config-header
        :mock-expect-config="mockExpectConfig"
        :show-head-table="showHeadTable"
        @saveMockExpectConfig="saveMockExpectConfig"
      />
    </template>
    <div>
      <!--  期望详情 -->
      <p class="tip">{{ $t('api_test.mock.request_condition') }}</p>
      <el-form :model="mockExpectConfig" :rules="rule" ref="mockExpectForm" label-width="80px" label-position="right">

        <div class="card">
          <div class="base-info">
            <el-row>
              <tcp-params
                v-if="isTcp" :show-operation-col="true"
                :request="mockExpectConfig.request" style="margin: 10px 10px;" ref="tcpParam"></tcp-params>
              <mock-request-param
                v-else
                :isShowEnable="false"
                :referenced="true"
                :is-read-only="false"
                :api-params="apiParams"
                :request="mockExpectConfig.request.params"/>
            </el-row>

            <el-row style="margin-top: 10px;">
              <el-col :span="12">
                <p class="tip">{{ $t('api_test.mock.rsp_param') }}</p>
              </el-col>
              <el-col :span="12">
                <el-button class="ms-right-buttion" size="small" @click="addPostScript">
                  +{{ $t('api_test.definition.request.post_script') }}
                </el-button>
              </el-col>
            </el-row>
            <el-row>
              <mock-response-param :api-id="apiId" :is-tcp="isTcp"
                                   :response="mockExpectConfig.response.responseResult" ref="mockResponseParam"/>
            </el-row>
          </div>
        </div>
      </el-form>
    </div>
  </ms-drawer>
</template>
<script>

import MsDrawer from "@/business/components/common/components/MsDrawer";
import {REQUEST_HEADERS} from "@/common/js/constants";
import MockRowVariables from "@/business/components/api/definition/components/mock/MockRowVariables";
import MsCodeEdit from "@/business/components/common/components/MsCodeEdit";
import MockConfigHeader from "@/business/components/api/definition/components/mock/MockConfigHeader";
import MockRequestParam from "@/business/components/api/definition/components/mock/Components/MockRequestParam";
import MockResponseParam from "@/business/components/api/definition/components/mock/Components/MockResponseParam";
import {getUUID} from "@/common/js/utils";
import TcpParams from "@/business/components/api/definition/components/request/tcp/TcpParams";

export default {
  name: 'MockEditDrawer',
  components: {
    MsDrawer, MockConfigHeader, MockRowVariables, MsCodeEdit, MockRequestParam, MockResponseParam, TcpParams
  },
  props: {
    apiParams: Object,
    apiId: String,
    mockConfigId: String,
    isTcp: {
      type: Boolean,
      default: false,
    }
  },
  data() {
    return {
      showDrawer: false,
      mockExpectConfig: {},
      showHeadTable: true,
      headerSuggestions: REQUEST_HEADERS,
      baseMockExpectConfig: {
        id: "",
        name: "",
        mockConfigId: "",
        request: {
          reportType: "raw",
          jsonParam: false,
          variables: [],
          jsonData: "{}",
          params: {
            headers: [],
            arguments: [],
            rest: [],
            body: {
              type: 'JSON',
              binary: [],
              kvs: [],
            }
          }
        },
        response: {
          httpCode: "",
          httpHeads: [],
          body: "",
          responseResult: {
            delayed: 0,
            headers: [],
            arguments: [],
            rest: [],
            body: {
              type: 'JSON',
              binary: []
            }
          }
        },
      },
      rule: {
        name: [
          {required: true, message: this.$t('test_track.case.input_name'), trigger: 'blur'},
          {max: 100, message: this.$t('test_track.length_less_than') + '100', trigger: 'blur'}
        ],
        response: {
          httpCode: [{required: true, message: this.$t('api_test.mock.rule.input_code'), trigger: 'blur'},],
          delayed: [{required: true, message: this.$t('test_track.case.input_name'), trigger: 'blur'},],
        },
      },
    };
  },
  watch: {},
  created() {
    this.mockExpectConfig = JSON.parse(JSON.stringify(this.baseMockExpectConfig));
  },
  computed: {},
  methods: {
    addPostScript() {
      this.$refs.mockResponseParam.setUsePostScript();
    },
    uuid: function () {
      return (((1 + Math.random()) * 0x100000) | 0).toString(16).substring(1);
    },
    open(param) {
      this.mockExpectConfig = JSON.parse(JSON.stringify(this.baseMockExpectConfig));
      if (param) {
        this.mockExpectConfig = param;
        if (!this.mockExpectConfig.request.params) {

          let requestParamsObj = {
            rest: [],
            headers: [],
            arguments: [],
            body: {
              type: "JSON",
              raw: "",
              kvs: [],
            },
          };
          this.$set(this.mockExpectConfig.request, "params", requestParamsObj);

          if (this.mockExpectConfig.request.jsonParam && this.mockExpectConfig.request.jsonData) {
            this.mockExpectConfig.request.params.body.type = "JSON";
            this.mockExpectConfig.request.params.body.raw = this.mockExpectConfig.request.jsonData;
          } else if (this.mockExpectConfig.request.variables) {
            this.mockExpectConfig.request.params.body.type = "Form Data";
            let headerItem = {};
            headerItem.enable = true;
            this.mockExpectConfig.request.params.headers.push(headerItem);
            this.mockExpectConfig.request.variables.forEach(item => {
              this.mockExpectConfig.request.params.arguments.push({
                description: "",
                type: "text",
                name: item.name,
                value: item.value,
                required: true,
                contentType: "text/plain",
                uuid: this.uuid(),
              });
              this.mockExpectConfig.request.params.body.kvs.push({
                description: "",
                type: "text",
                name: item.name,
                value: item.value,
                required: true,
                contentType: "text/plain",
                uuid: this.uuid(),
              });
            });
          }
        }
        if (!this.mockExpectConfig.response.responseResult) {
          let responseResultObj = {
            headers: [],
            arguments: [],
            rest: [],
            httpCode: this.mockExpectConfig.response.httpCode,
            delayed: this.mockExpectConfig.response.delayed,
            body: {
              type: "Raw",
              raw: this.mockExpectConfig.response.body,
              binary: []
            }
          };
          this.$set(this.mockExpectConfig.response, "responseResult", responseResultObj);
          if (this.mockExpectConfig.response.httpHeads) {
            this.mockExpectConfig.response.httpHeads.forEach(item => {
              this.mockExpectConfig.response.responseResult.headers.push({
                enable: true,
                name: item.name,
                value: item.value,
              });
            });
          }

        }
      }
      this.showDrawer = true;
      this.$nextTick(() => {
        this.$refs.mockDrawer.setfullScreen();
      });
    },
    close() {
      this.showDrawer = false;
    },
    saveMockExpectConfig() {
      if (this.isTcp && this.$refs.tcpParam) {
        this.$refs.tcpParam.saveData();
      }
      let mockConfigId = this.mockConfigId;
      this.mockExpectConfig.mockConfigId = mockConfigId;
      let formCheckResult = this.checkMockExpectForm("mockExpectForm", true);
    },
    cleanMockExpectConfig() {
      this.showHeadTable = false;
      this.mockExpectConfig = JSON.parse(JSON.stringify(this.baseMockExpectConfig));
      this.$nextTick(function () {
        this.showHeadTable = true;
      });
    },
    updateMockExpectConfig() {
      this.checkMockExpectForm("mockExpectForm");
    },
    uploadMockExpectConfig(clearForm) {
      let url = "/mockConfig/updateMockExpectConfig";
      let param = this.mockExpectConfig;
      if (!param.name || param.name === '') {
        this.$error(this.$t('test_track.case.input_name'));
        return false;
      } else if (param.name.length > 100) {
        this.$error(this.$t('test_track.length_less_than') + 100);
        return false;
      }

      if (!param.response.responseResult.httpCode) {
        param.response.responseResult.httpCode = 200;
      }

      if (!param.request.params.id) {
        param.request.params.id = getUUID();
      }
      let obj = {
        request: param.request.params,
        response: param.response.responseResult
      };
      let bodyFiles = this.getBodyUploadFiles(obj);

      this.$fileUpload(url, null, bodyFiles, param, response => {
        let returnData = response.data;
        this.mockExpectConfig.id = returnData.id;
        this.$emit('refreshMockInfo', param.mockConfigId);
        if (clearForm) {
          this.cleanMockExpectConfig();
        }
        this.$message({
          type: 'success',
          message: this.$t('commons.save_success'),
        });
        this.close();
      });
    },
    getBodyUploadFiles(data) {
      let bodyUploadFiles = [];
      data.bodyUploadIds = [];
      let request = data.request;
      let response = data.response;
      if (request.body) {
        if (request.body.kvs) {
          request.body.kvs.forEach(param => {
            if (param.files) {
              param.files.forEach(item => {
                if (item.file) {
                  let fileId = getUUID().substring(0, 8);
                  item.name = item.file.name;
                  item.id = fileId;
                  data.bodyUploadIds.push(fileId);
                  bodyUploadFiles.push(item.file);
                }
              });
            }
          });
        }
        if (request.body.binary) {
          request.body.binary.forEach(param => {
            if (param.files) {
              param.files.forEach(item => {
                if (item.file) {
                  let fileId = getUUID().substring(0, 8);
                  item.name = item.file.name;
                  item.id = fileId;
                  data.bodyUploadIds.push(fileId);
                  bodyUploadFiles.push(item.file);
                }
              });
            }
          });
        }
      }
      if (response.body) {
        if (response.body.kvs) {
          response.body.kvs.forEach(param => {
            if (param.files) {
              param.files.forEach(item => {
                if (item.file) {
                  let fileId = getUUID().substring(0, 8);
                  item.name = item.file.name;
                  item.id = fileId;
                  data.bodyUploadIds.push(fileId);
                  bodyUploadFiles.push(item.file);
                }
              });
            }
          });
        }
        if (response.body.binary) {
          response.body.binary.forEach(param => {
            if (param.files) {
              param.files.forEach(item => {
                if (item.file) {
                  let fileId = getUUID().substring(0, 8);
                  item.name = item.file.name;
                  item.id = fileId;
                  data.bodyUploadIds.push(fileId);
                  bodyUploadFiles.push(item.file);
                }
              });
            }
          });
        }
      }
      return bodyUploadFiles;
    },
    checkMockExpectForm(formName, clearForm) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.uploadMockExpectConfig(clearForm);
          return true;
        } else {
          return false;
        }
      });
    },
  }
};
</script>

<style scoped>

.ms-drawer >>> .ms-drawer-body {
  margin-top: 40px;
}

.base-info {
  width: 99%;
  margin-left: 5px;
}

.ms-right-buttion {
  float: right;
  margin: 6px 0px 8px 30px;
  color: #783887;
  background-color: #F2ECF3;
  border: #F2ECF3;
}
</style>
