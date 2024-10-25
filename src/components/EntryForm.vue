<template>
  <DialogForm ref="dialogForm">
    <template v-slot:header>
      <template v-if="isReferDialog">
        <h4 class="modal-title">{{ labels.title_view }}</h4>
      </template>
      <template v-else>
        <h4 class="modal-title" v-if="insertMode">{{ labels.title_new }}</h4>
        <h4 class="modal-title" v-if="updateMode">{{ labels.title_edit }}</h4>
      </template>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-height col-md-3">
            <label>{{ labels.transtime_label }}</label>
            <input type="text" v-model="showData.transtime" class="form-control input-md" disabled />
          </div>
          <div class="col-height col-md-8">
            <label>{{ labels.quotable_label }}</label>
            <input type="text" v-model="localData.quotable" class="form-control input-md" disabled />
          </div>
        </div>
        <div class="row row-height">
          <div class="col-height col-md-5">
            <label>{{ labels.email_label }}</label>
            <input type="text" v-model="localData.email" class="form-control input-md" disabled />
          </div>
          <div class="col-height col-md-6">
            <label>{{ labels.sender_label }}</label>
            <input type="text" v-model="localData.sender" class="form-control input-md" disabled />
          </div>
        </div>
        <div class="row row-height">
          <div class="col-height col-md-3">
            <label>{{ labels.statusname_label }}</label>
            <input type="text" v-model="localData.statusname" class="form-control input-md" disabled />
          </div>
        </div>
        <div class="row row-height">
          <div class="col-height col-md-12">
            <label>{{ labels.remark_label }}</label>
            <textarea v-model="localData.remark" class="form-control input-md" cols="100" rows="10" readonly></textarea>
          </div>
        </div>
        <div class="row row-height">
          <div class="col-height col-md-12" ref="mailcontents">
            <label>{{ labels.contents_label }}</label>
            <div ref="contents" class="form-control input-md contents-mailing" v-html="localData.contents"></div>
          </div>
        </div>
    </template>
    <template v-slot:footer>
      <button ref="referbutton" id="referbutton" class="btn btn-dark btn-sm pull-left" @click="referClick" v-if="hasRefer"><em class="fa fa-eye fa-btn-icon"></em>{{ labels.refer_button }}</button>
      <button ref="resendbutton" id="resendbutton" class="btn btn-dark btn-sm" @click="resendClick" v-if="displayResend"><em class="fa fa-send fa-btn-icon"></em>{{ labels.resend_button }}</button>
      <button class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<style>
#referbutton { min-width: 120px; }
div.contents-mailing { overflow: auto; min-height: 250px; }
</style>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, disableControls, Utilities }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, detectErrorResponse }  from '@willsofts/will-app';
import { confirmResend, successbox, serializeParameters } from '@willsofts/will-app';
import DialogForm from './DialogForm.vue';

const APP_URL = "/api/sftq003";
const defaultData = {
  keyid: "",
  transtime: "",
  quotable: "",
  email: "",
  sender: "",
  statusname: "",
  remark: "",
  contents: "",
};
const displayData = {
  transtime: "",
};

export default {
  components: {
    DialogForm
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object,
    referDialog: {
      type: [String,Boolean],
      default: true,
    },    
    canResend: {
      type: [String,Boolean],
      default: true,
    },    
  },
  emits: ["data-resended","data-refered"],
  setup(props) {
    const mode = ref({action: "new", ...props.modes});
    const localData = ref({ ...defaultData }); 
    const disabledKeyField = ref(false);
    const showData = ref({...displayData});
    return { mode, localData, disabledKeyField, showData };
  },
  computed: {
    insertMode() {
      return this.mode.action=="insert" || this.mode.action=="new";
    },
    updateMode() {
      return this.mode.action=="update" || this.mode.action=="edit";
    },
    hasRefer() {
      return this.localData.refer && this.localData.refer.trim().length > 0;
    },
    isReferDialog() { return Utilities.parseBoolean(this.$props.referDialog); },
    displayResend() { return Utilities.parseBoolean(this.$props.canResend); },
  },
  mounted() {
    this.$nextTick(function () {
      $("#modaldialog_layer").find(".modal-dialog").draggable();
    });
  },
  methods: {
    reset(newData,newMode) {
      if(newData) this.localData = {...newData};
      if(newMode) this.mode = {...newMode};
      let transtime = "";
      if(this.localData.transtime && this.localData.transtime.trim().length > 0) {
        let date = Utilities.parseDate(this.localData.transtime);
        if(date) transtime = Utilities.formatDateTime(date);
      }
      this.showData.transtime = transtime;
    },
    showDialog(callback) {
      if(callback) $(this.$refs.dialogForm.$el).on("shown.bs.modal",callback);
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    resetRecord() {
      //reset to default data 
      this.reset(defaultData,{action:"insert"});
      //enable key field
      this.disabledKeyField = false;
    },
    startResendRecord() {
      confirmResend(() => {
        this.resendRecord(this.localData);
      });
    },
    resendRecord(dataRecord) {
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getApiUrl()+APP_URL+"/resend",
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
            console.log("success",data);
            if(detectErrorResponse(data)) return;
            successbox(() => {
              this.hideDialog();
              this.$emit('data-resended',dataRecord,data);
            });
          }
      });
    },
    retrieveRecord(dataKeys) {
      let jsondata = {ajax: true};
      let formdata = serializeParameters(jsondata,dataKeys);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/retrieve",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("retrieveRecord: error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("retrieveRecord: success",data);
          if(data.body.dataset) {
            this.reset(data.body.dataset,{action:"edit"});
            this.disabledKeyField = true;
            this.showDialog(() => {
              this.diableLinkedDocument();
            });
          }
        }
      });	
    },
    diableLinkedDocument() {
      $("a",$(this.$refs.mailcontents.$el)).attr("disabled",true).addClass("disabled").on("click",function(e){
        e.preventDefault();
      });
    },
    resendClick() {
      console.log("click: resend");
      disableControls($("#resendbutton"));
      this.startResendRecord();
    },
    referClick() {
      console.log("click: refer");
      disableControls($("#referbutton"));
      if(this.hasRefer) {
        this.$emit('data-refered',this.localData);
      }
    },   
  }
};
</script>
