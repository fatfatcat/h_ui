<template>
  <div :class="wrapCls">
    <div
      :class="classes"
      @click="handleClick"
      @drop.prevent="onDrop"
      @dragover.prevent="dragOver = true"
      @dragleave.prevent="dragOver = false">
      <input
        ref="input"
        type="file"
        :class="[prefixCls + '-input']"
        @change="handleChange"
        :multiple="multiple"
        :accept="accept">
      <!-- 手动配置 -->
      <slot v-if="selfConfig" name="chooseFile"></slot>
      <slot v-else></slot>
    </div>
    <slot name="tip"></slot>
    <div  v-if="selfConfig" @click="handlePostFile" :class="postBtnCls">
      <slot name="postFile" ></slot>
    </div>    
    <div  v-if="selfConfig && $slots.cancleFile" @click="handleCancleFile" :class="cancleBtncls">
      <slot name="cancleFile" @click="handleCancleFile"></slot>
    </div>
    <div  v-if="selfConfig " @click="handleFileList" :class="listBtnCls">
      <slot name="showList"></slot>
    </div>

    <upload-list
      v-if = "showUploadList"
      v-show = "!showUploadedList"
      :files = "fileList"
      @on-file-remove = "handleRemove"
      @on-file-preview = "handlePreview"
      ref="postFileList"></upload-list>

    <!-- 已上传列表 -->
    <div v-if = "selfConfig && showUploadList"
         v-show = "showUploadedList"
         :class="[prefixCls + '-uploaded-wrap']">
      <div :class="[prefixCls + '-uploaded-title']">
        <span>
          已上传文件列表
        </span>
         <Icon
          name="close"
          :class="[prefixCls + '-uploaded-close']"
          @click.native ="handleDelAll"></Icon>
        <Icon
          name="smallscreen"
          :class="[prefixCls + '-uploaded-close']"
          @click.native ="handleCloseList"></Icon>
      </div>
      <div>
        <upload-list  
          :files = "uploadedFileList"
          showUploaded = "showLi"
          @on-file-remove = "handleUploadedRemove"
          @on-file-preview = "handlePreview"></upload-list>
        <!-- 其中showUploadList,用于控制是否显示列表，showUploaded用于控制是否显示已上传列表，当显示已上传列表时，isShow失效 -->
      </div>
    </div>
  </div>
</template>
<script>
import Icon from '../Icon/Icon.vue';
import UploadList from './Upload-list.vue';
import ajax from './ajax.js';
import { oneOf } from '../../util/tools.js';
import Emitter from '../../mixins/emitter';

const prefixCls = 'h-upload';

export default {
  name: 'Upload',
  mixins: [ Emitter ],
  components: { UploadList , Icon},
  props: {
    action: {
      type: String,
      required: true
    },
    headers: {
      type: Object,
      default () {
        return {};
      }
    },
    multiple: {
      type: Boolean,
      default: false
    },
    data: {
      type: Object
    },
    name: {
      type: String,
      default: 'file'
    },
    withCredentials: {
      type: Boolean,
      default: false
    },
    showUploadList: {
      type: Boolean,
      default: true
    },
    type: {
      type: String,
      validator (value) {
        return oneOf(value, ['select', 'drag']);
      },
      default: 'select'
    },
    format: {
      type: Array,
      default () {
        return [];
      }
    },
    accept: {
      type: String
    },
    maxSize: {
      type: Number
    },
    beforeUpload: Function,
    onProgress: {
      type: Function,
      default () {
        return {};
      }
    },
    onSuccess: {
      type: Function,
      default () {
        return {};
      }
    },
    onError: {
      type: Function,
      default () {
        return {};
      }
    },
    onRemove: {
      type: Function,
      default () {
        return {};
      }
    },
    onPreview: {
      type: Function,
      default () {
        return {};
      }
    },
    onExceededSize: {
      type: Function,
      default () {
        return {};
      }
    },
    onFormatError: {
      type: Function,
      default () {
        return {};
      }
    },
    defaultFileList: {
      type: Array,
      default() {
        return [];
      }
    },
    selfConfig: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      prefixCls: prefixCls,
      dragOver: false,
      fileList: [],
      tempIndex: 1,
      selfUploadStatus: '',
      xhr:'',
      uploadedFileList: [], // 已上传列表
      showUploadedList: false, // 是否显示已上传列表
    };
  },
  computed: {
    wrapCls () {
      return [
          `${prefixCls}`,
          {
            [`${prefixCls}-self`]: this.selfConfig
          }
        ]
    },
    classes () {
      return [
        `${prefixCls}`,
        {
          [`${prefixCls}-select`]: this.type === 'select',
          [`${prefixCls}-drag`]: this.type === 'drag',
          [`${prefixCls}-dragOver`]: this.type === 'drag' && this.dragOver
        }
      ];
    },
    postBtnCls () {
      return [
        `${prefixCls}`,
        `${prefixCls}-postBtn`,
      ];
    },
    cancleBtncls () {
      return [
        `${prefixCls}`,
        `${prefixCls}-cancleBtn`,
      ];
    },
    listBtnCls () {
      return [
        `${prefixCls}`,
        `${prefixCls}-listBtn`,
      ];
    },
    // 已上传列表长度，用于判定是否禁用该按钮
    uploadedFileListLength () {
      return this.uploadedFileList.length
    }
  },
  methods: {
    handlePostFile () {
      if(this.showUploadedList) this.showUploadedList = false
      this.fileList.forEach(file => {
        if(file.status !== 'finished'){
          this.startPost(file)
        }
      })
    },
    handleCancleFile () {
      if(this.xhr) this.xhr.abort()
      this.xhr =''
      this.fileList = this.fileList.filter((file, index) => {
        if(file.status === "finished") {
          return file
        }
      })
      
    },
    handleFileList () {
      this.showUploadedList = true
    },
    handleCloseList () {
      this.showUploadedList = false
    },
    handleDelAll () {
      this.uploadedFileList = []
      this.fileList = this.fileList.filter((file, index) => {
        if(file.status !== "finished") {
          return file
        }
      })
    },
    handleClick () {
      this.$refs.input.click();
    },
    handleChange (e) {
      const files = e.target.files;

      if (!files) {
          return;
      }
      this.uploadFiles(files);
      this.$refs.input.value = null;
    },
    onDrop (e) {
      this.dragOver = false;
      this.uploadFiles(e.dataTransfer.files);
    },
    uploadFiles (files) {
      let postFiles = Array.prototype.slice.call(files);
      if (!this.multiple) postFiles = postFiles.slice(0, 1);

      if (postFiles.length === 0) return;

      postFiles.forEach(file => {
        this.upload(file);
          
      });
    },
    upload (file) {
      if (!this.beforeUpload) {
          return this.post(file);
      }
      const before = this.beforeUpload(file);
      if (before && before.then) {
        before.then(processedFile => {
          if (Object.prototype.toString.call(processedFile) === '[object File]') {
            this.post(processedFile);
          } else {
            this.post(file);
          }
        }, () => {
          // this.$emit('cancel', file);
        });
      } else if (before !== false) {
        this.post(file);
      } else {
        // this.$emit('cancel', file);
      }
    },
    post (file) {
      // check format
      if (this.format.length) {
        const _file_format = file.name.split('.').pop().toLocaleLowerCase();
        const checked = this.format.some(item => item.toLocaleLowerCase() === _file_format);
        if (!checked) {
          this.onFormatError(file, this.fileList);
          return false;
        }
      }

      // check maxSize
      if (this.maxSize) {
        if (file.size > this.maxSize * 1024) {
          this.onExceededSize(file, this.fileList);
          return false;
        }
      }
      this.handleStart(file);
      // 手动控制，校验完文件显示待上传列表
      if(this.selfConfig) {
        if(this.showUploadedList) this.showUploadedList = false
        if(file.status === "finished") {
          file.isShow = false
          return 
        }
        return 
      } else {
        this.startPost(file)
      }
    },
    // 开始上传文件
    startPost (file) {
      let formData = new FormData();
      formData.append(this.name, file);
      this.xhr = ajax({
        headers: this.headers,
        withCredentials: this.withCredentials,
        file: file,
        data: this.data,
        filename: this.name,
        action: this.action,
        onProgress: e => {
          this.handleProgress(e, file);
        },
        onSuccess: res => {
            this.handleSuccess(res, file);
        },
        onError: (err, response) => {
            this.handleError(err, response, file);
        }
      });
    },
    handleStart (file) {
      file.uid = Date.now() + this.tempIndex++;
      const _file = {
        status: 'uploading',
        name: file.name,
        size: file.size,
        percentage: 0,
        uid: file.uid,
        showProgress: true,
        isShow: true
      };
      this.fileList.push(_file);
    },
    getFile (file) {
      const fileList = this.fileList;
      let target;
      fileList.every(item => {
        target = file.uid === item.uid ? item : null;
        return !target;
      });
      return target;
    },
    handleProgress (e, file) {
      const _file = this.getFile(file);
      this.onProgress(e, _file, this.fileList);
      _file.percentage = e.percent || 0;
    },
    handleSuccess (res, file) {
      const _file = this.getFile(file);

      if (_file) {
        _file.status = 'finished';
        _file.response = res;

        this.dispatch('FormItem', 'on-form-change', _file);
        this.onSuccess(res, _file, this.fileList);
        
        setTimeout(() => {
          _file.showProgress = false;
        }, 1000);
        if(this.selfConfig && this.$slots.showList) {
          _file.isShow = false
          this.uploadedFileList.push(_file)
        }
      }
    },
    handleError (err, response, file) {
      const _file = this.getFile(file);
      const fileList = this.fileList;

      _file.status = 'fail';

      fileList.splice(fileList.indexOf(_file), 1);

      this.onError(err, response, file);
    },
    handleRemove(file) {
      const fileList = this.fileList;
      fileList.splice(fileList.indexOf(file), 1);
      this.onRemove(file, fileList);
    },
    handleUploadedRemove(file) {
      const uploadedList = this.uploadedFileList;
      uploadedList.splice(uploadedList.indexOf(file), 1);
      this.handleRemove(file)
      // bug64478 upload组件在配置了selfConfig后on-remove钩子会触发两次
      // this.onRemove(file, uploadedList);
    },
    handlePreview(file) {
      if (file.status === 'finished') {
        this.onPreview(file);
      }
    },
    clearFiles() {
      this.fileList = [];
    }
  },
  watch: {
    uploadedFileListLength() {
      let el = this.$slots.showList[0].elm
      if(this.uploadedFileListLength > 0 && el && el.getAttribute('disabled')==='disabled'){
        el.removeAttribute('disabled')
      }  else if (this.uploadedFileListLength == 0 && el) {
        this.showUploadedList = false
        el.setAttribute('disabled', 'disabled')
      }
    },
    defaultFileList: {
      immediate: true,
      handler(fileList) {
        this.fileList = fileList.map(item => {
          item.status = 'finished';
          item.percentage = 100;
          item.uid = Date.now() + this.tempIndex++;
          return item;
        });
        // 手动上传时，更新列表
        if (this.selfConfig) {
          this.uploadedFileList = fileList.map(item => {
          item.status = 'finished';
          item.percentage = 100;
          item.uid = Date.now() + this.tempIndex++;
          return item;
          });
        }
      }
    }
  },
  mounted () {
    // 禁用已上传列表按钮
    if(this.uploadedFileList.length === 0 && this.$slots.showList) {
      this.$slots.showList[0].elm.setAttribute('disabled', 'disabled')
    }
  }
};
</script>
