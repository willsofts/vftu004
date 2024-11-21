<template>
  <DialogForm ref="dialogForm">
    <template v-slot:header>
      <h4 class="modal-title" v-if="insertMode">{{ labels.title_new }}</h4>
      <h4 class="modal-title" v-if="updateMode">{{ labels.title_edit }}</h4>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-height col-md-8">
            <label for="keyname">{{ labels.keyname_label }}</label>
            <div class="input-group has-validation" :class="{'has-error': v$.keyname.$error}">
              <input type="text" ref="keyname" v-model="localData.keyname" id="keyname" class="form-control input-md" maxlength="100" /> 
              <label class="required">*</label>
            </div>
            <span v-if="v$.keyname.$error" class="has-error">{{ v$.keyname.$errors[0].$message }}</span>
          </div>
        </div>
        <div class="row row-height hidden-layer">
          <div class="col-height col-md-4">
            <label for="expireflag">{{ labels.expireflag_label }}</label>
            <fieldset :disabled="!insertMode">
            <select ref="expireflag" v-model="localData.expireflag" id="expireflag" class="form-control input-md">
              <option v-for="item in dataCategory.tkexpire" :key="item.id" :value="item.id">{{item.text}}</option>
            </select>
            </fieldset>
          </div>
          <div class="col-height col-md-4">
            <label for="expireday">{{ labels.expireday_label }}</label>
            <div class="input-group has-validation" :class="{'has-error': v$.expireday.$error}">
              <InputNumber ref="expireday" v-model="localData.expireday" id="expireday" :disabled="!canEntry" /> 
              <label class="required" v-if="canEntry">*</label>
            </div>
            <span v-if="v$.expireday.$error" class="has-error">{{ v$.expireday.$errors[0].$message }}</span>
          </div>
        </div>
        <div class="row row-height hidden-layer copy-layer" v-show="displayCopy">
          <div class="col-height col-md-11">
            <label for="keypass" class="control-label">{{ labels.keypass_label }}</label>
            <div class="input-group">
              <textarea ref="keypass" id="keypass" class="form-control input-md" rows="5" v-model="localData.keypass" readonly></textarea>
              <div class="input-group-required input-group-addon input-group-append">
                <table>
                  <tr><td valign="top">
                  <a href="javascript:void(0);" class="input-group-text key-link" @click="copyTokenKey" tabindex="-1" title="Copy"><i class="fa fa-copy" aria-hidden="true"></i></a>
                  <a href="javascript:void(0);" class="input-group-text key-link" @click="downloadTokenKey" tabindex="-1" title="Download"><i class="fa fa-download" aria-hidden="true"></i></a>
                  </td></tr>
                </table>
              </div>
            </div>
					</div>
        </div>
        <div class="row row-height hidden-layer copy-layer" v-show="displayCopy">
          <div class="col-height col-md-4">
            <label>{{ labels.expiredate_label }}</label>
            <input type="text" ref="expiredate" v-model="localData.expiredate" id="expiredate" class="form-control input-md" disabled /> 
          </div>
          <div class="col-height col-md-4">
            <label>{{ labels.expiretime_label }}</label>
            <input type="text" ref="expiretime" v-model="localData.expiretime" id="expiretime" class="form-control input-md" disabled /> 
          </div>          
        </div>
    </template>
    <template v-slot:footer>
      <button ref="savebutton" id="savebutton" class="btn btn-dark btn-sm" @click="saveClick" v-if="insertMode"><em class="fa fa-save fa-btn-icon"></em>{{ labels.save_button }}</button>
      <button ref="updatebutton" id="updatebutton" class="btn btn-dark btn-sm" @click="updateClick" v-if="updateMode"><em class="fa fa-save fa-btn-icon"></em>{{ labels.update_button }}</button>
      <button class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<script>
import { ref, computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, disableControls }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, detectErrorResponse }  from '@willsofts/will-app';
import { confirmUpdate, confirmSave, confirmDelete, successbox, serializeParameters } from '@willsofts/will-app';
import { InputNumber } from '@willsofts/will-control';
import DialogForm from './DialogForm.vue';

const APP_URL = "/api/sftu004";
const defaultData = {
  keyid: "",
  keyname: "",
  expireflag: "0",
  expireday: "30",
  keypass: "",
  expiredate: "",
  expiretime: "",
};

export default {
  components: {
    DialogForm, InputNumber
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object
  },
  emits: ["data-saved","data-updated","data-deleted"],
  setup(props) {
    const mode = ref({action: "new", ...props.modes});
    const localData = ref({ ...defaultData }); 
    const disabledKeyField = ref(false);
    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const requiredField = () => { 
      let inserting = mode.value.action=="insert" || mode.value.action=="new";
      return inserting && (localData.value.expireflag == "0") ? helpers.withMessage(reqalert, required) : false; 
    };
    const validateRules = computed(() => { 
      return {
        keyname: { required: requiredMessage() },
        expireday: { required: requiredField() },
      } 
    });
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });
    const displayCopy = ref(false);
    const keepexpireday = ref("30");
    watch(() => localData.value.expireflag, (newFlag) => {
      if(newFlag == "0") {
        localData.value.expireday = keepexpireday.value;
      } else {
        keepexpireday.value = localData.value.expireday;
        if(keepexpireday.value=="") keepexpireday.value = "30";
        localData.value.expireday = "";
      }
    });
    return { mode, v$, localData, disabledKeyField, reqalert, displayCopy, keepexpireday };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
    });
  },
  computed: {
    insertMode() {
      return this.mode.action=="insert" || this.mode.action=="new";
    },
    updateMode() {
      return this.mode.action=="update" || this.mode.action=="edit";
    },
    canEntry() {
      return (this.mode.action=="insert" || this.mode.action=="new") && (this.localData.expireflag == "0");
    }
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
    },
    async saveClick() {
      console.log("click: save");
      disableControls($("#savebutton"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.startSaveRecord();
    },
    async updateClick() {
      console.log("click: update");
      disableControls($("#updatebutton"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.startUpdateRecord();
    },
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
      //reset validator
      this.v$.$reset();
      //enable key field
      this.disabledKeyField = false;
      this.displayCopy = false;
    },
    scrapeData(dataset) {
      let resData = {...defaultData};
      if(dataset) {
        resData = Object.keys(resData).reduce((data, key) => {
          if(Object.hasOwn(dataset,key)) {
            data[key] = dataset[key];
          }
          return data;
        },{});
      }
      if(resData.expireflag=="1") {
        resData.expiredate = "";
        resData.expiretime = "";
        resData.expireday = "";
      }
      return resData;
    },
    startInsertRecord() {
      this.resetRecord();
      this.showDialog(() => { this.$refs.keyname.focus(); });
    },
    startSaveRecord() {
      confirmSave(() => {
        this.saveRecord(this.localData);
      });
    },
    startUpdateRecord() {
      confirmUpdate(() => { 
        this.updateRecord({keyid: this.localData.keyid, keyname: this.localData.keyname});
      });
    },
    startDeleteRecord(dataKeys) {
      confirmDelete(Object.values(dataKeys),() => {
        this.deleteRecord(dataKeys);
      });
    },
    saveRecord(dataRecord) {
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getApiUrl()+APP_URL+"/insert",
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
            let dataset = this.scrapeData(data.body.dataset);
            this.reset(dataset,{action:"view"});
            this.v$.$reset();
            this.disabledKeyField = true;
            this.displayCopy = true;
            successbox(() => {
              //reset data for new record insert
              //this.resetRecord();
              this.$emit('data-saved',dataRecord,data);
            });
          }
      });
    },
    updateRecord(dataRecord) {
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getApiUrl()+APP_URL+"/update",
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
              this.$emit('data-updated',dataRecord,data);
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
            let dataset = this.scrapeData(data.body.dataset);
            this.reset(dataset,{action:"edit"});
            this.v$.$reset();
            this.disabledKeyField = true;
            this.displayCopy = true;
            this.showDialog(() => { this.$refs.keyname.focus(); });
          }
        }
      });	
    },
    deleteRecord(dataKeys) {
      let jsondata = {ajax: true};
      let formdata = serializeParameters(jsondata,dataKeys);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/remove",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("deleteRecord: error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("deleteRecord: success",data);
          if(detectErrorResponse(data)) return;
          successbox(() => {
            this.$emit('data-deleted',dataKeys,data);
          });
        }
      });	
    },
    async copyTokenKey() {
      try {
        let copyText = document.getElementById("keypass");
        copyText.select();
        await navigator.clipboard.writeText(copyText.value);
      } catch(ex) { console.error(ex); }
    },
    downloadTokenKey() {
      this.createDownloader(this.localData.keypass,this.localData.keyname+".txt");
    },
    createDownloader(data, filename) {
      let a = window.document.createElement('a');
      a.href = window.URL.createObjectURL(new Blob([data], { type: 'text/html' }));
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
    },
  }
};
</script>
