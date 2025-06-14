<template>
<div id="searchpanel" class="panel-body search-panel">
    <div class="row row-height">
      <div class="col-height col-md-2">
          <label for="datefrom">{{ labels.datefroms_label }}</label>
          <div class="input-group has-validation" :class="{'has-error': v$.datefrom.$error}">
              <div class="date-control">
                <InputDate v-model="localData.datefrom" id="datefrom" ref="datefrom" />
                <label class="required">*</label>
              </div>
          </div> 
          <span v-if="v$.datefrom.$error" class="has-error">{{ v$.datefrom.$errors[0].$message }}</span>
      </div>
      <div class="col-height col-md-2">
          <label for="dateto">{{ labels.datetos_label }}</label>
          <div class="input-group has-validation" :class="{'has-error': v$.dateto.$error}">
            <div class="date-control">
              <InputDate v-model="localData.dateto" id="dateto" ref="dateto" />
              <label class="required">*</label>
            </div>
          </div>
          <span v-if="v$.dateto.$error" class="has-error">{{ v$.dateto.$errors[0].$message }}</span>
      </div>
      <div class="col-height col-md-2">
        <label for="trxstatuss">{{ labels.trxstatuss_label }}</label>
        <select ref="trxstatuss" id="trxstatuss" v-model="localData.trxstatus" class="form-control input-md">
          <option value=""></option>          
          <option v-for="item in dataCategory.tkrxstatus" :key="item.id" :value="item.id">{{item.text}}</option>
        </select>
      </div>
      <div class="col-height col-md-3">
          <label>{{ labels.emails_label }}</label>
          <input type="text" v-model="localData.owner" class="form-control input-md" maxlength="100" />
      </div>
      <div class="col-height col-md-3">
          <label>{{ labels.quotables_label }}</label>
          <input type="text" v-model="localData.quotable" class="form-control input-md" maxlength="100" />
      </div>
    </div>
    <div class="row row-height">
      <div class="col-height col-md-12 pull-right text-right">
          <button id="searchbutton" @click="searchClick" class="btn btn-dark btn-sm btn-ctrl"><i class="fa fa-search fa-btn-icon" aria-hidden="true"></i>{{ labels.search_button }}</button>
          <button id="resetbutton" @click="resetClick" class="btn btn-dark btn-sm btn-ctrl"><i class="fa fa-refresh fa-btn-icon" aria-hidden="true"></i>{{ labels.reset_button }}</button>
      </div>
    </div>
    <div id="listpanel" class="table-responsive fa-list-panel">
        <DataTable ref="dataTable" :settings="tableSettings" :labels="labels" :dataset="dataset" @data-select="dataSelected" @data-sort="dataSorted" />
        <DataPaging ref="dataPaging" :settings="pagingSettings" @page-select="pageSelected" />
    </div>
</div>
</template>
<style>
#resetbutton { margin-right: 0px; }
</style>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, Utilities }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, serializeParameters }  from '@willsofts/will-app'
import { Paging } from "@willsofts/will-app";
import { InputDate, DataTable, DataPaging } from '@willsofts/will-control';
import { computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';

const curdate = Utilities.getDateNow();
const APP_URL = "/api/sftq003";
const defaultData = {
  trxstatus: "",
  datefrom: curdate,
  dateto: curdate,
  owner: "",
  quotable: "",
};

const tableSettings = {
    sequence: { label: "seqno_label" },
    columns: [
        {name: "transtime", type: "DATETIME", sorter: "transtime", label: "transtime_headerlabel", css: "text-center" },
        {name: "owner", type: "STRING", sorter: "owner", label: "email_headerlabel" },
        {name: "quotable", type: "STRING", sorter: "quotable", label: "quotable_headerlabel" },
        {name: "statusname", type: "STRING", sorter: "statusname", label: "statusname_headerlabel", css: "text-center" },
        {name: "remark", type: "STRING", sorter: "remark", label: "remark_headerlabel" }
    ],        
    actions: [
        {type: "button", action: "edit"},
        { 
          render: function(record) { 
            return record.refer && record.refer.trim().length > 0 ? { type: "a", action: "view", css: "btn-view fa-data-view", icon: "fa fa-eye" } : {};
          }
        }
    ],
};

export default {
  components: {
    InputDate, DataTable, DataPaging
  },
  props: {
    labels: Object,
    formData: Object,
    dataCategory: Object
  },
  emits: ["data-select","data-insert"],
  setup(props) {
    const localData = ref({ ...defaultData, ...props.formData });
    let paging = new Paging();
    let pagingSettings = paging.setting;
    let filters = {};
    const dataset = ref({});

    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        datefrom: { required: requiredMessage() },
        dateto: { required: requiredMessage() },
      } 
    });
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });    

    return { v$, localData, tableSettings, pagingSettings, paging, filters, dataset };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
    });
  },
  methods: {
    async validateForm(focusing=true) {
      const valid = await this.v$.$validate();
      console.log("validate form: valid",valid);
      console.log("error:",this.v$.$errors);
      if(!valid) {
        if(focusing) {
          this.focusFirstError();
        }
        return false;
      }
      return true;
    },
    focusFirstError() {
      if(this.v$.$errors && this.v$.$errors.length > 0) {
        let input = this.v$.$errors[0].$property;
        let el = this.$refs[input];
        if(el) el.focus(); //if using ref
        else $("#"+input).trigger("focus"); //if using id
      }
    },
    reset(newData) {
      if(newData) this.localData = {...newData};
    },
    getPagingOptions(settings) {
      if(!settings) settings = this.pagingSettings;
      return {page: settings.page, limit: settings.limit, rowsPerPage: settings.rowsPerPage, orderBy: settings.orderBy?settings.orderBy:"", orderDir: settings.orderDir?settings.orderDir:"" };
    },
    resetClick() {
      this.localData = {...defaultData};
      this.resetFilters();
      this.$refs.dataTable.clear();
      this.$refs.dataPaging.clear();
      this.pagingSettings.rows = 0;
    },
    async searchClick() {
      let valid = await this.validateForm();
      if(!valid) return;
      this.filters = {...this.localData};
      this.resetFilters();
      this.search();
    },
    resetFilters() {
      this.pagingSettings.page = 1;
      this.pagingSettings.orderBy = "";
      this.pagingSettings.orderDir = "";
    },
    search(ensurePaging=false) {
      if(ensurePaging) {
        if(this.pagingSettings.rows <= 1 && this.pagingSettings.page > 1) {
          this.pagingSettings.page = this.pagingSettings.page - 1;
        }
      }
      console.log("search: pagingSettings",this.pagingSettings);
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
    collecting(options,criterias) {
      console.log("collecting: options",options,", criteria",criterias);
      let jsondata = Object.assign({ajax: true},options);
      let formdata = serializeParameters(jsondata,criterias);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/collect",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("collecting: success",data);
          if(data.body) {
            this.$refs.dataTable.reset(data.body);
            this.$refs.dataPaging.reset(data.body?.offsets);
            this.pagingSettings.rows = data.body?.rows?.length;
          }
        }
      });	
    },
    pageSelected(item) {
      console.log("page selected:",item);
      this.pagingSettings.page = item.page;
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
    dataSelected(item,action) {
      console.log("dataSelected",item,"action",action);
      this.$emit('data-select', item, action);
    },
    dataSorted(sorter,direction) {
      console.log("dataSorted",sorter,"direction",direction);
      this.pagingSettings.orderBy = sorter;
      this.pagingSettings.orderDir = direction;
      let options = this.getPagingOptions();
      this.collecting(options,this.filters);
    },
  }
};
</script>
